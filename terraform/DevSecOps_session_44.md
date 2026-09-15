Friday, 13 March 2026

Session 44 - Auto Scaling, CPU Stress Test
Class Notes
aws ec2 terminate-instances --instance-ids 

.terraform.lock.hcl -> should be available in git

wherever you pull terraform code, we should run terraform init

curl http://catalogue.backend-alb-dev.daws88s.online/health

1. R53, if someone hits *.backend-alb-dev.daws88s.online -> forward this request to backend alb
2. Listener, this is http listener 80
3. Rule, catalogue rule matches
4. target group, catalogue instance port no 8080
5. before sending, alb sends only to healthy instances

terraform-roboshop-component

Region 		-> AZ
us-east-1	-> us-east-1a

TTL -> update TTL 1 day before to 1sec


Commands:
for I in 00-vpc/ 10-gs/ 20-sg-rules/ 30-bastion/ 50-backend-alb/; do cd $I; terraform apply -auto-approve; cd ..;done  =>
ssh ec2-user@<bastion-public-ip-address> => Connect to bastion.
git clone “roboshop-infra-dev-git-url” => 
cd roboshop-infra-dev/60-catalogue/ => 
terraform init -reconfigure => 
terrform plan =>   
terraform apply -auto-approve =>
Start writing infra code.
curl http://catalogue.backend-alb-dev.<domain-name>/health  => Inside Bastion host server. Response should be ‘OK’ otherwise will get error debug and fix.
 ssh ec2-user@<catalogue-private-ip>  =>
Google search for “stress cpu utilisation rhel”.
sudo yum install stress-ng -y  => inside Catalogue server.
stress-ng —cpu 2 —cpu-load 80  => This command creates 80 percent stress on 2 CPU cores.
Now to Catalogue AWS instance then click on “Monitoring” then observer “CPU utilisation”.
Click on “CPU utilisation”.
Click on Refresh button and check line graph, it should slowly raise.
When the CPU utilisation line graph raised to more than 70% click on “Auto Scaling Groups” is AWS EC2 instance side menu then check new “roboshop-dev-catalogue” instance should be created after some time.
Check “roboshop-dev-catalogue” instance “Activity”.
Under Auto Scaling Group after refreshing for “roboshop-dev-catalogue” instance status will be changes to “Updating capacity”.
Finally “roboshop-dev-catalogue” instance count should be increased to 2 after auto scaling applying to “roboshop-dev-catalogue” group.
Here observe the “Instance need” time is 300 seconds that’s why taking too much time so as a solution should involve instance into metrics ASAP or change to 120 seconds.
Now reduce Catalogue instance stress, then one “roboshop-dev-catalogue” instance should be deleted after CPU utilisation decreased to below 70%.
terraform apply will be run whenever we are creating new resources/infra or doing changes in infra or while production release.
exit  => inside Catalogue server.
In 60-catalogue we implemented catalogue instance as AMI so we should not run “terraform apply -auto-approve” every time because every time catalogue server will be created then convert as AWS AMI then instance will be deleted then in Auto Scaling group newly created catalogue AMI will be takes as grouping instance. So if any changes or prod releases then only should run “terraform apply -auto-approve” otherwise should not run. To avoid this issue should create parameterised Robosho Components.
Create “terraform-roboshop-component” folder and write Roboshop reusable parameterised components.
Write “terraform-roboshop-component” code.      


Diagrams:
roboshop-component


Timestamps:
Terraform lock file = 00:34:10
QA = 01:24:45

Mistakes & Learning:




Doubts Link Clarification AI chat link: