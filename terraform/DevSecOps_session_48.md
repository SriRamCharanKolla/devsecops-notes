Sunday, 29 March 2026

Session 48 - VPN, Open VPN, Terraform import, Terraform target, Terraform taint, Terraform lifecycle options.
Class Notes
CISCO VPN

OpenVPN Access Server Community Image-8fbe3379-63b6-43e8-87bd-0e93fd7be8f3
ami-03ea534e3e9b6f519

ssh access username: openvpnas
admin UI username: openvpn

manually in console

import existing infra into terraform

resource definition -> infra

infra -> resource

reverse engineering

1. terraform import is the option
2. create skelton syntax for all the resources and provider file
3. run import command to get the details from terraform state
4. we get full details, we can pick and place them into resource definition


terraform import aws_instance.web i-12345678

R53 -> frontend ALB -> frontend -> backend alb -> components -> db

create infra parellelly

stateless vs stateful
======================
stateful -> state(data) is important. databases are stateful applications
stateless -> backend and frontend applications created using programming languages are called stateless applications, we can easily even they crash

dummy R53 -> new frontend ALB -> new frontend -> new backend alb -> components -> db

R53 -> new frontend ALB

terraform target -> you can target specific resource for creation or deletion

terraform taint ->

taint -> paint/pollute/corrupt


Commands:
Generally companies use CISCO VPN, but here we are taking open VPN.
Create a Security Group for OpenVpn manually.
Create a AWS instance for OpenVpn manually.
Then once Add Rules for “roboshop-dev-openvpn” security group.
ssh -I <key-name> openvpnas@<roboshop-dev-openvpn-public-IP>  => To connect with “roboshop-dev-openvpn” AWS instance.
Then enter “yes” for “please enter ‘yes’ to indicate your agreement [no]” prompt.
Then press enter all prompts.
Enter “yes” for “Should client traffic be routed by default through VPN? > Press ENTER for default [no]: “.
Press ENTER for default [no]:   all these prompts should press enter.
Then set password for OpenVPN with one Capita case letter like Openvpn123.
Then confirm same password.
Then copy the url you got from open VPN then enter in you chrome browser.
Then Sign In with Username “openvpn” then enter password like Openvpn123 then click on "Sign In”.
Then click on “Agree” button for “License Agreement”.
Then dashboard will be displayed then click on “Advanced” in side menu then click on “Proceed” button for the popup you got again by checking checkbox.
Then search for “dns" in open VPN Advance settings search bar.
Then search for “traffic" in open VPN Advance settings search bar.
Settings can be changed in Advance Settings page.
Generally VPN Administrator will access Open VPN dashboard and they will manage.
This is open VPN server, to connect with open VPN then should have open VPN client software.
In chrome search for “openvpn client download” then click on official OpenVPN page then download open VPN software and install it.
Then open OpenVPN client software then enter “Client UI” url in https://www.whatismyip.com/“Enter your URL or Cloud ID” input box then click on ”Next” button then click on “Accept” button then enter user name and password.
Now check your current system IP in “https://www.whatismyip.com/" .
Now go to open VPN application then toggle on then enter password then click on “OK” button. Once you successfully connected to the OpenVPN server then refresh “https://www.whatismyip.com/" page then check now your IP address will be changed and location will be showing somewhere from your current location.
The IP address showing in “https://www.whatismyip.com/" is AWS “roboshop-dev-openvpn” IP address.
Now go to “roboshop-dev-bastion” security group and add shh security rule by selecting “roboshop-dev-openvpn” security group source for security rule of “roboshop-dev-bastion”.
Generally in a company all team members have access to VPN but they not have access to Backend ALB, Catalogue, Cart, User, Shipping, Payment, MongoDB, MySQL, Redis, RabbitMQ, Frontend ALB, Frontend.
Now check the connect with Backend ALB like “something.backend-alb-dev.<domain-name>. Generally we can’t connect with this URL and because we are connecting from VPN which don’t have access to Backend ALB, so we should enable connection from VPN then only Backend ALB will allow to connect.
So change “roboshop-dev-backend-alb” security group rules.
Then Add a SG Rule to allow connection through port number 80 with “robosho-dev-openvpn” SG.
Now refresh “something.backend-alb-dev.<domain-name> page, now can able to access Backend ALB.
Similarly do for Catalogue so that Catalogue developers can connect with VPN through their browser.
catalogue.backend-alb-dev.<domain-name>  => once enabled connect from VPN to Catalogue then enter this URL in the browser and check should able connect to Catalogue.
Now delete all manually created AWS infra and start implementing all above with Terraform.
Create “98-openvpn” folder and create required files and code for openVPN.
In interview we can say like for “Roboshop” development project created VPN with OpenVPN like that.
For OpenVPN should create SG, SG Rules, SSM Parameter store first.
Now write OpenVPN Terraform code.
Execute Terraform code to create OpenVPN infra.
Create “import” folder in “Terraform” project(learning & practice one).
Terraform Import is used to create Terraform code by importing existing infra like AWS instance. Terraform import resource/infra should not be managed with Terraform then only we can import resource/infra from AWS.
terraform import aws_instance.<resource-name> <instance-id>   => To import AWS instance with Terraform
Firstly you should on Terraform import folder then should run Terraform import command.
cd ../../terraform/import/   =>
terraform init   => 
terraform import aws_instance.import i-074f714a1cee05e12  => To import AWS instance infra to  aws_instance.import
instance with Terraform, here importing terraform code into import.
Now “terraform.tfstate” file will be created and from this file whatever infra code want that should be pick from here and use it.
Then terraform plan should be run to know which Terraform we missed from “terraform.tfstate” file to create Terraform code from existing infrastructure.
Based on the “terraform plan” command should add the required code.
From the import code we can do changes, delete infra, similarly like Terraform manages code.
With tag name or other code data we can able to know the infra is created with Terraform or not.
terraform plan  =>
terraform apply -auto-approve   =>
terraform destroy -auto-approve  =>
If project in large & complex then we will do Terraform import for databased because we can change databased infra remaining infra we can create parallely. We create dummy R53 records to test infra then once everything is working fine then we will announce down time and go for release.
Terraform target is used to destroy only particular resources. Terraform Target is dangerous because if any resource in dependent on other resource then it will delete both, so whenever situation came to use Terraform Target then should carefully analyse the dependencies then only should use it.
terraform destroy -target=<resource-type>.<resource-name>  => Syntax of Terraform Target.
Terraform Tiant is used to recreate corrupted resources/infra by deleting old one. We apply terraform tiant to a terraform module or folder then if any of that code is corrupted then it will delete previous one and create new infra. 
terraform tiant <resource-type>.<resource-name>  => Terraform Tiant Syntax
Terraform life cycle is used to create resource before destroy. 
Below code snippet need to add to use terraform life cycle feature.
lifecycle {
   create_before_destroy = true
}
 terraform fmt  => To format Terraform code.  If terraform code syntax is correct then only this command will format terraform code       



How to create Open VPN AWS instance Manually?
Go AWS EC2 then in side menu click on “Instances” then click on “Lunch instances” button then in “Launch an instance” page under “Name and tags” enter “roboshop-dev-openvpn” for “Name” field then under “Application and OS Images (Amazon Machine Image)” click on “Browse more AMIs” in search bar search for “openvpn access” and click on “Community AMIs” tab then select this “OpenVPN Access Server Community Image-8fbe3379-63b6-43e8-87bd-0e93fd7be8f3” VPN by clicking on “Select” button. t3.small is minimum requirement for this AWS AMI Community image. Then under “Key pair (login)” select “key-pairs-you-have” for “Key pair name - required” then under “Network settings” click on “Edit” button then select “roboshop-dev” option for “VPC - required” then select “roboshop-dev-public-us-east-1a” option for “Subnet” then create a security group for VPN. After creating a security group for VPN then click on “Compare security group rules” refresh icon to get Open VPN SG in “Common security groups” dropdown list then select “roboshop-dev-opevpn” security group option for “Common security groups” then click on “Launch instance” button.


How to create Security Group for Open VPN AWS instance Manually?
Go AWS EC2 then in side menu under “Network & Security” click on “Security Groups” then click on “Create security group” button then in “Create security group” page under “Basic details” enter “roboshop-dev-openvpn” for “Security group name” field then enter “roboshop-dev-openvpn” for “Description” then select “roboshop-dev” option for “VPC” then click on “Create security group” button.

How to create Rules for Open VPN AWS Security Group Manually?
Once security group created successfully then click on “Edit inbound rules” button then click on “Add rule” button then select “Custom TCP” for “Type” then enter “443” for “Port range” to allow connections from outside to inside then click on “Save rules” button then agin click on “Add rule” button to enable connection from port number “22” so select “Custom TCP” for “Type” then enter “22” for “Port range” to enable connection then click on “Save rules” button then agin click on “Add rule” button to enable connection from port number “943” so select “Custom TCP” for “Type” then enter “943” for “Port range” to enable connection then click on “Save rules” button. Select inbond traffic with “0.0.0.0/0” for all rules.

How to create Rules for Open VPN AWS Security Group Manually?
Go to AWS EC2 security groups from side menu then search for “roboshop-dev-bastion” then select it then click on “Edit inbound rules” button then click on “Add rule” button then select “SSH” for “Type” then in “Source”(Custom) search for “roboshop-dev-openvpn” and select it then click on “Save rules” button.


Diagrams:
roboshop-infra



Timestamps:
Terraform Import = 00:40:26
Interview questions = 01:08:10
Interview questions - Terraform Target = 01:15:52
QA = 01:26:46




Mistakes & Learning:




Doubts Link Clarification AI chat link: