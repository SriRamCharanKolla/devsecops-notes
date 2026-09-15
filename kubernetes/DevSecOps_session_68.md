Tuesday, 5 May 2026

Session 68 - EKS setup using custom module, Statefulset DBS, Backend through helm charts, Ingress controller vs Load Balancer controller vs Gateway.
Class Notes
ingress -> gateway

db in k8 require statefulset, headless service, normal service, PV, PVC and SC

1. EBS dynamic provisioning requires SC

install ebs drivers
make sure node has EBSCSIDriverPolicy
create storage class

make sure RDS is running and it allows 3306 from bastion and EKS node. 

helm charts
	deployment -> resources, liveness probe, readiness probe, etc..
	configmap/secret
	service
	hpa
	
1. service type is load balancer -> CLB

1. ingress controller -> ingress beta1
2. aws load balancer controller -> ingress v1
3. gateway controller

ingress -> API to connect with external resources like load balancer

ingress controller -> drivers. same across all the 3 versions. Service Account has IAM role mapping, it has IAM permissions to create/read/update/delete the resources

on-premise -> plain kubernetes -> nginx ingress controller

1. make sure OIDC provider exist
2. download IAM policy and create
3. create service account for load-balancer-controller
4. install aws load-balancer-controller drivers
5. create ingress resource.. annotations are useful to select external resources

1. apiVersion changes
2. initial version can't group load balancers -> over cost
3. target group didn't have IP type, you must add all the instances to the load balancer as targets and nodePort should be opened.

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

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=roboshop-dev \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
  
ingress API and ingress controller. ingress controller never changed
only ingress api changed
1. v1beta1 to v1

ingress-v1

https://roboshop-dev.daws88s.online/ -> ALB -> HTTPS Listener -> Rule -> roboshop-dev.daws88s.online -> TG -> pods

when ingress API changed from v1beta1 -> v1

1. apiVersion
2. grouping
3. registering pods as IP address in target group

cost optimisation, security(no need to open nodePort)

ingress api is frozen
1. no new features, no innovations
2. only bug fixes and security patches

python-2.X vs python-3.X

ingress API does not have role seperation between developer and devops

they created new API called gateway. but controller is same. aws-load-balancer-controller

devops/eks admins
==================
gatewayclass -> which LB to use ALB/NLB
loadbalancerconfiguration -> internal/public
gateway -> listener

rule and target-group ()

developer job
==============
TargetGroupConfiguration
HTTPRoute

1. delete ingress
2. delete drivers

helm uninstall aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system

hops

VPC Subnets

ALB -> public

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=roboshop-dev \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID \
  --set controllerConfig.featureGates.ALBGatewayAPI=true \
  --set controllerConfig.featureGates.NLBGatewayAPI=true
  
1. create ALB, Listener, Rule, Target Group
2. Then you just add targetgroupbinding to attach pods

kubectl delete \
  -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml
  
kubectl delete --server-side=true \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

git-ops. project and application at single place. if changes also single place. no chance of forgetting

ALB ACM -> git

check any infra changes ->1st step
check k8 manifest -> 2nd step

init containers
fetching secret from secretmanager by pod
networking

git

cicd using jenkins


Commands:
Create all resources of “roboshop-infra-eks” project upto “60-eks” clusters for Blue Green update process of Roboshop project.
Connect to Bastion server.
aws configure   => To configure AWS CLI configuration on Bastion server, after running this command provide required data related to AWS CLI configuration like AWS Access Key ID, Secrete Access Key, region name, Default output format - leave this empty.
aws eks update-kubeconfig —region us-east-1 —name roboshop-dev   => To generate Kubernetes config file.
kubectl get nodes   =>
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver   => 
helm repo update    =>
helm upgrade --install aws-ebs-csi-driver \ --namespace kube-system \ aws-ebs-csi-driver/aws-ebs-csi-driver     =>
We should always check drivers are properly installed or not. To check it duplicate Bastion connected Terminal tab then type “k9s” which is “kube-system” namespace.
k9s   => To open K9s interspace. Then search for “namespace” then select ““kube-system” by clicking on it, now we can check Kubernetes config is running or not. And its status is Running.
git clone https://github.com/daws-88s/roboshop-infra-eks.git  =>
cd  roboshop-infra-eks/60-eks/app/   =>
ls -l   =>
cd 01-storage-class  =>
cat ebs-sc.yaml   =>
kubectl apply -f ebs-sc.yaml   =>
kubectl get sc   =>
cd ..   =>
kubectl apply -f namespace.yaml   =>
cd 02-databases/  =>    
kubectl apply -f mongodb/manifest.yaml   => 
Now go to K9s tab then choose namespace then select “roboshop” then check “mongodb-0” status created or not.
How this mongodb is creating ? In Kubernets stateful set PVC is written then the PVC go to call storage class then storage-class uses drivers then created volumes of MongoDB. If the “mongodb-0” status is pending then the problem will be on PVC and its drivers of PVC then should check drivers also then should check AWS IAM policy is available for Drivers.
kubectl apply -f redis/manifest.yaml   =>
kubectl apply -f rabbitmq/manifest.yaml   =>  
For Enterprise application we always should be stick to custom modules because they can’t wait for the release of required modules and doing changes as per the Open source modules documentation.
Make sure RDS is running and it allows 3306 from bastion and EKS node.
Interview Scenario - To avoid audit issues related to Kubernetes nodes and Databases, I created a separate node to only accept traffic from MySQL node to MySQL cluster server, there may be some issue in this line I need to correct.
cd ../roboshop-docker/shipping/  => Run in your local system.
ls -l   =>  Run in your local system.
cd db/  =>  Run in your local system.
ls -l   =>  Run in your local system.
cat app-user.sql   =>  Run in your local system.  
How to transfer data securely from one server to another server? Answer - By using “scp <files/folders> ec2-user@<bastion-public-IP>:/folder-name>” .
scp app-user.sql master-data.sql ec2-user@100.24.124.184:/tmp    => To transfer app-user.sql master-data.sql MySQL database script files to Bastion AWS Linux server to “tmp” directory/folder. Type “yes” then enter password of DevOps Practive RedHat-RHEL 9 Linux Community AWS AMI.
Now check is Bastion Server Database Script files are transferred successfully or not.
ls -l /tmp/  =>  
mysql -h <AWS-Aurora and RDS-Databases-roboshop-dev-url> -u root -p<mysql-db-password>   => Command syntax
sudo dnf install mysql -y   =>  
mysql -h roboshop-dev.cw1eikkk698o.us-east-1.rds.amazonaws.com -u root -pRoboshop#123   =>  To connect with AWS Aurora and RDS MySQL Database.
mysql -h roboshop-dev.cw1eikkk698o.us-east-1.rds.amazonaws.com -u root -pRoboshop#123 < /tmp/master-data.sql   => To load Master data in MySQL.
mysql -h roboshop-dev.cw1eikkk698o.us-east-1.rds.amazonaws.com -u root -pRoboshop#123 < /tmp/app-user.sql   => To load users data in MySQL.
mysql -h roboshop-dev.cw1eikkk698o.us-east-1.rds.amazonaws.com -u root -pRoboshop#123   =>
show databases;     =>
use cities;    =>
show tables    =>
Select count(*) from cities;    =>
Select count(*) from codes;    =>    
cd ../03-backend/    =>
ls   =>
cd catalogue/    =>
helm upgrade —install catalogue .   =>
Now check in K9s. Catalogue should show.
kubectl apply -f ../debug/manifest.yaml   => To debug catalogue
Now check in K9s in “debug” then run “curl http://catalogue:8080/health” , output should be like app: “OK”, then run “exit” to came out from catalogue debug.
cd ..   =>
ls   =>
cd user/    =>
helm upgrade —install user .   =>      
Now check in K9s User is created or not.
cd ..cart/    =>
helm upgrade —install cart .   =>  
Now check in K9s Cart is created or not. 
cd ..shipping/    =>
helm upgrade —install shipping .   =>   Shipping is a Java application, Java application took more space and take time to start.
Now check in K9s Shipping is created or not.
cd ..payment/    =>
helm upgrade —install payment .   =>  
Now check in K9s Payment is created or not.
Now check in K9s then go to “debug shell”.
curl http://user:8080/health    => Run inside K9s debug
curl http://shipping:8080/health    => Run inside K9s debug 
curl http://payment:8080/health   => Run inside K9s debug
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.2.1/docs/install/iam_policy.json      =>
aws iam create-policy \ --policy-name AWSLoadBalancerControllerIAMPolicy \ --policy-document file://iam-policy.json   =>
eksctl create iamserviceaccount \ --cluster=roboshop \ --namespace=kube-system \ --name=aws-load-balancer-controller \ --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \ --override-existing-serviceaccounts \ --region us-east-1 \ --approve   => Syntax command
eksctl create iamserviceaccount \ --cluster=roboshop-dev \ --namespace=kube-system \ --name=aws-load-balancer-controller \ --attach-policy-arn=arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy \ --override-existing-serviceaccounts \ --region us-east-1 \ --approve     =>
helm repo add eks https://aws.github.io/eks-charts    =>
helm repo update    =>
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \ -n kube-system \ --set clusterName=roboshop-dev \ --set serviceAccount.create=false \ --set serviceAccount.name=aws-load-balancer-controller     =>
Now check in K9s “kube-system” all drivers are running properly or not.
cd ../roboshop-infra-eks/60-eks/app/   =>
cd 04-frontend/    =>
ls -l  =>
helm upgrade --install frontend .    =>
Now check in K9s then search for “namespace” then select “roboshop” then search for  “ingressClass” resource then check “frontend” load balancer should be created. Then delete “frontend” ingress to remove ingress and to use ingress API.
helm uninstall aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system     =>
helm uninstall aws-load-balancer-controller -n kube-system     =>
# Must be v1.5.0 — earlier versions missing TLSRoute v1 which LBC v3.x requires.   kubectl apply --server-side=true \ -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml                                                                # Verify — should see 8+ CRDs                                                                                 kubectl get crd | grep gateway     =>  To install GateWay API resources
kubectl apply \  -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml                   # Verify TLSRoute is v1 (required by LBC v3.x)                                                          kubectl get crd tlsroutes.gateway.networking.k8s.io -o yaml | grep "name: v1”                 # Must show: name: v1   =>
VPC_ID=$(aws ssm get-parameter --name /roboshop/dev/vpc_id \  --region us-east-1 --query Parameter.Value --output text)     =>
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \  -n kube-system \ --set clusterName=roboshop-dev \ --set serviceAccount.create=false \ --set serviceAccount.name=aws-load-balancer-controller \ --set region=us-east-1 \ --set vpcId=$VPC_ID \ --set controllerConfig.featureGates.ALBGatewayAPI=true \ --set controllerConfig.featureGates.NLBGatewayAPI=true     =>  To create gateway API enabled controller.
Now check in K9s “aws-load-balancer-controller” created or not.
cd ..   =>
cd 05-gateway/   =>
ls -l    =>
kubectl apply -f gatewayclass.yaml     =>
kubectl apply -f loadbalancerconfiguration.yaml     =>    
kubectl apply -f gateway.yaml     => 
Above 3 files or Kubernets resources responsibility is on DevOps Engineer.
 After create these resources then we will inform to applications team then they will include target group configuration, service group etc.
 kubectl apply -f frontend.yaml     =>
 Then you should update loadbalancer in AWS DNS by clicking on “Edit record” then toggle on for “Alias” hen select “Alias to Application and Classic Load Balancer” hen select “Us East (N. Virginia)” then select “robohop” for “Route traffic to” then click on “Save”.
 Now check “roboshop-dev.<your-domain-name>” in the browser, check “Roboshop” application end to end then everything is working fine.
 We need to create Load balancer outside of Kubernetes.
 kubectl delete -f gateway.yaml     =>
 kubectl delete -f gateway.yaml     =>
 kubectl delete -f loadbalancerconfiguration.yaml     =>  
 kubectl delete -f gatewayclass.yaml     =>
 cd ../../    =>  Run in your local system.
 ls -l     =>  Run in your local system.
 cd ..  =>
 ls -l  =>
 kubectl delete \ -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml    =>
 kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml    =>
 



Diagrams:
rbac
roboshop-infra-updated
terraform-eks
K8-roboshop-architecture


Git Repo Links:
https://github.com/daws-88s/roboshop-infra-eks.git
https://github.com/daws-88s/terraform-aws-eks.git


Timestamps:
Assignment -
Interview questions = 00:25,  00:19:45, 00:27:40, 01:53:32, 





Mistakes & Learning:




Doubts Link Clarification AI chat link: