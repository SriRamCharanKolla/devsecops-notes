Monday, 17 August 2026

Session 29 - Include vs import role, Dynamic inventory, Vault, SSM parameter store, Secrets manager, Parallel execution, Ansible disadvantages.
Class Notes
import_role and include_role
===========================
1. tags applied to import_role will be automatically applied to all the tasks inside it.
2. tags applied to include_role will not be cascaded.
3. import_roles parse the playbook before execution, it is static. include_role is dynamic, it will directly execute the playbook, it will not parse like import_role before execution.
4. conditions applied to import_role is applied to all the tasks, include_role will not care the condition applied to it.
5. You cannot use loops on 'import_role' statements. You should use 'include_role' instead

dynamic inventory
=================
scaling environment, servers are dynamically created and destroyed based on traffic.. we cant use static inventory file, we need to dynamically query the instances. We can use dynamic inventory plugin from AWS passing region, name of the instances, runnning state, etc.

go to us-east-1, search frontend-dev running, get the IP address

Vault
=====
ansible-vault create valut.yaml

code -> configuration(confidential, non-confidential)

AWS -> Platform

parellel execution
=================
forks = 8
each task is targeted on 8 servers
single playbook execution

serial=5
8 forks -> 5
run entire playbook
next 3 servers

ansible does not have the state

IaaC
======
I created infra -> I keep track of it in store
again I asked for some change -> compare state and proceed

Terraform -> creating servers
Ansible -> connect to servers and configure them