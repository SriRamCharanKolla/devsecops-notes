Monday, 17 August 2026

Session 29 - Include vs import role, Dynamic inventory, Vault, SSM parameter store, Secrets manager, Parallel execution, Ansible disadvantages.
Class Notes

import_role vs include_role
===========================
| Feature | `import_role` (Static) | `include_role` (Dynamic) |
| :--- | :--- | :--- |
| **Parsing Time** | Pre-parsed before playbook execution starts (Static). | Parsed dynamically at runtime when the task is reached (Dynamic). |
| **Tags Behavior** | Tags applied to `import_role` automatically cascade to all tasks inside the role. | Tags applied to `include_role` do not cascade to tasks inside the role. |
| **Conditions (`when`)** | Condition is evaluated and applied to every task inside the role. | Condition applies only to whether the role gets included, not individual tasks. |
| **Loops (`loop`)** | Cannot use loops directly on `import_role`. | Can use loops directly on `include_role`. |
| **Handlers** | Handlers are imported upfront. | Handlers are recognized only after the role is included. |

Dynamic Inventory
=================
In modern auto-scaling cloud environments, servers are created and destroyed dynamically based on traffic. 
- Static inventory files (`inventory.ini` or `hosts`) cannot keep up because IP addresses change constantly.
- We need to query instances dynamically from cloud providers like AWS using the `aws_ec2` inventory plugin.
- Filters can be applied by AWS Region, Instance Tags (e.g., `Name: frontend-dev`), and Instance State (`running`).

Example flow:
Go to `us-east-1` -> Search for instances matching `Name: frontend-dev` and state `running` -> Automatically fetch private/public IP addresses into Ansible.

Ansible Vault & Secret Management
=================================
Source code should be separated into:
1. Code (playbooks, roles, tasks) -> stored in Git repository.
2. Configuration:
   - Non-confidential: URLs, port numbers, environment names.
   - Confidential / Sensitive: Passwords, API keys, database credentials, certificates.

Methods to manage secrets:
1. **Ansible Vault**: Encrypts sensitive YAML files or variable files directly using AES-256 encryption.
2. **AWS Systems Manager (SSM) Parameter Store**: Cloud platform-level key-value store for configurations and SecureString secrets.
3. **AWS Secrets Manager**: Managed cloud service with automatic secret rotation, fine-grained IAM policies, and cross-account access.

Parallel Execution (Forks vs Serial)
====================================
By default, Ansible executes tasks in parallel across hosts using `forks`:

1. **Forks (`forks = 8`)**:
   - Defines how many servers Ansible communicates with simultaneously for each task.
   - Example: If `forks = 8` and you have 20 servers, Task 1 runs on the first 8 servers, then the next 8 servers, then the remaining 4 servers. Once all 20 finish Task 1, Ansible moves to Task 2.

2. **Serial (`serial = 5`)**:
   - Defines rolling execution at the entire playbook level (batch processing).
   - Example: If you have 8 servers and `serial = 5`:
     - Batch 1 (5 servers): Runs the complete playbook from start to finish.
     - Batch 2 (remaining 3 servers): Runs the complete playbook after Batch 1 succeeds.
   - Useful for zero-downtime rolling updates (e.g., updating web servers behind a load balancer).

Ansible Disadvantages & Need for IaaC (Terraform)
=================================================
1. **Ansible does not have state management**:
   - Ansible does not track what it created or the state of remote infrastructure.
   - If an instance is deleted outside Ansible, it does not automatically detect the drift or keep track of existing resources like a state file does.
2. **Configuration vs Provisioning**:
   - **Terraform (IaaC)**: Best suited for Infrastructure Creation / Provisioning (VPC, Subnets, EC2, RDS, IAM).
     - Flow: Create infra -> Track state in remote store (e.g., S3 bucket) -> Compare state on every run and apply only necessary changes.
   - **Ansible (Configuration Management)**: Best suited for connecting to already provisioned servers and configuring packages, files, services, and users.

Commands:
# Ansible Vault Commands:
ansible-vault create vault.yaml                       => To create a new encrypted file.
ansible-vault encrypt secrets.yaml                    => To encrypt an existing unencrypted file.
ansible-vault decrypt vault.yaml                      => To decrypt an encrypted file.
ansible-vault view vault.yaml                         => To view encrypted content without decrypting the file.
ansible-vault edit vault.yaml                         => To edit encrypted file content directly.
ansible-playbook -i inventory playbook.yaml --ask-vault-pass => To run a playbook with vault password prompt.
ansible-playbook -i inventory playbook.yaml --vault-password-file ~/.vault_pass => To supply vault password via file.

# Dynamic Inventory Commands:
ansible-inventory -i aws_ec2.yaml --graph             => To inspect the dynamic inventory hierarchy.
ansible-inventory -i aws_ec2.yaml --list              => To list all dynamic host details in JSON format.
ansible all -i aws_ec2.yaml -m ping                   => To ping all dynamically discovered hosts.

# Parallel Execution Commands:
ansible-playbook -i inventory playbook.yaml -f 10      => To override default forks and run with 10 parallel connections.

Timestamps:
Include vs Import role = 00:15:00
Dynamic inventory = 00:45:00
Ansible Vault = 01:05:00
Forks vs Serial = 01:20:00
Interview question = 01:30:00
QA = 01:32:00

Interview Questions:
1. What is the difference between `import_role` and `include_role`?
   - Answer: `import_role` is static (parsed before execution; tags/conditions cascade; cannot use loops). `include_role` is dynamic (evaluated at runtime; tags/conditions do not cascade; supports loops).
2. Why do we need Dynamic Inventory in AWS cloud environments?
   - Answer: Auto-scaling environments create and terminate EC2 instances dynamically. Static inventory files cannot track changing IP addresses. Dynamic inventory plugins query AWS APIs in real time to fetch running instances based on tags and regions.
3. What is the difference between `forks` and `serial` in Ansible?
   - Answer: `forks` controls task-level parallel thread execution across servers. `serial` controls playbook-level batching for rolling updates.
4. Why is Ansible not ideal for infrastructure creation compared to Terraform?
   - Answer: Ansible lacks a centralized state management engine to detect infrastructure drift. Terraform is purpose-built for declarative infrastructure provisioning with state tracking, dependency graphs, and lifecycle management.

Mistakes & Learning:
1. Typo in file naming: Named `valut.yaml` instead of `vault.yaml`. Always verify naming conventions before encrypting.
2. Loops on `import_role`: Tried using `loop` on `import_role` which caused a syntax error. Switched to `include_role` to iterate over roles dynamically.
3. Vault password prompt: Forgot `--ask-vault-pass` when running vault-protected playbooks, leading to authentication failure.

Doubts Link Clarification AI chat link: