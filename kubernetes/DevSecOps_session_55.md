Sunday, 5 April 2026

Session 55 - K8 setup, Layers in docker, Layer optimisation.
Class Notes
mongodb
redis
mysql
rabbitmq

logs
metrics

1. dont keep stateful apps in containers
2. if we want we can keep not end user stateful apps

shell vs ansible

with in docker host we are using bridge network


autoscaling, networking, storing secrets, managing volumes is difficult in docker swarm, heavy applications are not suitable

we need kubernetes

workstation
===========
1. Install docker -> build the images, push the images to ECR repo hub
2. eksctl -> create, update and delete the eks cluster
3. kubectl -> connect with eks cluster
4. aws cli (aws configure) -> to provide authentication while creating cluster, connecting to cluster


sudo growpart /dev/nvme0n1 4
sudo lvextend -L +30G /dev/RootVG/varVol
sudo xfs_growfs /var

sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl


curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

spot instances
===============
1TB RAM, 200TB HD

40GB RAM, 80TB HD -> not in use
70-80% discount they give us if we use this..
AWS will take out these spot instances with 2 min notice...

eksctl create cluster --config-file=eks.yaml

DOCKER_BUILDKIT=0 docker build -t user:1.0.0 --rm=false --no-cache .

FROM node:20 -> create container from it
login to that intermediate container

run this instruction
WORKDIR /app

create image out of this container -> image-abcfd1234

create container from abcd1234 -> container id is container-xyz1234

xyz1234 -> run next instruction COPY package.json . -> image-dfd5346

create container from image-dfd5346 called container-fadh1632

docker layers
=============
docker creates intermediate container for every instruction
it will run next following instruction inside this container -> create image out of it
then it creates another intermediate container from that image, run the next instruction -> create image out of it

so for every instruction new layers are created -> docker exports all these layers into hub while pushing the image
so for every instruction new layers are created -> docker exports all these layers into hub while pushing the image

we must keep frequently changing instructions at the bottom of Dockerfile to make the build process fast and storage

Commands:
Create “workstation” AWS EC2 instance with 50GB storage and use remaining 30GB by run running linux commands. Then install Docker in the instance then install kubectl in the instance.
Connect to “workstation” AWS EC2 instance.
sudo growpart /dev/nvme0n1 4  =>
sudo lvextend -L +30G /dev/RootVG/varVol  =>
sudo xfs_growfs /var =>
sudo dnf -y install dnf-plugins-core =>
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo =>
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y  =>
sudo systemctl start docker =>
sudo systemctl enable docker  =>
sudo usermod -aG docker ec2-user  =>
ARCH=amd64  =>
PLATFORM=$(uname -s)_$ARCH  =>
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"  =>
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz  =>
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl  =>
eksctl version  =>
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl  =>
chmod +x ./kubectl  =>
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH  =>
kubectl  version --client  =>
aws configure  =>
Then give AWS Configure keys.
Create “eksctl” git repo & clone it in your system then create “eke.yaml” file inside “eksctl”.
Write eks cluster creation code and push changes into “eksctl” git repo.
Clone “eksctl” git repo in “workstation” AWS EC2 instance.
cd eksctl/    =>
eksctl create cluster --config-file=eks.yaml   => Run inside Kubernetes workstation server.
After running cluster creation command then it will take 15 minutes time to create cluster.
Now connect with “docker” AWS EC2 instance.
Then clone “roboshop-docker” repo in AWS EC2 instance.
cd roboshop-docker/  =>
cd catalogue/   =>
docker build -t catalogue:1.0.0 .  =>
cd ../user/    =>
DOCKER_BUILDKIT=0 docker build -t user:1.0.0 --rm=false --no-cache .  => Command to see with Docker Layers.
cd  =>
Now clone “dockerfiles” repo in docker AWS EC2 instance.
git clone “<git-dockerfiles-repo>   =>
cd dockerfiles/  =>
cd FROM/   =>
docker build -t layer-demo .
docker push layer-demo  =>
docker images  =>
docker tag layer-demo <docker-user-name>/layer-demo  =>
docker push <docker-user-name>/layer-demo  =>
cd ..  =>
cp FROM test   =>
cd test/  =>
docker push <docker-user-name>/test  => Now check you can find “<container-id>:Mounted from <docker-user-name>/layer-demo” 
docker push <docker-user-name>/test  => Now check you can find “<container-id>: Layer already exits”
Docker check pushed DockerHub image content then if any image exits with same content then it will not create new memory for new image with same content instead it will refer available image content for the currently pushing image.
docker images  =>
docker tag user:1.0.0 <docker-user-name>/user:1.0.0  =>
docker push <docker-user-name>/user:1.0.0  =>
Do Docker code changes in user Dockerfile to enhance user Docker image as per reusable layering technique.
cd ../../roboshop-docker/
git pull  =>
cd user/  =>
docker build -t <docker-user-name>/user:1.0.0 .  =>
Do code changes in any .js file then push code.
git pull  =>
docker build -t <docker-user-name>/user:1.0.0 .  => Now check only .js files layer only update in user:1.0.0 Docker image.  
docker push <docker-user-name>/user:1.0.0  => Some layers already exits so Docker not allowing to push this image so do changes agin in any .js file then push code.
git pull  =>
docker build -t <docker-user-name>/user:1.0.0 .  =>
docker push <docker-user-name>/user:1.0.0  => check output now able to push image but not some layers are already exits.
Now do a changes in user image Dockerfile changes “WRKDIR /app” to “WRKDIR /app1” then push changes.
Now go to Kubernetes “workstation" server then check Kubernetes cluster is created or not.
kubectl get nodes  => Run inside workstation server. 
eksctl delete cluster --name=roboshop --region=us-east-1  => Command to delete Kubernetes Cluster.
eksctl delete cluster -f eks.yaml --force   => Command to force delete Kubernetes Cluster.



Diagrams:
docker-network
k8-setup
docker-layer


Timestamps:
Interview questions





Mistakes & Learning:




Doubts Link Clarification AI chat link: