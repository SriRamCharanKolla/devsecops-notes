Wednesday, 4 March 2026

Session 39 - SG rules, Bastion, IAM.
Class Notes
instance-1 -> instance-2

inbound
========
3306	instance-1 SG_ID

[
  "subnet-048c42aeba89987f1",
  "subnet-08d03c4487720a4c0",
] -> list

"subnet-048c42aeba89987f1", "subnet-08d03c4487720a4c0" -> string

"${var.project}-${var.environment}-${var.sg_name}"

roboshop-dev-mongodb
roboshop-dev-backend_alb
replace _ with -

public_subnet_ids
terraform -> convert string to List and access 0th element
AWS -> StringList (string)

IAM
====
User Group
Role
Permissions

devops team
===========
Role		Permissions
trainee -> only read access
junior engineer -> read + create
senior engineer -> read + create + update
team lead -> read + create + update + delete
architect -> read + create + update + delete
team manager -> read + create + update + delete

User -> roboshop-trainee(group) -> only read access

Nouns -> Names 
Verbs -> Actions

Everything is called resource/service

EC2 -> noun
createEC2, readEC2, updateEC2, deleteEC2

User(human) -> what services he can access(nouns) -> what are the permissions he have(actions)
access-key and secret-key (programatic access)
username and password (console)

User vs Role
1. You create user for humans
2. You create Role for non humans like for AWS services to act on behalf of us

Role for EC2

Role
I attached permission to Role -> EC2FullAccess


Commands:

cd /roboshop-infra-dev/00-vpc  =>
terraform plan   =>
terraform apply -auto-approve  =>
Write sum parameter code to store public, private, database ids.
terraform plan  =>
Will get issue due to sending StringList but AWS Parameter Store will accept single string only.
So use “join( )” function while sending value to sum parameter store. Join with coma(, )
terraform plan  => 
Terraform apply -auto  =>
Check AWS Parameter for VPC ids stores successfully or not.
Create “20-sg-rules” and write sg rules terraform code.
Create sg_rules.yaml file.
Write sg_rules terraform code.
Execute terraform code to create required security group rules.
Create “30-bastion” folder to store Terraform & code files.
Start writing code.
Execute Bastion code to create bastion infrastructure.
Create Role for EC2(Bastion EC2) and attach same role to bastion EC2 instance in “IAM instance profile” select EC2 Role you want(I will select “RoboshopDevBastion” EC2 role”) under “Advanced details”.
Now write Terraform code to attach EC2 role while creating AWS EC2 instance.
Then attach permission to EC2 role you created with Terraform code. 

How to create Role for EC2(Bastion EC2)?
Go to “IAM”.
Click on “Roles” in side menu under “Access Management”.
Click on “Create role” button then keep “AWS service” same for “Trusted entity type”.
Then select “EC2” option for “Use Case”.
Then click on “Next” button.
Then select “AmazonEC2FullAccess” option for “Permissions policies” under “Add permissions”.
Then click on “Next” button.
Then enter “RoboshopDevBastion” for “Role name” under “Name, review, and create”.
Then click on “Create role” button.
Now you successfully created Role for EC2(Bastion EC2).

Timestamps:
QA = 01:30:10

Mistakes & Learning:




Doubts Link Clarification AI chat link: