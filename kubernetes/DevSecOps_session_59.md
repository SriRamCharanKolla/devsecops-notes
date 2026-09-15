Tuesday, 14 April 2026

Session 59 - K8 Roboshop(MySQL, Shipping, Payment, Frontend), Nginx as config map.
Class Notes
1. Building images in Docker -> best practices
2. Running inside k8s as manifest files

NodeJs	node_modules + copy source_code -> almost platform(OS) independent
Java -> jar file = compiled code + dependencies -> platform(OS) independent
Python -> dependencies will be installed in python directory -> platform(OS) dependent

1. Install dependencies in different path, so that we can copy into another stage
2. Build stage python needs more packages build-base, linux-headers and pcre-dev
3. run time you need pcre package only


nginx.conf -> value is the nginx.conf file

k8S volumes
===========
1. static provisioning
2. dynamic provisioning

EBS/EFS

1. admin role
2. project role

RBAC -> role based access control

Commands:
git clone “roboshop-docker-git-link”   =>
cd roboshop-docker/mysql   =>
docker login -u <docker-user-name>   =>
docker build -t <docker-user-name>/mysql:1.4.0 .  =>   
docker push <docker-user-name>/mysql:1.4.0  =>
Now write Mysql Kubernetes code then push & pull code.
cd =>
git clone “k8-roboshop-git-link”   => 
cd k8-roboshop  =>
git pull  =>
kubectl apply -f 01-namespace.yaml   =>
kubectl apply -f mongodb/manifest.yaml   =>
kubectl apply -f redis/manifest.yaml   => 
kubectl apply -f catalogue/manifest.yaml   =>
kubectl apply -f user/manifest.yaml   =>
kubectl apply -f cart/manifest.yaml   =>
curl -sS https://webinstall.dev/k9s | bash   => This command should run with New Terminal tab with After connecting with “k8-workstation” AWS EC2 instance server. This package/software is used to monitor k8-clusters deployed and its status monitoring.
kubectl apply -f debug/manifest.yaml   =>
k9s  => Run inside K9s server terminal tab. 
Then enter “namespaces” => Run inside K9s server terminal tab.
These select “roboshop” => Run inside K9s server terminal tab.
Then check running Docker images in K9s server tab.
Select “mysql” then press “s” in the keyboard to go to mysql shell => Run inside K9s server terminal tab.
mysql -u root -pRoboshop@1 => Run inside K9s server terminal tab.
show databases;  => Run inside K9s server terminal tab. Inside mysql.
use cities;  => Run inside K9s server terminal tab. Inside mysql.
show tables;  => Run inside K9s server terminal tab. Inside mysql.   
select count(*) from codes;  => Run inside K9s server terminal tab. Inside mysql.
select count(*) from cities;  => Run inside K9s server terminal tab. Inside mysql.
exit  => Run inside K9s server terminal tab. Inside mysql.
exit  => Run inside K9s mysql shell.
kubectl apply -f shipping/manifest.yaml   =>
Go to “shipping” shell => Run inside K9s server terminal tab.
100m CPU & 126Mi Memory is not enough for Shipping so need to increase and check once, 200m CPU & 256Mi Memory increase.
Push & pull code.
kubectl apply -f shipping/manifest.yaml   =>
Go to “debug” shell => Run inside K9s server terminal tab.
curl http://shipping:8080/health  => Run inside K9s server terminal tab. Inside debug.
exit => Run inside K9s server terminal tab. Inside debug.
Now create “rabbitmq” folder inside “k8-roboshop” repo, then create “manifest.yaml” then write kubernetes code then push & pull code.
git pull  =>
kubectl apply -f rabbitmq/manifest.yaml   => 
Select “debug” => Run inside K9s server terminal tab.
telnet rabbitmq 5672   => Run inside K9s server terminal tab. Inside debug shell.
Then press ctrl+]   => Run inside K9s server terminal tab. Inside debug shell.
quit => Run inside K9s server terminal tab. Inside debug shell.
exit => Run inside K9s server terminal tab. Inside debug shell.
cd ../roboshop-docker/payment  =>
docker build -t <docker-user-name>/payment:1.4.0 .  => 
Optimise Docker payment image to reduce size and make it very light in size.
git pull  =>
docker build -t <docker-user-name>/payment:1.4.0 .  =>
docker images =>
Now create “payment” folder inside “k8-roboshop” repo, then create “manifest.yaml” then write kubernetes code then push & pull code.
Now go to “robodhop-documentatiom” verify payment service file & configuration setup.
cd ~/k8-roboshop/   =>
git pull  =>
kubectl apply -f payment/manifest.yaml   => 
docker push <docker-user-name>/payment:1.4.0  =>
Go to “payment” shell => Run inside K9s server terminal tab.
Select “describe” “payment”  => Run inside K9s server terminal tab. Inside payment.
Got error in payment
git pull  =>
kubectl apply -f payment/manifest.yaml   =>
Select “payment” => Run inside K9s server terminal tab.
ls -l  => Run inside K9s server terminal tab. Inside payment.
cd install  => Run inside K9s server terminal tab. Inside payment.
cd /usr/local/  => Run inside K9s server terminal tab. Inside payment.
ls -l  => Run inside K9s server terminal tab. Inside payment.
cd bin/  => Run inside K9s server terminal tab. Inside payment.
ls -l  => Run inside K9s server terminal tab. Inside payment.
cd => 
cd roboshop-docker/payment  =>
Here the issue is in “alpine” group ids are starting from 100 but we given 1001 that’s why got issue.
git pull  =>
docker build -t <docker-user-name>/payment:1.4.0 .  =>
docker push <docker-user-name>/payment:1.4.0  =>
cd ~/k8-roboshop/   =>
git pull  =>
kubectl apply -f payment/manifest.yaml   =>
Go to “debug” shell => Run inside K9s server terminal tab.
curl http://payment:8080/health  => Run inside K9s server terminal tab. Inside debug shell.
exit => Run inside K9s server terminal tab. Inside debug shell.
Now optimise Docker frontend image to reduce to very minimal and lightweight.
cd ../roboshop-docker/frontend  =>
git pull  =>
docker build -t <docker-user-name>/frontend:1.4.0 .  =>
docker images => Check frontend Docker image size.
docker push <docker-user-name>/frontend:1.4.0  =>
Now create “frontend” folder inside “k8-roboshop” repo, then create “manifest.yaml” then write kubernetes code then push & pull code.
kubectl apply -f frontend/manifest.yaml   =>
Go to “debug” shell => Run inside K9s server terminal tab.
curl http://frontend:80  => Run inside K9s server terminal tab. Inside debug shell.
exit => Run inside K9s server terminal tab. Inside debug shell.
Then go to services in K9s server then Unser frontend copy DNS and test DNS inside browser like http://Enter-DNS-provides by AWS EKS.
Now register as a user in Roboshop app then place order and check everything is working or not.
Now implement nginx config with Kubenetes configMap as a file with “|” then mount in desired location in the pod and make it readOnly, so that every time changing the nginx config location no need to rebuild Docker image.
git pull  =>
kubectl apply -f frontend/manifest.yaml   =>
 Got error because not added “subPath” in the fronted “manifest.yaml” file while setting up nginx config file from Kubernetes configMap and while mounting in the container.
 Add subpath in frontend/manifest.yaml file 
 git pull  =>
 kubectl apply -f frontend/manifest.yaml   =>
 eksctl delete cluster --name=roboshop --region=us-east-1  => Command to delete Kubernetes Cluster.
 eksctl delete cluster -f eks.yaml --force   => Command to force delete Kubernetes Cluster.



Diagrams:
k8-setup
nginx-conf 


Diagrams:
k8-roboshop
roboshop-docker

Timestamps:
Interview questions





Mistakes & Learning:




Doubts Link Clarification AI chat link: