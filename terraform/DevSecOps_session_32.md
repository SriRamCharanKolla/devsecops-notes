# Wednesday, 25 February 2026
# Session 32 - For_Each Loop, Dynamic Blocks, Data Sources & Built-in Functions
## Comprehensive Class Notes, Multi-Project Code Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [HCL Syntax & Variable Review](#hcl-syntax--variable-review)
   - [Count vs For_Each vs Dynamic Block](#count-vs-for_each-vs-dynamic-block)
   - [The `for_each` Meta-Argument](#the-for_each-meta-argument)
   - [The `dynamic` Block (DRY Principle for Nested Blocks)](#the-dynamic-block-dry-principle-for-nested-blocks)
   - [Terraform Data Sources (Read-Only Cloud Queries)](#terraform-data-sources-read-only-cloud-queries)
   - [Terraform Built-in Functions & `merge()`](#terraform-built-in-functions--merge)
   - [Saving JSON Outputs via CLI](#saving-json-outputs-via-cli)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Project 1: `for_each` Loop & DNS Automation (`terraform/for_each/`)](#project-1-for_each-loop--dns-automation-terraformfor_each)
     - [`variable.tf`](#variabletf-in-terraformfor_each)
     - [`ec2.tf`](#ec2tf-in-terraformfor_each)
     - [`r53.tf`](#r53tf-in-terraformfor_each)
     - [`outputs.tf`](#outputstf-in-terraformfor_each)
   - [Project 2: Dynamic Ingress Blocks (`terraform/dynamic/`)](#project-2-dynamic-ingress-blocks-terraformdynamic)
     - [`variables.tf`](#variablestf-in-terraformdynamic)
     - [`ec2.tf`](#ec2tf-in-terraformdynamic)
   - [Project 3: Data Sources (`terraform/data-sources/`)](#project-3-data-sources-terraformdata-sources)
     - [`data.tf`](#datatf-in-terraformdata-sources)
     - [`aws_ec2.tf`](#aws_ec2tf-in-terraformdata-sources)
     - [`outputs.tf`](#outputstf-in-terraformdata-sources)
   - [Project 4: Built-in Functions & `merge()` (`terraform/functions/`)](#project-4-built-in-functions--merge-terraformfunctions)
     - [`variables.tf`](#variablestf-in-terraformfunctions)
     - [`aws_ec2.tf`](#aws_ec2tf-in-terraformfunctions)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Running `for_each` & Exporting JSON Outputs](#1-running-for_each--exporting-json-outputs)
   - [2. Testing Dynamic Ingress Rules](#2-testing-dynamic-ingress-rules)
   - [3. Testing Data Source AMI Lookup](#3-testing-data-source-ami-lookup)
   - [4. Testing `merge()` Function Precedence](#4-testing-merge-function-precedence)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### HCL Syntax & Variable Review
- **Standard Resource Syntax**:
  ```hcl
  resource "type_of_resource" "name_of_resource" {
    key = value # Configuration arguments
  }
  ```
- **Input Variables**:
  ```hcl
  variable "name_of_variable" {
    type    = string
    default = ""
  }
  ```
- **Variable Precedence**:
  1. Command-line flags: `-var "var_name=value"` or `-var-file`
  2. Variable definition files: `*.auto.tfvars` and `terraform.tfvars`
  3. Environment variables: `export TF_VAR_VAR_NAME="value"`
  4. Default values inside `variables.tf`
- **Ternary Conditions**:
  ```hcl
  expression ? "true_val" : "false_val"
  ```
- **List vs Set Data Types**:
  - `list`: Ordered, index-based values (`[0]`, `[1]`), allows duplicate values.
  - `set`: Order not guaranteed, automatically strips duplicate values.

---

### Count vs For_Each vs Dynamic Block
The three primary iteration constructs in Terraform have distinct purposes:

```
+---------------------------------------------------------------------------------------------------+
| ITERATION CONSTRUCT   | APPLICABLE ON           | ITERATOR VARIABLE | COMMON USE CASE             |
| -------------------   | -------------           | ----------------- | ---------------             |
| 1. count              | Resources & Modules     | count.index       | Identical multiple copies   |
| 2. for_each           | Resources & Modules     | each.key,         | Distinct named resources    |
|                       | (Accepts Map or Set)    | each.value        | with stable state keys      |
| 3. dynamic            | Nested blocks INSIDE    | <block_name>.key, | Repeated sub-blocks (e.g.,  |
|                       | a single resource       | <block_name>.value| ingress, egress, tag blocks)|
+---------------------------------------------------------------------------------------------------+
```

- **`count`**: List-based. Creates resources indexed numerically (`aws_instance.example[0]`, `aws_instance.example[1]`). Deleting an item from the middle causes destructive index-shifting.
- **`for_each`**: Map or Set-based. Creates resources keyed by stable strings (`aws_instance.example["mongodb"]`, `aws_instance.example["redis"]`). Deleting an element only destroys that specific resource.
- **`dynamic`**: **Only for repeated code INSIDE a single resource** (e.g., repeating multiple `ingress {}` blocks inside `aws_security_group` without duplicating code).

---

### The `for_each` Meta-Argument
- **Syntax**:
  ```hcl
  resource "aws_instance" "example" {
    for_each = toset(var.instances) # or a map
    ami      = "ami-0220d79f3f480ecf5"

    tags = {
      Name = each.key # Holds current element string
    }
  }
  ```
- When using `for_each`, Terraform exposes a special read-only object called `each`:
  - `each.key`: The map key or set member.
  - `each.value`: The map value (for maps) or set member (for sets).
- **Converting List to Set**:
  - `for_each` does **not** accept a raw `list`. You must wrap lists with `toset(var.list_variable)` so Terraform can guarantee unique identifier keys.

---

### The `dynamic` Block (DRY Principle for Nested Blocks)
When defining resources like security groups, routing tables, or autoscaling groups, configurations often require repeating identical nested blocks:

```hcl
# The redundant, manual way:
resource "aws_security_group" "allow_tls" {
  ingress { from_port = 22, to_port = 22, ... }
  ingress { from_port = 443, to_port = 443, ... }
  ingress { from_port = 3306, to_port = 3306, ... }
}
```

With `dynamic`:
```hcl
dynamic "ingress" {
  for_each = toset(var.ingress_rules)
  content {
    from_port   = ingress.value.port
    to_port     = ingress.value.port
    protocol    = "tcp"
    cidr_blocks = ingress.value.cidr
    description = ingress.value.description
  }
}
```
- `dynamic "<BLOCK_NAME>"` specifies the nested block type to generate.
- `content { ... }` defines the body of each generated block.
- Inside `content`, attributes are accessed via `<BLOCK_NAME>.value.<field>`.

---

### Terraform Data Sources (Read-Only Cloud Queries)
- **Concept**:
  - While `resource` creates, updates, and destroys infrastructure, a `data` source performs **read-only queries** against the cloud provider API to fetch existing metadata.
  - Examples: Looking up the latest official Amazon Linux/RHEL AMI, querying existing default VPC IDs, subnets, or certificates.
- **Syntax**:
  ```hcl
  data "aws_ami" "joindevops" {
    most_recent = true
    owners      = ["973714476881"]

    filter {
      name   = "name"
      values = ["Redhat-9-DevOps-Practice"]
    }
  }
  ```
- **Referencing in Resources**:
  ```hcl
  resource "aws_instance" "example" {
    ami = data.aws_ami.joindevops.id # Dynamically fetched AMI ID
  }
  ```

---

### Terraform Built-in Functions & `merge()`
- **Key Rule**: **You CANNOT write custom functions in Terraform!**
  - Unlike Python or JavaScript, Terraform (HCL) is purely declarative and does not support user-defined functions (`def` or `function()`).
  - You must use Terraform's rich library of built-in functions (e.g., `length()`, `index()`, `lookup()`, `keys()`, `values()`, `merge()`, `toset()`, `concat()`).
- **The `merge()` Function**:
  - Combines two or more maps into a single map.
  - **Conflict Resolution Rule**: If the same key exists in multiple maps, **the rightmost map's value wins** (overwrites earlier values).

```hcl
# Example from Class:
map1 = { a = "b", c = "d" }
map2 = { d = "e", c = "z" }
map3 = { d = "e", e = "f" }

merged = merge(map1, map2, map3)
# Result:
# a = "b"
# c = "z"  (Overwritten by map2)
# d = "e"  (Set by map2 and confirmed by map3)
# e = "f"  (Set by map3)
```

---

### Saving JSON Outputs via CLI
In automated pipelines, downstream scripts (Ansible, Python, Shell) often need Terraform output data:

```bash
# Save complete Terraform output as formatted JSON
terraform output -json > ec2_output.json
```

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Session 32 code is organized into four separate folders under `terraform/`. Click the links below to open files directly in your IDE:

#### Project 1: `for_each` Loop & DNS
- **Directory**: [terraform/for_each/](../../terraform/for_each)
- **Provider**: [for_each/provider.tf](../../terraform/for_each/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/for_each/provider.tf)
- **Variables**: [for_each/variable.tf](../../terraform/for_each/variable.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/for_each/variable.tf)
- **EC2 & SG**: [for_each/ec2.tf](../../terraform/for_each/ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/for_each/ec2.tf)
- **Route 53 DNS**: [for_each/r53.tf](../../terraform/for_each/r53.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/for_each/r53.tf)
- **Outputs**: [for_each/outputs.tf](../../terraform/for_each/outputs.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/for_each/outputs.tf)

#### Project 2: Dynamic Ingress Blocks
- **Directory**: [terraform/dynamic/](../../terraform/dynamic)
- **Provider**: [dynamic/provider.tf](../../terraform/dynamic/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/dynamic/provider.tf)
- **Variables**: [dynamic/variables.tf](../../terraform/dynamic/variables.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/dynamic/variables.tf)
- **EC2 & Dynamic SG**: [dynamic/ec2.tf](../../terraform/dynamic/ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/dynamic/ec2.tf)

#### Project 3: Data Sources
- **Directory**: [terraform/data-sources/](../../terraform/data-sources)
- **Provider**: [data-sources/provider.tf](../../terraform/data-sources/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/data-sources/provider.tf)
- **Data Query**: [data-sources/data.tf](../../terraform/data-sources/data.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/data-sources/data.tf)
- **EC2 & SG**: [data-sources/aws_ec2.tf](../../terraform/data-sources/aws_ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/data-sources/aws_ec2.tf)
- **Outputs**: [data-sources/outputs.tf](../../terraform/data-sources/outputs.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/data-sources/outputs.tf)

#### Project 4: Built-in Functions & `merge()`
- **Directory**: [terraform/functions/](../../terraform/functions)
- **Provider**: [functions/provider.tf](../../terraform/functions/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/functions/provider.tf)
- **Variables**: [functions/variables.tf](../../terraform/functions/variables.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/functions/variables.tf)
- **EC2 & SG with Merge**: [functions/aws_ec2.tf](../../terraform/functions/aws_ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/functions/aws_ec2.tf)

---

### Architectural Diagrams

#### A. `count` vs `for_each` State Mapping
```
COUNT (Numeric Indexing - Brittle):
State Address: aws_instance.example[0] -> mongodb
State Address: aws_instance.example[1] -> redis
State Address: aws_instance.example[2] -> mysql
* PROBLEM: If you delete "redis", index [1] becomes "mysql", causing destructive recreation!

FOR_EACH (String Key Mapping - Resilient):
State Address: aws_instance.example["mongodb"] -> mongodb
State Address: aws_instance.example["redis"]   -> redis
State Address: aws_instance.example["mysql"]   -> mysql
* BENEFIT: If you delete "redis", only aws_instance.example["redis"] is destroyed.
           "mongodb" and "mysql" remain untouched!
```

#### B. Dynamic Block Unrolling Flow
```
+-------------------------------------------------------------+
|               variable "ingress_rules" (List of Maps)       |
|               - Port 22  (SSH)                              |
|               - Port 443 (HTTPS)                            |
|               - Port 3306 (MySQL)                           |
+-------------------------------------------------------------+
                              │
                              │ Iterated via dynamic "ingress"
                              ▼
+-------------------------------------------------------------+
|              aws_security_group.allow_tls                   |
|                                                             |
|   ├── Ingress Rule 1: TCP/22   from 0.0.0.0/0               |
|   ├── Ingress Rule 2: TCP/443  from 0.0.0.0/0               |
|   └── Ingress Rule 3: TCP/3306 from 0.0.0.0/0               |
+-------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Project 1: `for_each` Loop & DNS Automation (`terraform/for_each/`)

#### [`variable.tf` in `terraform/for_each/`](../../terraform/for_each/variable.tf)

```hcl
12: # This should be converted into set
13: variable "instances" {
14:   type    = list(any)
15:   default = ["mongodb", "redis"]
16: }
17: 
18: variable "zone_id" {
19:   default = "Z082192717Y56TLJLLOXS"
20: }
21: 
22: variable "domain_name" {
23:   default = "aitechapp.fun"
24: }
```
- **Lines 13-16 (`instances`)**: Defines the target components. Notice it is declared as a `list(any)`. In `ec2.tf`, `toset()` will convert this list into a set to supply unique string keys for `for_each`.

---

#### [`ec2.tf` in `terraform/for_each/`](../../terraform/for_each/ec2.tf)

```hcl
9: resource "aws_instance" "example" {
10:   ami = "ami-0220d79f3f480ecf5"
11:   # 'toset()' converts a list into a set of unique string elements
12:   for_each               = toset(var.instances)
13:   instance_type          = "t3.micro"
14:   vpc_security_group_ids = [aws_security_group.allow_tls.id]
15: 
16:   tags = {
17:     # 'each.key' contains the current element string (e.g., 'mongodb', 'redis')
18:     Name    = each.key
19:     Project = "roboshop"
20:   }
21: }
```
- **Line 12 (`for_each = toset(var.instances)`)**:
  - `toset()` converts `["mongodb", "redis"]` into a unique set.
  - Terraform creates two named resources in state:
    - `aws_instance.example["mongodb"]`
    - `aws_instance.example["redis"]`
- **Line 18 (`Name = each.key`)**:
  - In each iteration, `each.key` dynamically evaluates to `"mongodb"` or `"redis"`.

---

#### [`r53.tf` in `terraform/for_each/`](../../terraform/for_each/r53.tf)

```hcl
8: resource "aws_route53_record" "roboshop_internal" {
9:   for_each        = aws_instance.example
10:   zone_id         = var.zone_id
11:   name            = "${each.key}.${var.domain_name}"
12:   type            = "A"
13:   ttl             = 1
14:   records         = [each.value.private_ip]
15:   allow_overwrite = true
16: }
17: 
22: resource "aws_route53_record" "roboshop_public" {
23:   # Only create this record if 'frontend' exists in the instances map
24:   for_each        = contains(keys(aws_instance.example), "frontend") ? { "frontend" = aws_instance.example["frontend"] } : {}
25:   zone_id         = var.zone_id
26:   name            = "roboshop.${var.domain_name}"
27:   type            = "A"
28:   ttl             = 1
29:   records         = [each.value.public_ip]
30:   allow_overwrite = true
31: }
```
- **Line 9 (`for_each = aws_instance.example`)**:
  - Directly iterates over the resource map generated by `aws_instance.example`.
  - `each.key` is the instance name (`"mongodb"`).
  - `each.value` is the complete resource object; `each.value.private_ip` dynamically extracts the private IP for internal DNS.
- **Line 24 (`roboshop_public` conditional generation)**:
  - Uses functions `contains()` and `keys()`:
    `contains(keys(aws_instance.example), "frontend")`
  - If `"frontend"` is in the map, it passes `{ "frontend" = ... }` to create the public record.
  - If `"frontend"` is omitted from `var.instances`, it passes an empty map `{}` $\rightarrow$ **Zero public records are created**, preventing plan crashes!

---

### Project 2: Dynamic Ingress Blocks (`terraform/dynamic/`)

#### [`variables.tf` in `terraform/dynamic/`](../../terraform/dynamic/variables.tf)

```hcl
1: variable "ingress_rules" {
2:   default = [
3:     {
4:       port        = 22
5:       cidr        = ["0.0.0.0/0"]
6:       description = "allowing port number 22 from internet"
7:     },
8:     {
9:       port        = 443
10:       cidr        = ["0.0.0.0/0"]
11:       description = "allowing port number 443 from internet"
12:     },
13:     {
14:       port        = 3306
15:       cidr        = ["0.0.0.0/0"]
16:       description = "allowing port number 3306 from internet"
17:     }
18:   ]
19: }
```
- Declares a list of objects defining port numbers, CIDR ranges, and descriptions. Adding a new firewall rule in production simply means appending an object to this list!

---

#### [`ec2.tf` in `terraform/dynamic/`](../../terraform/dynamic/ec2.tf)

```hcl
33:   dynamic "ingress" {
34:     for_each = toset(var.ingress_rules)
35:     content {
36:       from_port   = ingress.value.port
37:       to_port     = ingress.value.port
38:       protocol    = "tcp"
39:       cidr_blocks = ingress.value.cidr
40:       description = ingress.value.description
41:     }
42:   }
```
- **Line 33 (`dynamic "ingress"`)**: Generates nested `ingress {}` blocks.
- **Line 34 (`for_each = toset(var.ingress_rules)`)**: Loops through the 3 rule definitions.
- **Lines 36-40**: Unrolls each item into `ingress.value.port`, `ingress.value.cidr`, `ingress.value.description`.

---

### Project 3: Data Sources (`terraform/data-sources/`)

#### [`data.tf` in `terraform/data-sources/`](../../terraform/data-sources/data.tf)

```hcl
9: data "aws_ami" "joindevops" {
10:   most_recent = true
11:   owners      = ["973714476881"] # AWS Account ID of AMI owner
12: 
13:   # Filter by AMI name pattern
14:   filter {
15:     name   = "name"
16:     values = ["Redhat-9-DevOps-Practice"]
17:   }
18: 
19:   filter {
20:     name   = "root-device-type"
21:     values = ["ebs"]
22:   }
23: 
24:   filter {
25:     name   = "virtualization-type"
26:     values = ["hvm"]
27:   }
28: }
29: 
31: data "aws_instance" "terraform_instance" {
32:   instance_id = "i-0aa84494a46dd5be9"
33: }
```
- **Lines 9-28 (`data "aws_ami" "joindevops"`)**:
  - Queries the AWS EC2 API for the latest AMI owned by account `973714476881`.
  - Filters by image name `"Redhat-9-DevOps-Practice"` and virtualization `"hvm"`.
  - Sets `most_recent = true` to automatically fetch newer patched images without changing code!
- **Lines 31-33 (`data "aws_instance" "terraform_instance"`)**:
  - Queries live metadata of an already running EC2 instance by its AWS Instance ID.

---

#### [`aws_ec2.tf` in `terraform/data-sources/`](../../terraform/data-sources/aws_ec2.tf)

```hcl
4: resource "aws_instance" "example" {
5:   # Dynamically use the AMI ID returned by the data source block in data.tf
6:   ami                    = data.aws_ami.joindevops.id
7:   instance_type          = "t3.micro"
8:   vpc_security_group_ids = [aws_security_group.allow_tls.id]
```
- **Line 6 (`ami = data.aws_ami.joindevops.id`)**: Dynamically injects the queried AMI ID without hardcoding string IDs.

---

### Project 4: Built-in Functions & `merge()` (`terraform/functions/`)

#### [`variables.tf` in `terraform/functions/`](../../terraform/functions/variables.tf)

```hcl
1: variable "common_tags" {
2:   default = {
3:     Project     = "roboshop"
4:     Terraform   = "true"
5:     Environment = "dev"
6:   }
7: }
8: 
9: variable "ec2_tags" {
10:   default = {
11:     Name        = "functions-demo"
12:     Environment = "prod"
13:   }
14: }
```
- Notice that both `common_tags` and `ec2_tags` define the key `Environment`.

---

#### [`aws_ec2.tf` in `terraform/functions/`](../../terraform/functions/aws_ec2.tf)

```hcl
14:   tags = merge(
15:     var.common_tags,
16:     var.ec2_tags
17:   )
```
- **Lines 14-17 (`tags = merge(...)`)**:
  - Evaluates `merge(var.common_tags, var.ec2_tags)`.
  - Base tags: `Project = "roboshop"`, `Terraform = "true"`.
  - Instance tag: `Name = "functions-demo"`.
  - **Key Collision**: `common_tags` specifies `Environment = "dev"`, but `ec2_tags` specifies `Environment = "prod"`.
  - **Result**: `Environment` becomes `"prod"` because `var.ec2_tags` is the rightmost argument.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are copyable and should be executed in terminal:

### 1. Running `for_each` & Exporting JSON Outputs
```bash
# Navigate to for_each directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/for_each

# Initialize and validate
terraform init
terraform validate

# Plan and inspect named resource addresses
terraform plan

# Apply changes to AWS
terraform apply -auto-approve

# Save the full state/output to JSON for downstream consumption
terraform output -json > ec2_output.json
cat ec2_output.json

# Clean up infrastructure
terraform destroy -auto-approve
```

---

### 2. Testing Dynamic Ingress Rules
```bash
# Navigate to dynamic directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/dynamic

# Initialize
terraform init

# Run plan and verify that 3 distinct ingress rules (22, 443, 3306) are generated
terraform plan

# Notice in plan output:
# + ingress { from_port = 22, to_port = 22, protocol = "tcp" ... }
# + ingress { from_port = 443, to_port = 443, protocol = "tcp" ... }
# + ingress { from_port = 3306, to_port = 3306, protocol = "tcp" ... }
```

---

### 3. Testing Data Source AMI Lookup
```bash
# Navigate to data-sources directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/data-sources

# Initialize
terraform init

# Plan and verify the resolved AMI ID
terraform plan

# Inspect outputs without applying changes
terraform apply -auto-approve
terraform output ami_id
terraform destroy -auto-approve
```

---

### 4. Testing `merge()` Function Precedence
```bash
# Navigate to functions directory
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/functions

# Initialize
terraform init

# Generate execution plan
terraform plan

# Verify that under 'tags', Environment is "prod" (overriding "dev"):
# tags = {
#   "Environment" = "prod"
#   "Name"        = "functions-demo"
#   "Project"     = "roboshop"
#   "Terraform"   = "true"
# }
```

---

## 5. Commands & CLI Flags Reference Table

| Command | Arguments / Flags | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `terraform output` | `-json` | Outputs all values formatted in strict JSON syntax for script automation. |
| `terraform output -json > file.json` | Shell redirection | Exports infrastructure outputs to a JSON file (consumed by Ansible inventory). |
| `terraform console` | Interactive shell | Opens an interactive HCL REPL to test functions like `merge()`, `length()`, `toset()`. |
| `terraform plan` | `-refresh=true` | Queries data sources and refreshes existing state against AWS before planning. |
| `terraform apply` | `-target=aws_instance.example["mongodb"]` | Applies changes strictly to a specific instance in the `for_each` map. |

---

## 6. Official Documentation & References
- **The `for_each` Meta-Argument**: [developer.hashicorp.com/terraform/language/meta-arguments/for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)
- **The `dynamic` Blocks**: [developer.hashicorp.com/terraform/language/expressions/dynamic-blocks](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks)
- **Data Sources Documentation**: [developer.hashicorp.com/terraform/language/data-sources](https://developer.hashicorp.com/terraform/language/data-sources)
- **Terraform Built-in Functions Reference**: [developer.hashicorp.com/terraform/language/functions](https://developer.hashicorp.com/terraform/language/functions)
- **`merge()` Function**: [developer.hashicorp.com/terraform/language/functions/merge](https://developer.hashicorp.com/terraform/language/functions/merge)
- **`toset()` Function**: [developer.hashicorp.com/terraform/language/functions/toset](https://developer.hashicorp.com/terraform/language/functions/toset)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why is `for_each` preferred over `count` when provisioning heterogenous cloud resources?
**Answer**:
`count` assigns integer indices (`[0]`, `[1]`, `[2]`). If an element is removed from the middle or beginning of the input list, all subsequent resource indices shift down. Terraform interprets this as modifying existing resources and triggers unintended destroy-and-recreate cycles across unrelated infrastructure.
In contrast, `for_each` binds each resource to a stable, unique string key (e.g., `["mongodb"]`). Removing an item deletes only that specific named resource, leaving all other resources completely untouched.

---

### Q2: Can we write custom user-defined functions in Terraform?
**Answer**:
**No.** Terraform does not support user-defined functions. All functions available in Terraform (such as `merge()`, `length()`, `lookup()`, `element()`, `cidrsubnet()`) are built into the Terraform core engine. For complex logic, engineers can combine built-in functions, use local values (`locals`), or employ external data sources/scripts.

---

### Q3: What happens when two maps merged using `merge()` share the same key?
**Answer**:
The `merge()` function resolves key collisions based on argument order: the value from the **rightmost map** takes precedence and overwrites any previous values for that key.
Example: `merge({ env = "dev" }, { env = "prod" })` evaluates to `{ env = "prod" }`.

---

### Q4: When should you use a `dynamic` block instead of multiple static blocks?
**Answer**:
A `dynamic` block should be used when the number of repeated nested blocks inside a resource varies dynamically based on input variables (e.g., security group ingress/egress rules, autoscaling tag blocks, EBS block device attachments). It upholds the DRY principle by maintaining rule definitions in a centralized variable list or map.

---

### Q5: What is the difference between a `resource` block and a `data` source block?
**Answer**:
- `resource`: Manages infrastructure lifecycle (Create, Read, Update, Delete). Terraform provisions and manages the cloud entity.
- `data`: Read-only query. Fetches information from existing cloud resources (created outside Terraform or by another team) without managing or modifying their lifecycle.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Passing a Raw `list` Directly to `for_each`**:
   - *Error*: `The given "for_each" argument value is unsuitable: "for_each" supports maps and sets of strings, and does not support lists.`
   - *Fix*: Wrap the list with `toset()`: `for_each = toset(var.my_list)`.
2. **Overusing Dynamic Blocks for Simple Code**:
   - *Problem*: Using `dynamic` blocks for one or two static arguments degrades code readability.
   - *Best Practice*: Use static blocks for 1-2 standard rules; reserve `dynamic` blocks only for lists of rules that vary across environments.
3. **Data Source Query Finding Zero or Multiple Resources**:
   - *Error*: `Error: Your query returned no results.` or `Error: multiple AMIs matched; use most_recent to choose one.`
   - *Fix*: Add `most_recent = true` or refine `filter` attributes to ensure deterministic matching.
4. **Modifying Sensitive Maps without Understanding Precedence**:
   - *Problem*: Passing parameters in wrong order in `merge(var.specific_tags, var.common_tags)` causes enterprise default tags to overwrite component-specific tags.
   - *Fix*: Always place specific or overriding maps on the right side: `merge(var.common_tags, var.specific_tags)`.

---

## 9. Session Metadata & Timestamps

- **Recap & For_Each Loop**: `00:00 - 25:00`
- **Dynamic Blocks (Ingress Rules)**: `25:00 - 45:00`
- **Terraform Data Sources**: `45:00 - 01:05:00`
- **Built-in Functions & `merge()`**: `01:05:00 - 01:25:00`
- **Interview Questions & Answers**: `01:25:00`
- **Session Q&A**: `01:26:28`

### Doubts & AI Clarification Link
- [Session 32 AI Clarification Chat](https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049)