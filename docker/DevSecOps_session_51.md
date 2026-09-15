Saturday, 4 April 2026

Session 51 - Dockerfile(CMD, LABEL, EXPOSE, ENV, COPY, ADD, ENTRYPOINT, ARG), RUN vs CMD, CMD vs ENTRYPOINT, ENV vs ARG.
Class Notes
1. Build the image -> Bare min OS + System packages + App run time + Code + dependencies
2. Run the image -> port, name, background, other component urls, etc.

docker ps -> running containers
docker ps -a -> all containers
docker images -> show the images in server
docker pull image-name:version
docker create image-name:version
docker start container-id
docker run image-name:version = pull + create + start
docker rm container-id -> remove the container

docker run -d
docker run -p host-port:container-port
docker run --name <container-name>
docker stats -> show the resources consumed by containers
docker inspect container-id

Dockerfile -> declarative way of creating custom images by using docker instructions

FROM -> should be 1st instruction, to refer base OS
RUN -> used to configure image install packages, create users, etc. used at the time of building the image

docker build -t url/username/image-name:version .
docker login
docker tag image-name:version username/image-name:version
docker push username/image-name:version

CMD
====
used to run the container. executed at the time of running the container. it will not execute at the time of building..

we can have multiple RUN instructions inside dockerfile. but only one CMD instruction

["executable","param1","param2"]

systemctl start nginx -> won't work inside images, systemctl is kernel level process

your command inside dockerfile should run in foreground, but you can take it into background

LABEL
=====
adds the metadata to the image, used for filteration

EXPOSE
=====
useful to display the port information of container. it will not really open the port but provides information only to the users

80
8080 -> can't open

ENV
====
to provide env variables to the container. app code can access these env variables and use them

COPY
====
used to copy the files from local to image

ADD
===
used to copy the files from local to image, but it has 2 extra capabilities
1. It can fetch the content directly from internet
2. It can untar the content directly in the image

ENTRYPOINT
===========
ENTRYPOINT is used to provide the command executed by the container when it is started.

ping google.com ping facebook.com

CMD vs ENTRYPOINT  => Important interview question
=========
We can override the command inside CMD instruction at run time... We can't override ENTRYPOINT command if you try to do so it will go and append.. for best results we can always use both CMD and ENTRYPOINT

CMD can be used to provide the default args/inputs to ENTRYPOINT.. if we want our own args we can always override CMD at runtime

never run container with root access..

ENV vs ARG
==========
ENV -> provide variables to container
ARG -> provide variables to image

ARG variables can't be accessed inside container, they are accessible only at the of image building. ENV variables can be accessed in container

ARG can be first instruction in a special case to supply the version to base image


Commands:
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
Now install Docker with Docker installation & setup commands.
Exit from Docker server then login again to get Docker Sudo access as already run sudo access for docker.
Docker CMD instruction is used to run the container. executed at the time of running the container. it will not execute at the time of building. we can have multiple RUN instructions inside dockerfile. but only one CMD instruction.
create “CMD” folder in local machine & create a “Dockerfile” file inside “CMD” folder.
Write Docker instructions of CMD and push code into git then pull inside Docker server.
git clone <dockerfiles-git-repo-link>  => Run inside Docker Server.
cd dockerfiles/CMD  => Run inside Docker Server.
docker build -t <image-name>:<tag> <path> => Command to build Docker Image.
docker build -t  cmd:1.0.0 .  =>  To build Docker image.
Check the logs here, only two Docker instructions are executed and Docker CMD instruction will not run after running doctor build command, Docker CMD instruction will only run when after running Docker run command.
docker run cmd:1.0.0 .  =>  To run Docker image.
your command inside dockerfile should run in foreground, but you can take it into background.
docker run -d -p 80:80 cmd:1.0.0 .  =>  To run Docker image in Background in port number 80.
docker logs bf  =>
docker logs -f bf  =>
Docker LABEL is used to display the port information of container. it will not really open the port but provides information only to the users.
Create “LABEL” folder in local machine & create a “Dockerfile” file inside “LABEL” folder.
Write Docker instructions of LABEL and push code into git then pull inside Docker server.
git pull  => 
cd ../LABEL/  =>
docker build -t  label:1.0.0 .  =>  To build Docker image.
docker images   =>
docker images —filter “label=COURSE=Docker”  => To filter Docker image with Label name.
docker images —help  =>   
docker inspect label:1.0.0   =>
docker run label:1.0.0   =>
docker ps   =>
docker ps -a  =>
Here label container got exit because we did not keep the container busy(running in background with execution command). Here “/bin/bash” should not keep the container to run in background(busy).
docker run -d label:1.0.0 sleep 15   =>
docker ps   =>
docker ps -a  =>
Docker EXPOSE is used to display the port information of container. it will not really open the port but provides information only to the users.
Create “EXPOSE” folder in local machine & create a “Dockerfile” file inside “EXPOSE” folder.
Write Docker instructions of EXPOSE and push code into git then pull inside Docker server.
git pull  =>
cd ../EXPOSE/  =>
docker build -t  expose:1.0.0 .  =>  To build Docker image.
docker run -d -p 8080:80 expose:1.0.0   =>
docker inspect expose:1.0.0 or <docker-container-id>   =>
docker ps   =>
docker rm -f <docker-id>   =>
Change Docker EXPOSE port 80 to 8080. Then push & pull changes.
docker build -t  expose:1.0.0 .  =>  To build Docker image.
docker inspect expose:1.0.0  =>
docker run -d -p 8080:80 expose:1.0.0   =>
docker inspect <expose-docker-id>   =>
Docker ENV is used to provide environment variables to the container. app code can access these env variables and use them.
Create “ENV” folder in local machine & create a “Dockerfile” file inside “ENV” folder.
Write Docker instructions of ENV and push code into git then pull inside Docker server.
git pull  =>
cd ../ENV/  =>
docker build -t  env:1.0.0 .  =>  To build Docker image.
docker run -d env:1.0.0 sleep 100   =>
docker run -d -p 8080:80 env:1.0.0 sleep 100   =>
docker exec -it <env-container-id> bash  =>
Docker COPY is used to provide env variables to the container. app code can access these env variables and use them.
Create “COPY” folder in local machine & create a “Dockerfile” & “index.html” files inside “COPY” folder.
Write Docker instructions of COPY and push code into git then pull inside Docker server.
git pull  =>
cd ../COPY/  =>
docker build -t  copy:1.0.0 .  =>  To build Docker image.
docker ps   =>
docker ps -a -q => to get all docker container IDs 
docker rm -f `docker ps -a -q` => To get all docker containers. This `docker ps -a -q` provide docker container IDs then “docker rm -f” remove containers.
docker ps   =>
docker ps -a -q => to get all docker container IDs
docker run -d -p 80:80 copy:1.0.0   =>
Now enter Docker Server Public IP in browser and check should show “CPOY” folder “index.html” output.
Add any website code in COPY folder like qi website which is old one I could add here then comment “index.html” Docker command code then write new like to copy qi website to nginx/html/ folder.
If any changes did in Docker image then should rebuild Docker image.
docker build -t  copy:1.0.0 .  =>
docker rm -f `docker ps -a -q` =>
docker run -d -p 80:80 copy:1.0.0   =>
Docker ADD is used to copy the files from local to image, but it has 2 extra capabilities. 1st one is it can fetch the content directly from internet. 2nd one is it can untar the content directly in the image.
Create “ADD” folder in local machine & create a “Dockerfile” file inside “ADD” folder.
Write Docker instructions of ADD and push code into git then pull inside Docker server.
git pull  =>
cd ../ADD/  =>
docker build -t  add:1.0.0 .  =>  To build Docker image.
docker run -d add:1.0.0 sleep 100   =>
docker exec -it <add-container-id> bash   =>
cd /tmp/  => run inside ADD container after above bash exec
ls -l   => run inside ADD container
cat 02-catalogue   =>
Now download sample tar file from internet and add inside “Add” folder. Then write Docker Add untar instruction push code and pull inside docker server.
git pull  =>
docker build -t  add:1.0.0 .  =>  To build Docker image. 
docker run -d add:1.0.0 sleep 100   => 
docker exec -it <add-container-id> bash   =>
cd /tmp/  => run inside ADD container after above bash exec
ls -l   => run inside ADD container
cd sample-1/  => run inside ADD container
ls -l   => run inside ADD container
Docker ENTRYPOINT is used to provide the command executed by the container when it is started.
Create “ENTRYPOINT” folder in local machine & create a “Dockerfile” file inside “ENTRYPOINT” folder.
Write Docker instructions of ENTRYPOINT and push code into git then pull inside Docker server.
git pull  =>
cd ../ENTRYPOINT/  =>
docker build -t  entry:1.0.0 .  =>  To build Docker image.
docker run entry:1.0.0   => with CMD
Press ctrl+c => To stop pinging
docker run entry:1.0.0 ping Facebook.com  => with CMD, we can override CMD
Write ENTRYPOINT code in Dockerfile in ENTRYPOINT folder.
git pull  =>
docker build -t  entry:1.0.0 .  => with entrypoint
docker run entry:1.0.0   => with entrypoint
docker run entry:1.0.0 ping Facebook.com  => with entrypoint, now will get error because we can't override ENTRYPOINT command if you try to do so it will go and append. For best results we can always use both CMD and ENTRYPOINT.
docker ps -a —no-trunc  => To display all information.
Now use both CMD & ENTRYPOINT commands in Dockerfile in ENTRYPOINT folder.
git pull  =>
ping google.com ping Facebook.com.
docker build -t  entry:1.0.0 .  =>
docker run entry:1.0.0   =>
docker run entry:1.0.0 facebook.com  => Now this “facebook.com" overrides “google.com" of CMD command then run ping command.
CMD can be used to provide the default args/inputs to ENTRYPOINT.. if we want our own args we can always override CMD at runtime
docker ps   =>
docker exec -it <entry-container-id> bash   => 
Never run container with root access.
exit
Docker USER is used to .
Create “USER” folder in local machine & create a “Dockerfile” file inside “USER” folder.
Write Docker instructions of USER and push code into git then pull inside Docker server.
git pull  =>
cd ../USER/  =>
docker build -t  user:1.0.0 .  =>  To build Docker image.
docker run -d user:1.0.0   => 
docker exec -it <user-container-id> bash   =>
id => inside user-container to check docker roboshop user id after building & running user container.
Docker WORKDIR is used to .
Create “WORKDIR” folder in local machine & create a “Dockerfile” file inside “WORKDIR” folder.
Write Docker instructions of WORKDIR and push code into git then pull inside Docker server.
git pull  =>
cd ../WORKDIR/  =>
docker build -t  work:1.0.0 .  =>  To build Docker image.
docker run -d work:1.0.0 sleep 100  =>
docker exec -it <work-container-id> bash   =>
pwd  => inside work container
exit  => inside work container
Docker CMD instruction not work for change directory so should use WORKDIR to change directory.
git pull =>
docker build -t  work:1.0.0 .  =>
docker run -d work:1.0.0 sleep 100  =>
docker exec -it <work-container-id> bash   =>
pwd  => inside work container
ls -l   => inside work container   
cat hello.txt   => inside work container   
Docker ARG is used to provide variable to Docker image. ARG can be first instruction in a special case to supply the version to base image. ARG variables can't be accessed inside container, they are accessible only at the of image building. ENV variables can be accessed in container.
Create “ARG” folder in local machine & create a “Dockerfile” file inside “ARG” folder.
Write Docker instructions of ARG and push code into git then pull inside Docker server.
git pull =>
cd ../ARG/  =>
docker build -t  arg:1.0.0 .  =>  To build Docker image.
docker images   =>
docker build -t  arg:1.0.0 —build-arg <arg-variable-name>=<version-number> . => syntax
docker build -t  arg:1.0.0 —build-arg version=9 . => example
Add some more arg variables with multi-line docker variables declaration using backslash(\), push & pull code.
docker build -t  arg:1.0.0 —build-arg version=9 . =>
docker run -d arg:1.0.0  =>
docker exec -it <work-container-id> bash  =>
env  => run inside work-container
Add “version is: ${version}” in Dockerfile in ARG folder.
git pull =>
docker build -t  arg:1.0.0 —build-arg version=9 . =>
 ARG variables can't be accessed inside container, they are accessible only at the of image building. ENV variables can be accessed in container.


Diagrams:
1. docker-user


Timestamps:
Interview questions =





Mistakes & Learning:




Doubts Link Clarification AI chat link: