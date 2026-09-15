Tuesday, 3 March 2026

Session 38 - Bastion Host, Security Groups, VPC peering test, RoboShop infra in DEV, SG module, SSM parameter store.
Class Notes
1. Project Infra -> one time infra
2. Application Infra -> 

1. VPN
2. Bastion/Jump host -> instance in public subnet used to connect instances in private subnets for security purpose.

bastion(roboshop-dev-bastion) -> roboshop-dev(roboshop-dev)
								22 10.0.1.94/32
								22 mention source of traffic roboshop-dev-bastion
10.0.11.233							
telnet 172.31.9.80 22

use variables
use data sources
use locals
use outputs

1. module should output the required info to users
2. users should catch and store it in SSM parameter store

/roboshop/dev/vpc_id


Commands:
cd ../vpc-module-test/ => In Terraform projects folder change your director to “vpc-module-test”.
terraform init -reconfigure    / terraform init => 
Create “roboshop-infra-dev”.
Create “00-vpc” folder & create “main.tf” file in “roboshop-infra-dev” git repo.
Start writing code to create VPC infra with VPC module with main.tf, variables.tf, 
cd ../roboshop-infra-dev/00-vpc/ =>
terraform init  =>
terraform plan  =>
terraform apply -auto-approve  => 
Create Security Groups for “roboshop-dev” with “roboshop-dev” VPC manually.
Whenever VPC is created default security group will be created automatically by AWS.
Create security group with name “default-vac” with “default” VPC manually.
Create “roboshop-dev” instance manually with “RHEL9 DevOps Practice” AMI with “t3.micro” then edit network then select “roboshop-dev” VPC for VPC required field then select “roboshop-dev-private-us-east-1a” subnet for Subnet field then select “Select existing security group” for “Firewall(security groups)” option then select “roboshop-dev” security group for “Common security groups” field then click on “Lunch instance” button.
Create another instance with name “default” manually with “RHEL9 DevOps Practice” AMI with “t3.micro” then edit network then select “Disable” for “Auto-assign public IP” option then select “Select existing security group” for “Firewall(security groups)” option then select “default-vpc” security group for “Common security groups” field then click on “Lunch instance” button.
Create “bastion” instance manually with “RHEL9 DevOps Practice” AMI with “t3.micro” then edit network then select “roboshop-dev” VPC for VPC required field then select “roboshop-dev-public-us-east-1a” subnet for Subnet field then select “Create security group” for “Firewall(security groups)” option then enter “roboshop-dev-bastion” as security group name for “Security group name” field then under “Inbound Security Group Rules” select “My IP” for “Source type” click on “Lunch instance” button.
Copy “bastion” server IP & connect from your system using terminal.
ssh ec2-user@<bation-public-IP> =>
telnet <roboshop-dev-Private-IP> 22  => should not be connect because not we are not allowing from “roboshop-dev” security group.
Go to security group from “Security” tab then click on “roboshop-dev” security group ID.
Then click on “Edit inbound rules” button then click on “Add rule” button select “SSH” for “Type” option then select “Custom” for “Source” then enter “bastion” host server private IP address like “10.0.1.94/32”(/32 means single IP) in the input field then click on “Save rules” button.  => Not a good practice, below one is good practice.
But here the issue is every time bastion host private IP will be changed so that we need to change it again & again, so to solve this problem we need provide “roboshop-dev” security ID. So delete previous one create new one with “roboshop-dev” security ID.
telnet <roboshop-dev-Private-IP> 22  => now it will work.
To connect to “default” VPC 1st connect to “roboshop-dev” server with “roboshop-dev” private IP address.
ssh ec2-user@<roboshop-private-IP>  =>
telnet <default-VPC-server-private-IP> 22  => should not be connect to “default” VPC server because default VPC security group not allowing “roboshop-dev” server to connect with “default” VPC server, to resolve this do below changes in default VPC server.
Select “default” EC2 instance click on security group then click on “default-vpc” server IP which is under “Security groups”.
Then click on “Edit inbound rules” button then click on “Add rule” button select “SSH” for “Type” option then select “Custom” for “Source” then enter “roboshop-dev” security ID in the input field then click on “Save rules” button.
telnet <default-VPC-server-private-IP> 22  =>
Create “terraform-aws-sg” module to create only security group infra.
Create main.tf & variables.tf files.
Write code for sg module.
In AWS search for “Systems Manager” then click on it.
Then click on “Parameter Store” in side menu.
Click on “Create parameter”, we already created “Parameter store” so write terraform code.
cd ../roboshop-infra-dev/10-sg/ =>     
terraform init  =>
terraform plan => 
terraform apply -auto-approve  =>        


Diagrams:
roboshop-infra
vpc-module
store

Timestamps:
QA = 01:28:40

Mistakes & Learning:




Doubts Link Clarification AI chat link:


techolution.com interview question
Design a type-safe User model in TypeScript for create, update, and fetch operations, ensuring proper use of generics and utility types without using any.