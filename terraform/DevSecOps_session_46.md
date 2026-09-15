# Saturday, 28 March 2026
# Session 46 - SG Rules Completed, Roboshop Terraform Component, 90-Components Multi-Service Orchestration & NGINX Reverse Proxy Teardown
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Production Engineering Guide

---

## 1. Original Class Notes & Real-Time Log

> [!NOTE]
> Below is the verbatim, unedited transcription of the original class notes, commands, mistakes, and architectural checklists captured during Session 46. Nothing has been truncated or omitted.

```text
Saturday, 28 March 2026

Session 46 - SG Rules completed, Roboshop terraform component.
Class Notes
private key
certificate
ca_bundle

certificate+ca_bundle = final_certificate_chain -> public
private key
-
- Mistakes will be happen while:
1. Check SG rules
2. Ansible Roles service files
3. Ansible Roles host URL's

for dvo in [“one”,"two","three"]

Commands:
for in in 00-vpc/ 10-sg/ 20-sg-rules/ 50-backend-alb/ 70-acm/ 80-fronend-alb/; do cd $I; terraform init -reconfigure; terraform apply -auto-approve; cd ..;done
Do changes in “20-sg-rules” to create sg-rules for Redis, MongoDB, MySql, Rabbitmq Components.
Execute Terraform code.
cd 30-bastion/  =>
Execute Bastion Terraform code to create bastion infra resources.
ssh ec2-user@<bastion-public-IP>   =>
git clone <git-roboshop-infra-dev-link>   => Inside bastion server
cd roboshop-infra-dev/40-databases/     => Inside bastion server 
terraform init   => Inside bastion server
terraform plan  => Inside bastion server
terraform apply -auto-approve   => Inside bastion server
Then Write “terraform-roboshop-component” and push and pull changes.
 Do changes in “ansible-roboshop-roles-tf” for DNS, port number etc as per the flow decided for “roboshop-infra-dev” as per “roboshop-infra” Architecture diagram.
Create “90-components” folder and create provide.tf, main.tf etc files and write Terraform infra code.
Push “terraform-roboshop-component” code into GitHub.
Get “terraform-roboshop-component”  git url and use in “90-components” to execute “terraform-roboshop-component” from git.
cd ../90-components/   => Inside your system terminal in VS code.
terraform init  => Inside your system terminal in VS code.
Push changes along with terraform generated lock file. 
git pull =>  Inside bastion server
cd ../90-components/   => Inside bastion server
terraform init  => Inside bastion server
terraform plan  => Inside bastion server
terraform apply -auto-approve  => Inside bastion server
This will take about 15 minutes to create all resources and infra.  
Frontend got issue due to port number 8080 using in “nginx.conf” file in “ansible-roboshop-roles-tf” in Catalogue, Cart, User, Payment URLs at the end like :8080, actually no need to write :8080 in these URLs, remove :8080 then frontend issue will be resolved. While debugging infrastructure code need to check NGINX file also.


Diagrams:
roboshop-infra



Timestamps:
QA = 01:20:00





Mistakes & Learning:




Doubts Link Clarification AI chat link:
```

---

## 2. Workspace Projects Mapping & Architecture

Click any file link below to view it directly in your IDE:

#### Project 1: Multi-Tier Foundation (`roboshop-infra-dev`)
- **Security Group Rules Layer (`20-sg-rules/`)**: [roboshop-infra-dev/20-sg-rules/](../../roboshop-infra-dev/20-sg-rules)
  - [20-sg-rules/provider.tf](../../roboshop-infra-dev/20-sg-rules/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/provider.tf)
  - [20-sg-rules/main.tf](../../roboshop-infra-dev/20-sg-rules/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/main.tf)
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

## 3. Visual Architectural Blueprints & Diagrams

### A. TLS Certificate Chain Structure & Secret Management
```
+-----------------------------------------------------------------------------------------+
|                        PUBLIC TRUST CHAIN vs PRIVATE KEY STORAGE                       |
+-----------------------------------------------------------------------------------------+

  [ CLIENT BROWSER / cURL ]
           │
           │ (1) Sends HTTPS Request (TLS ClientHello)
           ▼
  +─────────────────────────────────────────────────────────────────────────────+
  |              SERVER PUBLIC CERTIFICATE CHAIN (Transmitted Over Wire)         |
  |                                                                             |
  |  +───────────────────────────────────────────────────────────────────────+  |
  |  | 1. Leaf Server Certificate (e.g. *.aitechapp.fun)                    |  |
  |  |    - Contains Public Key & Identity Info                              |  |
  |  |    - Signed by Intermediate CA                                        |  |
  |  +───────────────────────────────────────────────────────────────────────+  |
  |                                 ▲ (Signed by)                               |
  |  +───────────────────────────────────────────────────────────────────────+  |
  |  | 2. Intermediate CA Certificate (ca_bundle.crt)                        |  |
  |  |    - Issued to AWS ACM / Let's Encrypt / DigiCert                     |  |
  |  |    - Signed by Trusted Root CA                                        |  |
  |  +───────────────────────────────────────────────────────────────────────+  |
  |                                 ▲ (Signed by)                               |
  |  +───────────────────────────────────────────────────────────────────────+  |
  |  | 3. Root CA Certificate (Pre-installed in OS / Browser Trust Store)   |  |
  |  +───────────────────────────────────────────────────────────────────────+  |
  +─────────────────────────────────────────────────────────────────────────────+
                                        │
                                        │ Validates Chain of Trust
                                        ▼
  +─────────────────────────────────────────────────────────────────────────────+
  |                   SECURE SERVER STORAGE (NEVER Transmitted)                 |
  |                                                                             |
  |  [ PRIVATE KEY (.key) ]                                                     |
  |  - Kept strictly confidential on server / AWS ACM KMS HSM                   |
  |  - Decrypts pre-master secret during asymmetric key exchange                |
  +─────────────────────────────────────────────────────────────────────────────+
```

### B. Multi-Tier Layer Pipeline Execution & Bastion Architecture
```
                               +-----------------------------+
                               | 00-vpc (VPC, Subnets, NAT)  |
                               +-----------------------------+
                                              │
                                              ▼
                               +-----------------------------+
                               | 10-sg (Base Security Groups)|
                               +-----------------------------+
                                              │
                                              ▼
                               +-----------------------------+
                               | 20-sg-rules (Interservice)  |
                               +-----------------------------+
                                              │
                                              ▼
                               +-----------------------------+
                               | 30-bastion (Public Ingress) |
                               +-----------------------------+
                                              │
                                              │ SSH Jump Host
                                              ▼
                    ====================================================
                    PRIVATE ISOLATED VPC NETWORK (No Direct Internet In)
                    ====================================================
                                              │
                     ┌────────────────────────┴────────────────────────┐
                     ▼                                                 ▼
        +-------------------------+                       +-------------------------+
        | 40-databases (Stateful) |                       | 50-backend-alb (Internal|
        | MongoDB, Redis, MySQL,  |                       | Listener & Host Rules)  |
        | RabbitMQ Provisioning   |                       +-------------------------+
        +-------------------------+                                    │
                     │                                                 ▼
                     │                                    +-------------------------+
                     │                                    | 70-acm (TLS Public Cert)|
                     │                                    +-------------------------+
                     │                                                 │
                     ▼                                                 ▼
        +─────────────────────────+                       +-------------------------+
        | 90-components (Fleet)   | <==================== | 80-frontend-alb (Public)|
        | Catalogue, User, Cart,  |  ALB Target Groups    +-------------------------+
        | Shipping, Payment, Web  |
        +─────────────────────────+
```

### C. The NGINX Reverse Proxy Routing Bug & Port 80 vs 8080 Resolution
```
+---------------------------------------------------------------------------------------+
|                THE NGINX REVERSE PROXY ROUTING DILEMMA (PORT 80 vs 8080)             |
+---------------------------------------------------------------------------------------+

                               +--------------------------+
                               | Client Browser / Public  |
                               +--------------------------+
                                            │
                                            │ HTTPS :443
                                            ▼
                               +--------------------------+
                               | Frontend ALB (Port 443)  |
                               +--------------------------+
                                            │
                                            │ Target Group Forward :80
                                            ▼
                               +--------------------------+
                               | NGINX Web Server (Port 80|
                               | on Frontend ASG Instance)|
                               +--------------------------+
                                            │
                                            │ Evaluates /api/catalogue/
                                            │
        ┌───────────────────────────────────┴───────────────────────────────────┐
        │                                                                       │
        ▼ [ INCORRECT CONFIGURATION ]                                           ▼ [ CORRECT CONFIGURATION ]
proxy_pass http://catalogue.backend...:8080/;                           proxy_pass http://catalogue.backend.../;
        │                                                                       │
        ▼ (Port 8080 explicitly requested)                                      ▼ (Defaults to Standard Port 80)
+───────────────────────────────────────────+                           +───────────────────────────────────────────+
| Internal Backend ALB (Listening on :80)   |                           | Internal Backend ALB (Listening on :80)   |
|                                           |                           |                                           |
| ❌ CONNECTION REFUSED!                    |                           |  REQUEST ACCEPTED!                       |
| The ALB listener ONLY accepts Port 80.    |                           | Matches Host: catalogue.backend...        |
| Port 8080 has NO active listener on ALB!  |                           | Evaluates Listener Rule Priority 10       |
+───────────────────────────────────────────+                           +───────────────────────────────────────────+
                                                                                        │
                                                                                        │ ALB Target Group Forwards
                                                                                        ▼
                                                                        +───────────────────────────────────────────+
                                                                        | Catalogue EC2 Instance (Listens on :8080) |
                                                                        +───────────────────────────────────────────+
```

---

## 4. Deep-Dive Mental Models & Engineering Principles

### Mental Model 1: The Anatomy of Modern PKI (Public Key Infrastructure)
In enterprise DevSecOps, TLS security rests upon a strict separation of public verification assets and private cryptographic keys:
1. **The Private Key (`.key`)**:
   - Generated using RSA 2048/4096 or ECDSA (P-256/P-384).
   - Never leaves its origin boundary. In AWS ACM, the private key is managed directly inside AWS hardware security modules (HSMs) and cannot be exported.
2. **The Server Certificate (`.crt`)**:
   - Contains the server's public identity (Common Name, Subject Alternative Names) and public key.
   - Issued and digitally signed by an authorized Certificate Authority.
3. **The CA Bundle (`ca_bundle.crt`)**:
   - The cryptographic proof chain linking the intermediate CA that signed your certificate back to a globally recognized Root CA trusted by all client operating systems.
   - Formula: $\text{Certificate} + \text{CA Bundle} = \text{Complete Public Chain}$.

### Mental Model 2: The Multi-Layer Decoupled Infrastructure Strategy
Enterprise cloud architectures avoid monolithic Terraform codebases. Instead, infrastructure is decomposed into independent layers:
- **Blast Radius Containment**: If a bug occurs while modifying `90-components`, the foundational VPC (`00-vpc`) and core networking remain completely untouched.
- **Role-Based Isolation**: Infrastructure engineers manage `00-vpc` and `10-sg`, DBAs manage `40-databases`, and application developers manage `90-components`.
- **SSM Parameter Store as Contract Bus**: Layers communicate strictly through SSM parameters (e.g. `/roboshop/dev/vpc_id`, `/roboshop/dev/backend_alb_listener_arn`).

### Mental Model 3: Bastion as the Secure Terraform Engine
Production database servers and backend instances must never possess public IP addresses. However, Terraform provisioners (`remote-exec`, `local-exec` running Ansible) require direct IP reachability to configure MySQL, MongoDB, Redis, and RabbitMQ:
- The **Bastion Host** acts as the trusted runner within the VPC.
- Engineers clone `roboshop-infra-dev` inside the Bastion server.
- Because Bastion resides in a public subnet with routing into the private subnets, running `terraform apply` on Bastion allows Ansible to configure databases over private IP addresses directly, avoiding public internet exposure.

### Mental Model 4: Provider Lock File (`.terraform.lock.hcl`) Discipline
When initializing Terraform modules across diverse environments (Mac development machine vs Amazon Linux Bastion host):
- `.terraform.lock.hcl` records the exact cryptographic checksums of provider binaries (`registry.terraform.io/hashicorp/aws`).
- Committing the lock file ensures that both the developer's laptop and the Bastion server run identical provider versions, preventing subtle breaking changes or schema drift.

---

## 5. End-to-End Line-by-Line Code Teardown

### Layer 1: Database Ingress Rules (20-sg-rules)
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
11: # Redis Ingress from User
12: resource "aws_security_group_rule" "redis_user" {
13:   type                     = "ingress"
14:   from_port                = 6379
15:   to_port                  = 6379
16:   protocol                 = "tcp"
17:   source_security_group_id = local.user_sg_id
18:   security_group_id        = local.redis_sg_id
19: }
20: 
21: # MySQL Ingress from Shipping
22: resource "aws_security_group_rule" "mysql_shipping" {
23:   type                     = "ingress"
24:   from_port                = 3306
25:   to_port                  = 3306
26:   protocol                 = "tcp"
27:   source_security_group_id = local.shipping_sg_id
28:   security_group_id        = local.mysql_sg_id
29: }
30: 
31: # RabbitMQ Ingress from Payment
32: resource "aws_security_group_rule" "rabbitmq_payment" {
33:   type                     = "ingress"
34:   from_port                = 5672
35:   to_port                  = 5672
36:   protocol                 = "tcp"
37:   source_security_group_id = local.payment_sg_id
38:   security_group_id        = local.rabbitmq_sg_id
39: }
```

#### Line-by-Line Breakdown:
- **Lines 1-9 (`mongodb_catalogue`)**: Grants the Catalogue service security group (`local.catalogue_sg_id`) network ingress to MongoDB on Port 27017. No other component can access MongoDB directly.
- **Lines 11-19 (`redis_user`)**: Grants the User microservice security group ingress to the Redis in-memory cache on Port 6379 for session state handling.
- **Lines 21-29 (`mysql_shipping`)**: Allows the Java-based Shipping microservice to query the relational MySQL database on Port 3306.
- **Lines 31-39 (`rabbitmq_payment`)**: Allows the Python-based Payment microservice to publish and consume AMQP messaging queues on RabbitMQ Port 5672.

---

### Layer 2: Microservice Fleet Variables (90-components)
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

#### Line-by-Line Breakdown:
- **Lines 1-24 (`variable "components"`)**: Defines a centralized configuration map defining every service in the Roboshop architecture.
- **Lines 4-18 (Backend Services)**: Defines priorities 10 through 50 for the internal ALB listener rules. In AWS Application Load Balancers, every listener rule requires a unique numerical priority. Lower numbers are evaluated first.
- **Lines 20-22 (Frontend Service)**: Configures the public-facing Web UI on rule priority 10 attached to the Frontend Application Load Balancer.

---

### Layer 3: Dynamic Multi-Component Instantiation (90-components)
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
- **Line 1 (`module "component"`)**: Creates child module instances dynamically.
- **Line 2 (`for_each = var.components`)**: Iterates through each key in the components map (`catalogue`, `user`, `cart`, `shipping`, `payment`, `frontend`).
- **Line 3 (`source = "git::https://..."`)**: Points directly to the remote GitHub repository holding the reusable module code. When `terraform init` runs, Terraform clones this repository into `.terraform/modules/`.
- **Line 4 (`component = each.key`)**: Passes the service name into the module.
- **Line 5 (`rule_priority = each.value.rule_priority`)**: Passes the unique rule priority to configure the ALB listener rule.

---

### Layer 4: Polymorphic Component Logic (terraform-roboshop-component)
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
- **Line 2 (`is_frontend`)**: Evaluates a boolean flag checking if the component being instantiated is the frontend web server.
- **Line 3 (`app_port`)**: Uses ternary operator logic: if `is_frontend` is true, the target group and security group expect traffic on Port 80; for all backend services (NodeJS, Java, Python), traffic is directed to Port 8080.
- **Lines 5-6 (`alb_listener_arn`, `alb_dns_name`)**: Dynamically switches between the Frontend Public ALB and the Backend Internal ALB based on the component type.
- **Line 8 (`health_check_path`)**: Assigns `/` for the NGINX web root and `/health` for backend microservice health checks.

---

### Layer 5: NGINX Reverse Proxy Reverse Engineering
> **File**: [ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2](../../ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2)

```nginx
45:         location /images/ {
46:           expires 5s;
47:           root   /usr/share/nginx/html;
48:           try_files $uri /images/placeholder.jpg;
49:         }
50:         location /api/catalogue/ { proxy_pass http://{{ CATALOGUE_HOST }}/; }
51:         location /api/user/ { proxy_pass http://{{ USER_HOST }}/; }
52:         location /api/cart/ { proxy_pass http://{{ CART_HOST }}/; }
53:         location /api/shipping/ { proxy_pass http://{{ SHIPPING_HOST }}/; }
54:         location /api/payment/ { proxy_pass http://{{ PAYMENT_HOST }}/; }
55: 
56:         location /health {
57:           stub_status on;
58:           access_log off;
59:         }
```

#### Line-by-Line Breakdown:
- **Lines 50-54 (`proxy_pass http://{{ SERVICE_HOST }}/;`)**:
  - `{{ CATALOGUE_HOST }}` evaluates to the Route53 wildcard DNS record pointing to the Internal Backend ALB: `catalogue-dev.backend-alb-dev.aitechapp.fun`.
  - **The Crucial Fix**: In earlier versions, this was configured as `proxy_pass http://{{ CATALOGUE_HOST }}:8080/;`. Because the Backend ALB listener operates on Port 80, requesting `:8080` caused connection timeouts (`HTTP 502 Bad Gateway`). Removing `:8080` directs traffic to the ALB on Port 80, which then routes it to the target group on Port 8080!

---

## 6. Step-by-Step Hands-on Execution Walkthrough

```
+───────────────────────────────────────────────────────────────────────────────────────────+
|                           THE 10-STEP RE-ORCHESTRATION PIPELINE                           |
+───────────────────────────────────────────────────────────────────────────────────────────+
```

### Step 1: Batch Reconfiguration Across Foundational Layers
Run this compound Bash loop to re-initialize and synchronize backend states across all layers:
```bash
for dir in 00-vpc/ 10-sg/ 20-sg-rules/ 50-backend-alb/ 70-acm/ 80-frontend-alb/; do
  echo "===> Processing Layer: $dir <==="
  cd "$dir"
  terraform init -reconfigure
  terraform apply -auto-approve
  cd ..
done
```

### Step 2: Implement Interservice Security Group Rules
Navigate to `20-sg-rules/` and verify that all ingress rules connecting backend microservices to stateful datastores are active:
```bash
cd roboshop-infra-dev/20-sg-rules/
terraform plan
terraform apply -auto-approve
```

### Step 3: Deploy the Bastion Host
Deploy the Bastion instance in the public subnet to establish a management bridge into the private network:
```bash
cd ../30-bastion/
terraform init
terraform apply -auto-approve
BASTION_IP=$(terraform output -raw bastion_public_ip 2>/dev/null || aws ec2 describe-instances --filters "Name=tag:Name,Values=bastion-dev" --query "Reservations[].Instances[].PublicIpAddress" --output text)
echo "Bastion Public IP: $BASTION_IP"
```

### Step 4: Access Bastion and Clone the Infrastructure Code
Log into the Bastion server and pull the repository:
```bash
ssh -i ~/.ssh/id_rsa ec2-user@"$BASTION_IP"

# Inside Bastion Host:
git clone https://github.com/SriRamCharanKolla/roboshop-infra-dev.git
cd roboshop-infra-dev/
```

### Step 5: Provision Database Tier from Bastion
Run Terraform inside the Bastion server to spin up MongoDB, Redis, MySQL, and RabbitMQ:
```bash
cd 40-databases/
terraform init
terraform plan
terraform apply -auto-approve
```
> [!IMPORTANT]
> Because the Bastion host shares the VPC network, its Ansible provisioners can reach the private IP addresses of the database instances without traversing the public internet.

### Step 6: Create & Publish the Reusable Component Module
In your local workstation, verify that `terraform-roboshop-component` contains the launch template, ASG, target group, and listener rule logic:
```bash
cd /Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component
git status
git add .
git commit -m "feat: complete reusable roboshop microservice module with polymorphic ports"
git push origin main
```

### Step 7: Configure 90-Components & Lock File Generation
In your local system, initialize `90-components` to download the Git module and generate the provider lock file:
```bash
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/90-components/
terraform init

# Commit the generated lock file:
git add provider.tf main.tf variables.tf .terraform.lock.hcl
git commit -m "feat: configure 90-components fleet and commit lock file"
git push origin main
```

### Step 8: Pull Updated Code Inside Bastion
Return to the Bastion SSH session and pull the latest changes:
```bash
# Inside Bastion:
cd ~/roboshop-infra-dev/
git pull origin main
cd 90-components/
```

### Step 9: Launch the Microservices Fleet
Execute Terraform inside the Bastion terminal to deploy all 6 services simultaneously:
```bash
terraform init
terraform plan
terraform apply -auto-approve
```
*Note: This execution takes approximately 12 to 15 minutes as it bakes golden AMIs, builds launch templates, provisions EC2 instances, and registers target groups.*

### Step 10: Debug & Resolve the Frontend NGINX Port Issue
If the frontend UI displays `502 Bad Gateway` when fetching catalog items:
1. Inspect the NGINX configuration template in `ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2`.
2. Locate the proxy directives:
   ```nginx
   # BEFORE (BROKEN):
   location /api/catalogue/ { proxy_pass http://catalogue-dev.backend-alb-dev.aitechapp.fun:8080/; }
   
   # AFTER (RESOLVED):
   location /api/catalogue/ { proxy_pass http://catalogue-dev.backend-alb-dev.aitechapp.fun/; }
   ```
3. Restart NGINX on the frontend instances:
   ```bash
   sudo systemctl restart nginx
   ```

---

## 7. Verification & Production Health Checks

Run these CLI commands to validate end-to-end connectivity across all tiers:

```bash
# 1. Verify Public Frontend Access via HTTPS
curl -Iv https://dev.aitechapp.fun

# 2. Verify Internal ALB Catalogue Endpoint via Frontend Host
curl -s http://catalogue-dev.backend-alb-dev.aitechapp.fun/health

# 3. Inspect Ingress Security Group Rules for MongoDB
aws ec2 describe-security-group-rules \
  --filters "Name=group-id,Values=$(aws ssm get-parameter --name "/roboshop/dev/mongodb_sg_id" --query "Parameter.Value" --output text)" \
  --query "SecurityGroupRules[?FromPort==\`27017\`].[SecurityGroupRuleId,GroupId,CidrIpv4,ReferencedGroupInfo.GroupId]" \
  --output table

# 4. Verify Route53 DNS Resolution for Microservices
dig +short catalogue-dev.backend-alb-dev.aitechapp.fun
```

---

## 8. Common Pitfalls, Anti-Patterns & Best Practices

| Category | Anti-Pattern | Recommended Production Practice |
| :--- | :--- | :--- |
| **Reverse Proxy** | Adding `:8080` to internal ALB host URLs in `proxy_pass`. | ALB listeners operate on standard Port 80; omit `:8080` in client/proxy requests. |
| **Module Sourcing** | Referencing local relative paths (`../../module`) across different hosts. | Use versioned Git repository URLs (`git::https://github.com/...`) for child modules. |
| **Bastion Security** | Keeping Port 22 open to `0.0.0.0/0` continuously. | Restrict Port 22 ingress to dynamic VPN CIDRs or engineer public IPs (`${local.my_ip}/32`). |
| **Provider Locks** | Omitting `.terraform.lock.hcl` from Git repository. | Always track the lock file to guarantee cryptographic checksum alignment across machines. |
| **State Decoupling** | Hardcoding ARNs and resource IDs across layers. | Publish outputs to AWS SSM Parameter Store and consume them via `data "aws_ssm_parameter"`. |