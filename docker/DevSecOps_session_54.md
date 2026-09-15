Sunday, 5 April 2026

Session 54 - Docker volumes(Un-named volumes, Named volumes), Docker best practices.
Class Notes
1. Build image
	best practices implement
2. Run image
	best practices implement

Docker architecture
===================
1. docker checks image is available in local or not
2. if available it will run
3. if not available, it will pull from the hub
4. creates container out of it
5. send the output to user

Docker client == Docker CLI
Docker deamon == docker software running

docker run nginx -> checks whether nginx is available in local
if available creates container out of it and show to client

if not available pull from hub, store it in local, create container and show to client

volumes
========
1. un-named volumes -> managed by us
2. named volumes -> use docker commands to create and manage the container volumes

Docker best practices
======================
1. Use Minimal Base Images. alpine images are minimal, but use official
	node:20 built on ubuntu/debian
2. Use Multi-Stage Builds
	multi stage builds have multiple FROM instructions, basically looks multiple Dockerfiles. One Dockerfile is used to provide the artifact. we can copy the artifact into run time image... when you are installing dependencies there might be cache, dev related tools, etc are installed. those are not required for running into production..
	
	node artifact = code + node_modules
	java = .jar file
	
3. use dockerignore -> don't un necessary files/folders
4. Don’t Run as Root
5. Avoid using latest -> we will not be sure which version actually running
6. Use volumes if containers are stateful, prefer stateless apps inside containers not stateful
7. Use ENV variables at runtime instead of build time.. no need to rebuild the image
8. Health Checks
9. Scan Images for Vulnerabilities
10. Use Proper Logging
11. Limit Container Resources
12. Use ENTRYPOINT + CMD Correctly
13. Version and Tag Images Properly prefer semantic version
14. Optimise layers
15. Use secrets, dont hardcodes creds

1.0.0 -> major-version.minor-version.patch-version

20
20.1
20.2
20.3 -> released public
20.3.1 -> 20 major-version 3 minor 1 patch-version

Docker disadvantages to run images
==================================
1. you can't rely on single docker host
2. if run multiple hosts, there needs to be some orchestrator
3. container to container communication is difficult if we run in docker
4. if there is load on application, there should be load balancing
5. autoscaling is not there
6. networking...
7. where to store secrets?
8. volumes implementation is not good

Docker swarm, kubernetes, mesos, etc..

EKS, AKS, GKE, Openshift
Commands:
git clone “roboshop-docker-git-url”   =>
docker compose up -d  => This command will directly pull docker images from docker hub.
df -hT  =>
sudo su -  =>
lsblk  =>
Due to Higher Docker images sizes server storage is not enough, so create new server with 50GB then extend storage.
Connect to Docker Server.
docker run -d -p 80:80 nginx  =>   
docker exec -it <docker-image-id> bash  =>
cd /usr/share/nginx/html/  => Run inside nginx docker image.
echo “hi” > hi.html  =>  Run inside nginx docker image.
Now go to browser then enter “http://<docker-server-ip>/hi.html”. Now check hi text should display as we already given this file to nginx.
exit  =>  Run inside nginx docker image.
docker rm -f <docker-image-id>  =>
docker run -d -p 80:80 nginx  => 
docker exec -it <docker-image-id> bash  => 
cd /usr/share/nginx/html/  => Run inside nginx docker image.
ls -l  => Run inside nginx docker image.
docker rm -f <docker-image-id>  =>
pwd   =>
mkdir nginx-data   =>
cd nginx-data   =>
echo “<h1>Hi, from volumes</h1>” hi.html   =>
ls -l  =>
echo “<h1>Hi, from volumes</h1>” > hi.html   =>   
ls -l  =>
echo “<h1>Hi, from volumes</h1>” > index.html   =>   
ls -l  =>
cd  =>
docker run -d -p 80:80 -v /home/ec2-user/nginx-data:/usr/share/nginx/html nginx  =>  -v is for Docker volume, “/home/ec2-user/nginx-data” is source & “:/usr/share/nginx/html” is destination giving for nginx.
docker rm -f <docker-image-id>  =>
ls -l  =>
cd nginx-data   =>
ls -l  =>
cd  =>
docker run -d -p 80:80 -v /home/ec2-user/nginx-data:/usr/share/nginx/html nginx  =>
un-named volumes => managed by us
docker volume <volume-name> =>
docker volume nginx =>
docker volume create <volume-name> =>
docker volume create nginx =>
docker rm -f <docker-image-id>  =>
docker volume ls =>
docker volume inspect nginx =>
named volumes => use docker commands to create and manage the container volumes
docker run -d -p 80:80 -v nginx:/usr/share/nginx/html nginx  =>
sudo su -  =>
cd /var/lib/docker/volumes/nginx/_data   =>
ls -l  =>
echo “<h1>Hello from named volumes</h1>” > hello.html   =>
Check in Browser.
exit  =>
docker ps  =>    
docker rm -f <docker-image-id>  =>
sudo su -  =>
cd /var/lib/docker/volumes/nginx/_data   =>
ls -l  =>
exit  =>
cd  =>
Always Docker Named volumes are better suggestion because managing is tough job.
Now add volumes docker code in “docker-compose.yaml” file in “roboshop-docker”.
To find the Docker images data storage location go to Docker Hub check data storage location of a required image by visiting Docker official image documentation in “overview” tab in Docker Hub like for nginx, mysql, redis etc.
git clone “roboshop-docker-git-llink”   =>  
cd roboshop-docker  =>
docker compose up -d  =>
docker images =>
Now in Browser check roboshop application then place an order
docker ps  =>
docker logs shipping  =>
docker compose down  =>
docker compose up -d  =>
Now in Browser roboshop application by login into the application with previously created user credentials before docker compose down then it should displayed previously placed order.
In Enterprise application no one will keep docker data in docker container volumes.
docker images -a =>
docker pull node:20 =>
docker images =>
docker pull alpine => “alpine” is one of the linux os.
docker images  =>
Now do changes in catalogue Dockerfile with “alpine” to reduce node size to very minimal. Push & pull code.
docker compose build catalogue =>
docker images => now check “catalogue” DISK USAGE like 344MB.
docker compose up -d  => After reducing size should run this command to rebuild catalogue container gain.
Now do changes in catalogue Dockerfile to implement multi stage build. Push & pull code.
docker compose build catalogue =>
docker images => now check “catalogue” DISK USAGE like 246MB.
docker compose up -d  =>
Under “shipping” folder create “.dockerignore” files/folders to ignore unwanted files, in shipping we don’t need “db” so we should keep in “.dockerignore” file.
docker run -d -p 8080:80 -v /:/nginx-root nginx  =>
docker exec -it <docker-image-id> bash  =>
cd /nginx-root/   => run inside nginx after above bash exec.
ls -l   => run inside nginx after above bash exec.
Now create group & user then assign user to group as per “alpine”(linux os) OS commands which we used to build catalogue to reduce image size. Push & pull code.
docker compose build catalogue =>
docker compose up -d  =>
docker exec -it catalogue sh  =>
ls -l   => run inside catalogue. Check now everything under roboshop user then check once functionality also.


Diagrams:

Timestamps:
Interview questions = 01:11:00





Mistakes & Learning:




Doubts Link Clarification AI chat link: