Sunday, 29 March 2026

Session 49 - Legacy vs Monolithic Microservices, Baremetal vs Virtulisation vs Containerisation, Installing Docker, Docker commands
Class Notes
Family History

100 years back

joint familities

5 couples
20 children
25-30 members
individual big houses

advantages
==========
no shared, we have freedom
privacy
everything in our control, we can design

disadvantages
==========
too much cost
maintainance on us
time taking process

1 couple
2 or 3 kids
a flat is enough

advantages
==========
low maintainance
less cost
just pay and joint
less time

disadvantages
==========
less privacy
space is shared
we dont have control
more rules

individuals
==========
single person
shared room/pg

advantages
==========
less time
no maintainance at all
no responsibility
too less cost

disadvantages
=========
no privacy
everything is shared
0 control


Legacy = Frontend + Backend
JSP+Servlets+HTML
150Mb

disadvantages
=============
a small problem in frontend/backend crash everything
a small change should create new version.. new application release
size is very big
physical baremetal servers

monolitic
============
frontend  -> API -> backend(catalogue+user+cart+ratings etc.)
angular js

advantages
==========
problem in frontend can't create problems in backend..both are different
release process is different
size is 50% reduced
speed of provisioning(boot time) is reduced

Virtulisation -> 100GB RAM, 2TB HD, 16 cores, host OS(vmware esxi)
				VM -> 1GB RAM, 8GB HD, 1 processor, Guest OS(windows)
				VM -> 2GB RAM, 20GB HD, 2 processor, Guest OS(Linux)

disadvantages
==========
all components must use same programming language
single component problem can crash entire backend


MicroServices
==========
frontend -> user, cart, catalogue, ratings, etc..

advantages
==========
multiple programming languages
release process is different
problem in a single component still keeps the application running
less downtime
connected through API
	URL, Method, Sample request, expected response, status code
	

containerisation
===============
1. no resource allocation, can use dynamically
2. less cost
3. less boot time
4. less size
5. portable and immutable
6. scalable and reliable
7. less privacy, but we can address this

Docker
======
AMI = OS + System packages + App run time + App code + App dependencies -> Instance/Server/VM -> AMI ID
Docker Image = Bare minimum OS + System packages + App run time + App code + App dependencies -> Container

OS + System packages

Docker Installation in AWS EC2 instance server & enable, start, status check docker & 
======================================================
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker 
sudo usermod -aG docker ec2-user   => Adding “ec2-user” to “docker” group to enable “ec2-user” to access Docker directly without sudo access.

logout and login again

Docker commands
===============
docker ps -> running containers
docker images -> images available
docker pull nginx -> pull the nginx image from dockerhub
docker create nginx -> creates the containers, status is created
docker start container-id -> start the container
docker rm container-id
docker rmi nginx
docker run nginx = pull + create + start

nginx = OS + system packages + nginx software


Commands:
Create AWS EC2 t2.micro Linux instance with Redhat-9-DevOps-Practice AMI.
sudo dnf -y install dnf-plugins-core   =>
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo   =>
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y  =>
sudo systemctl start docker  =>
sudo systemctl enable docker  =>
sudo systemctl status docker  =>
sudo docker ps  =>
To run Docker command in AWS EC2 instance should have sudo access like “sudo doctor ps”.
Add “ec2-user” to “docker” group to enable “ec2-user” to access Docker directly without sudo access.
sudo usermod -aG docker ec2-user   => Adding “ec2-user” to “docker” group to enable “ec2-user” to access Docker directly without sudo access.
After running above running command disconnect from AWS EC2 instance then again connect to docker EC2 server then user will get direct access to Docker.
docker ps   => Now this command will work. To check running containers.
docker images  => To check images available
docker pull nginx  => To pull the nginx image from dockerhub
docker images   =>
docker create nginx:latest    => To creates the containers, status is created
docker ps   => 
docker ps -a  => To check list of created status containers
docker start <container-id>   => To start a container   
docker ps -a  =>
docker ps   =>
docker rm <container-id>   => Remove container
docker rm -f <container-id>   => Force remove container
docker rmi nginx   => To remove Docker image, if not specified “:latest” then docker will automatically take latest
docker images   =>
docker run nginx    =>  pull + create + start.



Diagrams:



Timestamps:
Interview questions = 
QA = 01:24:30




Mistakes & Learning:




Doubts Link Clarification AI chat link: