Friday, 13 March 2026

Session 41 - IAM role form MySQL, Roboshop databases, Load Balancer
Class Notes
Iam Role
Iam policy -> it can get the SSM parameter of mysql password
Attach policy to Role
instance_profile
attach to mysql instance

roboshop -> Roboshop
dev -> Dev
mysql -> Mysql

join

Deliver Manager/Project Manger/Team Lead -> features should be delivered on time
Client Business Analyst
IT Business Analyst -> Architects+managers+team leads -> technical requirements

DM -> frontend -> frontend team lead -> frontend team members
   -> backend -> backend team
   -> db -> db team
   -> reports team -> reports team -> reports team member
   
they will check he is available or not and is he occupied or not?

Load Balancer == DM/Team Lead
Target Group == Team == a group of instances
Rule = LB Rules
Listener == 80/443
Health check
Team member == ec2 instance
4 instances -> query for running instances -> round robin -> 1-1, 2-2, 3-3, 

click on categories -> catalogue.daws88s.online
click on user submit -> user.daws88s.online -> post -> {"user":"sivakumar", "password":"siva123"}

catalogue.daws88s.online -> forward that cataloug group

8hr -> 6hr
12hr, 4hr

DM -> HR -> they will recruit new person and add to team

HR == AutoScaling
JD == Launch template -> AMI, SG, NAME, etc

4 instances -> Avg CPU utilisation -> >80%

launch another instance using launch template, add to target group
if avg cpu utlisation is reduced autoscaling terminate the instance
notice period

Commands:
for I in 00-vpc/ 10-sg-rules/ 30-bastion/; do cd $I; terraform apply -auto-approve; cd ..;done => 
ssh ec2-user@<bastion-server-ip>  => Connect to Bastion EC2 instance.
git clone <roboshop-infra-dev-repo-url>    => Clone “roboshop-infra-dev” repo.
cd roboshop-infra-dev/40-database/   => 
terraform init =>
terraform plan  =>
Write terraform code to create MySql infrastructure.
terraform apply -auto-approve => 
Will get MySQL root password issue/error due to Python botocore & boto3, do changes in Ansible MySQL yaml file.
Then run “terraform plan” then terraform will only create MySQL resources.
Again may get same issue because Ansible code not pulling from Git, then add “git pull”  after “cd ansible-roboshop-roles-tf” line in bootstrap.sh file.
Write AWS IAM infra code before that create IAM & its policy manually.
Then write “iam.tf” infra code.
Write Terraform code to create AWS IAM Role for MySql because mysql server need to access ssm parameter store to get mysql root password. And also create Systems Manager Policy with List & GetParameter.
Ssh ec2-user@mysql-dev.aitechapp.fun ‘netstat -lntp’  => To check sql is running on default port number 33060.
Create Load balancer Manually.
Under EC2 instance you will find “Load Balancer”, click onit.
Then click on “Create Load Balancer” button
Then select “Application load balancer.
Then enter “Load balancer name” and other required information.
Then create Target Group to group instances like all Catalogue Instances in one group.


Timestamps:
MySQL AWS IAM User setup =
MySQL AWS IAM Policy setup = 
Assignment = 01:01:47
Load Balancer = 01:18:24
QA = 01:27:00

Mistakes & Learning:




Doubts Link Clarification AI chat link: