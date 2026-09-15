Thursday, 19 February 2026

Session 30 - Ansible using keys, Terraform advantages, Installation and setup, Ec2 and SG creation.
Class Notes
Ansible -> node

node -> public key

Ansible -> private key

ssh -i private-key ec2-user@IP

servers -> create one user for ansible

get the private key access to ansible server
configure the path in ansible.cfg
make sure keys are RSA based

Terraform
============
IaaC tool
Pulumi, Cloudformation(AWS), Arm, etc..

multi-cloud
connects to any platform

1. Version control -> We can have history of our infra..easy to restore. can track who changed and review
2. Consistent infra -> DEV, UAT and PROD
3. CRUD -> inventory management
4. Cost optimisation 
5. Dependency management -> automatic in terraform
6. Reusable infra -> terraform modules


1. Install terraform
2. Setup the PATH in ENV variables
3. Install AWS CLI V2
4. Run aws configure

Providers
==========

HCL -> Hashicorp configuration language

block_name "type" "name" {

}

resource, data, output, variable, module, terraform, etc..


resource "aws_instance" "name-of-the-instance" {
	key = value
}

aws_instance -> syntax
name-of-the-instance -> we can change, this is for terraform reference
ami, name, sg, instance_type

terraform init => intialise terraform
terraform plan

terraform apply

AWS
Terraform

Commands:
Create AWS IAM user to work with Terraform.
Generate Access Key and download it for feature reference.
Install Terraform in your system.
Install AWS CLI v2.
aws configure => Run this command on system terminal.
Enter “AWS Access Key ID”(take form access key file you already downloaded).
Enter “AWS Secret Access Key”(take form access key file you already downloaded).
Enter “Default region name” => us-east-1
Leave “Default output format” as empty.
Create new git Repo to work with Terraform on EC2-instance server.
Start writing Terraform code.
1st write provider.
Here we are going to write Terraform AWS provider.
Everything are resource in Terraform.
To run Terraform code should always be in .tf files folder.
Go to .tf files location then start running Terraform code with below steps.






How to run Terraform code?
17. terraform init => 1st run this command to initialise Terraform into the folder. “terraform init” check for “terraform” block and download give aws provider from internet into below .terraform folder and its files.
18. terraform plan => To plan infrastructure & confirm resource going to create. It will show infra it is going to create.
 19. terraform apply => To apply plan and execute code and create infrastructure. After running this command it will ask confirmation then enter “yes” then aws ec2 instance will be created.
20. terraform plan => 2nd Time if you want to execute same code then run this command directly then below one. 
21. terraform apply -auto-approve => To avoid prompting and enable auto approve. 






Timestamps:
Terraform = 38:50
Interview question = 01:31:00

QA = 01:32:38

Mistakes & Learning:

https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049


Doubts Link Clarification AI chat link: 
https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049