# Wednesday, 25 February 2026
# Session 34 - Multi-Environment Deployments (Workspaces vs Tfvars vs Separate Repos) & Terraform Module Development
## Comprehensive Class Notes, Project Code Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Managing Multiple Environments in Terraform](#managing-multiple-environments-in-terraform)
   - [Approach 1: Terraform Workspaces (`terraform.workspace`)](#approach-1-terraform-workspaces-terraformworkspace)
   - [Approach 2: Tfvars & Partial Backend Configuration](#approach-2-tfvars--partial-backend-configuration)
   - [Approach 3: Complete Isolation (Separate Repos & AWS Accounts)](#approach-3-complete-isolation-separate-repos--aws-accounts)
   - [Comparative Analysis: Workspaces vs Tfvars vs Separate Accounts](#comparative-analysis-workspaces-vs-tfvars-vs-separate-accounts)
   - [Terraform Module Development Fundamentals](#terraform-module-development-fundamentals)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Project 1: Multi-Env using Workspaces (`terraform/terraform-multienv/workspace/`)](#project-1-multi-env-using-workspaces-terraformterraform-multienvworkspace)
     - [`locals.tf`](#localstf-in-workspace)
     - [`variable.tf`](#variabletf-in-workspace)
     - [`ec2.tf`](#ec2tf-in-workspace)
   - [Project 2: Multi-Env using Tfvars & Backends (`terraform/terraform-multienv/tfvars/`)](#project-2-multi-env-using-tfvars--backends-terraformterraform-multienvtfvars)
     - [`provider.tf` (Partial Backend)](#providertf-partial-backend-in-tfvars)
     - [`dev/backend.tf` & `dev/terraform.tfvars`](#devbackendtf--devterraformtfvars)
     - [`prod/backend.tf` & `prod/terraform.tfvars`](#prodbackendtf--prodterraformtfvars)
     - [`ec2.tf`](#ec2tf-in-tfvars)
   - [Project 3: Custom EC2 Module Development (`terraform-aws-instance/`)](#project-3-custom-ec2-module-development-terraform-aws-instance)
     - [`ec2.tf`](#ec2tf-in-terraform-aws-instance)
     - [`variables.tf` & `locals.tf`](#variablestf--localstf-in-terraform-aws-instance)
     - [`outputs.tf`](#outputstf-in-terraform-aws-instance)
   - [Project 4: Consuming the Module (`ec2-module-test/`)](#project-4-consuming-the-module-ec2-module-test)
     - [`ec2.tf`](#ec2tf-in-ec2-module-test)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Testing Multi-Env via Workspaces](#1-testing-multi-env-via-workspaces)
   - [2. Testing Multi-Env via Tfvars & Remote Backends](#2-testing-multi-env-via-tfvars--remote-backends)
   - [3. Testing Custom Module Consumer](#3-testing-custom-module-consumer)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Managing Multiple Environments in Terraform
In enterprise software delivery, applications must run across multiple environments:
- **DEV**: Development testing, small footprints (`t3.micro`), rapid iterations.
- **UAT / STAGING**: User acceptance testing, mirrors production sizing (`t3.small`).
- **PROD**: Live user traffic, high availability, larger instances (`t3.medium`), maximum security.

There are **three distinct architectural approaches** to managing multiple environments in Terraform:
1. **Workspaces**
2. **Tfvars with Separate Backends**
3. **Separate Repositories & Dedicated AWS Accounts**

---

### Approach 1: Terraform Workspaces (`terraform.workspace`)
- **Concept**:
  - Workspaces allow a single Terraform configuration directory to manage multiple separate state files.
  - Terraform provides a built-in read-only variable: `terraform.workspace`.
  - Default workspace name is `"default"`.
  - Additional workspaces can be created (e.g., `dev`, `uat`, `prod`).
- **Dynamic Sizing via Map Lookup**:
  ```hcl
  variable "instance_type" {
    default = {
      dev  = "t3.micro"
      uat  = "t3.small"
      prod = "t3.medium"
    }
  }

  resource "aws_instance" "example" {
    # Dynamically selects instance size based on active workspace
    instance_type = lookup(var.instance_type, terraform.workspace)
  }
  ```
- **Advantages**:
  - Single codebase: Identical HCL code provisions all environments.
  - Consistency: Guarantees DEV, UAT, and PROD use the same resource definitions.
- **Disadvantages & Security Pitfalls**:
  - **Shared Blast Radius**: All workspaces typically run within the same AWS account and share the same S3 backend bucket (under `env:/<workspace_name>/`).
  - **Human Error**: An engineer intending to test changes in DEV forgets to switch workspaces and accidentally applies destructive changes to PROD!
  - Not recommended by HashiCorp for complete PROD vs NON-PROD security isolation.

---

### Approach 2: Tfvars & Partial Backend Configuration
- **Concept**:
  - Maintains a single set of `.tf` resource files, but organizes environment-specific settings into dedicated directories (`dev/`, `prod/`).
  - Each environment folder contains:
    1. `backend.tf`: Configures the remote state S3 bucket and key.
    2. `terraform.tfvars`: Supplies environment-specific variable values.
- **Partial Backend Configuration**:
  - The root `provider.tf` leaves the `backend "s3" {}` block empty.
  - Backend parameters are injected dynamically at initialization time via CLI:
    ```bash
    # For DEV:
    terraform init -backend-config=dev/backend.tf
    terraform plan -var-file=dev/terraform.tfvars

    # For PROD:
    terraform init -reconfigure -backend-config=prod/backend.tf
    terraform plan -var-file=prod/terraform.tfvars
    ```
- **Advantages**:
  - Separate S3 buckets and state files for DEV and PROD (zero state contamination).
  - Explicit parameterization via dedicated `.tfvars` files.

---

### Approach 3: Complete Isolation (Separate Repos & AWS Accounts)
- **Concept**:
  - Organizations strictly isolate Non-Prod and Prod at the cloud account level:
    - **Dev/Non-Prod AWS Account**: `111111111111`
    - **Prod AWS Account**: `222222222222`
  - Dedicated Git repositories:
    - `terraform-ec2-dev`
    - `terraform-ec2-prod`
- **Advantages**:
  - **Blast Radius is ZERO**: Accidental actions in the DEV repository or AWS account cannot physically affect PROD resources.
  - Separate IAM permissions, billing, and compliance auditing.
- **Disadvantages**:
  - Code duplication across repositories if not utilizing shared remote modules.

---

### Comparative Analysis: Workspaces vs Tfvars vs Separate Accounts

| Evaluation Criteria | 1. Workspaces | 2. Tfvars + Partial Backend | 3. Separate Repos & Accounts |
| :--- | :--- | :--- | :--- |
| **Code Duplication** | None (Single codebase) | None (Single codebase) | High (unless using Modules) |
| **State Storage** | Single S3 bucket (`env:/`) | Separate S3 buckets/keys | Completely isolated S3 buckets |
| **Blast Radius** | High (Same AWS account) | Medium (Shared repo, diff state) | **Zero (Different AWS accounts)** |
| **Security / IAM Isolation**| Weak | Moderate | **Maximum / Enterprise Grade** |
| **Switching Workflow** | `terraform workspace select` | `terraform init -reconfigure` | `aws configure` / IAM AssumeRole |
| **Recommended For** | Ephemeral feature branches | Staging vs QA environments | **Production vs Non-Production** |

---

### Terraform Module Development Fundamentals
- **What is a Module?**
  - Just like a function in shell scripting (`common.sh` vs `a.sh`), a Terraform module is a reusable container for multiple resources used together.
  - Every Terraform configuration has at least one module, known as the **Root Module**.
  - A **Child Module** is called by another configuration using a `module` block.
- **Core Advantages of Modules**:
  1. **Enforce Standards**: Centralizes security policies, mandatory tagging, and network topologies.
  2. **Code Reuse (DRY)**: Write once, consume across dozens of application repositories.
  3. **Simplified Maintenance**: Bug fixes and version upgrades are published centrally without rewriting client code.
- **Standard Module File Structure**:
  ```text
  terraform-aws-instance/
  ├── ec2.tf        # Underlying cloud resources
  ├── variables.tf  # Input arguments accepted by the module
  ├── outputs.tf    # Output attributes returned to the caller
  ├── locals.tf     # Internal tag merges and transformations
  └── README.md     # Documentation and usage examples
  ```

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Session 34 implementation code spans four interconnected projects across your workspace:

#### Project 1: Multi-Env Workspaces Demo
- **Directory**: [terraform/terraform-multienv/workspace/](../../terraform/terraform-multienv/workspace)
- **Provider**: [workspace/provider.tf](../../terraform/terraform-multienv/workspace/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/workspace/provider.tf)
- **Locals**: [workspace/locals.tf](../../terraform/terraform-multienv/workspace/locals.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/workspace/locals.tf)
- **Variables**: [workspace/variable.tf](../../terraform/terraform-multienv/workspace/variable.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/workspace/variable.tf)
- **EC2 Resource**: [workspace/ec2.tf](../../terraform/terraform-multienv/workspace/ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/workspace/ec2.tf)

#### Project 2: Multi-Env Tfvars & Partial Backend Demo
- **Directory**: [terraform/terraform-multienv/tfvars/](../../terraform/terraform-multienv/tfvars)
- **Provider (Partial Backend)**: [tfvars/provider.tf](../../terraform/terraform-multienv/tfvars/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/tfvars/provider.tf)
- **EC2 & SG**: [tfvars/ec2.tf](../../terraform/terraform-multienv/tfvars/ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/tfvars/ec2.tf)
- **Variables**: [tfvars/variables.tf](../../terraform/terraform-multienv/tfvars/variables.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/terraform-multienv/tfvars/variables.tf)
- **DEV Config**: [tfvars/dev/backend.tf](../../terraform/terraform-multienv/tfvars/dev/backend.tf) & [tfvars/dev/terraform.tfvars](../../terraform/terraform-multienv/tfvars/dev/terraform.tfvars)
- **PROD Config**: [tfvars/prod/backend.tf](../../terraform/terraform-multienv/tfvars/prod/backend.tf) & [tfvars/prod/terraform.tfvars](../../terraform/terraform-multienv/tfvars/prod/terraform.tfvars)

#### Project 3: Reusable EC2 Child Module
- **Directory**: [terraform-aws-instance/](../../terraform-aws-instance)
- **Resource Definition**: [terraform-aws-instance/ec2.tf](../../terraform-aws-instance/ec2.tf)
- **Variables**: [terraform-aws-instance/variables.tf](../../terraform-aws-instance/variables.tf)
- **Locals**: [terraform-aws-instance/locals.tf](../../terraform-aws-instance/locals.tf)
- **Outputs**: [terraform-aws-instance/outputs.tf](../../terraform-aws-instance/outputs.tf)
- **Module Readme**: [terraform-aws-instance/readme.md](../../terraform-aws-instance/readme.md)

#### Project 4: Module Consumer (Root Module)
- **Directory**: [ec2-module-test/](../../ec2-module-test)
- **Caller Config**: [ec2-module-test/ec2.tf](../../ec2-module-test/ec2.tf)
- **Provider**: [ec2-module-test/provider.tf](../../ec2-module-test/provider.tf)
- **Variables**: [ec2-module-test/variables.tf](../../ec2-module-test/variables.tf)

---

### Architectural Diagrams

#### A. Multi-Environment Isolation Architectures
```
1. WORKSPACES (Logical Isolation):
   Single S3 Bucket ──┬── env:/dev/terraform.tfstate  ───> AWS Account (Shared)
                      └── env:/prod/terraform.tfstate ───> AWS Account (Shared)

2. TFVARS + PARTIAL BACKENDS (Bucket Isolation):
   S3 Bucket: remote-state-daws-dev  ──> dev.tfstate  ───> AWS Account
   S3 Bucket: remote-state-daws-prod ──> prod.tfstate ───> AWS Account

3. SEPARATE ACCOUNTS (Physical Isolation - Zero Blast Radius):
   Repo: terraform-ec2-dev  ──> AWS Account (111111111111 - Non-Prod)
   Repo: terraform-ec2-prod ──> AWS Account (222222222222 - Production)
```

#### B. Terraform Module Call Pattern
```
+-------------------------------------------------------------+
|         CONSUMER ROOT MODULE (ec2-module-test/ec2.tf)       |
|                                                             |
|   module "ec2" {                                            |
|     source        = "../terraform-aws-instance"             |
|     instance_type = "t3.small"                              |
|     project       = "roboshop"                              |
|   }                                                         |
+-------------------------------------------------------------+
                              │
                              │ Passes input arguments
                              ▼
+-------------------------------------------------------------+
|        CHILD MODULE (terraform-aws-instance/ec2.tf)         |
|                                                             |
|   resource "aws_instance" "this" {                          |
|     ami           = var.ami_id                              |
|     instance_type = var.instance_type                       |
|     tags          = local.ec2_final_tags                    |
|   }                                                         |
|                                                             |
|   output "instance_id" { value = aws_instance.this.id }     |
+-------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Project 1: Multi-Env using Workspaces (`terraform/terraform-multienv/workspace/`)

#### [`locals.tf` in `workspace/`](../../terraform/terraform-multienv/workspace/locals.tf)

```hcl
7: locals {
8:   ami_id      = data.aws_ami.joindevops.id
9:   environment = terraform.workspace
10: }
```
- **Line 9 (`environment = terraform.workspace`)**: Captures the active workspace string (`"default"`, `"dev"`, `"prod"`) into a local value.

---

#### [`variable.tf` in `workspace/`](../../terraform/terraform-multienv/workspace/variable.tf)

```hcl
5: variable "instance_type" {
6:   default = {
7:     dev  = "t3.micro"
8:     uat  = "t3.small"
9:     prod = "t3.medium"
10:   }
11: }
```
- Maps environment names to their corresponding hardware footprints.

---

#### [`ec2.tf` in `workspace/`](../../terraform/terraform-multienv/workspace/ec2.tf)

```hcl
1: resource "aws_instance" "example" {
2:   ami                    = local.ami_id
3:   instance_type          = lookup(var.instance_type, local.environment)
4:   vpc_security_group_ids = [aws_security_group.allow_tls.id]
5: 
6:   tags = {
7:     Name        = "${var.project}-${local.environment}"
8:     Project     = "roboshop"
9:     Environment = local.environment
10:   }
11: }
```
- **Line 3 (`instance_type = lookup(var.instance_type, local.environment)`)**:
  - Evaluates `lookup()`. When workspace is `"dev"`, selects `"t3.micro"`. When workspace is `"prod"`, selects `"t3.medium"`.
- **Line 7 (`Name = "${var.project}-${local.environment}"`)**:
  - Dynamically names the instance `roboshop-dev` or `roboshop-prod`, preventing naming collisions in AWS.

---

### Project 2: Multi-Env using Tfvars & Backends (`terraform/terraform-multienv/tfvars/`)

#### [`provider.tf` (Partial Backend) in `tfvars/`](../../terraform/terraform-multienv/tfvars/provider.tf)

```hcl
9:   backend "s3" {
10:     # Enable S3-native state locking
11:   }
```
- Declares an unconfigured S3 backend block. All configuration parameters are passed during `terraform init`.

---

#### [`dev/backend.tf` & `dev/terraform.tfvars`](../../terraform/terraform-multienv/tfvars/dev/backend.tf)
```hcl
# dev/backend.tf:
bucket       = "remote-state-daws-dev"
key          = "remote-state.tfstate"
region       = "us-east-1"
encrypt      = true
use_lockfile = true

# dev/terraform.tfvars:
environment   = "dev"
instance_type = "t3.micro"
```

#### [`prod/backend.tf` & `prod/terraform.tfvars`](../../terraform/terraform-multienv/tfvars/prod/backend.tf)
```hcl
# prod/backend.tf:
bucket       = "remote-state-daws-prod"
key          = "remote-state.tfstate"
region       = "us-east-1"
encrypt      = true
use_lockfile = true

# prod/terraform.tfvars:
environment   = "prod"
instance_type = "t3.small"
```

---

#### [`ec2.tf` in `tfvars/`](../../terraform/terraform-multienv/tfvars/ec2.tf)

```hcl
1: resource "aws_instance" "example" {
2:   ami                    = "ami-0220d79f3f480ecf5"
3:   instance_type          = var.instance_type
4:   vpc_security_group_ids = [aws_security_group.allow_tls.id]
5: 
6:   tags = {
7:     Name    = "terraform-state-demo-${var.environment}"
8:     Project = "roboshop"
9:   }
10: }
```
- Parameterizes `instance_type` and `tags` using `var.instance_type` and `var.environment` supplied via `-var-file`.

---

### Project 3: Custom EC2 Module Development (`terraform-aws-instance/`)

#### [`ec2.tf` in `terraform-aws-instance/`](../../terraform-aws-instance/ec2.tf)

```hcl
1: resource "aws_instance" "this" {
2:   ami                    = var.ami_id
3:   instance_type          = var.instance_type
4:   vpc_security_group_ids = var.sg_ids
5: 
6:   tags = local.ec2_final_tags
7: }
```
- **Standard Module Naming Convention**: Reusable child resources are commonly named `"this"` or `"main"`.
- All resource arguments are parameterized as variables.

---

#### [`variables.tf` & `locals.tf` in `terraform-aws-instance/`](../../terraform-aws-instance/variables.tf)
```hcl
# locals.tf:
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    Terraform   = "true"
  }
  ec2_final_tags = merge(local.common_tags, var.tags)
}
```
- Automatically injects mandatory organizational tags (`Project`, `Environment`, `Terraform`) while allowing consumers to append custom tags via `merge()`.

---

#### [`outputs.tf` in `terraform-aws-instance/`](../../terraform-aws-instance/outputs.tf)

```hcl
output "instance_id" {
  value = aws_instance.this.id
}

output "private_ip" {
  value = aws_instance.this.private_ip
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
```
- Exposes critical computed attributes back to the caller.

---

### Project 4: Consuming the Module (`ec2-module-test/`)

#### [`ec2.tf` in `ec2-module-test/`](../../ec2-module-test/ec2.tf)

```hcl
1: module "ec2" {
2:   source        = "../terraform-aws-instance"
3:   project       = var.project_name
4:   environment   = var.env
5:   ami_id        = data.aws_ami.joindevops.id
6:   sg_ids        = ["sg-0d955cb0bc2d3310a"]
7:   instance_type = "t3.small"
8:   tags = {
9:     Name      = "${var.project_name}-${var.env}-${var.component}"
10:     Component = var.component
11:   }
12: }
```
- **Line 2 (`source = "../terraform-aws-instance"`)**:
  - Points to the local filesystem path of the child module. (In production, this would be a Git URL or Terraform Registry reference: `git::https://github.com/org/terraform-aws-instance.git?ref=v1.0.0`).
- **Lines 3-11**: Supplies input arguments fulfilling all module variable requirements.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

### 1. Testing Multi-Env via Workspaces
```bash
# Navigate to workspaces directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/terraform-multienv/workspace

# 1. Initialize
terraform init

# 2. View available workspaces (notice active * on 'default')
terraform workspace list

# 3. Create and switch to 'dev' workspace
terraform workspace new dev
terraform workspace list

# 4. Plan in DEV -> verifies instance_type = "t3.micro"
terraform plan

# 5. Create and switch to 'prod' workspace
terraform workspace new prod
terraform workspace list

# 6. Plan in PROD -> verifies instance_type = "t3.medium"
terraform plan

# 7. Switch back to dev
terraform workspace select dev
```

---

### 2. Testing Multi-Env via Tfvars & Remote Backends
```bash
# Navigate to tfvars directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/terraform-multienv/tfvars

# 1. Initialize with DEV S3 Backend
terraform init -backend-config=dev/backend.tf

# 2. Plan DEV Infrastructure
terraform plan -var-file=dev/terraform.tfvars

# 3. Apply DEV
terraform apply -auto-approve -var-file=dev/terraform.tfvars

# 4. Switch to PROD Backend (MANDATORY: Use -reconfigure flag!)
terraform init -reconfigure -backend-config=prod/backend.tf

# 5. Plan PROD Infrastructure
terraform plan -var-file=prod/terraform.tfvars

# 6. Apply PROD
terraform apply -auto-approve -var-file=prod/terraform.tfvars

# 7. Clean up environments
terraform destroy -auto-approve -var-file=prod/terraform.tfvars
terraform init -reconfigure -backend-config=dev/backend.tf
terraform destroy -auto-approve -var-file=dev/terraform.tfvars
```

---

### 3. Testing Custom Module Consumer
```bash
# Navigate to module test directory
cd /Users/sriramcharankolla/Desktop/DevOps/ec2-module-test

# Initialize (Terraform downloads/links the local child module)
terraform init

# Validate configuration
terraform validate

# Plan module resources
terraform plan
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Flag | Example Usage | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `terraform workspace list` | `terraform workspace list` | Lists all existing workspaces in the active backend. |
| `terraform workspace new <name>` | `terraform workspace new dev` | Creates a new workspace and automatically switches into it. |
| `terraform workspace select <name>` | `terraform workspace select prod` | Switches the active context to an existing workspace. |
| `terraform workspace show` | `terraform workspace show` | Prints the name of the currently active workspace. |
| `terraform workspace delete <name>` | `terraform workspace delete dev` | Deletes an empty workspace (cannot delete default or active). |
| `-backend-config="<path>"` | `terraform init -backend-config=dev/backend.tf` | Supplies external backend configuration parameters dynamically. |
| `-reconfigure` | `terraform init -reconfigure -backend-config=...` | Reinitializes backend ignoring cached settings when switching envs. |
| `terraform get -update` | `terraform get -update` | Downloads or updates modules referenced in the root configuration. |

---

## 6. Official Documentation & References
- **Terraform Workspaces**: [developer.hashicorp.com/terraform/language/state/workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)
- **Partial Backend Configuration**: [developer.hashicorp.com/terraform/language/backend#partial-configuration](https://developer.hashicorp.com/terraform/language/backend#partial-configuration)
- **Creating Terraform Modules**: [developer.hashicorp.com/terraform/language/modules/develop](https://developer.hashicorp.com/terraform/language/modules/develop)
- **Module Sources (Local, Git, Registry)**: [developer.hashicorp.com/terraform/language/modules/sources](https://developer.hashicorp.com/terraform/language/modules/sources)
- **HCL `lookup()` Function**: [developer.hashicorp.com/terraform/language/functions/lookup](https://developer.hashicorp.com/terraform/language/functions/lookup)

---

## 7. High-Yield Interview Questions & Answers

### Q1: How do you manage multiple environments in Terraform? What are the pros and cons of each approach?
**Answer**:
There are three standard methods:
1. **Workspaces**: Uses `terraform.workspace`. Single codebase, but states share the same S3 bucket and AWS account. High blast radius; suitable only for short-lived ephemeral environments.
2. **Directory / Tfvars with Partial Backends**: Separate folders (`dev/`, `prod/`) containing `backend.tf` and `.tfvars`. Run `terraform init -reconfigure -backend-config=...`. Keeps state buckets strictly isolated while sharing `.tf` files.
3. **Dedicated Repositories & Separate AWS Accounts**: Physically distinct AWS accounts (`dev-account-id`, `prod-account-id`). Eliminates blast radius entirely; enterprise gold standard for production isolation.

---

### Q2: Why should Terraform Workspaces NOT be used for Production vs Non-Production isolation?
**Answer**:
1. **Single AWS Account Trap**: Workspaces typically operate under the same AWS provider credentials. A single typo or misplaced `apply` can accidentally mutate or delete production resources.
2. **Shared Backend Storage**: State files reside in the same bucket under `env:/`. S3 bucket policies cannot easily grant granular write permissions to Dev while restricting Prod.
3. **Blast Radius**: Workspaces provide logical state isolation, **not physical or security boundary isolation**.

---

### Q3: What is Partial Backend Configuration and why is it used?
**Answer**:
Partial configuration allows leaving the `backend "s3" {}` block in `.tf` files partially or completely empty.
Specific backend arguments (`bucket`, `key`, `region`, `encrypt`, `dynamodb_table`) are injected dynamically during `terraform init -backend-config="<path_or_key_value>"`. This enables a single generic codebase to target different S3 buckets and remote state files across environments.

---

### Q4: What is the difference between a Root Module and a Child Module?
**Answer**:
- **Root Module**: The working directory containing the top-level `.tf` files where `terraform init` and `terraform apply` are executed.
- **Child Module**: A self-contained, parameterized package of `.tf` files located in a separate folder, Git repository, or public registry, called from a root module via a `module "<name>" { source = "..." }` block.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Forgetting `-reconfigure` When Switching Backends**:
   - *Error*: `Error: Backend configuration changed` or state migration prompts.
   - *Fix*: Whenever switching from `dev/backend.tf` to `prod/backend.tf`, always execute `terraform init -reconfigure -backend-config=prod/backend.tf`.
2. **Applying in the Wrong Workspace**:
   - *Problem*: Running `terraform apply` assuming you are in `dev`, but `prod` workspace is active.
   - *Mitigation*: Run `terraform workspace show` in your terminal prompt or build scripts to assert the active workspace before applying.
3. **Hardcoding Resource Names in Shared Code**:
   - *Problem*: Hardcoding `name = "roboshop-instance"` causes AWS API errors when creating DEV and PROD in the same region/account because names conflict.
   - *Fix*: Always append dynamic environment suffixes: `name = "${var.project}-${var.environment}"`.
4. **Modifying Child Module Without Re-initializing**:
   - *Problem*: Adding new provider dependencies or variables to a child module throws errors during planning.
   - *Fix*: Run `terraform init` or `terraform get -update` whenever child module schemas are updated.

---

## 9. Session Metadata & Timestamps

- **Terraform Workspace Fundamentals**: `00:00 - 30:00`
- **Tfvars & Partial Backend Isolation**: `30:00 - 55:00`
- **Interview Question Breakdown**: `59:28` (How to manage multiple environments in Terraform)
- **Module Architecture & EC2 Module Development**: `01:00:00 - 01:25:00`
- **Session Q&A**: `01:28:30`

### Doubts & AI Clarification Link
- [Session 34 AI Clarification Chat](https://chatgpt.com/share/69a067dc-dd60-8007-94a0-5cab91cbe883)
