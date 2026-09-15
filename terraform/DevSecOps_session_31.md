# Friday, 20 February 2026
# Session 31 - Git Ignore, Terraform Variables, Variable Precedence, Conditional Expressions, Count-Based Loops & Route53 Records
## Comprehensive Class Notes, Project Code Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Core Workflow Refresher](#core-workflow-refresher)
   - [Arguments vs Attributes vs Outputs](#arguments-vs-attributes-vs-outputs)
   - [Git Repository Setup & `.gitignore` Best Practices](#git-repository-setup--gitignore-best-practices)
   - [Terraform Variables & Data Types](#terraform-variables--data-types)
   - [Variable Precedence Order (Evaluation Hierarchy)](#variable-precedence-order-evaluation-hierarchy)
   - [Conditional Expressions (Ternary Operator)](#conditional-expressions-ternary-operator)
   - [Loops in Terraform: Count-Based Loop](#loops-in-terraform-count-based-loop)
   - [List vs Set Data Structures](#list-vs-set-data-structures)
   - [Terraform Outputs](#terraform-outputs)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architecture Diagram: Count Loop & Route 53 DNS Mapping](#architecture-diagram-count-loop--route-53-dns-mapping)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Project 1: Variables Demo (`terraform/variables/`)](#project-1-variables-demo-terraformvariables)
     - [`variables.tf`](#variablestf-in-terraformvariables)
     - [`aws_ec2.tf`](#aws_ec2tf-in-terraformvariables)
     - [`terraform.tfvars`](#terraformtfvars-in-terraformvariables)
   - [Project 2: Conditions Demo (`terraform/conditions/`)](#project-2-conditions-demo-terraformconditions)
     - [`variable.tf`](#variabletf-in-terraformconditions)
     - [`aws_ec2.tf`](#aws_ec2tf-in-terraformconditions)
   - [Project 3: Count Loop & DNS Automation (`terraform/count/`)](#project-3-count-loop--dns-automation-terraformcount)
     - [`variables.tf`](#variablestf-in-terraformcount)
     - [`aws_ec2.tf`](#aws_ec2tf-in-terraformcount)
     - [`r53.tf`](#r53tf-in-terraformcount)
     - [`output.tf`](#outputtf-in-terraformcount)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [Executing the Variables Project](#executing-the-variables-project)
   - [Testing Variable Precedence Hands-on](#testing-variable-precedence-hands-on)
   - [Executing the Conditions Project](#executing-the-conditions-project)
   - [Executing the Count & Route53 Project](#executing-the-count--route53-project)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Core Workflow Refresher
- `terraform init` $\rightarrow$ Scans `.tf` files, downloads provider plugins from the Terraform Registry into `.terraform/`, and writes lock file.
- `terraform plan` $\rightarrow$ Plans the infrastructure changes and confirms resources to be created (Dry Run). It does **not** create real resources.
- `terraform apply` $\rightarrow$ Creates the real cloud infrastructure via AWS API calls. Prompts for confirmation (`yes`).
- `terraform apply -auto-approve` $\rightarrow$ Applies changes without interactive confirmation (ideal for CI/CD pipelines).
- `terraform destroy` $\rightarrow$ Destroys and cleans up all managed resources tracked in state.
- `terraform destroy -auto-approve` $\rightarrow$ Deletes all infrastructure without prompt confirmation.

---

### Arguments vs Attributes vs Outputs
Understanding the three pillars of Terraform resource data flow:

```
+-----------------------------------------------------------------------------------+
|                                  RESOURCE BLOCK                                   |
|                                                                                   |
|   ARGUMENTS (Inputs we supply)             ATTRIBUTES (Outputs AWS returns)       |
|   ----------------------------             --------------------------------       |
|   - ami = "ami-0220d7..."                  - id = "i-09ab12cd34ef56"              |
|   - instance_type = "t3.micro"             - private_ip = "172.31.16.5"           |
|   - tags = { Name = "db" }                 - public_ip = "54.210.12.89"           |
|                                            - arn = "arn:aws:ec2:us-east-1:..."    |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
                      OUTPUTS (Surfaced to Terminal / Modules)
                      ----------------------------------------
                      output "db_ip" { value = aws_instance.db.private_ip }
```

1. **Arguments (Inputs)**: Configuration settings you pass into a block (e.g., `ami`, `instance_type`, `name`).
2. **Attributes (Exported Values)**: Values computed by the cloud provider after resource creation (e.g., `.id`, `.arn`, `.private_ip`, `.public_ip`).
3. **Outputs**: User-defined exports exposed on the command line or consumed by remote state / other modules.

---

### Git Repository Setup & `.gitignore` Best Practices
When initializing a repository for Terraform code:

```bash
rm -rf .git
git init
git branch -M main
git remote add origin <GIT_REPO_URL>
git add .
git commit -m "initial commit"
git push -u origin main
```

#### What MUST be added to `.gitignore`?
```gitignore
# 1. Local provider binary caches (large downloads)
.terraform/

# 2. State files (CRITICAL: Contains plaintext secrets and IPs)
*.tfstate
*.tfstate.*
*.tfstate.backup

# 3. Crash logs and overrides
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# 4. Sensitive variable definitions
secrets.tfvars
*.secret.tfvars
```
> [!IMPORTANT]
> The `.terraform.lock.hcl` file **should be committed** to Git so all team members use identical provider versions. However, `terraform.tfstate` must **never** be pushed to Git.

---

### Terraform Variables & Data Types
Variables prevent hardcoding and make infrastructure modular, reusable, and secure.

#### Basic Syntax:
```hcl
variable "variable_name" {
  type        = string       # string, number, bool, list(any), map(any), set(string)
  default     = "dev"        # Optional default fallback
  description = "Help text"  # Documentation for users
}
```

#### Core Data Types:
- `string`: Single string value (e.g., `"t3.micro"`, `"ami-0220d79f3f480ecf5"`).
- `number`: Numerical values without quotes (e.g., `0`, `22`, `443`).
- `bool`: `true` or `false` without quotes.
- `list(any)`: Ordered sequence of items accessible via index `[0]`, `[1]` (allows duplicates).
- `set(string)`: Unordered collection of **unique** items (automatically de-duplicates).
- `map(any)`: Key-value lookup dictionary (e.g., `{ Name = "web", Project = "roboshop" }`).

---

### Variable Precedence Order (Evaluation Hierarchy)
When a variable is declared in `variables.tf`, its final value can be supplied from multiple sources. Terraform resolves conflicts using strict precedence (highest priority wins):

```
HIGHEST PRIORITY
▲   1. Command Line Flags:        terraform apply -var="key=value" or -var-file="custom.tfvars"
│   2. Variable Definition Files: terraform.tfvars or *.auto.tfvars (Alphabetical)
│   3. Environment Variables:     export TF_VAR_variable_name="value"
│   4. Default Values:            default = "value" inside variables.tf
LOWEST PRIORITY
```

#### Detailed Precedence Breakdown:
1. **Command Line Flag `-var` / `-var-file` (Highest)**:
   - Example: `terraform plan -var="instance_type=t3.medium"`
   - Overrides all other sources.
2. **`*.auto.tfvars` & `terraform.tfvars`**:
   - Automatically loaded by Terraform if placed in the working directory.
   - Example: `instance_type = "t3.small"` inside `terraform.tfvars`.
3. **Environment Variables `TF_VAR_<name>`**:
   - Prefix any variable with `TF_VAR_`:
   - Linux/macOS: `export TF_VAR_instance_type="t3.large"`
   - Windows Command Prompt: `set TF_VAR_instance_type="t3.large"`
   - Windows PowerShell: `$env:TF_VAR_instance_type="t3.large"`
4. **Default Value in `variables.tf` (Lowest)**:
   - Used only if none of the above sources provide a value.
   - If no default is provided and no source supplies a value, Terraform prompts interactively on the CLI.

---

### Conditional Expressions (Ternary Operator)
Terraform supports ternary conditionals to dynamically toggle configurations:

```hcl
condition ? true_value : false_value
```

#### Real-World Example:
```hcl
instance_type = var.environment == "dev" ? "t3.micro" : "t3.small"
```
- If `var.environment` is `"dev"` $\rightarrow$ assigns `"t3.micro"`.
- Otherwise (for `prod`, `uat`, etc.) $\rightarrow$ assigns `"t3.small"`.

---

### Loops in Terraform: Count-Based Loop
Terraform offers three looping mechanisms:
1. **`count`**: Numeric-based replication of resources or modules.
2. **`for_each`**: Key/value map or set-based replication (Session 32).
3. **`dynamic` block**: Repeated nested configuration blocks within a single resource (Session 32).

#### The `count` Meta-Argument:
When `count` is specified inside a `resource` block:
- Terraform creates multiple instances of that resource.
- An integer variable `count.index` is exposed (0, 1, 2, ..., N-1).
- Resources become an array indexed from 0: `aws_instance.example[0]`, `aws_instance.example[1]`.

```hcl
resource "aws_instance" "example" {
  count = length(var.instances) # Evaluates to 10
  ami   = "ami-0220d79f3f480ecf5"

  tags = {
    Name = var.instances[count.index] # mongodb, mysql, rabbitmq...
  }
}
```

---

### List vs Set Data Structures
In Session 31, list and set are contrasted using `fruits`:

```hcl
variable "fruits" {
  type    = list(string)
  default = ["apple", "banana", "apple", "orange"]
}

variable "fruits_set" {
  type    = set(string)
  default = ["apple", "banana", "apple", "orange"]
}
```

| Feature | `list(string)` | `set(string)` |
| :--- | :--- | :--- |
| **Duplicates** | Allowed (Keeps both `"apple"` entries) | Disallowed (Automatically removes duplicates $\rightarrow$ `["apple", "banana", "orange"]`) |
| **Ordering** | Preserved (0-indexed: `fruits[0]` is `"apple"`) | Unordered (Cannot access by numerical index `set[0]`) |
| **Use with `count`**| Direct index access `var.instances[count.index]` | Cannot index directly with `count.index` |

---

### Terraform Outputs
Outputs query and expose infrastructure attributes on the CLI after `terraform apply`, or pass data to CI/CD pipelines:

```hcl
output "roboshop_instances" {
  value       = aws_instance.example
  description = "Complete metadata map of all 10 provisioned instances"
}
```
- Print outputs anytime: `terraform output`
- Print specific output in JSON: `terraform output -json roboshop_instances`

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Session 31 contains three distinct hands-on projects inside the workspace. Click to open them directly in your IDE:

#### Project A: Variables & Precedence Demo
- **Directory**: [terraform/variables/](../../terraform/variables)
- **Provider**: [variables/provider.tf](../../terraform/variables/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/variables/provider.tf)
- **Variables**: [variables/variables.tf](../../terraform/variables/variables.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/variables/variables.tf)
- **EC2 & SG**: [variables/aws_ec2.tf](../../terraform/variables/aws_ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/variables/aws_ec2.tf)
- **Variable Values File**: [variables/terraform.tfvars](../../terraform/variables/terraform.tfvars) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/variables/terraform.tfvars)

#### Project B: Conditional Expressions Demo
- **Directory**: [terraform/conditions/](../../terraform/conditions)
- **Provider**: [conditions/provider.tf](../../terraform/conditions/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/conditions/provider.tf)
- **Variables**: [conditions/variable.tf](../../terraform/conditions/variable.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/conditions/variable.tf)
- **EC2 & SG**: [conditions/aws_ec2.tf](../../terraform/conditions/aws_ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/conditions/aws_ec2.tf)

#### Project C: Count Loop & Route 53 DNS Records (Roboshop 10 Components)
- **Directory**: [terraform/count/](../../terraform/count)
- **Provider**: [count/provider.tf](../../terraform/count/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/count/provider.tf)
- **Variables**: [count/variables.tf](../../terraform/count/variables.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/count/variables.tf)
- **EC2 & SG**: [count/aws_ec2.tf](../../terraform/count/aws_ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/count/aws_ec2.tf)
- **Route 53 DNS Records**: [count/r53.tf](../../terraform/count/r53.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/count/r53.tf)
- **Outputs**: [count/output.tf](../../terraform/count/output.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/count/output.tf)

---

### Architecture Diagram: Count Loop & Route 53 DNS Mapping

```
+---------------------------------------------------------------------------------------------------+
|                                      AWS VPC (us-east-1)                                          |
|                                                                                                   |
|   +-------------------------------------------------------------------------------------------+   |
|   |                         Security Group: "allow-all-terraform"                             |   |
|   |                         (Applied to all 10 Roboshop EC2 Instances)                        |   |
|   +-------------------------------------------------------------------------------------------+   |
|                                                   ▲                                               |
|                                                   │ Attached to all instances                     |
|                                                   ▼                                               |
|   +-------------------------------------------------------------------------------------------+   |
|   |                       aws_instance.example[count.index] (10 Nodes)                        |   |
|   |                                                                                           |   |
|   |  [0] mongodb     [1] mysql       [2] rabbitmq    [3] redis       [4] catalogue            |   |
|   |  [5] shipping    [6] cart        [7] user        [8] payment     [9] frontend             |   |
|   +-------------------------------------------------------------------------------------------+   |
|                                │                                            │                     |
|          Private IP Mapping    │                                            │ Public IP Mapping   |
|                                ▼                                            ▼                     |
|   +-----------------------------------------------------------+  +----------------------------+   |
|   |       AWS Route 53 (Hosted Zone: aitechapp.fun)           |  |     Public Frontend DNS    |   |
|   |             Internal Private DNS Records                  |  |                            |   |
|   |                                                           |  | roboshop.aitechapp.fun     |   |
|   | - mongodb.aitechapp.fun   -> 172.31.X.X (Private IP)      |  |         │                  |   |
|   | - mysql.aitechapp.fun     -> 172.31.X.X (Private IP)      |  |         ▼                  |   |
|   | - catalogue.aitechapp.fun -> 172.31.X.X (Private IP)      |  | Points to Frontend Public  |   |
|   | - shipping.aitechapp.fun  -> 172.31.X.X (Private IP)      |  | IP: 54.X.X.X               |   |
|   | - (... 10 internal records generated via count)           |  |                            |   |
|   +-----------------------------------------------------------+  +----------------------------+   |
+---------------------------------------------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Project 1: Variables Demo (`terraform/variables/`)

#### [`variables.tf` in `terraform/variables/`](../../terraform/variables/variables.tf)

```hcl
1: variable "env" {
2:   default = "dev"
3: }
4: 
5: variable "ami_id" {
6:   type        = string
7:   default     = "ami-0220d79f3f480ecf5"
8:   description = "RHEL 9 Image"
9: }
10: 
11: variable "instance_type" {
12:   type    = string
13:   default = "t3.micro"
14: }
15: 
16: variable "ec2_tags" {
17:   type = map(any)
18:   default = {
19:     Name        = "variables-demo"
20:     Project     = "roboshop"
21:     Terraform   = true
22:     Environment = "dev"
23:   }
24: }
25: 
26: variable "sg_name" {
27:   type    = string
28:   default = "allow-all-terraform-default"
29: }
30: 
31: variable "sg_description" {
32:   type    = string
33:   default = "Allow TLS inbound traffic and all outbound traffic"
34: }
35: 
36: variable "sg_from_port" {
37:   type    = number
38:   default = 0
39: }
40: 
41: variable "sg_to_group" {
42:   type    = number
43:   default = 0
44: }
45: 
46: variable "cidr_blocks" {
47:   type    = list(any)
48:   default = ["0.0.0.0/0"]
49: }
50: 
51: variable "sg_tags" {
52:   type = map(any)
53:   default = {
54:     Name        = "allow-all-terraform"
55:     Project     = "roboshop"
56:     Terraform   = true
57:     Environment = "dev"
58:   }
59: }
```

##### Line-by-Line Breakdown:
- **Lines 1-3 (`variable "env"`)**: Declares environment name with default `"dev"`.
- **Lines 5-9 (`variable "ami_id"`)**: Strongly typed as `string`, provides default RHEL-9 AMI and documentation `description`.
- **Lines 11-14 (`variable "instance_type"`)**: Default hardware type `"t3.micro"`.
- **Lines 16-24 (`variable "ec2_tags"`)**: Uses `type = map(any)`. Collects common metadata tags (`Name`, `Project`, `Terraform`, `Environment`).
- **Lines 26-29 (`variable "sg_name"`)**: Default name `"allow-all-terraform-default"`. (Notice how this is overridden by `terraform.tfvars`).
- **Lines 36-44 (`sg_from_port`, `sg_to_group`)**: Typed as `number`, defaulting to `0` for all ports.
- **Lines 46-49 (`cidr_blocks`)**: Typed as `list(any)`, allowing an array of IP ranges.

---

#### [`aws_ec2.tf` in `terraform/variables/`](../../terraform/variables/aws_ec2.tf)

```hcl
1: resource "aws_instance" "example" {
2:   ami           = var.ami_id
3:   instance_type = var.instance_type
4:   # if dev t3.micro, otherwise t3.small
5:   # instance_type = var.env == "dev" ? "t3.micro" : "t3.small"
6:   vpc_security_group_ids = [aws_security_group.allow_tls.id] # creating aws security group & applying to instance going to create.
7: 
8:   tags = var.ec2_tags
9: }
10: 
11: # Creating aws security group
12: resource "aws_security_group" "allow_tls" {
13:   name        = var.sg_name # this is for AWS account
14:   description = var.sg_description
15: 
16:   egress { # outbound
17:     from_port        = var.sg_from_port
18:     to_port          = var.sg_to_group
19:     protocol         = "-1"
20:     cidr_blocks      = var.cidr_blocks
21:     ipv6_cidr_blocks = ["::/0"]
22:   }
23: 
24:   ingress { # inbound
25:     from_port        = var.sg_from_port
26:     to_port          = var.sg_from_port
27:     protocol         = "-1"
28:     cidr_blocks      = var.cidr_blocks
29:     ipv6_cidr_blocks = ["::/0"]
30:   }
31: 
32:   tags = var.sg_tags
33: }
```

##### Line-by-Line Breakdown:
- **Lines 2-3**: Instead of hardcoding `"ami-0220..."` or `"t3.micro"`, attributes reference `var.ami_id` and `var.instance_type`.
- **Line 6 (`vpc_security_group_ids = [aws_security_group.allow_tls.id]`)**: Dynamic dependency reference to the Security Group ID.
- **Line 8 (`tags = var.ec2_tags`)**: Passes the complete map of tags defined in `variables.tf`.
- **Lines 13-14 (`name = var.sg_name`, `description = var.sg_description`)**: Replaces hardcoded strings with variables.
- **Lines 16-30 (`egress` & `ingress` blocks)**: Replaces port numbers and CIDR blocks with `var.sg_from_port`, `var.cidr_blocks`.

---

#### [`terraform.tfvars` in `terraform/variables/`](../../terraform/variables/terraform.tfvars)

```hcl
1: # instance_type = "t3.small"
2: sg_name = "allow-all-terraform-tfvars"
```

##### Breakdown:
- Line 2 overrides `var.sg_name` ("allow-all-terraform-default") with `"allow-all-terraform-tfvars"`.
- Because `terraform.tfvars` has higher precedence than `variables.tf`, `terraform plan` will use this value unless overridden on the command line via `-var`.

---

### Project 2: Conditions Demo (`terraform/conditions/`)

#### [`variable.tf` in `terraform/conditions/`](../../terraform/conditions/variable.tf)
- Declares `variable "environment" { default = "prod" }`.
- Sets up standard variables for AMI, instance tags, and security group.

#### [`aws_ec2.tf` in `terraform/conditions/`](../../terraform/conditions/aws_ec2.tf)

```hcl
5: resource "aws_instance" "example" {
6:   ami                    = var.ami_id
7:   instance_type          = var.environment == "dev" ? "t3.micro" : "t3.small"
8:   vpc_security_group_ids = [aws_security_group.allow_tls.id]
9: 
10:   tags = var.ec2_tags
11: }
```

##### Line-by-Line Breakdown:
- **Line 7 (`instance_type = var.environment == "dev" ? "t3.micro" : "t3.small"`)**:
  - Evaluates `var.environment == "dev"`.
  - In `variable.tf`, `environment` defaults to `"prod"`.
  - Condition evaluates to `false` $\rightarrow$ Terraform picks `"t3.small"`.
  - If you run `terraform plan -var="environment=dev"`, condition evaluates to `true` $\rightarrow$ Terraform picks `"t3.micro"`.

---

### Project 3: Count Loop & DNS Automation (`terraform/count/`)

#### [`variables.tf` in `terraform/count/`](../../terraform/count/variables.tf)

```hcl
1: variable "instances" {
2:   type    = list(any)
3:   default = ["mongodb", "mysql", "rabbitmq", "redis", "catalogue", "shipping", "cart", "user", "payment", "frontend"]
4: }
5: 
6: variable "zone_id" {
7:   default = "Z082192717Y56TLJLLOXS"
8: }
9: 
10: variable "domain_name" {
11:   default = "aitechapp.fun"
12: }
13: 
14: variable "fruits" {
15:   type    = list(string)
16:   default = ["apple", "banana", "apple", "orange"]
17: }
18: 
19: variable "fruits_set" {
20:   type    = set(string)
21:   default = ["apple", "banana", "apple", "orange"]
22: }
```

##### Line-by-Line Breakdown:
- **Lines 1-4 (`instances`)**: An ordered list of the 10 microservices required for the Roboshop e-commerce architecture.
- **Lines 6-12 (`zone_id`, `domain_name`)**: Route 53 Hosted Zone ID and domain name (`aitechapp.fun`) for automated DNS registration.
- **Lines 14-22 (`fruits`, `fruits_set`)**: Demonstrates duplicate handling in `list` vs `set`.

---

#### [`aws_ec2.tf` in `terraform/count/`](../../terraform/count/aws_ec2.tf)

```hcl
7: resource "aws_instance" "example" {
8:   count                  = length(var.instances) # Iterates dynamically through all items in var.instances
9:   ami                    = "ami-0220d79f3f480ecf5"
10:   instance_type          = "t3.micro"
11:   vpc_security_group_ids = [aws_security_group.allow_tls.id]
12: 
13:   tags = {
14:     # Dynamically assign instance name using count.index to lookup list elements
15:     Name    = var.instances[count.index]
16:     Project = "roboshop"
17:   }
18: }
```

##### Line-by-Line Breakdown:
- **Line 8 (`count = length(var.instances)`)**:
  - `length()` function counts the items in `var.instances` (10 items).
  - Creates 10 instances: `aws_instance.example[0]` through `aws_instance.example[9]`.
- **Line 15 (`Name = var.instances[count.index]`)**:
  - When `count.index = 0` $\rightarrow$ `Name = "mongodb"`
  - When `count.index = 1` $\rightarrow$ `Name = "mysql"`
  - ... up to `count.index = 9` $\rightarrow$ `Name = "frontend"`

---

#### [`r53.tf` in `terraform/count/`](../../terraform/count/r53.tf)

```hcl
5: resource "aws_route53_record" "roboshop_internal" {
6:   count   = length(var.instances)
7:   zone_id = var.zone_id
8:   name    = "${var.instances[count.index]}.${var.domain_name}"
9:   type    = "A"
10:   ttl     = 1
11:   records = [aws_instance.example[count.index].private_ip]
12: }
13: 
18: resource "aws_route53_record" "roboshop_public" {
19:   zone_id = var.zone_id
20:   name    = "roboshop.${var.domain_name}"
21:   type    = "A"
22:   ttl     = 1
23:   # Finds the index of 'frontend' in var.instances and extracts its public_ip
24:   records = [aws_instance.example[index(var.instances, "frontend")].public_ip]
25: }
```

##### Line-by-Line Breakdown:
- **Lines 5-12 (`roboshop_internal`)**:
  - `count = length(var.instances)`: Generates 10 internal DNS records.
  - `name = "${var.instances[count.index]}.${var.domain_name}"`: Evaluates to `mongodb.aitechapp.fun`, `mysql.aitechapp.fun`, etc.
  - `type = "A"`: Maps domain name to an IPv4 address.
  - `ttl = 1`: Time to live (1 second for immediate propagation in testing).
  - `records = [aws_instance.example[count.index].private_ip]`: Points the DNS record to the corresponding EC2 instance's **Private IP** for secure VPC communication.
- **Lines 18-25 (`roboshop_public`)**:
  - Single public DNS record (`roboshop.aitechapp.fun`) for user-facing browser traffic.
  - `index(var.instances, "frontend")`: Built-in function looks up the list index of `"frontend"` (index `9`).
  - `aws_instance.example[...].public_ip`: Points to the Frontend instance's **Public IP**.

---

#### [`output.tf` in `terraform/count/`](../../terraform/count/output.tf)

```hcl
1: output "roboshop_instances" {
2:   value       = aws_instance.example
3:   description = "learning how output works"
4: }
5: 
6: output "fruits_names" {
7:   value = var.fruits
8: }
9: 
10: output "fruits_names_set" {
11:   value = var.fruits_set
12: }
```

##### Breakdown:
- `aws_instance.example`: Outputs a list of all 10 created instance objects containing their IPs, IDs, and ARNs.
- `fruits_names` vs `fruits_names_set`: Directly compares the list output with duplicates against the deduplicated set output.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

### Executing the Variables Project
```bash
# 1. Navigate to the variables project folder
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/variables

# 2. Initialize provider plugins
terraform init

# 3. Format and validate code
terraform fmt
terraform validate

# 4. Plan and review variables substitution
terraform plan
```

---

### Testing Variable Precedence Hands-on
Follow this exact sequence to observe how Terraform prioritizes variable sources:

```bash
# Level 4: Default value in variables.tf is "allow-all-terraform-default"
# Level 2: But terraform.tfvars has: sg_name = "allow-all-terraform-tfvars"
terraform plan
# Notice: Security group name will be "allow-all-terraform-tfvars"

# Level 3: Set Environment Variable
export TF_VAR_sg_name="allow-all-terraform-env"
terraform plan
# Notice: terraform.tfvars STILL wins over environment variables!

# Level 1: Command-line -var flag (Ultimate Winner)
terraform plan -var="sg_name=allow-all-terraform-cmd"
# Notice: Security group name will now be "allow-all-terraform-cmd"

# Clean up exported env var
unset TF_VAR_sg_name
```

---

### Executing the Conditions Project
```bash
# 1. Navigate to conditions directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/conditions

# 2. Initialize Terraform
terraform init

# 3. Test default (environment = "prod") -> Expects t3.small
terraform plan | grep instance_type

# 4. Test dev condition -> Expects t3.micro
terraform plan -var="environment=dev" | grep instance_type
```

---

### Executing the Count & Route53 Project
```bash
# 1. Navigate to count directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/count

# 2. Initialize Terraform
terraform init

# 3. Dry run: Notice it plans 10 EC2 instances + 10 internal DNS records + 1 public DNS record (Total 22 resources)
terraform plan

# 4. Apply infrastructure to AWS
terraform apply -auto-approve

# 5. Inspect outputs
terraform output fruits_names
terraform output fruits_names_set

# 6. Clean up all 22 resources to avoid AWS costs
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Flag | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `terraform init` | `terraform init` | Downloads provider plugins and initializes `.terraform/` directory. |
| `terraform plan` | `terraform plan` | Generates and displays an execution plan without modifying cloud resources. |
| `terraform apply` | `terraform apply` | Executes the configuration to provision or update real cloud infrastructure. |
| `terraform apply -auto-approve` | `terraform apply -auto-approve` | Applies changes skipping the interactive `yes` prompt confirmation. |
| `terraform destroy` | `terraform destroy` | Tears down all resources managed by the state file. |
| `-var` Flag | `terraform plan -var="instance_type=t3.small"` | Passes or overrides a single variable value directly from terminal. |
| `-var-file` Flag | `terraform apply -var-file="prod.tfvars"` | Loads variable values from a custom-named `.tfvars` file. |
| `TF_VAR_<name>` (Linux/Mac) | `export TF_VAR_env="prod"` | Sets a Terraform input variable using OS shell environment variables. |
| `TF_VAR_<name>` (Windows) | `set TF_VAR_env="prod"` | Sets an environment variable in Windows Command Prompt. |
| `terraform output` | `terraform output` | Displays all output values defined in `output.tf`. |
| `terraform output <name>` | `terraform output roboshop_instances` | Prints only the specified output value. |

---

## 6. Official Documentation & References
- **Terraform Input Variables Documentation**: [developer.hashicorp.com/terraform/language/values/variables](https://developer.hashicorp.com/terraform/language/values/variables)
- **Variable Definition Files (`.tfvars`)**: [developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files](https://developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files)
- **Conditional Expressions in HCL**: [developer.hashicorp.com/terraform/language/expressions/conditionals](https://developer.hashicorp.com/terraform/language/expressions/conditionals)
- **The `count` Meta-Argument**: [developer.hashicorp.com/terraform/language/meta-arguments/count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)
- **Terraform Built-in Functions (`length()`, `index()`)**: [developer.hashicorp.com/terraform/language/functions](https://developer.hashicorp.com/terraform/language/functions)
- **AWS Route 53 Record Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record)

---

## 7. High-Yield Interview Questions & Answers

### Q1: What is the exact Variable Precedence order in Terraform?
**Answer**:
From highest priority to lowest:
1. Command line arguments: `-var` and `-var-file`.
2. Variable files: `*.auto.tfvars` (processed alphabetically) followed by `terraform.tfvars`.
3. Environment variables: `TF_VAR_<variable_name>`.
4. Default values defined in `variables.tf`.

---

### Q2: What is the difference between `list` and `set` in Terraform?
**Answer**:
- `list`: An ordered collection of elements indexed by number (`0, 1, 2...`). Allows duplicate values. Supports indexing syntax like `var.instances[count.index]`.
- `set`: An unordered collection of unique elements. Automatically removes duplicate items. Elements cannot be accessed by numerical index (`set[0]` will throw an error).

---

### Q3: What is the major drawback or risk of using the `count` loop in production?
**Answer**:
The `count` loop references resources by their numerical index in state (e.g., `aws_instance.example[0]`, `aws_instance.example[1]`).
- **The Index-Shifting Problem**: If you remove an item from the middle of the list (e.g., deleting `"redis"` from position `3`), all subsequent resources shift down one index (`catalogue` becomes index `3`, `shipping` becomes `4`).
- Terraform detects that the resource at index `3` changed its name and configurations, causing it to **destroy and re-create all subsequent resources** instead of deleting just the targeted instance!
- **Solution**: Use `for_each` (introduced in Session 32) which binds resources to stable string keys instead of integer indices.

---

### Q4: What is the difference between an Argument, an Attribute, and an Output?
**Answer**:
- **Argument**: An input configuration passed into a block (e.g., `ami = var.ami_id`).
- **Attribute**: An export computed by AWS upon resource creation (e.g., `aws_instance.example[0].private_ip`).
- **Output**: A custom-defined export block in Terraform that surfaces selected attributes to the CLI or allows other modules to consume them.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **The `count` Removal Trap (Index Shifting)**:
   - *Problem*: Removing an item from `var.instances` destroys and rebuilds multiple unrelated production servers.
   - *Fix*: For production workloads with independent lifecycles, prefer `for_each` over `count`.
2. **Bash Quotation Issues with `-var`**:
   - *Problem*: Running `terraform plan -var=instance_type=t3.small` fails on some shells.
   - *Fix*: Always wrap key-value pairs in quotes: `terraform plan -var="instance_type=t3.small"`.
3. **Environment Variable Stale State**:
   - *Problem*: Terraform behaves unexpectedly because an old `export TF_VAR_...` is still lingering in the terminal session.
   - *Fix*: Run `env | grep TF_VAR` to inspect active environment variables, or use `unset TF_VAR_<name>` to clear them.
4. **Referencing Route 53 Resources Before Instances Exist**:
   - *Problem*: Using dynamic IPs in Route 53 records causes plan errors if instances aren't tracked.
   - *Fix*: Terraform automatically builds a dependency graph when you reference `aws_instance.example[count.index].private_ip`, ensuring the instances are provisioned before attempting to register DNS records.

---

## 9. Session Metadata & Timestamps

- **Terraform Workflow & Git Ignore**: `00:00 - 15:00`
- **Variables & Data Types**: `15:00 - 35:00`
- **Variable Precedence Hierarchy**: `35:00 - 50:00`
- **Conditional Expressions**: `50:00 - 01:05:00`
- **Count Loop & Route 53 Automation**: `01:05:00 - 01:17:00`
- **Interview Questions**: `01:17:00`
- **Session Q&A**: `01:17:20`

### Doubts & AI Clarification Link
- [Session 31 AI Clarification Chat](https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049)