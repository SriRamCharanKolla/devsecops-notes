Friday, 3 April 2026

Session 50 - NACL, Docker commands, Dockerfile(FROM, RUN), Build image.
Class Notes
NACL
=====

IP:port

destination IP and destination port
source IP and source port

https://facebook.com:443

0-65,535

source ip: 192.168.1.3
source port: 32,897

0-1023 -> system ports
1024-5000  -> ephemeral range

32768-65535 -> ephemeral port range

NACL vs SG
===========
SG -> attached to instance
   -> source can be IP address or other SG ID
   -> we can't add DENY rule
   -> Rules are stateful, if we are allowing port number 8080 from 10.0.1.34. while sending response back to the source an ephemeral port is opened. since SG is stateful we no need to write seperate outbound rule
   
NACL -> attached to subnet level
	 -> source can be subnet ID
	 -> NACL is first layer of defence, then SG
	 -> You can add DENY rule..
	 -> NACL are stateless. you have to write outbound rules also...
	 
catalogue -> mongodb

source IP: 10.0.11.26
source port: 32,789

destination IP: 10.0.21.5
destination port: 27017

request -> inbound
response -> outbound

cart -> redis

source IP: 10.0.11.28
source port: 32,792

destination IP: 10.0.21.7
destination port: 6379


for i in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/ 50-backend-alb/ 70-acm/ 80-frontend-alb/ ; do cd $i; terraform apply -auto-approve; cd ..;done

for i in $(ls -dr *); do cd $i; terraform init; terraform destroy -auto-approve; cd ..;done

DOCKER_HOME = /var/lib/docker

When “docker run nginx” is run then below tasks are performed by Docker
1. it checks image available locally or not. if available it will create container from it, otherwise it will download from docker hub, keep it in local and then create the container
2. start the container

containers run in foreground, but we should take them to background

root 3337 3311  0 02:39 ?  00:00:00 nginx: master process nginx -g daemon off;
PID: 3337
PPID: 3311 -> docker engine


docker run -d -p 80:80 nginx
docker run -d -p 8080:80 nginx

docker exec -it cdc00ddf3b81 bash

Dockerfile -> we can create our own customised images

FROM
=====
FROM is used to mention the base OS of the image.. it should be first instruction in Dockerfile
rhel -> redhat image

build the image
===============
docker build -t from:1.0.0 .

docker login

docker push from:1.0.0

url/username/image-name:version
docker.io/joindevops/from:1.0.0

https://hub.docker.com/repositories/joindevops

RUN
===
RUN instruction is used to install packages or configure image

16.20.2
Commands:
Whenever request sent to a website then in our system(laptop, desktop, EC2 server) one source port will be opened to process the internet required once request processed successfully then this source port will be closed automatically.
Create AWS Security Network ACL.
After creating AWS NACL now in “Network ACLs” page check “Inbound rules” see here its denying All Traffic. Test with Roboshop Dev deployment with Card required.
This issue is because AWS NACL is stateless so we need to add Inbound & Outbound rules, here we only setup Inbound rules no outbound rules mentioned. And also MongoDB will send request through ephemeral port to process the request inside system in AWS outbound we should send traffic ephemeral port.
Setup Outbound Rules similarly like Inbound rules setup.

Create an AWS EC2 instance for Docker with 50GB storage.
DOCKER_HOME = /var/lib/docker
ls          => Run inside Bastion server  
ls -dr *  => To get all directories in reverse order. Run inside Bastion server.
for I in $(ls -dr *); do cd $i; terraform init; terraform destroy -auto-approve; ce —;done   => Run inside Bastion server.
Delete manually created AWS NACLs.
Connect to Docker instance.
df -hT   => Run inside Docker server.
sudo -su => Run inside Docker server.
lsblk  => Run inside Docker server.
sudo growpart /dev/nvme0n1 4  => Run inside Docker server.
sudo lvextend -L +30G /dev/RootVG/varVol   => Run inside Docker server.
sudo xfs_growfs /var  => Run inside Docker server.
df -hT   => Run inside Docker server.
Crate an Account in Docker Hub.
Now install Docker with Docker installation & setup commands.
sudo dnf -y install dnf-plugins-core   =>
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo   =>
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y  =>
sudo systemctl start docker  =>
sudo systemctl enable docker  =>
sudo systemctl status docker  =>
sudo usermod -aG docker ec2-user   => Adding “ec2-user” to “docker” group to enable “ec2-user” to access Docker directly without sudo access.
Exit from Docker server then login again to get Docker Sudo access as already run sudo access for docker.
docker ps    => Run inside Docker server.
docker images    => Run inside Docker server.
docker run nginx    => Run inside Docker server.   
docker run -d nginx    => nginx pulled & created & then went to background. -d for detach mode.
Containers run in foreground, but we should take them to background.
Container is nothing but another process in Linux.
ps -ef  =>
docker ps    =>
docker rm -f b3    =>
docker run -d -p 80:80 nginx    => To run Docker on Background and run nginx on port number 80.
http://<docker-IP>:80    => To check nginx with docker run
docker run -d -p 8080:80 nginx    => To run Docker on Background and run nginx on port number 8080.
http://<docker-IP>:8080    => To check nginx with docker run
Multiple nginx can be run in this way.
docker ps    =>
docker exec -it <container-id>    => To give execution permission with interactive terminal access to a Docker Container.
docker exec -it <container-id> bash   => To give execution permission with interactive terminal access and also giving Bash prompt access to a Docker Container. After running this command you be to went inside the container.
cd /usr/share/nginx/html/   => Run inside Docker Container, continuation to above command.
ls -l => Run inside Docker Container.
echo “<h1>Hi from docker</h1>” hi.html  => Run inside Docker Container.
ls -l  => => Run inside Docker Container.
echo “<h1>Hi from docker</h1>” > hi.html  => Run inside Docker Container. 
ls -l  => => Run inside Docker Container.
http://<docker-IP>:8080/hi.html    => To check nginx with docker run
exit  => To Exit from Docker Container
docker inspect <container-ID>    => Run inside Docker server. To know all information about container.
docker stats    => Run inside Docker server. To know memory usage of each container.
docker ps   => Run inside Docker server.
docker rm -f cd <containter-ID>   => Run inside Docker server. At least 1st 2 digits to container ID should provide.
docker ps   => Run inside Docker server.
If no name provide to a Docker Container then Docker will give random name.
docker run -d -p 80:80 —name <container-name> nginx   => Run inside Docker server. To run & provide name to a container.
docker run -d -p 80:80 —name frontend nginx   => Run inside Docker server. To run & provide name frontend to a container.
docker ps   => Run inside Docker server.
docker exec -it <container-name> bash   => Run inside Docker server.
docker exec -it frontend bash   => Run inside Docker server.
exit => Exit fron Docker Container.
docker container prune  => To remove all stopped Docker containers.
docker logs <container-name>  => To verify all logs of a Docker Container.
docker stop <container-name>  => To stop a running container. 
To know Docker container host port check in DockerHub docker image documentation.
Dockerfile -> we can create our own customised images.
Check Docker Reference Documentation to learn syntax of Docker files to write docker files.
Create a “dockerfiles” git repo.
Clone it in Docker Server.
1st should learn “FROM”.
FROM is used to mention the base OS of the image.. it should be first instruction in Dockerfile.
docker pull redhat/ubi9   => Run inside Docker server.
docker images   => Run inside Docker server.
Clone “dockerfiles” git repo into local machine(Laptop/Desktop).
git clone https://github.com/SriRamCharanKolla/dockerfiles.git   => Run this command in your local system.
Create “FROM” folder.
Then create “Dockerfile” file inside “FROM” folder. Docker file name should give “Dockerfile” only.
cd dockerfiles  => Run in your local system.
git add . ; git commit -m “docker”; git push origin main  =>  Run in your local system.
git clone <dockerfiles-git-repo-link>  => Run inside Docker Server.
git clone https://github.com/SriRamCharanKolla/dockerfiles.git => Run inside Docker Server.
cd dockerfiles  => Run inside Docker Server.
cd FROM => Run inside Docker Server. 
ls -l  => Run inside Docker Server.
docker build -t from:1.0.0 .  => To build docker image. Run inside Docker Server. Dot(.) represents current folder.
docker images   => Run inside Docker server.
Run Below Commands to push build Docker image into Docker Hub.
docker login  =>
docker login -u <Docker-user-name>  => After running this command then enter password.
docker push from:1.0.0  => To push docker image into docker hub, but this is not correct syntax do docker will reject.
docker images   =>
docker tag from:1.0.0.0 <docker-user-name>/from:1.0.0   =>   
docker tag from:1.0.0.0 ramcharankola2/from:1.0.0   =>
Now check in Docker Hub Repositories under “from” latest one.
Create “RUN” folder in VS-Code “dockerfiles” repo clone & then create “Dockerfile” inside “RUN” folder to write Docker RUN command.
RUN instruction is used to install packages or configure image.
Write Docker RUN command to install dnf.
 git add . ; git commit -m “docker”; git push origin main  =>  Run in your local system.
 git pull => Run inside Docker Server.
 cd ../RUN/  => 
 docker build -t  <docker-user-name>/from:1.0.0 .  =>  To build Docker image.
 docker images   =>
 


How to create AWS Security Network ACL?
Go to VPC then in side menu under Security click on “Network ACL” then click on “Create network ACL” then “Create network ACL” page will be appearing then enter “Name”(roboshop-dev-roboshop) under  “Network ACL settings” then select “roboshop-dev” for “VPC” then click on “Create network ACL” then in “Network ACLs” page select “roboshop-dev-roboshop” then click on “Subnet associations” tab then click on “Edit subnet associations” then select “roboshop-dev-database-us-east-1a” & “roboshop-dev-database-us-east-1b” subnets then click on “Save changes”.


How to Add AWS Security Network ACL Inbound Rules?
“Network ACLs” page select “Inbound rules” then click on “Edit inbound rules” then in “Edit inbound rules” page click on “Add new rule” then enter “Rule number”(10) then select “Custom TCP” for “Type” then enter “Port”(27017) then enter “Source”(10.0.11.0/24) for source always should provide subnet IP starting range otherwise if only specific IP address is provided then due to auto scaling this IP address will always be changes, then select “Allow” for “Allow/Deny” then click on “Add new rule” then enter “Rule number”(15) then select “Custom TCP” for “Type” then enter “Port”(27017) then enter “Source”(10.0.12.0/24) for source always should provide subnet IP starting range otherwise if only specific IP address is provided then due to auto scaling this IP address will always be changes, then select “Allow” for “Allow/Deny” then click on “Save changes”.


How to Add AWS Security Network ACL Outbound Rules?
“Network ACLs” page select “Outbound rules” then click on “Edit outbound rules” then in “Edit outbound rules” page click on “Add new rule” then enter “Rule number”(10) then select “Custom TCP” for “Type” then enter “Port range”(32768-65535) then enter “Source”(10.0.11.0/24) for source always should provide subnet IP starting range otherwise if only specific IP address is provided then due to auto scaling this IP address will always be changes, then select “Allow” for “Allow/Deny” then click on “Add new rule” then enter “Rule number”(15) then select “Custom TCP” for “Type” then enter “Port range”(32768-65535) then enter “Source”(10.0.12.0/24) for source always should provide subnet IP starting range otherwise if only specific IP address is provided then due to auto scaling this IP address will always be changes, then select “Allow” for “Allow/Deny” then click on “Save changes”.



Diagrams:
containers
roboshop-infra-update





Timestamps:
Interview questions = 
QA = 01:25:30





Mistakes & Learning:




Doubts Link Clarification AI chat link: