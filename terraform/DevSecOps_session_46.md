Saturday, 28 March 2026

Session 46 - SG Rules completed, Roboshop terraform component.
Class Notes
private key
certificate
ca_bundle

certificate+ca_bundle = final_certificate_chain -> public
private key
-
- Mistakes will be happen while:
1. Check SG rules
2. Ansible Roles service files
3. Ansible Roles host URL's

for dvo in [“one”,"two","three"]

Commands:
for in in 00-vpc/ 10-sg/ 20-sg-rules/ 50-backend-alb/ 70-acm/ 80-fronend-alb/; do cd $I; terraform init -reconfigure; terraform apply -auto-approve; cd ..;done
Do changes in “20-sg-rules” to create sg-rules for Redis, MongoDB, MySql, Rabbitmq Components.
Execute Terraform code.
cd 30-bastion/  =>
Execute Bastion Terraform code to create bastion infra resources.
ssh ec2-user@<bastion-public-IP>   =>
git clone <git-roboshop-infra-dev-link>   => Inside bastion server
cd roboshop-infra-dev/40-databases/     => Inside bastion server 
terraform init   => Inside bastion server
terraform plan  => Inside bastion server
terraform apply -auto-approve   => Inside bastion server
Then Write “terraform-roboshop-component” and push and pull changes.
 Do changes in “ansible-roboshop-roles-tf” for DNS, port number etc as per the flow decided for “roboshop-infra-dev” as per “roboshop-infra” Architecture diagram.
Create “90-components” folder and create provide.tf, main.tf etc files and write Terraform infra code.
Push “terraform-roboshop-component” code into GitHub.
Get “terraform-roboshop-component”  git url and use in “90-components” to execute “terraform-roboshop-component” from git.
cd ../90-components/   => Inside your system terminal in VS code.
terraform init  => Inside your system terminal in VS code.
Push changes along with terraform generated lock file. 
git pull =>  Inside bastion server
cd ../90-components/   => Inside bastion server
terraform init  => Inside bastion server
terraform plan  => Inside bastion server
terraform apply -auto-approve  => Inside bastion server
This will take about 15 minutes to create all resources and infra.  
Frontend got issue due to port number 8080 using in “nginx.conf” file in “ansible-roboshop-roles-tf” in Catalogue, Cart, User, Payment URLs at the end like :8080, actually no need to write :8080 in these URLs, remove :8080 then frontend issue will be resolved. While debugging infrastructure code need to check NGINX file also.


Diagrams:
roboshop-infra



Timestamps:
QA = 01:20:00




Mistakes & Learning:




Doubts Link Clarification AI chat link: