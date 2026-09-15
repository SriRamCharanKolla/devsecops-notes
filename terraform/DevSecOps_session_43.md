Friday, 13 March 2026

Session 43 - Load balancer terraform approach(Load Balancer, Listener, Rule, Health Check, Catalogue instance and configuration, Route53 record), Auto Scaling and Auto Scaling Group Manual & Terraform Approach
Class Notes
1. Launch catalogue instance
2. Configure using terraform_data and remote-exec provisioner
3. Stop instance before taking AMI
	aws_instance creates first
	terraform_data and aws_ec2_instance_state both are dependent on aws_instance
	when instance created, terraform immidiately triggers both terraform_data and aws_ec2_instance_state parellely

4. create target group 
5. create launch template
6. input to autoscaling group
7. ask autoscaling to launch instances into target group
8. create a rule in listener for catalogue
9. delete  instance used for ami

1a instance 1b instance

Min -> at any point of time how many instances you want
Desired -> How many you want right now.
Max -> Maximum instances autoscaling group can create

Min = 2
Desired = 4
Max = 10


Commands:
for I in 00-vpc/ 10-gs/ 20-sg-rules/ 30-bastion/ 50-backend-alb/; do cd $I; terraform apply -auto-approve; cd ..;done  =>
ssh ec2-user@<bastion-public-ip-address> => Connect to bastion.
git clone “roboshop-infra-dev-git-url” => 
cd roboshop-infra-dev/40-databases/ => To create databases infra & deploy & run databases.
terraform init => 
terrform plan =>   
terraform apply -auto-approve =>
Create “60-catalogue” folder to create Catalogue backend Terraform code files and to writes catalogue infra code.
Write steps to create Catalogue infra.
Write Catalogue Terraform code like data. Locals, variables, main, provider etc.
git pull  => If you pushed away infra code changes.
cd ../60-catalogue  =>
terraform init  =>
terraform plan  =>
terraform apply -auto-approve  =>
ssh ec2-user@<catalogue-private-IP> ‘systemctl status catalogue’  =>  
ssh ec2-user@<catalogue-private-IP> ‘netstat -lntp’  =>  To check Catalogue is running on port number 8080.
ssh ec2-user@<catalogue-private-IP> ‘curl http://localhost:8080/health’  => Health check.
Create AWS Launch Template and edit Target group and provide 60 for “deregistration delay”.
2nd time creating Launch Template and Auto Scaling and Auto Scaling group.
This time 2 Auto Scaling instances will be created because to handle traffic 2 desired capacity was given. 1 will be created on use-east-1a zone and the other one will be create inside use-east-1b zone.
Check 2 AWS instances are created are not with “robosho-dev-catalogue” name in us-east-1a and in us-east-1b zones.
This is called Auto Scaling.
If cost matters then chose one availability zone is also fine, but 2 availability zone is recommended.
Now the catalogue is ready so now should create Rule in Load Balancer.
Then Click on in Load Balancer Listener page click on “Add rule” button then under “Host header” section enter “Host header condition value”(catalogue.backend-alb-dev.<domain-name>) then under “Actions” select “roboshop-dev-catalogue” for “Target group” then click on “Next” button then under “Set rule priority” page under “Listener rules” enter “Priority”(10) then click on “Next” button then click on “Add rule” button, that’s it.
curl http://catalogue.backend-alb-dev.<domain-name>/health  =>
 curl http://catalogue.backend-alb-dev.<domain-name>/categories =>
Now start writing Terraform code for target group, launch template, auto scaling, auto scaling, auto scaling group.


How to create AWS Launch Template Manually?
Go AWS EC2 then under “Instances” click on “Lunch Template” then click on “Create launch template” button then enter “Launch template name and description”(roboshop-dev-catalogue) then enter “Template version description”(A dev catalogue server for roboshop) keep “Auto Scaling guidance” check box empty then under “Application and OS Images(Amazon Machine Image)” select “My AMIs” tab then select “Owned by me” then select “roboshop-dev-catalogue” for “Amazon Machine Image(AMI) then select “t3.micro” for “Instance type” then select “Don’t include in launch template” option for “Key pair name” then under network settings select “roboshop-dev-private-us-east-1a” for “Subnet” then select “roboshop-dev-catalogue” for “Common security groups” then click on “Create launch template” button, that’s it.




How to create AWS Auto Scaling Group Manually?
Go AWS EC2 then under “Auto Scaling” click on “Auto Scaling Group” then click on “Create Auto Scaling Group” button then enter “Auto Scaling group name”(roboshop-dev-catalogue) then select “Launch template”(roboshop-dev-catalogue) then select “Latest(1)” for “Version” then click on “Next” button then under “Network” select “roboshop-dev” for “VPC” then select “roboshop-dev-private-us-east-1a” for “Availability Zones and subnets” then click on “Next” button then under “Load balancing” select “Attach to existing load balancer” then select “roboshop-dev-catalogue” for “Existing load balancer target group” then enter “Health check grace period”(120 seconds) then click on “Next” button then enter under “Group size” enter “Desired capacity”(2) then under “Scaling” enter “Min desired capacity”(1) then enter “Max desired capacity”(10) then select “Target tracking scaling policy” for “Choose whether to use a target tracking policy” under “Automatic scaling” then select/keep “Average CPU utilisation” then enter “Target value”(70) then enter “Instance warmup”(120 seconds) then click on “Next” button then again click on “Next” button then in “Review” page review Auto Scaling Group details once then click on “Create Auto Scaling group” button.

Here we are creating 2 Auto Scaling groups if once instance traffic is reached 70%  then it will keep both other only keep 1. 



Diagrams:


Timestamps:
QA = 01:28:31

Mistakes & Learning:




Doubts Link Clarification AI chat link: