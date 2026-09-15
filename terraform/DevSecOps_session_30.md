# Thursday, 19 February 2026
# Session 30 - Ansible Using Keys, Terraform Advantages, Setup & Installation, EC2 & SG Creation
## Comprehensive Class Notes, Practical Code Teardown & Execution Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Ansible SSH Key-Based Authentication](#ansible-ssh-key-based-authentication)
   - [Terraform & Infrastructure as Code (IaaC)](#terraform--infrastructure-as-code-iaac)
   - [Top 6 Advantages of Terraform](#top-6-advantages-of-terraform)
   - [Prerequisites & Environment Setup](#prerequisites--environment-setup)
   - [Terraform HCL Syntax & Block Anatomy](#terraform-hcl-syntax--block-anatomy)
2. [Workspace Project Code Mapping & Architecture](#2-workspace-project-code-mapping--architecture)
   - [Local File Links & Repository Mapping](#local-file-links--repository-mapping)
   - [Resource Dependency Architecture (DAG Diagram)](#resource-dependency-architecture-dag-diagram)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Provider Configuration (`provider.tf`)](#provider-configuration-providertf)
   - [Security Group (`aws_ec2.tf`: Lines 20-45)](#security-group-virtual-firewall-aws_ec2tf)
   - [EC2 Instance (`aws_ec2.tf`: Lines 5-14)](#ec2-compute-instance-aws_ec2tf)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [Step 1: AWS IAM User & Credentials Configuration](#step-1-aws-iam-user--credentials-configuration)
   - [Step 2: Navigate to Project Directory](#step-2-navigate-to-project-directory)
   - [Step 3: Terraform Initialization (`init`)](#step-3-terraform-initialization-init)
   - [Step 4: Format & Validate (`fmt` & `validate`)](#step-4-format--validate-fmt--validate)
   - [Step 5: Generate Execution Plan (`plan`)](#step-5-generate-execution-plan-plan)
   - [Step 6: Provision Infrastructure (`apply`)](#step-6-provision-infrastructure-apply)
   - [Step 7: Clean Up Resources (`destroy`)](#step-7-clean-up-resources-destroy)
5. [Terraform CLI Commands & Flags Teardown](#5-terraform-cli-commands--flags-teardown)
   - [Core Lifecycle Commands Table](#core-lifecycle-commands-table)
   - [CLI Flags Deep-Dive Table](#cli-flags-deep-dive-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Ansible SSH Key-Based Authentication
In automated DevOps pipelines, interactive password prompts block execution. Key-based authentication provides secure, non-interactive SSH communication:

```
+------------------------------------+                  +------------------------------------+
|       Ansible Control Server       |                  |        Target Managed Node         |
|                                    |   SSH (TCP:22)   |                                    |
| - Private Key:                     | ---------------> | - Public Key:                      |
|   /home/ec2-user/.ssh/id_rsa       |   PrivateKey     |   /home/ec2-user/.ssh/             |
|   (chmod 400 - strict permissions) |    Matches       |   authorized_keys                  |
| - ansible.cfg (private_key_file)   |   PublicKey      | - /etc/sudoers.d/ansible           |
|                                    |                  |   (NOPASSWD: ALL)                  |
+------------------------------------+                  +------------------------------------+
```

- **Core Rule**:
  - `Ansible Control Node` -> Holds the **Private Key** (`id_rsa`).
  - `Managed Node` -> Holds the **Public Key** (`authorized_keys`).
  - Command: `ssh -i <private-key> ec2-user@<IP>`
- **Server Setup Steps**:
  1. Create a dedicated automation user for Ansible on all target servers (`useradd ansible`).
  2. Grant passwordless sudo access in `/etc/sudoers.d/ansible` (`ansible ALL=(ALL) NOPASSWD: ALL`).
  3. Ensure SSH key pairs are RSA-based (`ssh-keygen -t rsa -b 4096`).
  4. Place the private key on the Ansible controller and set permission `chmod 400 /path/to/private-key`.
  5. Configure the key path in `ansible.cfg` under `private_key_file`.

---

### Terraform & Infrastructure as Code (IaaC)
- **What is Terraform?**
  - An open-source **Declarative Infrastructure as Code (IaaC)** tool developed by HashiCorp.
  - Written in Go; uses **HashiCorp Configuration Language (HCL)**.
  - Multi-cloud and platform-agnostic: Manages AWS, Azure, GCP, Kubernetes, GitHub, VMware, etc. via modular plugins called **Providers**.
- **Alternative IaaC Tools**:
  - **AWS CloudFormation**: AWS-only, uses JSON/YAML.
  - **Azure ARM / Bicep**: Azure-only declarative templates.
  - **Pulumi**: Multi-cloud, uses imperative programming languages (TypeScript, Python, Go, C#).

---

### Top 6 Advantages of Terraform
1. **Version Control & Auditability**:
   - Infrastructure configurations are tracked in Git.
   - History of every change is recorded; easy to review pull requests, trace who changed what, and roll back safely.
2. **Consistent Infrastructure (Environment Parity)**:
   - Same parameterized code templates deploy DEV, UAT, and PROD without manual drift.
3. **Automated Lifecycle Management (CRUD)**:
   - Tracks real-world state and inventory. Detects differences between desired `.tf` files and actual cloud state, managing Create, Read, Update, and Delete operations automatically.
4. **Cost Optimization & Ephemeral Environments**:
   - Spin up complete environments for testing and destroy them immediately after validation with `terraform destroy`, eliminating idle resource costs.
5. **Automatic Dependency Management**:
   - Builds a Directed Acyclic Graph (DAG) under the hood. Automatically calculates what resource must be created first (e.g., Security Group before EC2 instance) and parallelizes independent resources.
6. **Reusable Infrastructure (Modules)**:
   - Avoids repetitive code by packaging standard architectures into DRY (Don't Repeat Yourself) reusable Terraform modules.

---

### Prerequisites & Environment Setup
To begin working with Terraform on AWS:
1. **Install Terraform**:
   - Download the Terraform binary from [developer.hashicorp.com/terraform/install](https://developer.hashicorp.com/terraform/install).
   - Set up the binary directory in your system `PATH` environment variable.
2. **Install AWS CLI v2**:
   - Follow the official AWS CLI v2 installer for Mac, Linux, or Windows.
3. **Create AWS IAM User & Access Keys**:
   - Create an IAM User with required infrastructure permissions (e.g., `AmazonEC2FullAccess`).
   - Generate an **Access Key ID** and **Secret Access Key**; download the `.csv` file.
4. **Configure AWS Credentials**:
   - Run `aws configure` on your terminal and provide the downloaded keys, setting default region to `us-east-1`.
5. **Create Git Repository**:
   - Create a dedicated repository for your Terraform code to version-control all `.tf` files.

---

### Terraform HCL Syntax & Block Anatomy
Terraform uses declarative HCL blocks with standard syntax:

```hcl
block_type "resource_type" "local_name" {
  argument_key = "argument_value" # Configuration attributes
}
```

- **Block Types**:
  - `terraform`: Configures Terraform engine settings and required provider versions.
  - `provider`: Configures the target cloud platform plugin (e.g., `aws`, `azurerm`, `google`).
  - `resource`: Declares infrastructure components to create and manage (e.g., `aws_instance`, `aws_security_group`).
  - `data`: Queries existing cloud infrastructure without creating new resources.
  - `variable`: Declares input parameters to make configurations dynamic.
  - `output`: Exposes values (like Public IP, ARN) to CLI or other modules.
  - `locals`: Defines local temporary variables within a module.
  - `module`: Calls reusable child modules.

#### Resource Block Breakdown:
```hcl
resource "aws_instance" "example" {
  ami           = "ami-0220d79f3f480ecf5" # Argument: OS image
  instance_type = "t3.micro"              # Argument: Size
}
```
- `resource` -> Core Terraform keyword.
- `"aws_instance"` -> Resource type defined by the AWS Provider API (Fixed syntax).
- `"example"` -> Local reference name chosen by the engineer for internal Terraform graph referencing.
- Arguments -> Key-value pairs (`ami`, `instance_type`, `vpc_security_group_ids`, `tags`).

---

## 2. Workspace Project Code Mapping & Architecture

### Local File Links & Repository Mapping
The practical code for Session 30 resides in the project workspace. Click on the file links below to open them directly in your IDE:

- **Project Directory**: [terraform/aws_ec2/](../../terraform/aws_ec2)
- **Provider File**: [provider.tf](../../terraform/aws_ec2/provider.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/aws_ec2/provider.tf)
- **Infrastructure File**: [aws_ec2.tf](../../terraform/aws_ec2/aws_ec2.tf) | [GitHub Source](https://github.com/RamCharanKolaDevelopment/terraform/blob/main/aws_ec2/aws_ec2.tf)
- **Dependency Lock File**: [.terraform.lock.hcl](../../terraform/aws_ec2/.terraform.lock.hcl)
- **State File**: [terraform.tfstate](../../terraform/aws_ec2/terraform.tfstate)

---

### Resource Dependency Architecture (DAG Diagram)

```
+---------------------------------------------------------------------------------+
|                               AWS Cloud (us-east-1)                             |
|                                                                                 |
|   +-------------------------------------------------------------------------+   |
|   |                  Security Group: "allow_tls"                            |   |
|   |                  AWS SG Name: "allow-all-terraform"                     |   |
|   |                                                                         |   |
|   |   Inbound (Ingress):   Port 0-65535, Protocol ALL (-1) from 0.0.0.0/0  |   |
|   |   Outbound (Egress):   Port 0-65535, Protocol ALL (-1) to 0.0.0.0/0    |   |
|   +-------------------------------------------------------------------------+   |
|                                        ▲                                        |
|                                        │ Implicit Dependency via DAG            |
|                                        │ vpc_security_group_ids = [id]          |
|                                        ▼                                        |
|   +-------------------------------------------------------------------------+   |
|   |                     EC2 Instance: "example"                             |   |
|   |                     AMI: ami-0220d79f3f480ecf5 (RHEL-9)                 |   |
|   |                     Type: t3.micro                                      |   |
|   |                     Tags: Name = "roboshop", Project = "roboshop"       |   |
|   +-------------------------------------------------------------------------+   |
|                                                                                 |
+---------------------------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Provider Configuration: [provider.tf](../../terraform/aws_ec2/provider.tf)

```hcl
1: terraform {
2:   required_providers {
3:     aws = {
4:       source  = "hashicorp/aws"
5:       version = "~> 6.0" # Terraform AWS provider version
6:     }
7:   }
8: }
9: 
10: # Configure the AWS Provider
11: provider "aws" {
12:   region = "us-east-1"
13: }
```

#### Line-by-Line Teardown:
- **Lines 1-8 (`terraform { required_providers { ... } }`)**:
  - `terraform`: Global meta-block that configures the behavior of Terraform itself.
  - `required_providers`: Declares which provider plugins this configuration depends on.
  - `source = "hashicorp/aws"`: Fully qualified registry address (`registry.terraform.io/hashicorp/aws`). Instructs Terraform to fetch HashiCorp's official AWS provider plugin.
  - `version = "~> 6.0"`: Pessimistic version constraint operator (`~>`). Restricts updates to non-breaking minor versions (allows `>= 6.0.0` and `< 7.0.0`). Prevents unexpected breaking changes when new major versions release.
- **Lines 10-13 (`provider "aws" { region = "us-east-1" }`)**:
  - `provider "aws"`: Initializes the downloaded AWS plugin.
  - `region = "us-east-1"`: Sets Northern Virginia as the target AWS region for all resources in this directory.
  - **Zero Hardcoded Secrets**: Notice there are no `access_key` or `secret_key` attributes written here. Terraform adheres to the standard AWS credential chain, automatically reading credentials from `~/.aws/credentials` (populated via `aws configure`) or environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

---

### Security Group (Virtual Firewall): [aws_ec2.tf](../../terraform/aws_ec2/aws_ec2.tf)

```hcl
20: resource "aws_security_group" "allow_tls" {
21:   name        = "allow-all-terraform" # Name identifier in AWS VPC
22:   description = "Allow TLS inbound traffic and all outbound traffic"
23: 
24:   # Outbound Rules: Allow all outbound traffic to anywhere (Internet & VPC)
25:   egress {
26:     from_port        = 0
27:     to_port          = 0
28:     protocol         = "-1" # -1 signifies all protocols (TCP, UDP, ICMP, etc.)
29:     cidr_blocks      = ["0.0.0.0/0"]
30:     ipv6_cidr_blocks = ["::/0"]
31:   }
32: 
33:   # Inbound Rules: Allow all incoming traffic (Demo setup)
34:   ingress {
35:     from_port        = 0
36:     to_port          = 0
37:     protocol         = "-1"
38:     cidr_blocks      = ["0.0.0.0/0"]
39:     ipv6_cidr_blocks = ["::/0"]
40:   }
41: 
42:   tags = {
43:     Name = "allow-all-terraform"
44:   }
45: }
```

#### Line-by-Line Teardown:
- **Line 20**:
  - `resource "aws_security_group" "allow_tls"`: Creates a security group resource. `"allow_tls"` is the internal Terraform identifier used by other resources in code to access its attributes (e.g., `aws_security_group.allow_tls.id`).
- **Lines 21-22**:
  - `name = "allow-all-terraform"`: The external name shown in the AWS Management Console and AWS CLI.
  - `description`: Explains the firewall's purpose for security audits.
- **Lines 24-31 (`egress` block - Outbound Rules)**:
  - **Crucial Terraform Distinction**: Unlike the AWS Web Console (which silently injects an allow-all outbound rule), Terraform creates **completely empty** security groups by default. If you omit the `egress` block in Terraform, your EC2 instance will have no outbound internet access and commands like `yum install` or `apt update` will hang indefinitely!
  - `protocol = "-1"`: Special identifier representing **all IP protocols** (TCP, UDP, ICMP).
  - `from_port = 0`, `to_port = 0`: Required when protocol is `"-1"` to denote all port numbers (0-65535).
  - `cidr_blocks = ["0.0.0.0/0"]`: Permits outbound packets to any IPv4 internet address.
  - `ipv6_cidr_blocks = ["::/0"]`: Permits outbound packets to any IPv6 internet address.
- **Lines 33-40 (`ingress` block - Inbound Rules)**:
  - Defines incoming firewall permissions. Here, `protocol = "-1"` with `cidr_blocks = ["0.0.0.0/0"]` opens all ports for training demonstration. In production, restrict this to specific ports (e.g., port 22 for SSH from a bastion IP, port 80/443 for web traffic).
- **Lines 42-44 (`tags`)**:
  - Key-value metadata attached to the AWS Security Group for tracking and billing.

---

### EC2 Compute Instance: [aws_ec2.tf](../../terraform/aws_ec2/aws_ec2.tf)

```hcl
5: resource "aws_instance" "example" {
6:   ami                    = "ami-0220d79f3f480ecf5"           # Amazon Machine Image (AMI) ID for the OS
7:   instance_type          = "t3.micro"                        # Instance size determining CPU and RAM
8:   vpc_security_group_ids = [aws_security_group.allow_tls.id] # Reference security group ID dynamically
9: 
10:   tags = {
11:     Name    = "roboshop" # "Name" tag assigns the display name in AWS Management Console
12:     Project = "roboshop"
13:   }
14: }
```

#### Line-by-Line Teardown:
- **Line 5**:
  - `resource "aws_instance" "example"`: Instructs the AWS provider to provision an Amazon EC2 virtual machine referenced internally as `example`.
- **Line 6 (`ami = "ami-0220d79f3f480ecf5"`)**:
  - The Amazon Machine Image ID containing the bootable operating system (RHEL-9 dev image in `us-east-1`).
  - *Rule*: AMI IDs are region-specific; this AMI ID is valid specifically in `us-east-1`.
- **Line 7 (`instance_type = "t3.micro"`)**:
  - Specifies the instance hardware footprint (2 vCPUs, 1 GiB RAM). Free-tier eligible.
- **Line 8 (`vpc_security_group_ids = [aws_security_group.allow_tls.id]`)**:
  - **Implicit Dependency & Dynamic Attribute Reference**: Instead of hardcoding a raw ID like `"sg-01a2b3c4d5"`, Terraform dynamically reads the `.id` attribute generated by the `aws_security_group.allow_tls` resource.
  - **DAG Resolution**: Because `aws_instance.example` requires `aws_security_group.allow_tls.id`, Terraform's Directed Acyclic Graph automatically enforces that the security group must be created **first**. Once AWS returns the security group ID, Terraform passes it into the EC2 launch API call.
- **Lines 10-13 (`tags`)**:
  - `Name = "roboshop"`: Populates the Name column in the AWS EC2 Console.
  - `Project = "roboshop"`: Custom tag for resource filtering and cost allocation.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable and must be executed in order:

### Step 1: AWS IAM User & Credentials Configuration
Ensure your terminal is authenticated with AWS:

```bash
# Configure your AWS CLI credentials
aws configure
# AWS Access Key ID [None]: <YOUR_ACCESS_KEY_ID>
# AWS Secret Access Key [None]: <YOUR_SECRET_ACCESS_KEY>
# Default region name [None]: us-east-1
# Default output format [None]: json

# Verify authentication identity
aws sts get-caller-identity
```

---

### Step 2: Navigate to Project Directory
> **Golden Rule**: You must always change your working directory to the folder where your `.tf` files reside before running any Terraform commands.

```bash
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2

# Verify files are present
ls -la
```

Expected files:
```text
aws_ec2.tf
provider.tf
```

---

### Step 3: Terraform Initialization (`init`)
Run this first when starting with a new or cloned repository:

```bash
terraform init
```

*What Terraform does under the hood*:
1. Reads `provider.tf` and identifies `hashicorp/aws` constraint `~> 6.0`.
2. Connects to `registry.terraform.io` and downloads the AWS provider plugin binary into the local hidden directory `.terraform/providers/`.
3. Creates or updates `.terraform.lock.hcl` with SHA256 checksums to lock provider dependencies across team members.

---

### Step 4: Format & Validate (`fmt` & `validate`)
Ensure code is syntactically sound and follows HashiCorp formatting conventions:

```bash
# Canonicalize indentation and layout
terraform fmt

# Validate internal syntax and references without cloud API calls
terraform validate
```

Expected output:
```text
Success! The configuration is valid.
```

---

### Step 5: Generate Execution Plan (`plan`)
Perform a dry-run execution to preview proposed infrastructure changes before touching live cloud resources:

```bash
terraform plan
```

*Key details to inspect*:
- Symbol `+` denotes resources that will be newly created.
- Review attributes: `ami`, `instance_type`, `vpc_security_group_ids`.
- Summary line at bottom: `Plan: 2 to add, 0 to change, 0 to destroy.`

---

### Step 6: Provision Infrastructure (`apply`)
Apply the configuration to create the real AWS resources:

```bash
terraform apply
```

- When prompted: `Do you want to perform these actions?`, type: `yes`
- For non-interactive automated pipelines (CI/CD):
```bash
terraform apply -auto-approve
```

*Execution Result*:
1. AWS creates the Security Group `allow-all-terraform`.
2. AWS assigns an ID (e.g., `sg-0abc123456789`).
3. Terraform injects that ID into the EC2 instance launch request.
4. AWS launches the EC2 instance `roboshop`.
5. Terraform writes all metadata and IDs into `terraform.tfstate`.

---

### Step 7: Clean Up Resources (`destroy`)
Delete all created cloud infrastructure to prevent ongoing AWS billing:

```bash
terraform destroy
```

- When prompted, type: `yes`
- Or run with auto-approval:
```bash
terraform destroy -auto-approve
```

*Teardown Result*:
- Terraform reads `terraform.tfstate`.
- Reverses dependency order: Terminates the EC2 instance **first**, waits until fully terminated, and then deletes the Security Group.

---

## 5. Terraform CLI Commands & Flags Teardown

### Core Lifecycle Commands Table
| Command | Primary Function | When to Use in Workflow |
| :--- | :--- | :--- |
| `terraform init` | Downloads provider plugins, configures backend, locks versions. | First step in any folder, or after updating providers/modules. |
| `terraform fmt` | Rewrites HCL files to standard indentation and canonical layout. | Before every git commit to maintain consistent code style. |
| `terraform validate` | Verifies syntax, arguments, and internal consistency offline. | In pre-commit hooks, CI pipelines, and before planning. |
| `terraform plan` | Compares `.tf` code against state and live cloud to create a diff. | Before every deployment to preview additions, edits, deletes. |
| `terraform apply` | Provisions or updates real-world cloud resources via provider APIs. | To deploy infrastructure changes to target environments. |
| `terraform destroy` | Deletes all resources managed by the current state file. | For tearing down temporary demo, test, or feature environments. |
| `terraform show` | Displays human-readable output of current state or a plan file. | To inspect provisioned attributes (Public IPs, ARNs, VPC IDs). |
| `terraform state list` | Lists all resource addresses currently recorded in the state file. | Quick audit of resources managed by Terraform. |

---

### CLI Flags Deep-Dive Table
| Command | Flag | What It Does | Practical Real-World Use Case |
| :--- | :--- | :--- | :--- |
| `terraform init` | `-upgrade` | Upgrades all provider plugins and modules to the newest allowed version. | Upgrading AWS provider from `5.x` to `6.x`. |
| `terraform init` | `-reconfigure` | Disregards existing backend settings and reinitializes backend configuration. | Switching between different remote S3 backend buckets. |
| `terraform init` | `-migrate-state` | Reinitializes backend while migrating existing state data to the new backend. | Migrating from local `terraform.tfstate` to remote S3 backend. |
| `terraform plan` | `-out=<filename>` | Writes the generated plan to an encrypted binary file. | CI/CD pipelines: Guarantees `apply` runs the exact previewed plan. |
| `terraform plan` | `-detailed-exitcode` | Returns exit code `0` (no changes), `2` (changes present), or `1` (error). | In drift detection cron jobs to alert on unauthorized changes. |
| `terraform apply` | `-auto-approve` | Skips interactive approval prompt (`yes`). | Automated CI/CD pipelines (Jenkins, GitHub Actions). |
| `terraform apply` | `-replace="<address>"` | Marks a specific resource for recreation (destroy & re-create). | Replacing a corrupted or tainted EC2 instance. |
| `terraform apply` | `-refresh-only` | Updates state file with real-world infrastructure drift without modifying cloud. | Syncing state after someone modified a tag via AWS Console. |
| `terraform apply` | `-var="key=value"` | Sets an input variable value directly on the CLI. | `terraform apply -var="instance_type=t3.small"` |
| `terraform apply` | `-var-file="<file>"` | Loads variable values from a specific `.tfvars` file. | Environment deployments: `-var-file="prod.tfvars"`. |
| `terraform destroy` | `-target="<address>"` | Destroys only the specified resource and its dependents. | Deleting a standalone test instance without destroying the VPC. |
| `terraform fmt` | `-check` | Checks formatting without modifying files; exits non-zero if misformatted. | CI code quality linting gates. |
| `terraform fmt` | `-diff` | Shows line-by-line whitespace and formatting diffs. | Reviewing what `terraform fmt` wants to change. |

---

## 6. Official Documentation & References
- **Terraform CLI Official Documentation**: [developer.hashicorp.com/terraform/cli](https://developer.hashicorp.com/terraform/cli)
- **Terraform AWS Provider Registry**: [registry.terraform.io/providers/hashicorp/aws/latest/docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- **AWS Instance Resource Docs**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)
- **AWS Security Group Resource Docs**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)
- **AWS CLI v2 Quickstart Guide**: [docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)

---

## 7. High-Yield Interview Questions & Answers

### Q1: What is the purpose of `.terraform.lock.hcl` and should it be committed to Git?
**Answer**:
Introduced in Terraform 0.14, `.terraform.lock.hcl` is the Dependency Lock File. It records the exact version and cryptographic checksums (hashes) of all provider plugins used in the configuration.
- **Yes, it MUST be committed to Git**.
- It guarantees that every team member, build agent, and CI/CD pipeline downloads the exact same provider binaries, preventing "upstream dependency drift" and breaking changes.

---

### Q2: What is the difference between Implicit and Explicit Dependencies in Terraform?
**Answer**:
- **Implicit Dependency**: Created automatically when one resource references an attribute exported by another resource (e.g., `vpc_security_group_ids = [aws_security_group.allow_tls.id]`). Terraform's DAG engine deduces that the security group must be created first without any manual instructions.
- **Explicit Dependency**: Declared manually using the `depends_on` meta-argument when Terraform cannot automatically deduce the relationship (e.g., an EC2 instance that requires an S3 bucket or IAM Role policy attachment to exist before launching).

---

### Q3: What is `terraform.tfstate` and why should it NEVER be stored in public Git?
**Answer**:
`terraform.tfstate` is Terraform's state database and source of truth. It maps declared HCL resources to real-world cloud provider resource IDs, tracking metadata and attributes.
- Storing state in Git is a major security and operational anti-pattern because:
  1. **Secrets Exposure**: State files store resource attributes in plain text, including sensitive database passwords, private keys, and environment variables.
  2. **No State Locking**: Git cannot prevent race conditions when multiple engineers or CI pipelines run `terraform apply` concurrently, leading to state corruption.
  3. **Best Practice**: Use an encrypted Remote Backend (e.g., AWS S3 with KMS encryption and DynamoDB state locking).

---

### Q4: How does Terraform detect and handle Configuration Drift?
**Answer**:
Configuration Drift occurs when infrastructure is modified outside of Terraform (e.g., manual edits in the AWS Web Console or via AWS CLI).
- When `terraform plan` or `terraform apply` runs, Terraform first executes a **Refresh phase** against the cloud provider's API.
- It compares live cloud reality against `terraform.tfstate` and your `.tf` code.
- If drift is detected, Terraform generates a plan to revert the cloud resource back to match the desired state declared in your `.tf` files.

---

### Q5: Can you destroy only a single resource without tearing down the entire infrastructure?
**Answer**:
Yes, using targeted destruction: `terraform destroy -target=aws_instance.example`.
- *Caution*: Targeted operations bypass normal dependency graph resolution and should be reserved only for emergency triage or debugging to avoid leaving orphaned resources.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Running Commands Outside the Project Folder**:
   - *Error*: `Error: No configuration files found.`
   - *Fix*: Always `cd` into the directory containing your `.tf` files before executing `terraform init/plan/apply`.
2. **Missing `egress` Rule in Custom Security Groups**:
   - *Symptom*: EC2 instance starts successfully, but SSH hangs, package updates (`yum update`) time out, or applications cannot reach external APIs.
   - *Cause*: Default Terraform security groups block all outbound traffic unless explicitly declared.
   - *Fix*: Always include an `egress` block with `protocol = "-1"`, `from_port = 0`, `to_port = 0`, and `cidr_blocks = ["0.0.0.0/0"]`.
3. **Regional AMI Mismatch**:
   - *Error*: `InvalidAMIID.NotFound: The image id '[ami-xxxx]' does not exist.`
   - *Cause*: AMI IDs are unique to each AWS region. An AMI ID from `us-east-1` does not exist in `ap-south-1` or `us-west-2`.
   - *Fix*: Ensure the AMI ID matches your configured provider region, or query AMIs dynamically using a `data "aws_ami"` block.
4. **Committing State and Secret Files to Git**:
   - *Risk*: Plaintext credentials exposed in version control.
   - *Fix*: Ensure your `.gitignore` includes:
     ```gitignore
     *.tfstate
     *.tfstate.*
     .terraform/
     .terraform.lock.hcl # (Optional: keep lock file, ignore state)
     crash.log
     override.tf
     *.tfvars
     ```

---

## 9. Session Metadata & Timestamps

- **Ansible Key Authentication**: `05:20`
- **Terraform Introduction & IaaC**: `38:50`
- **Terraform Top 6 Advantages**: `52:10`
- **Terraform Installation & AWS Configure**: `01:10:00`
- **HCL Syntax & Providers Breakdown**: `01:20:00`
- **Interview Questions**: `01:31:00`
- **Session Q&A**: `01:32:38`

### Doubts & AI Clarification Link
- [Session 30 AI Clarification Chat](https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049)