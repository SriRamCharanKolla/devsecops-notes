Wednesday, 25 February 2026

Session - 33 - Terraform - State, Collaboration infra(Maintaining State in S3 bucket), locals, provisioners
Notes
count based loop
	- list
	- count.index
for based loop
	- set or map
	- each, each.key and each.value
dynamic block
	- we get special variable with block name, for example ingress
	
data sources
- query the provider for existing information

functions

state
=====
terraform stores/tracks what it had created in the state file, it is like memory to terraform

.tf files -> desired/declared/expected/asked infra
actual infra -> what is actually creted inside provider
state -> what terraform created

desired infra == actual infra

first time
=========
declare infra
actual infra not created
state is empty

terraform apply

terraform creates the resources in provider and note them in state file

apply again
.tf files, state file, actual infra
state file == actual infra

scan .tf files

.tf files == actual infra

update tf files
===============
.tf, actual infra, state file

state file == actual infra -> means nothing changed outside of terraform

.tf files != actual infra

changes outside terraform
==============
.tf, actual infra, state file

state file != actual infra -> means someone changed outside of terraform
desired != actual infra

destroy
=======
delete the infra, update state file

state file -> for terraform
tf files -> us

state file is locked by terraform when it is performing actions on that

remote state
==========
in colloboration environment, state should be stored in remote like S3 so that we can prevent errors and duplicates, it should be locked parellel modifications should not be allowed

remote state secure
=================
1. users should not have access to delete except terraform
2. you should enable versions
3. data replication in another bucket

locals
=======
locals are like variables but has extra capabilities
1. you can refer other variables inside locals
2. values inside locals cant be overridden.
3. we can store functions or expressions in locals

Provisioners
=============
1. local-exec -> where terraform executes
2. remote-exec -> executes inside the resources created by terraform

if we want to take some actions or run scripts then we can use provisioners

provisioners will be executed at the time of destroy or creation but not at the time of updating the resources

openssl ssh version mismatch


Commands:
1. Create S3 Bucket in AWS to store terraform remote state files, and the bucket always should be unique across AWS like domain name.



Timestamps:
Interview question = 
QA = 01:31:18

Mistakes & Learning:
1. I missed “-y” in remote-exec provisioner inside inline command of “sudo dnf install nginx -y” to install nginx inside ec2 server because of this mistake Terraform unable to finish task. Terraform got confused to take decision for prompt(y/N) continuously tried and went to loop then I terminated process with “ctrl+c”.


Doubts Link Clarification AI chat link: 