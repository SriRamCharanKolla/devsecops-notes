Wednesday, 6 May 2026

Session 70 - Init containers, Fetching secrets from secret manager, Networking in EKS, Network Policy, PDB and TopologySpreadConstraint.
Class Notes
1. recreate
2. rolling update -> 2 versions running at same time
3. blue/green -> double cost, we can reduce replicas to 0
4. canary -> send traffic to diff version randomly, slowly ramping up from 0 to 100%
5. A/B testing -> sending some traffic to other version based on location, headers, etc
6. shadow -> v2 is running but not accepting live traffic, we mock live traffic live traffic from v1 to ve, but we dont send response to end user

switching route

blue
main service

green -> preview service -> endpoint testing

switch main service to green

1. upgrade control plan to next version, keeping blue current version
2. upgrade add-ons
3. create green ng with new version
4. cordon the blue node group, scheduling disabled
5. drain the blue node, workloads will go to green
6. remove blue ng

init containers
================
we must secrets in secretsmanager
MySQL pod, MYSQL_ROOT_PASSWORD is required to setup

we can use init containers, these are containers that before main container runs. it prepares the env for main container like fetching secrets, checking the dependency services, running some sql scripts

camping -> main
tent -> init container

init container runs to completion, you can have multiple init containers, they will be executed sequentially


1. we should have IAM policy that can fetch the secrets
2. create service account with this policy, IAM role will be created
3. run the pod with this service account.
4. create init container that fetches secret and make it available in certain location -> containers in the can share storage
5. main container reads it from shared location, export it as env variable and remove that file

/secrets/mysql-root-password.txt -> this is where init container fetches and stores the secret

arn:aws:iam::160885265516:policy/RoboshopMYSQLRootPasswordReader

eksctl create iamserviceaccount \
  --name mysql-secret-reader \
  --namespace roboshop \
  --cluster roboshop-dev \
  --attach-policy-arn arn:aws:iam::160885265516:policy/RoboshopMYSQLRootPasswordReader \
  --role-name "RoboshopMySQLReader" \
  --approve
  
aws secretsmanager get-secret-value --secret-id roboshop/dev/mysql_root_password --query SecretString --output text | jq -r .MYSQL_ROOT_PASSWORD

Networking
===========
VPC handles EKS networking, it treats pods as first class citizens. it allocates pod IP in VPC CIDR range...

Network interface
===========
How your system gets IP address. Ethernet, WiFi, Mobile Data, etc...
c3.large -> 3 ENI each interface can give 10 Ip = 30
1 Ip addres is for primary
29 secondary IP addresses we can have for pods

How many pods we are planning -> 100 pods

4 nodes

c3.xlarge -> 2

NetworkPolicy

1. shouldn't allows pods from differnet namespace. default deny for every pod in roboshop namespace

aws eks update-addon --cluster-name roboshop-dev --addon-name vpc-cni \
--configuration-values '{"enableNetworkPolicy": "true"}'

mongodb should accept connections from catalogue
mongodb should accept connections from redis

Pod Disruption Budget
=====================
1 pod in some node

if node down -> app down

node upgrade
cordon
drain
autoscaling -> nodes will be deleted

in voluntary disruptions pods may go down

we can define max how many number of pods can be unavailable

2 pods -> 1 pod can go to unavailable

2 nodes

node-1 -> pod-1 -> success
node-2 -> pod-2

node-2 -> pod-1 and pod-2
before draining PDB comes into action...
drain will be failed

node-3 -> pod-2 should come here

single pod replica -> 25% of pods 
maxUnavailable -> 25%
minAvailable -> 25%

4 pods -> 1 pod should be always available

1 pod ->25% max unavailable -> 0.25 pods should be always available -> 0 pods unavailable
2 pods -> 25% max unavailable -> 0.5 -> 0 pods unavailable
3 pods -> min available 25% -> 0.75 -> 0 pods can be available -> can have downtime
4 -> 1 pod can go available
0.anything -> 0

10 pods -> 2

4 nodes

node-1 -> 4 pods -> 6 pods are available -> success

2 nodes
=======
node-1 -> 9 pods -> cant drain

PDB
TopologySpreadConstraints -> spreding pods across nodes based on topologyspread constraint

host as topology(border)
single AZ
2 nodes
NODE-1 ->  1 pod is here
NODE-2 -> 2 pod is here

us-east-1a and us-east-1b

1 pod in us-east-1a az and another pod in us-east-1b
topology is zone
host level topology

us-east-1a-> 2 nodes and us-east-1b-> 2nodes

1st topology is zone -> k8s spreads pods across multi AZ
2nd topology is host -> 2 pods in same az spread across multi hosts

exec /bin/node /app/server.js

Commands:
Ghgdhghd
In AWS search for “Secrets Manager” then click on it click on “roboshop-dev/MySQL-Root-Password”. We need to create an IAM policy for this.
Go IAM then go to “Policies” then click on “Create policy” then select “Secrete Manager” for “Select a service” then select “GetSecretValue” for “Read” under “Access level” then select “Specific” for “Resources” then click on “Add ARN” then enter “us-east-1” in “Resource region” then enter “roboshop/mysql_root_password-BGanno” then enter “mysql_root_password-secrete-manager-ARN” in “Resource ARN” then click on “Add ARNs” then click on “Next” then enter “RoboshopMySQLRootPasswordReader” for “Policy name” then click on “Create policy” then in Policies listing page search with “RoboshopMySQLRootPasswordReader” then copy its ARN.
aws eks update-kubeconfig —region us-east-1 —name roboshop-dev   =>
kubectl get nodes     =>
]eksctl create iamserviceaccount \ --name mysql-secret-reader \ --namespace roboshop \ --cluster roboshop-dev \ --attach-policy-arn arn:aws:iam::160885265516:policy/RoboshopMYSQLRootPasswordReader \ --role-name "RoboshopMySQLReader" \ --approve     =>
kubectl get sa mysql-secret-reader -n roboshop -o yaml    =>
Create new GitHub Repo “k8s-init”.
git clone https://github.com/daws-88s/k8s-init.git     => Run on your local system.
Create “Dockerfile” file in “k8s-init” repo then write Docker code in this file.
Create “secrete-reader.sh” file in “k8s-init” repo then write Shell Scripting code in this file.
cd k8s-init/    => Run on your local system.
cp ../roboshop-docker/mysql/db .     =>  Run on your local system.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
git clone https://github.com/daws-88s/k8s-init.git     =>
cd k8s-init/    =>
ls -l   =>
docker build -t <Docker-hub-user-name>/mysql:1.5.0 .     =>
docker login -u <Docker-hub-user-name>    =>    
docker push <Docker-hub-user-name>/mysql:1.5.0     =>
Create “manifest.yaml” file in “k8s-init” repo then write Ansible code in this file.
aws secretsmanager get-secret-value --secret-id roboshop/dev/mysql_root_password --query SecretString --output text | jq -r .MYSQL_ROOT_PASSWORD   => Run on your local system.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
git pull   =>
Take new Terminal Tab then again connect to “Bastion” to connect to “K9s” to monitor Kubernetes resources. Run “k9s” command then search for “namespace” then select “roboshop” .
kubectl apply -f manifest.yaml     =>
Now verify in K9s mysql resource is created or not. Then click on “mysql” then verify the      Kubernetes able to access and place MySQL secrete key.
Now in K9s check mysql pod status, if it is running then click on it and then run “mysql -u root -pRoboShop@1” then check can able to connect with “mysql” Database or not. Then run “show database;” then run “use cities” then run “show tables;” then run “exit” to logout from mysql then agin run “exit” to came out from “mysql” pod.
Networking:  In Kubernetes VPC handles EKS networking, it treats pods as first class citizens. it allocates pod IP in VPC CIDR range.
Network interface:   How your system gets IP address. Ethernet, WiFi, Mobile Data, etc.  c3.large -> 3 ENI each interface can give 10 Ip = 30, 1 Ip addres is for primary, 29 secondary IP addresses we can have for pods.
Create “k8-network” repo in GitHub.
git clone https://github.com/daws-88s/k8-network.git     =>  Run on your local system.
Create “default.yaml” & “roboshop.yaml” files in “k8-network” repo then write required code.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
cd ..     =>
git clone https://github.com/daws-88s/k8-network.git     =>
cd k8-network.     => 
kubectl apply -f default.yaml     =>
kubectl apply -f manifest.yaml     => 
Now in K9s select default then run “curl <roboshop-pod-IP>” then run “exit”.
NetworkPolicy:   shouldn't allows pods from differnet namespace. default deny for every pod in roboshop namespace.
Create “deney-default.yaml” file in “k8-network” repo then write Kubernetes network policy code like deny all incoming requests to allow specific ports or IP addresses. 
git pull    =>
kubectl apply -f deney-default.yaml     =>
Now in K9s go to nginx then run “curl <roboshop-pod-IP>” then run “exit”.
kubectl get networkpolicy -n roboshop    =>
aws eks update-addon --cluster-name roboshop-dev --addon-name vpc-cni \ --configuration-values '{"enableNetworkPolicy": “true”}’    =>
Now in K9s go to nginx then run “curl <roboshop-pod-IP>” then run “exit”.
Create “mongodb-catalogue.yaml” file in “k8-network” repo then write Kubernetes network policy code to allow network from catalogue to mongodb.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
git pull    =>
kubectl apply -f mongodb-catalogue.yaml     => 
kubectl get networkpolicy -n roboshop    => After runing this command then verify POD-SELECTOR - it should be like “component=mongodb, catalogue=roboshop,tier=db”.
Pod Disruption Budget  



Diagrams:
rbac
roboshop-infra-updated
terraform-eks
K8-roboshop-architecture


Git Repo Links:
https://github.com/daws-88s/roboshop-infra-eks.git
https://github.com/daws-88s/k8s-init.git
https://github.com/daws-88s/k8-network.git


Timestamps:
Assignment -
Interview questions:
Kubernets Target Group Binding = 19:50
Do you know EKS cluster grouping as per version changes? = 01:16:33





Mistakes & Learning:




Doubts Link Clarification AI chat link: