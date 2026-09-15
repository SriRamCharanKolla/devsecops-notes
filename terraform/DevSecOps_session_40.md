Monday, 9 March 2026

Session 40 - Bastion to database using provisioner, Terraform ansible integration, Bastion user data.
Class Notes
1. S3 access, state file retrieval, creation, update
2. Ec2 creation access
3. R53 entries creation, deletion, update

1. connect to the instance and configure it
2. update r53 record

null resource -> earlier
terraform_data -> follows standard resource lifecycle, but it will not create any resources. we use this to configure instance using local-exec, remote-exec. we have one argument trigger_replace. we can decide which one to trigger the configuration again


bastion -> creates ec2 instance

connecting to mongodb

install ansible inside bastion -> mongodb
or
install ansible inside mongodb
configure it as localhost

bootstrap.sh
============
install ansible
configure it using playbooks

copy this script to mongodb
execute through remote-exec

user-data
==========
1. once the system is provisioned, AWS executes the commands inside user_data script
2. if user_data script is failed, terraform is not aware. terraform is still success
3. since AWS runs user_data we cant get immidiate log
4. useful for simple installations

provisioner
==========
1. It is terraform resource
2. if remote-exec is failed, terraform is also failed.
3. we can get immidiate log on the console about what is going on


Commands:
cd /roboshop-infra-dev/
ls
for I in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/; do cd $i; terraform apply -auto-approve; cd ..;done
Create “40-databases” folder & start writing Terraform & other code files.
ssh ec2-user@<roboshop-dev-bastion-public-IP>   => Connect to “roboshop-dev-bastion” ec2 instance with public IP address.
git clone <git-roboshop-infra-dev-link> => Clone “roboshop-infra-dev” repo inside “roboshop-dev-bastion” ec2 instance.
cd roboshop-infra-dev/  => Change directory to “roboshop-infra-dev” “roboshop-dev-bastion” ec2 instance.
cd 40-databases/  => “roboshop-dev-bastion” ec2 instance.
terraform init => “roboshop-dev-bastion” ec2 instance.
Install Terraform in “roboshop-dev-bastion” ec2 instance by following Terraform RHEL commands.
sudo yum install -y yum-utils  => “roboshop-dev-bastion” ec2 instance.   
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo   => “roboshop-dev-bastion” ec2 instance.
sudo yum -y install terraform   => “roboshop-dev-bastion” ec2 instance.
terraform init  => “roboshop-dev-bastion” ec2 instance.
git pull => “roboshop-dev-bastion” ec2 instance if you push any Terraform code changes.
terraform plan  => “roboshop-dev-bastion” ec2 instance.
terraform apply -auto-approve   => “roboshop-dev-bastion” ec2 instance.
ls -la => “roboshop-dev-bastion” ec2 instance.
cd .terraform  => “roboshop-dev-bastion” ec2 instance.
ls -l   => “roboshop-dev-bastion” ec2 instance.
cd providers/   => “roboshop-dev-bastion” ec2 instance.
ls  => “roboshop-dev-bastion” ec2 instance.  
cd registry.terraform.io/  => “roboshop-dev-bastion” ec2 instance.
ls  => “roboshop-dev-bastion” ec2 instance.
cd hashicorp/   => “roboshop-dev-bastion” ec2 instance.
cd aws/   => “roboshop-dev-bastion” ec2 instance.
ls   => “roboshop-dev-bastion” ec2 instance.
cd 6.33.0/   => “roboshop-dev-bastion” ec2 instance.
ls  => “roboshop-dev-bastion” ec2 instance.
cd linux_amd64/   => “roboshop-dev-bastion” ec2 instance.
ls -l   => “roboshop-dev-bastion” ec2 instance.
pwd  => “roboshop-dev-bastion” ec2 instance.
ls -l  =>  “roboshop-dev-bastion” ec2 instance.
df -hT  => “roboshop-dev-bastion” ec2 instance.
Write Terraform code to root block device size & extend volume then do partisan.

ssh ec2-user@<roboshop-dev-bastion-public-IP>   => Connect to “roboshop-dev-bastion” ec2 instance with public IP address.
sudo su -    =>
cd /var/log/  =>
ls -l  =>
less cloud-init-output.log
ctrl+c  =>
df -hT  => To check “/dev/mapper/RootVG-homeVol” storage volume size.
exit  =>
sudo less /var/log/cloud-init-output.log  =>
terraform version  => 
Now with the help of Ansible Roboshop Roles we need to configure MongoDB. So that create new repo with name “ansible-roboshop-roles-tf” to get Ansible code inside “roboshop-dev-mongoldb” server.
Here we are connecting to “roboshop-dev-mongoldb” server from “roboshop-dev-bastion” server(connect to with you system as we configure with my_ip4 then bastion will allow you to connect) then with the help of Ansible we are installing required mongodb packages into roboshop-dev-mongoldb”server with localhost.
git clone <git-roboshop-infra-dev-link>  =>  <ansible-roboshop-roles-tf-git-link>
cd roboshop-infra-dev/40-databases/  => 
terraform init  =>
terraform plan  =>
terraform apply -auto-approve   =>
ssh ce2-user@<roboshop-dev-mongodb-private-IP-address>  => connect to “roboshop-dev-mongodb” server from “roboshop-dev-bastion” server.
netstat -lntp   => After connecting to mongodb server to check mongodb running on its default port number 27017.
exit => To get exit from mondodb server from bastion server. 
Write Route53 Terraform code for custom DNS.
Create zone_id variable with AWS r53 DNS record ID and create domain_name with purchased domain name like aitechapp.fun.
Write code for redis, mysql, rabbitmq databases aws resource creation.
Do required local host and other changes in Ansible roles playbook.
ssh ec2-user@<redis-dev.<domain-name> ‘netstat -lnpt’  => To connect with Redis DB and check running on its default port number 6379.                  


Diagrams:
Bastion-databases

Timestamps:
QA = 01:28:42

Mistakes & Learning:

1. for i in 00-vpc/ 10-sg/ 20-sg-rules 30-bastion/; do cd $i; terraform apply -auto-approve; cd ..;done   => In this command I missed / after 20-sg-rules and also if any network issues they this command creating issues like resource not creating properly, in my case ssm parameters error I got due to network issues.

Sol:  for i in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/; do
  echo "Running $i"
  cd $i || exit 1

  [ ! -d ".terraform" ] && terraform init
  terraform apply -auto-approve || exit 1

  cd ..
done

Note: With above script the terminal is deleting then unable to track error


for i in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/; do
  echo "========================="
  echo "Running $i"
  echo "========================="

  cd $i || exit 1

  if [ ! -d ".terraform" ]; then
    terraform init
  fi

  terraform apply -auto-approve
  if [ $? -ne 0 ]; then
    echo "❌ Error occurred in $i"
    exit 1
  fi

  cd ..
done

To Delete All Resources:
for i in 30-bastion/ 20-sg-rules/ 10-sg/ 00-vpc/; do
  echo "Destroying $i"
  cd $i || exit 1

  [ ! -d ".terraform" ] && terraform init
  terraform destroy -auto-approve || exit 1

  cd ..
done


2. In bootstrap.sh file .yaml typo caused below issue.

terraform_data.bootstrap_redis (remote-exec): ERROR! the playbook: roboshop.ymal could not be found terraform_data.mongodb (remote-exec): ERROR! the playbook: roboshop.ymal could not be found

3. Error: adding IAM Role (arn:aws:iam::069416262340:role/Roboshop-Dev-Mysql) to IAM Instance Profile (roboshop-dev-mysql): operation error IAM: AddRoleToInstanceProfile, https response error StatusCode: 400, RequestID: 94a98668-7340-477f-9bcd-630df5333b48, api error ValidationError: The specified value for roleName is invalid. It must contain only alphanumeric characters and/or the following: +=,.@_-
│ 
│   with aws_iam_instance_profile.mysql,
│   on iam.tf line 41, in resource "aws_iam_instance_profile" "mysql":
│   41: resource "aws_iam_instance_profile" "mysql" {

Sol:- I already created “Roboshop-Dev-Mysql” AWS IAM Role Manually but not deleted the role and after executing “roboshop-infra-dev” - 40-databases then I got this issue, solution if a resource already exists in AWS then Terraform always through conflict error, so should remove. I deleted “Roboshop-Dev-Mysql” AWS IAM Role then executed below commands.  terraform state list | grep iam

Output:  aws_iam_instance_profile.mysql
aws_iam_policy.mysql
aws_iam_role.mysql
aws_iam_role_policy_attachment.mysql   terraform destroy -target=aws_iam_role.mysql
terraform destroy -target=aws_iam_instance_profile.mysql 




Doubts Link Clarification AI chat link: