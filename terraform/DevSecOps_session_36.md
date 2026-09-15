Friday, 27 February 2026

Session - 36 - VPC manual creation, VPC module through terraform( VPC, IGW, Public Subnets).
Notes
2 AZ
1 subnet for 1 AZ
public - roboshop-public-us-east-1a

public 10.0.1.0/24 10.0.2.0/24 -> 256
private 10.0.11.0/24 10.0.12.0/24
database 10.0.21.0/24 10.0.22.0/24

ingress/inbound -> incoming traffic
egress/outbound -> traffic/data goes out from VPC/servers

dnf install mysql-server -y -> outgoing traffic

NAT gateway -> Ec2 instance
Elastic IP -> Static IP is mandatory to create NAT gateway

1. create VPC
2. create IGW and attach to VPC

roboshop-dev


Commands:

How to create VPC in AWS?
Search for VPC click on you VPC then you will be redirected to your VPCs page in that you will find one default VPC, which is provided by AWS that will help while creating easy to instances. Now click on, “Create VPC” button then enter “Name tag” & “IPv4 CIDR” then click on “Create VPC” button.  How to create Internet gateway in AWS?
Click on “Internet gateways” then click on “Create internet gateway” button then enter “Name tag” then click on “Create internet gateway” then click on “Attach to VPC” button then choose “Available VPCs” (previously created VPC) then click on “Attach internet gateway” button.

How to create Subnet in AWS?
Go to subsets then click on “Create subnet” then choose “VPC ID” then, under “Subnet settings” enter “Subnet name” then choose “Availability Zone” then choose “IPv4 VPC CIDR block” then enter “IPv4 subnet CIDR block” then click on “Create subnet” button.

How to create Route tables in AWS?
Go to “Route Tables” then click on “Create route table” then enter “Name” then choose “VPC” then click on “Create route table” button. Then should attach subnets to Route so click on “Subnet associations” tab then click on “Edit subnet associations” button then choose “Available subnets” public one for public Route table, private one for private route table then click on “Save associations” button to save. Then click on “Edit routes” button then click on “Add route” button choose “0.0.0.0/0”(same for both Public & Private) and  then choose “Internet gateway”(for private & datbse it should be NAT gateway mean only outgoing traffic should allowed) below choose roboshop then click on “Save changes”.

How to create NAT Gateway in AWS?
Go to “NAT gateways” then click on “Create NAT gateway” button then enter “Name” then choose “Availability mode” I am choosing  “Zonal” then choose “Subnet” I am choosing “roboshop-public-us-east-1a” then choose “Connectivity type” I am choosing “Public” then choose “Elastic IP allocation ID” I am choosing previously created Elastic IP then click on “Create NAT gateway” button to save.

How to create Elastic IP Address in AWS?
Go to “Elastic IPs” then click on “Allocate Elastic IP address” button then choose “Network border group” I am choosing “us-east-1” then if you want add tags then click on “Allocate” button to save.


No charges for VPC.
terraform get -update => To get updated code from git. 

Timestamps:
Interview question = 
QA = 01:31:40

Mistakes & Learning:




Doubts Link Clarification AI chat link: