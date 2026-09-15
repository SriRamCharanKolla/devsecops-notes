Thursday, 19 February 2026

Session 30 - Ansible using keys, Terraform advantages, Installation and setup, EC2 and SG creation.
Class Notes & Project Code Teardown

================================================================================
Section 1: Workspace Project Mapping & Architecture
================================================================================

This session's hands-on implementation is located directly in the workspace repository under:
- **Project Folder**: [`terraform/aws_ec2/`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2)
- **Provider Configuration**: [`provider.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/provider.tf#L1-L13)
- **Infrastructure Code**: [`aws_ec2.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L1-L45)
- **Dependency Lock File**: [`.terraform.lock.hcl`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/.terraform.lock.hcl)
- **State File**: [`terraform.tfstate`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/terraform.tfstate)

### Architecture & Resource Dependency Flow:
```
+---------------------------------------------------------------------------------+
|                               AWS Cloud (us-east-1)                             |
|                                                                                 |
|   +-------------------------------------------------------------------------+   |
|   |                  Security Group: "allow_tls"                            |   |
|   |                  (Name: "allow-all-terraform")                          |   |
|   |                                                                         |   |
|   |   Inbound (Ingress):   Port 0-65535, Protocol ALL (-1) from 0.0.0.0/0  |   |
|   |   Outbound (Egress):   Port 0-65535, Protocol ALL (-1) to 0.0.0.0/0    |   |
|   +-------------------------------------------------------------------------+   |
|                                        ▲                                        |
|                                        │ Dynamic Reference via DAG              |
|                                        │ vpc_security_group_ids = [id]          |
|                                        ▼                                        |
|   +-------------------------------------------------------------------------+   |
|   |                     EC2 Instance: "example"                             |   |
|   |                     AMI: ami-0220d79f3f480ecf5                          |   |
|   |                     Type: t3.micro                                      |   |
|   |                     Tags: Name = "roboshop", Project = "roboshop"       |   |
|   +-------------------------------------------------------------------------+   |
|                                                                                 |
+---------------------------------------------------------------------------------+
```

---

================================================================================
Section 2: End-to-End Line-by-Line Code Teardown
================================================================================

### 1. Provider Configuration: [`provider.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/provider.tf#L1-L13)

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

#### Line-by-Line Breakdown:
- **[Lines 1-8](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/provider.tf#L1-L8) (`terraform { required_providers { ... } }`)**:
  - The `terraform` block configures core Terraform behavior and specifies dependencies.
  - `source = "hashicorp/aws"`: Tells Terraform to fetch the official AWS provider plugin maintained by HashiCorp from the public Terraform Registry (`registry.terraform.io/hashicorp/aws`).
  - `version = "~> 6.0"`: Pessimistic version constraint operator (`~>`). Allows minor updates (e.g., `6.1.0`, `6.2.0`) but locks the major version to avoid breaking API changes.
- **[Lines 10-13](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/provider.tf#L10-L13) (`provider "aws" { ... }`)**:
  - `provider "aws"`: Instantiates the AWS provider plugin downloaded during `terraform init`.
  - `region = "us-east-1"`: Sets the target AWS data center region (N. Virginia). All resources declared in this folder will be created in this region unless explicitly overridden with an alias provider.
  - Authentication: Notice no access keys are hardcoded here. Terraform automatically discovers credentials configured via `aws configure` (`~/.aws/credentials`) or environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

---

### 2. Infrastructure Resources: [`aws_ec2.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L1-L45)

#### Part A: Virtual Firewall - Security Group ([Lines 20-45](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L20-L45))

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

#### Line-by-Line Breakdown:
- **[Line 20](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L20)**:
  - `resource`: Core HCL keyword indicating a real cloud infrastructure component.
  - `"aws_security_group"`: Resource Type recognized by the AWS provider API.
  - `"allow_tls"`: Local Resource Identifier used within Terraform configurations to reference this security group's attributes (e.g., `aws_security_group.allow_tls.id`).
- **[Lines 21-22](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L21-L22)**:
  - `name = "allow-all-terraform"`: The actual security group name displayed in AWS Console / CLI.
  - `description`: Explains the firewall's purpose for auditing and team visibility.
- **[Lines 24-31](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L24-L31) (`egress` block - Outbound Traffic)**:
  - In AWS, security groups are stateful. However, default Terraform security groups have no egress rules unless explicitly declared.
  - `protocol = "-1"`: Special flag signifying **all network protocols** (TCP, UDP, ICMP).
  - `from_port = 0`, `to_port = 0`: When protocol is `"-1"`, ports must be set to `0` to encompass all port ranges.
  - `cidr_blocks = ["0.0.0.0/0"]`: Permits outbound packets to any IPv4 internet destination.
  - `ipv6_cidr_blocks = ["::/0"]`: Permits outbound packets to any IPv6 internet destination.
- **[Lines 33-40](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L33-L40) (`ingress` block - Inbound Traffic)**:
  - Configures incoming network access. In this practice demo, `protocol = "-1"` with `0.0.0.0/0` allows full open ingress traffic.
  - *Production Note*: In real-world environments, restrict `from_port` and `to_port` to specific application ports (e.g., 22 for SSH, 80 for HTTP, 443 for HTTPS) and specific CIDR IP ranges.
- **[Lines 42-44](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L42-L44)**:
  - Assigns AWS resource tags for billing, cost allocation, and console searching.

---

#### Part B: Compute Resource - EC2 Instance ([Lines 5-14](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L5-L14))

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

#### Line-by-Line Breakdown:
- **[Line 5](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L5)**:
  - Declares an EC2 virtual server named `"example"` for Terraform internal referencing.
- **[Line 6](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L6) (`ami = "ami-0220d79f3f480ecf5"`)**:
  - Specifies the base Amazon Machine Image containing the operating system (RHEL-9 / CentOS / Amazon Linux).
  - *Critical Rule*: AMI IDs are unique per AWS region. This ID must exist in `us-east-1`.
- **[Line 7](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L7) (`instance_type = "t3.micro"`)**:
  - Defines hardware specifications (2 vCPUs, 1 GiB Memory, burstable performance). Eligible for AWS Free Tier.
- **[Line 8](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L8) (`vpc_security_group_ids = [aws_security_group.allow_tls.id]`)**:
  - **Implicit Dependency / DAG Connection**: Instead of hardcoding an SG ID like `"sg-12345678"`, Terraform references the attribute `.id` of `aws_security_group.allow_tls`.
  - **Execution Order Determination**: Because `aws_instance.example` requires `aws_security_group.allow_tls.id`, Terraform automatically calculates that the Security Group **must be created first**, waits for AWS to return its generated ID, and then passes it to the EC2 API call.
- **[Lines 10-13](file:///Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2/aws_ec2.tf#L10-L13)**:
  - Key-value map of tags. The `Name` tag sets the EC2 instance name in the AWS Management Console list.

---

================================================================================
Section 3: Conceptual Theory & Architecture
================================================================================

### 1. Ansible Key-Based Authentication Architecture:
```
+-----------------------------------+               +-----------------------------------+
|      Ansible Control Server       |               |        Target Managed Node        |
|                                   |  SSH (TCP:22) |                                   |
| - /home/ec2-user/.ssh/id_rsa      | ------------> | - /home/ec2-user/.ssh/            |
|   (Private Key, chmod 400)        |  PrivateKey   |   authorized_keys (Public Key)    |
| - ansible.cfg (configured path)   |  Matches      | - /etc/sudoers.d/ansible          |
|                                   |  PublicKey    |   (Passwordless Sudo)             |
+-----------------------------------+               +-----------------------------------+
```
- Remote nodes never hold the private key; only the public key (`.pub`) is installed on targets.
- Passwordless SSH authentication prevents interactive prompt blocking in automation pipelines.

### 2. Infrastructure as Code (IaaC) Comparison:
| Feature | Terraform | AWS CloudFormation | Azure Bicep / ARM | Pulumi |
| :--- | :--- | :--- | :--- | :--- |
| **Cloud Support** | Multi-Cloud (AWS, Azure, GCP, K8s) | AWS Only | Azure Only | Multi-Cloud |
| **Language** | Declarative (HCL) | Declarative (YAML / JSON) | Declarative (Bicep / JSON) | Imperative (Python, TS, Go) |
| **State Storage** | Explicit (`terraform.tfstate` / S3) | Implicit (AWS Managed) | Implicit (Azure Managed) | Pulumi Service / S3 |
| **Ecosystem & Reusability**| Public Terraform Registry Modules | CloudFormation Registry | Azure Verified Modules | Pulumi Registry Packages |

### 3. Top 6 Advantages of Terraform:
1. **Version Control & Auditability**: Infra changes follow software engineering practices (Git branching, PR reviews, commit history, blame/audit tracking).
2. **Environment Parity**: Eliminates configuration discrepancies between DEV, UAT, and PROD by parameterizing identical templates.
3. **Automated Lifecycle (CRUD)**: Detects differences between desired code and real-world infrastructure, orchestrating accurate Create, Read, Update, and Delete operations.
4. **Cost Control & Ephemeral Infra**: Enables automated provisioning for temporary testing and immediate destruction (`terraform destroy`) to eliminate idle resource billing.
5. **Graph-Based Dependency Resolution**: Constructs a Directed Acyclic Graph (DAG) automatically to parallelize independent resource creations and sequence dependencies correctly.
6. **Modularity**: Promotes DRY (Don't Repeat Yourself) principle through reusable modules across multiple enterprise teams.

---

================================================================================
Section 4: Step-by-Step Hands-on Execution Walkthrough
================================================================================

Follow this exact sequence to deploy the infrastructure from terminal:

### Step 1: AWS CLI Authentication
```bash
# 1. Configure AWS CLI credentials
aws configure
# AWS Access Key ID [None]: <YOUR_ACCESS_KEY_ID>
# AWS Secret Access Key [None]: <YOUR_SECRET_ACCESS_KEY>
# Default region name [None]: us-east-1
# Default output format [None]: json

# 2. Verify connection to AWS
aws sts get-caller-identity
```

### Step 2: Navigate to Project Folder
```bash
cd /Users/sriramcharankolla/Desktop/DevOps/terraform/aws_ec2
ls -la
# Verify aws_ec2.tf and provider.tf are present
```

### Step 3: Initialize Terraform
```bash
terraform init
```
*What happens*:
- Scans `provider.tf`.
- Contacts `registry.terraform.io` and downloads the AWS provider plugin into `.terraform/providers/`.
- Creates or updates `.terraform.lock.hcl` with cryptographic hashes.

### Step 4: Validate and Format Code
```bash
terraform fmt      # Rewrites code to standard canonical formatting
terraform validate # Verifies syntax and internal consistency
```

### Step 5: Generate Execution Plan (Dry Run)
```bash
terraform plan
```
*Output Analysis*:
- Shows `Plan: 2 to add, 0 to change, 0 to destroy.`
- Resources marked with `+` symbol will be created.

### Step 6: Apply Infrastructure to AWS
```bash
terraform apply
# When prompted: Enter a value: yes
```
*Automation Alternate*:
```bash
terraform apply -auto-approve
```
*Result*:
- Security group `allow-all-terraform` is created first.
- EC2 instance `roboshop` is launched with the attached security group.
- State is committed to `terraform.tfstate`.

### Step 7: Teardown & Destroy
```bash
terraform destroy -auto-approve
```
*Result*:
- Terraform reads `terraform.tfstate`.
- Terminates the EC2 instance first, then deletes the security group in reverse dependency order.

---

================================================================================
Section 5: Commands & CLI Flags Teardown Table
================================================================================

### Core Lifecycle Commands:
| Command | Primary Function | When to Use |
| :--- | :--- | :--- |
| `terraform init` | Initializes directory, downloads provider plugins & modules. | First time running code, or after adding new providers/modules. |
| `terraform validate` | Checks configuration syntax and semantic validity without cloud calls. | In pre-commit hooks, CI pipelines, and before running plan. |
| `terraform fmt` | Formats HCL files to HashiCorp standard indentation and style. | Before every git commit to maintain code cleanliness. |
| `terraform plan` | Compares `.tf` code against state and actual cloud resources. | Before every deployment to preview additions, changes, deletions. |
| `terraform apply` | Provisions or updates real-world cloud resources via API calls. | To deploy infrastructure changes. |
| `terraform destroy` | Deletes all infrastructure tracked in the state file. | For tearing down temporary demo/testing environments to stop costs. |
| `terraform show` | Prints human-readable output of current state or a plan file. | To inspect provisioned attributes (IPs, ARNs, IDs). |
| `terraform state list` | Lists all resource addresses currently recorded in state. | To audit resources tracked by Terraform. |

### CLI Flags Teardown:
| Command | Flag | What It Does | Common Real-World Use Case |
| :--- | :--- | :--- | :--- |
| `terraform init` | `-upgrade` | Upgrades all providers and modules to the newest version allowed by version constraints. | When upgrading AWS provider from `5.x` to `6.x`. |
| `terraform init` | `-reconfigure` | Ignores existing backend configuration and reinitializes state backend. | When switching remote S3 backend buckets. |
| `terraform init` | `-migrate-state` | Reinitializes backend and copies existing state to the new backend. | Migrating local state to remote S3 backend. |
| `terraform plan` | `-out=tfplan` | Saves the execution plan to an encrypted binary file. | In CI/CD pipelines to guarantee `apply` runs the exact previewed plan. |
| `terraform plan` | `-detailed-exitcode` | Returns exit code `0` (no changes), `2` (changes present), or `1` (error). | In automated drift-detection cron jobs. |
| `terraform apply` | `-auto-approve` | Bypasses the interactive `yes` prompt confirmation. | In automated CI/CD pipelines (Jenkins, GitHub Actions). |
| `terraform apply` | `-replace="resource"` | Forces recreation (destroy & re-create) of a specific resource. | When an EC2 instance is corrupted or tainted. |
| `terraform apply` | `-refresh-only` | Updates state file with real-world infrastructure drift without altering cloud resources. | When resources were modified manually in AWS console. |
| `terraform apply` | `-var="key=value"` | Passes an input variable value directly via CLI. | `terraform apply -var="instance_type=t3.small"` |
| `terraform apply` | `-var-file="path"` | Loads variable values from an external `.tfvars` file. | Multi-env deployments: `-var-file=dev.tfvars`. |
| `terraform destroy` | `-target="resource"` | Restricts destruction to a specific resource address. | Deleting an experimental resource without touching core infra. |
| `terraform fmt` | `-check` | Checks if files are formatted, exits with non-zero if formatting is needed. | In CI linting pipelines to enforce code style. |
| `terraform fmt` | `-diff` | Displays the exact formatting diffs without altering files. | Reviewing whitespace adjustments. |

---

================================================================================
Section 6: Official Documentation & Reference Links
================================================================================

- **Terraform CLI Official Documentation**: [https://developer.hashicorp.com/terraform/cli](https://developer.hashicorp.com/terraform/cli)
- **Terraform AWS Provider Registry**: [https://registry.terraform.io/providers/hashicorp/aws/latest/docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- **`aws_instance` Resource Documentation**: [https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)
- **`aws_security_group` Resource Documentation**: [https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)
- **AWS CLI v2 Installation & Configuration Guide**: [https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)

---

================================================================================
Section 7: High-Yield Interview Questions & Answers
================================================================================

1. **What is the significance of the `.terraform.lock.hcl` file?**
   - *Answer*: Introduced in Terraform 0.14, the dependency lock file records the exact provider versions and cryptographic checksums used for the configuration. When multiple team members or CI/CD pipelines run `terraform init`, the lock file guarantees everyone uses identical provider binary hashes, preventing "upstream dependency drift."

2. **What is the difference between Implicit and Explicit Dependencies in Terraform?**
   - *Answer*:
     - **Implicit Dependency**: Created automatically when one resource references an attribute of another (e.g., `vpc_security_group_ids = [aws_security_group.allow_tls.id]`). Terraform automatically orders security group creation before the EC2 instance.
     - **Explicit Dependency**: Declared manually using the `depends_on = [resource]` meta-argument when Terraform cannot deduce the relationship from attribute references (e.g., an EC2 instance requiring an IAM role or S3 bucket to exist first).

3. **What is `terraform.tfstate`, why is it critical, and what risk comes with storing it in Git?**
   - *Answer*: The state file acts as the "source of truth" and memory for Terraform, mapping declared HCL resources to real-world cloud provider resource IDs and metadata. It should **never be committed to Git** because it may contain sensitive plain-text data (database passwords, private keys) and Git does not provide state locking during concurrent team executions. Instead, use a Remote Backend (AWS S3 with DynamoDB locking).

4. **What happens when an engineer modifies a resource manually in the AWS Console? How does Terraform handle it?**
   - *Answer*: This is called **Configuration Drift**. When `terraform plan` or `terraform apply` is executed, Terraform first runs a refresh phase against the cloud API. It detects differences between the real cloud state and the `.tfstate` file, proposing changes to revert the infrastructure back to the desired configuration declared in the `.tf` code.

5. **Can `terraform destroy` be restricted to delete only one specific resource?**
   - *Answer*: Yes, using the `-target` flag: `terraform destroy -target=aws_instance.example`. However, `-target` should be used with extreme caution because it bypasses normal graph resolution and can leave orphaned dependencies.

---

================================================================================
Section 8: Production Mistakes & Troubleshooting
================================================================================

1. **Executing commands from the wrong working directory**:
   - *Symptom*: Error: `No configuration files found.`
   - *Fix*: Terraform commands must always be executed from the folder containing the target `.tf` files (`cd terraform/aws_ec2`).
2. **Missing `egress` block in custom Security Groups**:
   - *Symptom*: EC2 instance cannot connect to the internet, `yum`/`dnf` installs hang, or package downloads time out.
   - *Cause*: Unlike security groups created via the AWS Web Console (which automatically inject an allow-all egress rule), Terraform creates a completely empty security group by default. Always explicitly declare an `egress` block with `protocol = "-1"` and `cidr_blocks = ["0.0.0.0/0"]`.
3. **AMI ID region mismatch**:
   - *Symptom*: Error: `InvalidAMIID.NotFound: The image id '[ami-xxxx]' does not exist.`
   - *Cause*: AMIs are regional. An AMI ID copied from `us-east-1` will fail if the provider is set to `us-west-2` or `ap-south-1`.
4. **Unencrypted sensitive state in Git repositories**:
   - *Pitfall*: Forgetting to add `*.tfstate`, `*.tfstate.backup`, and `.terraform/` to `.gitignore`. Always verify `.gitignore` contains these entries before running `git add`.

---

Timestamps:
Ansible SSH Key Authentication = 05:20
Terraform Introduction & IaaC = 38:50
Terraform Advantages = 52:10
Terraform Installation & AWS Configure = 01:10:00
HCL Syntax & Providers = 01:20:00
Interview Questions = 01:31:00
QA = 01:32:38

Doubts Link Clarification AI chat link: 
https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049