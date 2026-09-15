Monday, 20 April 2026

Session 67 - EKS Cluster creation using open source module, RDS creation for MySQL.
Class Notes
ingress controller(old ingress api/resources) -> load balancer controller(new ingress api/resources) -> gateway api(new gateway resources)

api -> controller


1. target group type only instance.. we must attach all the eks nodes to targetgroup and a nodeport must be opened
2. no grouping of load balancers, every app needs its own load balancer, too much cost and every load balancer should have all the worker nodes attached

RoboShop#123

db subnet group -> what are the subnets available for databases

ecr repos

Commands:
In VS code open integrated Terminal in “roboshop-infra-eks”.
cd 00-vpc   =>
terraform apply -auto-approve   =>
Create AWS Database subnet for RDS database create. In google search for “aws database subnet group terraform” to get code snippet. Then add copied code in “main.tf” file in “terraform-aws-vpc” project. Then push code.
terraform get -update  => To update existing VPC module resource.
 terraform plan  =>
terraform apply -auto-approve   =>
Create an AWS RDS service, go to aws search for RDS then in “Aurora and RDS” page side menu click on “Databases” then click on “Full confirmation” under “Create database” dropdown then choose “MySQL” database then choose “Full configuration” for “Choose a database creation method” then choose “Dev/Test”(if you want less cost) or else choose “Free tier”(no cost) for “Templates” then choose “Single-AZ DB instance deployment (1 instance) under “Availability and durability” for “Deployment options” then choose “MySQL Community” for Edition under “Settings then choose “MySQL 8.4.8” for “Engine version” then enter “roboshop-dev-mysql” for “DB instance identifier” then enter “root” for “Master username” then choose “Self managed” for “Credentials management” then enter “RoboShop#123” for “Master password” then enter same “RoboShop#123” for “Confirm master password” then remaining things keep default settings same then under  “Storage” uncheck “Enable storage autoscaling” if you want auto scaling with required scale like 100 then under “Connectivity” choose “Don’t connect to an EC2 compute resource” for “Compute resource” then choose “roboshop-dev(vpc-vpcID)” for “Virtual private cloud (VPC)” then for “DB subnet group” then select “roboshop-dev” then choose “No” for “Public access” then choose “Choose existing” for “VPC security group (firewall)” then select “default” for “Existing VPC security groups” then choose “us-east-1a” for “Availability Zone” then click on “Create database”.
Now search for Terraform AWS RDS data subnet group code then copy code snippets and use in in “terraform-aws-vpc” module in main.tf file in new lines of code.
In “roboshop-infra-eks” project create “30-rds” folder then create “main.tf” file under the folder then create “provider.tf” file then copy provider code and paster here and do required changes then create “variable.tf” file then add required variables then create “data.tf” file add terraform data code.
Add parameter code for Database VPC subnets IDs in “parameters.tf” in “00-vpc” module.
Add database subnets ID output code in “outputs.tf” file in “terraform-aws-vpc” module.
Now push all code.
cd roboshop-infra-eks/00-vpc   =>
terraform get -update  => To update existing VPC module resource.
terraform plan  => 
Got Error in “terraform-aws-vpc” module fix it, the push & pull code.
cd ../../terraform-aws-vpc  =>
git add . ; git commit -m “k8”; git push origin main  => To add “terraform-aws-vpc” module fix code changes then commit & push changes.
cd roboshop-infra-eks/00-vpc   =>
terraform get -update  => To update existing VPC module resource.
terraform plan  =>
terraform apply -auto-approve   =>
Now go to AWS “Aurora and RDS” then click on “Subnet groups” then verify “roboshop-dev” subnet group is created or not.
Now in AWS “Aurora and RDS” create MySQL database full configuration like given below “How to create AWS RDS Database Manually”. This same AWS “Aurora and RDS” create MySQL database full configuration we can create with Terraform so in Google search for “terraform aws rds” in Terraform Registry you will see Terraform AWS RDS module GitHub link click on and check code. In interview they may ask like “Already we have open source terraform module right why are you creating your own module?” then we have say like “Actually as per our client requirement we want full control on Terraform that’s why are creating own Terraform modules.
In “roboshop-infra-eks” project in “30-rds” folder in “main.tf” file do terraform code changes as per AWS RDS Database requirements.
In “roboshop-infra-eks” project in “30-rds” folder create “provide.tf” then add terraform aws provider code and create “variable.tf” file then add required terraform variables code then create “data.tf” file to add required terraform data code like to get database subnet group name.
cd ../../terraform-aws-vpc  =>
git add . ; git commit -m “k8”; git push origin main  => To add “terraform-aws-vpc” module fix code changes then commit & push changes.
cd roboshop-infra-eks/00-vpc   =>
terraform get -update  => To update existing VPC module resource.
terraform apply -auto-approve   =>
cd ../10-sg/  =>
terraform init -reconfigure   =>  
terraform apply -auto-approve   =>
cd ../20-sg-rules/   =>
terraform init -reconfigure   =>
terraform apply -auto-approve   =>
Write required code and do required changes in “roboshop-infra-eks” project in “30-rds” folder.
Got Error in “20-sg-rules” because not “locals.tf” file. To fix this issue add code file and necessary code.
terraform apply -auto-approve   => To to create “20-sg-rules” as per error fix changes.
cd ../30-rds/   =>
terraform init  => To initialize AWS RDS MySQL database configuration.
terraform plan  =>
Got password version issue add password version equal to 1.
terraform plan  =>
terraform apply -auto-approve   =>
Now write “mysql_eks_node” sg-rule to allow traffic from Kubernetes EKS node to MySQL Database(server).
Now need to develop module for EKS so in Google search for “terraform aws eks” for open source “EKS Manages Node Group” then copy the code snippet then create new “40-eks” folder then create “main.tf” file then paste copied code in this file and do required changes as per the your requirements.
Create “variable.tf”, “data.tf”, “locals.tf” files in “40-eks” folder then add necessary variables code.
cd ../40-eks/   =>
terraform init  =>
terraform plan  =>
Roboshop MySQL Project Database scripts should be load with Bastion server. We can connect with port number 22 should connect with port number “3306” so change from & to port numbers of “mysql_bastion” sg-rule.
From now onwards we use Amazon ECR repos as an alternative to Docker Hub, for more safety for Docker images.
Create “70-acm” & “80-frontend-alb” folders in “roboshop-infra-eks” project then add required files and necessary Terraform code to create Kubenetes based “Roboshop” project infrastructure.




How to create AWS ECR Repo Manually?
	Go to AWS and then search for “Elastic Container Registry” then click on it then click on “Create” then enter “roboshop/catalogue” for “Repository name” under “General settings” then click on “Create”. Now AWS ECR repo will be created, to verify it click on “Repositories” in side menu under “Private registry”. Then you can find “roboshop/catalogue” and in “URI” tab you can also find URI of that Repo. Click on “roboshop/catalogue” then you can find push commands to build & push images in AWS ECR Repos.
How to create AWS RDS Database Manually?
Create an AWS RDS service, go to aws search for RDS then in “Aurora and RDS” page side menu click on “Databases” then click on “Full confirmation” under “Create database” dropdown then choose “MySQL” database then choose “Full configuration” for “Choose a database creation method” then choose “Dev/Test”(if you want less cost) or else choose “Free tier”(no cost) for “Templates” then choose “Single-AZ DB instance deployment (1 instance) under “Availability and durability” for “Deployment options” then choose “MySQL Community” for Edition under “Settings then choose “MySQL 8.4.8” for “Engine version” then enter “roboshop-dev-mysql” for “DB instance identifier” then enter “root” for “Master username” then choose “Self managed” for “Credentials management” then enter “RoboShop#123” for “Master password” then enter same “RoboShop#123” for “Confirm master password” then remaining things keep default settings same then under  “Storage” uncheck “Enable storage autoscaling” if you want auto scaling with required scale like 100 then under “Connectivity” choose “Don’t connect to an EC2 compute resource” for “Compute resource” then choose “roboshop-dev(vpc-vpcID)” for “Virtual private cloud (VPC)” then for “DB subnet group” then select “roboshop-dev” then choose “No” for “Public access” then choose “Choose existing” for “VPC security group (firewall)” then select “default” for “Existing VPC security groups” then choose “us-east-1a” for “Availability Zone” then click on “Create database”.


Diagrams:
rbac
roboshop-infra-updated
terraform-eks


Git Repo Links:
https://github.com/daws-88s/roboshop-infra-eks.git
https://github.com/daws-88s/k8s-ingress.git



Timestamps:
Assignment -
Interview questions = 00:32:50





Mistakes & Learning:




Doubts Link Clarification AI chat link: