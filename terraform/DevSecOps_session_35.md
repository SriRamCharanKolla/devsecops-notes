Friday, 27 February 2026

Session 35 - Terraform Naming convention, Documentation, VPC Concept(Subnets, Internet gateway, Routes, IP address)
Class Notes
workspace -> terraform.workspace

terraform workspace new dev
terraform workspace select dev

variables keys should be based on the env names


terraform init -reconfigure -backend-config=dev/backend.config
terraform plan -var-file=dev/dev.tfvars
terraform apply -auto-approve -var-file=dev/terraform.tfvars

CICD tools are useful to reduce human errors here

advantages
==========
code reuse

disadvantages
===========
mistakes can be taken into production
no isolation
blast radius is very high

you can create multiple repos for NON-PROD and PROD

disadvantages
=============
duplicate code

roboshop-infra-dev
roboshop-infra-prod

we can use modules feature to
- reuse
- enforce standards across projects
- central place to push the updates
- easy maintanance

module "name_of_module" {
	source = "git-repo"
}

name_of_module -> programs

name-of-module -> humans

resource "aws_instance" "catalogue" {


}

Ramesh
Human Ramesh

only one resource => this/main

main -> where program starts executing (main code is here)

list of sg_id -> sg_ids

1 resource -> singular
multiple/list -> plural name

VPC
====
Virtual private cloud is like a data center where we can create resources to our project in an isolated space. it is completely in our control.

data center
network connection
subnets
servers

select one village
select one street
get one house number
then construct

subnets -> we create partitions in our network for maintanance and security. if not secure or end users connect directly then it is public subnet. if secure like backend or databases then it is private subnet.

arch -> internet gateway/modem

pincode -> CIDR 

IPV4 Address

32 bits binary system

unary -> 1, 11, 111, etc
binary -> 0 1 -> 2
decimal -> 0 1 2 3 4 5 6 7 8 9 -> 10

168 -> face value and place value
8 -> 8*10^0 = 8
6 -> 6*10^1 = 60
1 -> 1*10^2 = 100

1	6	8
=	=	=
2	1	0

4 parts each part contains 8 bits(Octate)

192.168.145.132 -> decimal number system

128 64  32  16  8   4   2   1
7	6	5	4	3	2	1	0
=	=	=	=	=	=	=	=
1 	0	1	1	0	1	1	1 = 1*2^0 + 1*2^1 + 1*2^2 + 0*2^3 + 1*2^4 + 1*2^5 + 0*2^6 + 1*2^7
							  = 1 + 2 + 4 + 0 + 16 + 32 + 0 + 128
							  = 183
							  
192.168.145.132 = 11000000.10101000.10010001.10000100
							
0.0.0.0
.
.
.
.
255.255.255.255

1bit
====
0
-
1
-

2 = 2^number-of-bits

2 bits
====
0	0
-	-
0	1
-	-
1	0
-	-
1	1
-	-

4 possibilities

32 bits = 2^32 = 4.29 billion IP address

NAT -> 1 -> private ip addresses inside network

IP = Network+Host

500082-20-30

10.0.0.0/16 -> 16 bits are reserved for network, 16 bits are reserved for host/server
2^16 = 65,536 IP address possible

10.0.0.0/24 -> 24 network bits, remaining 8 are allocated for host -> 2^8 -> 256 Ip address possible

192.168.1.0/24 -> 256 devices can be connected to your modem

192.168.1.0 -> modem
192.168.1.1
192.168.1.2
.
.
.
.
192.168.1.255

16 - 28 range CIDR

10.0.0.0/16 -> VPC CIDR

10.0.0.0/24 -> 8 bits for subnets -> 256 subnets -> each subnet can have 2^8 IP address
10.0.1.0/24
10.0.2.0/24
.
.
.
.
.
10.0.255.0/24

Commands:
1. 

Timestamps:
Interview question = 
Virtual Private Cloud(VPC) = 45:04
QA = 01:27:21

Mistakes & Learning:




Doubts Link Clarification AI chat link: 
