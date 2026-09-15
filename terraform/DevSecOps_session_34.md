Wednesday, 25 February 2026

Session 34 -  Multi environments using( Workspaces, Tfvars), Module development, EC2 module development.
Notes
state and remote state
Terraform interview code writing format:
—————————————————-
data.tf
ec2.tf
locals.tf
variables.tf

resource "aws_instance" "frontend" {

}

locals
—————————-
local-exec
remote-exec

1. workspaces
2. tfvars
3. seperate repos

workspaces
==========
if we want to create multiple environments we can make use of terraform workspaces. terraform will give us special variable called terraform.workspace

terraform.workspace=dev
terraform.workspace=default
terraform.workspace=prod

terraform workspace select dev/prod

dev -> t3.micro
uat -> t3.small
prod -> t3.medium

advantages
=========
1. same code
2. consistent environments

disadvantages
=========
since code is same, we should be very careful, because it applies to prod also.

tfvars
======
advantages
=========
1. same code
2. consistent environments

disadvantages
=========
since code is same, we should be very careful, because it applies to prod also.

I am in prod
I applied dev values

1. command line
2. tfvars
3. env variable
4. default values
5. prompt

- dev
- prod

isolation -> non-prod and prod
different accounts for different environments
different repository

aws configure
terraform workspace select dev

terraform-ec2-dev
terraform-ec2-prod

disadvantages
==============
repeat code

advantages
==============
clear isolation
blast radius is zero
seperate states


Terraform Modules
=================
functions -> a repeated task, it will be executed whenever we call. it will take inputs and give outputs

a.sh
common.sh

a -> function common.sh
Advantages
—————————
1. enforce best standards
2. code reuse
3. easy maintainance

branch names -> dev, uat and prod
feature and main
code and configuration

tfvars/
	dev.tfvars
	uat.tfvars
	prod.tfvars


Commands:
terraform workspace —help => To check terraform workspace commands.
Then change directory to workspace folder then initialise terraform.
terraform workspace list => To check Terraform workspace list. 1st time you will be at “default” workspace with start mark(*) indicated.
terraform workspace new <workspace-name> => To create new workspace.
Example: terraform workspace new dev => To create dev workspace.
terraform workspace list => To check Terraform workspace list.  Now you will be at “dev” workspace with start mark(*) indicated.
terraform plan =>
terraform workspace new prod => To create “prod” workspace.
terraform workspace list => To check Terraform workspace is changed to “prod”.
terraform plan =>
terraform init —help => 
terraform init -backend-config=dev/backend.tf => To config remote state backend with dev s3 bucket.
terraform init -reconfigure -backend-config=prod/backend.tf => To config remote state backend with prod s3 bucket.
terraform plan -var-file=dev/terraform.tfvars => To provide remote state tfvars variables file location of dev.
terraform plan -var-file=prod/terraform.tfvars => To provide remote state tfvars variables file location of prod.
terraform apply -auto-approve -var-file=dev/terraform.tfvars => 
terraform apply -auto-approve -var-file=prod/terraform.tfvars => 
terraform destroy -auto-approve -var-file=dev/terraform.tfvars =>
terraform destroy -auto-approve -var-file=prod/terraform.tfvars =>



Timestamps:
Interview question = 59:28 => How can you manage Terraform Multiple Environments?
A: There are 3 ways workspaces, tfvars, separate repos for dev, UAT, prod.
QA = 01:28:30

Mistakes & Learning:
https://chatgpt.com/share/69a067dc-dd60-8007-94a0-5cab91cbe883



Doubts Link Clarification AI chat link: 
https://chatgpt.com/share/69a067dc-dd60-8007-94a0-5cab91cbe883
