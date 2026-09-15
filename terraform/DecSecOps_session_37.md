Monday, 2 March 2026

Session - 37 - VPC Module completed, VPC Peering.
Notes
1. VPC -> CIDR
2. IGW with VPC association
3. Subnets -> public private database
4. Route tables -> Publlic private database
5. Associtaions and Routes
6. EIP
7. NAT gateway -> to provide egress internet access to the resources in private subnets

public RT
=========
0.0.0.0/0 IGW

private RT
=========
0.0.0.0/0 NAT

By default 2 VPCs in AWS can't connect with each other.

10.0.11.34 -> VPC-1 Backend server

10.0.0.0/16 -> VPC-1 CIDR

VPC-2 -> 10.1.0.0/16

10.1

VPC-Peering
==========
connecting two VPC in AWS. They can be in 
-> same region and same account
-> diff region same account
-> diff account same region
-> diff account diff region

only condition is CIDR of VPC should not overlap.

requestor -> who asks for connection -> roboshop-dev
acceptor -> who can accept -> default

public subnet -> 54.78.98.123 10.0.1.23 -> 192.168.2.1

172.31.1.3

1. Project infra -> one time infra

2. Application infra -> frequently changing

How to create VPC peering connection?
Go Peering Connections then enter “Name” I am giving “roboshop-dev-default” then select “VPC ID(Requester)” I an selecting “roboshop-dev” then select “Select another VPC to peer with” I am selecting “My account” then select “VPC ID(Accepter)” I am select same account default VPC ID then finally click on “Create peering connection” button. Then click on “Actions” in “Peering connections” list the click on “Accept request” because previously selected same account VPC ID. If raised for other account then we need to ask others to accept request.

Commands:
1. 

Timestamps:
VPC peering = 34:00
QA = 01:14:30

Mistakes & Learning:




Doubts Link Clarification AI chat link:
https://chat.z.ai/c/566b2042-5791-4a86-8ddb-484d8f10d276