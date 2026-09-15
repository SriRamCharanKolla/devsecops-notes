Wednesday, 6 May 2026

Session 69 - Target group binding, Blue green deployment, EKS blue green upgrade.
Class Notes
ingress old api vs new api
ingress api vs gateway api

target group binding

1. OIDC provider
2. policy and service account creation
3. aws load balancer controller drivers

eksctl create iamserviceaccount \
--cluster=roboshop-dev \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve

eksctl delete iamserviceaccount \
--cluster=roboshop-dev \
--namespace=kube-system \
--name=aws-load-balancer-controller

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=roboshop-dev --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

blue green deployment
==================
Recreate -> stop the servers, remove old code, download new code, then start again
RollingUpdate -> slowly installing new servers/pods and removing old server/pods
no downtime, at this point of upgrade app serves both old and new version
Blue/Green ->

1. main service -> running version
2. preview service -> pointing to upcoming version

running version = blue
main service -> blue

new version = green
attach preview service to green
check internally, some health checks
if response is good
switch the route
then point main service to green -> live traffic
make blue replicas as 0

if rollback
make blue replicas as 2
again point main service to blue

double resources

version 0.0.1 -> blue

kubectl patch service nginx -p '{"spec":{"selector":{"version":"green"}}}'
kubectl patch deployment nginx-blue -p '{"spec":{"replicas":0}}'

present running -> green

new version blue -> 0.0.3
point preview service to blue
test it

if fine, then switch main service to blue

1. EKS platform, blue-nodegroup

2. Upgrade platform to new version 1.35 keep the blue-ng running in 1.34
3. Create another nodegroup green 1.35(this is automatic, because eks in 1.35)
4. cordon(scheduling disabled) blue nodes, then drain blue node slowly..

when there are platform upgrades, we announce defnitely downtime. 30min - 6hours
you face intermittent issues ->no downtime

terraform apply \
  -var="eks_version=1.35" \
  -var="eks_nodegroup_blue_version=1.34"
 Process to Upgrade EKS cluster:
1. internal project teams, send your schedule -> before 3 months
2. update SG,cut the access to other teams

terraform apply \
  -var="eks_version=1.35" \
  -var="eks_nodegroup_blue_version=1.34" \
  -var="enable_green=true" \
  -var=“eks_nodegroup_green_version=1.35"


Commands:
Open integrated terminal on “roboshop-infra-eks” project.
ls    => Run in your local system.
for I in 70-acm/ 80-frontend-alb/; do cd $I; terraform apply -auto-approve; cd ..;done    =>  
Search for “Kubernetes AWS Load Balancer Controller Installation” then open documentation 
Then Go daws.88s GitHub “k8-ingress” repo and open it.
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.2.1/docs/install/iam_policy.json      =>
eksctl create iamserviceaccount \ --cluster=roboshop \ --namespace=kube-system \ --name=aws-load-balancer-controller \ --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \ --override-existing-serviceaccounts \ --region us-east-1 \ --approve       => Some times we will get exits output, so at that time we need to delete and recreate it by running this command.
eksctl delete iamserviceaccount \ --cluster=roboshop-dev \ --namespace=kube-system \ --name=aws-load-balancer-controller      =>
eksctl create iamserviceaccount \ --cluster=roboshop \ --namespace=kube-system \ --name=aws-load-balancer-controller \ --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \ --override-existing-serviceaccounts \ --region us-east-1 \ --approve       =>
helm repo add eks https://aws.github.io/eks-charts     =>
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=roboshop-dev --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller      =>
aws eks update-kubeconfig —region us-east-1 —name roboshop-dev   =>
kubectl get notes   =>
aws eks update-kubeconfig —region us-east-1 —name roboshop-dev   =>
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=roboshop-dev --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller      =>
aws eks update-kubeconfig —region us-east-1 —name roboshop-dev   => 
kubectl get notes   =>
aws eks update-kubeconfig —region us-east-1 —name roboshop-dev   =>
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=roboshop-dev --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller      =>
In New Tab open “K9s” to check drivers installed successfully or not. Search for “namespace” in K9s then select “kube-system” 
Now in google search for Kubernetes target group binding then get code syntax.
git clone https://github.com/daws-88s/k8-roboshop.git    =>
cd k8-roboshop/    =>
ls -l     =>
kubectl apply -f 01-namespace.yaml   =>
for I in mongodb/ redis/ mysql/ rabbitmq/ catalogue/ user/ cart/ shipping/ payment/ frontend/;do cd $I; kubectl apply -f manifest.yaml; cd ..;done     =>
Go K9s and select roboshop then check all roboshop Kubernetes components are creating are not. Then after frontend is created and displaying in K9s then select frontend then press e to edit “.yaml” file then comment out loadbalancer code then “:wq!” To save changes.
Go AWS “Load Balancers” list then there “roboshop-dev-frontend” will be created as per our changes in frontend Terraform file, Copy ARN.
Now create “tab.yaml” file in “60-eks” folder in “roboshop-infra-eks” repo. Use Kubernetes target group binding then get code in this file as per the requirement.
git clone https://github.com/daws-88s/roboshop-infra-eks.git     =>
cd roboshop-infra-eks/60-eks/      =>
ls -l      =>
cd app/    =>
ls -l      =>  
Push “tab.yaml” file then pull in Bastion server.
git pull  =>
kubectl apply -f tab.yaml    =>
Now verify “roboshop-dev-frontend” load balancer target group.
Now verify Roboshop application by placing an order, if everything working fine then we successfully deployed.
kubectl delete -f tab.yaml    =>
kubectl namespace roboshop    =>
Create “k8s-blue-green” repo in GitHub. Then clone it in your local system. 
cd ..  => Run in your local system.
git clone https://github.com/daws-88s/k8s-blue-green.git    => Run in your local system.
Create “Dockerfile” file in “k8s-blue-green” repo. Then write Docker code.
Create “01-blue-deployment.yaml” file in “k8s-blue-green” repo. Then write Docker code.  
Create “02-main.service.yaml” file in “k8s-blue-green” repo. Then write Docker code.   
git add . ; git commit -m “k8”; git push origin main   => => Run in your local system.
Now clone in Bastion Server.
git clone https://github.com/daws-88s/k8s-blue-green.git    =>
cd k8s-blue-green/     =>
ls    =>
docker build -t <your-docker-hub-user-name>/blue-greem:0.0.1 .     =>
docker login -u <your-docker-hub-user-name>   =>
docker push <your-docker-hub-user-name>/blue-greem:0.0.1    =>     
kubectl apply -f 01-blue-deployment.yaml   =>
Now go to K9s and search for namespace then select default check blue-green docker image is running or not.
kubectl apply -f 02-main.service.yaml   =>
Now go to K9s and search for namespace then select nginx check it. Then go to AWS then check Load Balancers, SG. We are getting NLB instead of ALB, this is because of AWS Loan Balancer Controller Drivers, we should uninstall drivers.
kubectl delete -f 02-main.service.yaml   =>
helm uninstall aws-load-balancer-controller -n kube-system  =>
kubectl apply -f 02-main.service.yaml   =>
Now go to K9s and search for namespace then select nginx check it and its should be elf load balancer.
Now go to AWS “roboshop-dev-frontend” load balancer then click on “Target instances” then check Target instances”.
Copy URL and check the repose of the “roboshop-dev-frontend” load balancer response in browser.
Do version 0.0.2 change in 02-main.service.yaml and push changes.
git add . ; git commit -m “k8”; git push origin main   => Run in your local system.
git pull  =>
docker build -t <your-docker-hub-user-name>/blue-greem:0.0.2 .     =>
docker push <your-docker-hub-user-name>/blue-greem:0.0.2    =>     
Create “03-green-deployment.yaml” file in “k8s-blue-green” repo. Then write Docker code.
Create “04-preview.service.yaml” file in “k8s-blue-green” repo. Then write Docker code.
git add . ; git commit -m “k8”; git push origin main   => => Run in your local system.
kubectl apply -f 03-green-deployment.yaml   =>
In K9s search for “pods” then verify green pods should display.
kubectl apply -f 02-preview.service.yaml   =>
In K9s search for “pods” then select green pods then run health check with “curl http:preview-nginx” response should whatever html code we given in “04-preview.service.yaml” file in “k8s-blue-green” repo.
kubectl patch service nginx -p ‘{"spec":{"selector":{"version":"green"}}}'   => To deploy new version green, output version should changes as per green deployment changes like “Hi, I am version 0.0.2”.
kubectl patch deployment nginx-blue -p ‘{“spec”:{"replicas":0}}'   => Once everything is working fine then we will delete all pod replicas with 0 replicas to reduce cost.
kubectl patch deployment nginx-blue -p ‘{“spec”:{"replicas":4}}'   => When we have any issue with newly deployed green version then we will recreate 4 pod replicas then we will deploy back previous which is blue.
kubectl patch service nginx -p ‘{"spec":{"selector":{"version":"blue"}}}'   =>  If any issue in new green deploy ment we can revert back to to previous version and deploy back blue version.
Now check in browser version should display like “Hi, I am version 0.0.1”.
kubectl patch deployment nginx-green -p ‘{“spec”:{"replicas":0}}'   =>
cd ../roboshop-infra-eks/40-rds/    => Run in your local system.
cd 60-eks  => Run in your local system.
terraform apply \ -var="eks_version=1.35" \ -var=“eks_nodegroup_blue_version=1.34" -auto-approve  => Run in your local system. Reply yes if any prompt confirmation required.
Process to Upgrade EKS cluster: 
Internal project teams, send your schedule -> before 3 months.
Update SG,cut the access to other teams.
cd  =>
kubectl get nodes   =>
for addon in vpc-cni eks-pod-identity-agent coredns kube-proxy metrics-server; do                  LATEST=$(aws eks describe-addon-versions \    --kubernetes-version 1.35 \ --addon-name $addon \ --query 'addons[0].addonVersions[0].addonVersion' \ --output text) aws eks update-addon \ --cluster-name roboshop-dev \ --addon-name $addon \ --addon-version $LATEST \ --resolve-conflicts OVERWRITE \ --region us-east-1 done  =>
kubectl get nodes   =>
kubectl cordon -l nodegroup=blue    =>
kubectl get nodes   =>
Check pods are running or not in K9s.
All Pods should not update at a time.
kubectl drain <node-id> —ignore-demonsets —delete-emptydir-data   => Should update each pods one by one with this command.
Check pods are changed to green or not in K9s.
 kubectl drain <node-id> —ignore-demonsets —delete-emptydir-data   => Should update each pods one by one with this command.
 terraform apply \ -var="eks_version=1.35" \ -var="enable_blue=false" \ -var=“enable_green=true" -auto-approve  => With this command blue pods will be deleted as green is running and in live.
 




Diagrams:
rbac
roboshop-infra-updated
terraform-eks
K8-roboshop-architecture


Git Repo Links:
https://github.com/daws-88s/roboshop-infra-eks.git
https://github.com/daws-88s/terraform-aws-eks.git
https://github.com/daws-88s/k8-ingress.git


Timestamps:
Assignment -
Interview questions:
Kubernets Target Group Binding = 19:50
Do you know EKS cluster grouping as per version changes? = 01:16:33





Mistakes & Learning:




Doubts Link Clarification AI chat link: