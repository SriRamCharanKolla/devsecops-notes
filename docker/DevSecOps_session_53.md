Sunday, 5 April 2026

Session 53 - Docker compose, Roboshop docker completed.
Class Notes
1. Build the image
2. Run the image, before running the image we need to make sure custom bridge network is created. 

there are dependencies between components

docker compose -> YAML file
-> It can build the images
-> It can define the dependencies between the components
-> It can starts all the components at a time
-> it can remove all the components at a time
-> you can define network and volume also here

It will make our job easy and accurate

docker run -d --network roboshop --name mongodb joindevops/mongodb:1.0.0 

docker compose up -d --build

by default, docker containers are ephemeral. once we remove container it will remove entire data

we need to use volumes to keep the data even containers are removed.

Commands:
Docker Compose is a YAML file. It can build the images. It can define the dependencies between the components. It can starts all the components at a time. It can remove all the components at a time. You can define network and volume also here. It will make our job easy and accurate.
Create “docker-compose.yaml” file in “roboshop-docker” then write Docker Compose yaml code then push code and pull inside docker server.
Connect to Docker Server.
git clone “roboshop-docker-git-link”   =>
docker compose up -d  => Create docker compose with all Containers.
docker ps   =>
docker compose down  => Remove Docker Compose for all containsers
Similarly write Docker compose code for Redis & other containers.
Push & pull code.
docker compose up -d  =>
Check in browser with Docker server IP, not working because port numbers not provides.
Now provide port number for frontend in Docker Compose file. Push & pull code.
docker rm frontend  =>
docker compose up -d  =>
Now check in browser frontend is running or not.
docker ps   => To check frontend is running
docker ps -a  =>
docker logs frontend   =>
docker compose up -d —build   =>
ls -l  =>
cat mongodb/Dockerfile  =>
May to due to Dockerfile path issue in Docker Compose File.
git pull => 
docker compose build   => To create Docker images.
docker compose up -d  =>  To create Docker Containers from docker images.
Now refresh frontend in browser and then check. Now Catalogue images are not showing in the app.
docker logs catalogue   =>
docker ps  =>
docker ps -a  =>
docker logs mongodb   =>
Unable to find mongodb issue, so try with running mongodb manually.
docker run -d —network roboshop —name mongodb <your-docker-hub-name>/mongodb:1.0.0  =>
docker rm mongodb  =>
docker run -d —network roboshop —name mongodb <your-docker-hub-name>/mongodb:1.0.0  =>
docker ps  =>
docker compose down  =>
docker rm -f mongodb  =>
docker compose down  =>
docker images  =>
docker rmi -f `docker images -aq`  => Removing all images once to do cleanup.
docker compose up -d   => 
Actually Docker images are not building its pulling only because of this facing issue.
docker compose down  =>
docker rmi -f `docker images -aq`  =>
docker compose up -d —build  => This command will build everything.
docker compose build <componet-name>   => To build specific component.
docker compose build frontend   =>
Now add cart in compose file then push & pull code.
docker compose build cart  =>
Now in “nginx.conf” file change cart location from localhost to cart. Then push & pull code.
docker compose build frontend   =>
Now browse roboshop frontend app then login/register & then check.
Getting Error in Cart, check logs.
docker logs frontend   =>
docker logs cart  =>
docker exec -it frontend bash  =>
cat /etc/nginx/nginx.conf   => Run inside frontend. Cart location is still in localhost because of this getting error.
cd /etc/nginx/   => Run inside frontend.
Git pull  =>
docker compose build frontend   =>
docker compose up -d  =>
Now create “mysql” folder then create “Dockerfile” then go to Robosho documentation to get required information about MySQL configuration & Docker code writing for required MySQL image version and copy MySQL db folder and paste in mysql folder, “schema.sql” file is not required delete the file and write docker code for user docker image then push changes and pull changes inside docker server.
git pull =>
docker compose build mysql  =>
docker compose up -d  =>
docker ps  =>
Now create “shipping” folder then create “Dockerfile” then go to Robosho documentation to get required information about Shipping configuration & Docker code writing for required Shipping image version and copy db, src folders & pom.xml file and paste in shipping folder and write docker code for user docker image then push changes and pull changes inside docker server.
git pull => 
docker compose build shipping =>
Got error due to src copy path should ./src not .
git pull  =>
docker compose build shipping => 
docker compose build frontend   =>
docker compose up -d  =>
Now check application shipping is working fine or not.
Now write compose code for rabbitmq then push & pull code.
docker compose build rabbitmq  =>
git pull =>
docker compose up -d  =>
Now create “payment” folder then create “Dockerfile” then write docker code for user docker image then push changes and pull changes inside docker server. 
git pull  =>
docker compose build payment =>
git pull  =>
docker compose build frontend => 
docker compose up -d  =>
docker login -u <docker-user-name>  => To push all composed container images into Docker Hub.
docker compose push  => Pushing images.
docker compose down  =>
docker rmi -f `docker images -aq`  => Removing all images.
docker images => 
docker compose up -d  => Now no need to build images again, we can directly run this command to directly deploy & run application with Docker container images.
docker images => Now the size of the images are high we need to reduce size with Docker Volumes.
By default, docker containers are ephemeral. once we remove container it will remove entire data.
we need to use volumes to keep the data even containers are removed.
docker compose restart frontend   =>
docker compose up -d --build frontend  =>




Diagrams:


Timestamps:
Interview questions
QA = 





Mistakes & Learning:




Doubts Link Clarification AI chat link: