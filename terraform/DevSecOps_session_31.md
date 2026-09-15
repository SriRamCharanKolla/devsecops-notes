Friday, 20 February 2026

Session 31 - Git ignore, Variables, Precedence of variables, Conditions, Count based loop.
Notes
terraform init -> downloads the provider
terraform plan -> plan the infra but it will not create
terraform apply -> create the infra
terraform destroy -> destroy the infra

resource "type_of_resource" "name_of_resource"{
	key = value # arguments
}

attributes -> outputs

rm -rf .git
git init
git branch -M main
git remote add origin <URL>
git add . ; git commit -m ""; git push origin main

variables
=========

variable "name_of_variable" {
	default = ""
}

conditions
===========
expression ? "true-value" : "false-value"

1. default values inside variables.tf
2. terraform.tfvars
3. command line
4. env variables

1. command line
2. tfvars
3. env variables -> export TF_VAR_VAR_NAME="value"
4. default values

loops
=======
1. count based loops
2. for_each
3. dynamic block

1. sg and ec2 instances -> 10 instances
2. r53 records

count.index

output "name_of_the_output" {
	value = 
}



Commands:
Steps to Run Terraform Code?
1st change directory to the folder in which you wrote Terraform code files.
terraform init => Run this 1st only
terraform plan => To plan and check infrastructure going to create. 2nd time onwards run this command directly.
terraform play => To create infrastructure . Type “yes” for the prompt.
terraform play -auto-approve => To create infrastructure with auto approval(with prompt confirmation)
terraform destroy => To delete infrastructure created. Type “yes” for the prompt.
terraform destroy -auto-approve => o delete infrastructure created. with auto approval(with prompt confirmation)

Create “terraform.tfvars” file and declare same “instance_type = “t3.small” “ in that file.
set TF_VAR_varable_name=“value you want to give” => Syntax for Environment variable declaration.
Example : set TF_VAR_sg_name=“allow-all-terraform-env”
export TF_VAR_sg_name=“allow-all-terraform-env” => after creating Environment variable should export it to use.
terraform plan -var=“varable_name=value-of-variable” => Syntax to create command line variable.
terraform plan -var=“sg_name=allow-all-terraform-cmd” => Example of command line variable.

Timestamps:
Interview question = 
QA = 01:17:20

Mistakes & Learning:

https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049


Doubts Link Clarification AI chat link: 
https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049