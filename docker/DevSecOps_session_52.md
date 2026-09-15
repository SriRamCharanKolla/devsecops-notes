Sunday, 5 April 2026

Session 52 - Dockerfile(ONBUILD), Docker networking(Mongodb, Catalogue, Frontend, User, Cart).
Class Notes
FROM -> refer base OS
RUN -> to configure image
CMD -> command executed at the time of container creation
LABEL -> metadata of the image
EXPOSE -> port information
ADD -> same as COPY but 2 extra capabilities. can download content directly from internet and untar the file
COPY -> copy the files from local to image
USER -> to mention the user that container should be running on
WORKDIR -> working directory of the image/container
ENV -> env variables to the container
ARG -> variables at build time, cant be accessed inside container. ARG can be first instruction before 	FROM to supply the version
ENTRYPOINT -> command to be executed while creating container. it cant be overridden, if we try it will go append. CMD can be overridden. CMD can supply default args for ENTRYPOINT. we can always override default args

ONBUILD
=======
our image can be used by someone else. but we want to put some conditions to use our image. that can be enforced using ONBUILD instruction..

our nginx image expects index.html while it is used by others.

OS -> 8,9,10, etc..
configurations -> packages update
programming languages -> nodejs 20,21, etc..

Ansible

docker uses by default bridge network
default bridge network containers can't communicate with each other..

you should create your own bridge network


roboshop bridge: 172.18.0.1
172.18.0.2
172.18.0.3

when you install docker, it creates one bridge network with name also as bridge
by default containers get IP address from this network with name bridge
if you use default bridge network you cant communicate between the containers using names
you should create your own bridge network


Commands:
ONBUILD is used to our image can be used by someone else. but we want to put some conditions to use our image. that can be enforced using ONBUILD instruction.
Create “ONBUILD” folder in local machine & create a “Dockerfile” file inside “ONBUILD” folder.
Write Docker instructions of ONBUILD and push code into git then pull inside Docker server.
git pull  =>
cd ../ONBUILD/  =>
docker build -t  onbuild:1.0.0 .  =>  To build Docker image.
docker build -t  nginx:1.0.0 .  =>
Create “test” folder inside “ONBUILD” folder in local machine & create a “Dockerfile” file inside “test” folder.
Write Docker instructions to test ONBUILD and push code into git then pull inside Docker server.
git pull  =>
cd ../test/  =>
docker build -t  onbuild-test:1.0.0 .  =>
Create “index.html” inside “test” folder then write HTML code then push code and pull changes inside “server”.
docker build -t onbuild-test:1.0.0 .  =>
Create “roboshop-docker” git repo then clone it in your local machine then create “mongodb” then create “Dockerfile” then write docker code push code.
Git clone “roboshop-docker” repo inside docker AWS EC2 instance server.
cd roboshop-docker   =>
cd mongodb   =>
docker build -t  mongodb:1.0.0 .  => 
docker images =>
docker run -d —name mongodb mongodb:1.0.0     =>
docker ps   => To check mongodb is running with Docker or not. 
Now create “catalogue” folder inside “roboshop-docker” folder then create “Dockerfile” then write docker code and push code.
docker inspect mongodb   =>
docker ps   =>
git pull => run inside docker server.
cd ../catalogue/   =>
docker build -t  catalogue:1.0.0 .  =>     
docker run -d —name catalogue catalogue:1.0.0     =>
docker ps   =>
docker exec -it catalogue bash  =>
curl http://localhost:8080/health  => Run inside catalogue. Here mongodb is not connected with catalogue because mongodb & catalogue are using Docker default bridge network, should create own network for the project then should connect mongodb & catalogue to the created network.
exit   => Run inside catalogue.
docker logs catalogue   => To check catalogue logs, to find error in logs.
ifconfig   => To check ip information.
docker inspect mongodb   =>
docker inspect catalogue   =>  
docker network ls   =>
Docker used by default bridge network.
docker run —network host nginx   => To run nginx with Docker Host network.
docker run —network host -d nginx   => Running in background.
docker inspect <nginx-docker-id>  =>
docker network create roboshop   =>
docker network ls   =>
docker network disconnect bridge mongodb   =>
docker network disconnect bridge catalogue  =>
docker network connect roboshop mongodb   =>
docker network connect roboshop catalogue   =>
ifconfig   => To check ip information.
docker inspect mongodb   => To check mongodb IP address.
docker inspect catalogue   => To check catalogue IP address.
docker exec -it catalogue bash  =>
curl http://localhost:8080/health  => Run inside catalogue. Now mongodb is connected with catalogue.
exit   => Run inside catalogue.
when you install docker, it creates one bridge network with name also as bridge by default containers get IP address from this network with name bridge
If you use default bridge network you cant communicate between the containers using names.
You should create your own bridge network
Now create “frontend” folder then create “Dockerfile” then write Docker code and go to Roboshop documentation repo then copy “nginx.conf” code then create “nginx.conf” paste nginx.conf copied code do required code changes and created “static” folder then copy and paster all Roboshop frontend code files and push code into git.
git pull  =>
cd ../frontend/   =>
docker build -t  frontend:1.0.0 .  =>
docker run -d -p 80:80 —network roboshop —name frontend frontend:1.0.0  =>  May get error because we already running an image so remove it
docker ps   => To check running images with IDs
docker rm -f <docker-image-id>   =>
docker run -d -p 80:80 —network roboshop —name frontend frontend:1.0.0  => Docker image name should be unique.
docker ps -a  =>
docker rm frontend  => Already created frontend image, so removing it and creating new frontend.
docker run -d -p 80:80 —network roboshop —name frontend frontend:1.0.0  =>
docker ps   =>
Now check in browser can able to load frontend app.
docker logs frontend   => To check frontend logs, to find error in logs.
docker exec -it frontend bash  =>
cat /etc/nginx/nginx.conf   => Run inside frontend.
cd /etc/nginx/   => Run inside frontend. 
ls -l   => Run inside frontend.
cd conf.d/   => Run inside frontend.
ls -l   => Run inside frontend.
cat default.conf    => Run inside frontend. Here not removed default.config should remove default.conf.
Do docker code changes to remove default.conf file then push code and pull inside docker server.
git pull  =>
docker build -t  frontend:1.0.0 .  => If any changes did inside docker image code then should build image again.
docker rm -f frontend  =>
docker run -d -p 80:80 —network roboshop —name frontend frontend:1.0.0  =>
Now check Catalogue components will be shown in the application.
Now create “user” folder then create “Dockerfile” then write docker code for user docker image and copy roboshop use component code files and paste in “user” folder and check Roboshop documentation repo & write “user” micro service/component code then push changes and pull changes inside docker server.
docker run -d —name redis —network roboshop redis:7   =>
docker ps   =>
Now create “cart” folder then create “Dockerfile” then go to Roboshop documentation git repo to follow configuration and to get cart code files and write docker code for user docker image then push changes and pull changes inside docker server.
git pull  =>
cd ../user/  => 
docker build -t user:1.0.0 . =>
cd ../cart/  =>
docker build -t cart:1.0.0 . =>
docker run -d —name user —network roboshop user:1.0.0  =>
docker run -d —name cart —network roboshop cart:1.0.0  =>
docker ps  =>  
Now in “frontend” docker image then in “nginx.conf” file for user & cart remove localhost and add user & cart as its locations.
Push code and pull changes inside docker server.
git pull  =>
docker build -t  frontend:1.0.0 .  =>
docker rm -f frontend  =>
docker run -d —network roboshop —name frontend frontend:1.0.0  =>
Now refresh frontend page in browser and check, may page not loaded because port not provided.
docker ps   =>
docker run -d -p 80:80 —network roboshop —name frontend frontend:1.0.0  =>
Now check the frontend should work.
docker images  => Check images DISK USAGE is very high we reduce images DISK USAGE size with the help of Docker best practices. 




Diagrams:
docker-network  


Timestamps:
Interview questions
QA = 





Mistakes & Learning:




Doubts Link Clarification AI chat link: