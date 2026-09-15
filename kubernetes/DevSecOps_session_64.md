Wednesday, 15 April 2026

Session 64 - Helm Charts.
Class Notes
Selectors
=========
nodeSelector -> based on node labels
nodeAffinity/AntiAffinity -> In, NotIn, Exists, Gt, Lt, etc.
PodAffinity/AntiAffinity -> attract or repel the pods
Taints and tolerations -> firewall rules, specific hardware nodes are tainted.. workloads should have tolerations


Helm Charts
===========
1. How to build the image -> nginx, nodejs, jre, etc..
2. How to run the image -> manifest files

1. you can use as package manager in k8s
2. we can templatize the k8s manifest files

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

Chart.yaml
==========
apiVersion: v2 # K8S related
name: nginx
version: 0.0.1 # This is chart version
appVersion: latest # This is application version like catalogue version

values.yaml -> we can maintain the placeholder values here
templates/ -> all k8s manifest files here with placeholders

helm install <chart-name> .
helm list -> list of the charts installed
helm uninstall <chart-name> -> removes the application

1. application -> code
2. configuration -> change

mongodb-dev.daws88s.online
mongodb.daws88s.online

values.yaml -> default values same across all environments
values-dev.yaml -> values for dev specific environment

build image, push image, run helm command, check app is running fine or not, if not rollback

EBS drivers, EFS drivers

dnf repo add docker.repo
dnf update
dnf install docker-ce -y

helm repo add <url>
helm repo update
helm 

prometheus/grafana
run the image -> manifest

statefulset, svc, role and rolebindings, 

official helm repos -> statefulset, svc, role and rolebindings, 
values.yaml

helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm search repo grafana-community/grafana
helm upgrade --install roboshop-grafana grafana-community/grafana --set service.type=LoadBalancer --namespace monitoring

RBAC -> Role based access control
=====
user who created EKS cluster by default gets admin access

DevOps engineer for roboshop project -> admin access to roboshop namespace

Nouns(resources), Verbs(actions)

authentication(prove your identity) and authorisation(scan your id at ODC)
User
Group

api group
pod, deployment, configmap, etc..
create pod, list pod, watch pod, update, delete, etc..

an api group have multiple resources, you can perform multiple actions on resources

trainee -> only read access
junior -> only read access+core(api/v1) deployments update access..
senior -> full access to app/v1 group
TL -> roboshop namesapce all api groups, all resources, all actions

Role -> Binds to users through rolebinding for example Suresh user has trainee role

Kubernetes -> PaaS -> has its own authentication mechanism

You can integrate IAM user to EKS

namesapce level and cluster level

Role and RoleBinding -> Namespace level access
PV -> cluster level
ClusterRole and ClusterRoleBinding -> cluster level resources

Roboshop devops engineer sends a email to K8 admin
create roboshop namespace and give access to us..

roboshop-eks-client -> kubectl install -> authenticate with cluster -> run their workloads


1.30 -> build

--set imageVersion=1.30 --description "upgrading to 1.30"



Commands:
Create “helm” new git repo & clone it in local.
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3    =>
chmod 700 get_helm.sh   =>
./get_helm.sh  =>
helm version  =>
Create “Chart.yaml” file in “helm” git repo local location. Then write Helm code & then push & pull code.
Create “templates” folder then create “deployment.yaml” & “service.yaml” files in “templates”folder then create “values.yaml” file in “helm” git repo local location. Then write Helm code & then push & pull code. 
git clone “helm-git-repo-link”   =>
cd helm   =>
git pull  => 
helm install nginx .   =>
helm list   =>
helm uninstall nginx   =>
git pull  => 
helm install nginx .   =>
helm list   =>
Now update nginx version.
git pull  =>
helm upgrade nginx .   =>   
helm history nginx  =>
Again update nginx version & do required changes.
helm upgrade nginx .   =>
helm list   =>
helm history nginx  =>
helm upgrade nginx —set deployment.imageVersion=“1.29.8.trixie” —version 0.0.4 —description “upgrading to 1.29.8.trixie” .   =>
helm history nginx  =>
kubectl get pods   =>
kubectl get svc   => 
helm rollback nginx  =>
helm history nginx  =>
helm rollback nginx 1  =>
helm history nginx  =>
Create “values-prod.yaml” file in “helm” git repo local location. Then write Helm code & then push & pull code.
git pull  =>
helm upgrade nginx .   =>   
helm history nginx  =>
helm uninstall nginx  =>   
helm list =>
helm upgrade —install nginx .   =>   If 1st time installing nginx then it will install nginx else it will upgrade nginx.
kubectl get pods   =>
helm upgrade —install nginx -f values-prod.yaml .   =>
kubectl get pods   =>
helm upgrade —install nginx -f values-dev.yaml .   => We can give anything after values-  
kubectl get pods   =>
Create “roboshop-helm” new git repo & clone it in local.
Create “mongodb” folder then create “templates” folder then create “deployment.yaml” & “service.yaml” files in “templates”folder then create “values.yaml” file in “roboshop-helm” git repo local location. Then write Helm code & then push & pull code.
cd  =>
git clone “roboshop-helm-git-repo-link”   => 
cd roboshop-helm  =>
kubectl create namespace roboshop  =>
Kubens roboshop   =>
cd mongodb  =>
helm upgrade —install mongodb .   =>
kubectl get pods   =>
kubectl get svc   =>
Create “catalogue” folder then create “templates” folder then create “deployment.yaml”, “service.yaml” & “configmap.yaml” files in “templates”folder then create “values.yaml” file in “roboshop-helm” git repo local location. Then write Helm code & then push & pull code.
git pull  =>
helm upgrade —install catalogue .   =>
kubectl get pods   =>
kubectl get svc   =>
helm upgrade catalogue —set deployment.imageVersion=“1.4.0” —version 0.0.1 —description “upgrading to 1.4.0” .   =>
helm history catalogue  =>
kubectl get deployment catalogue -o yaml   =>
helm rollback catalogue   =>
helm history catalogue   =>
kubectl get deployment catalogue -o yaml   =>
helm upgrade --install aws-ebs-csi-driver \ --namespace kube-system \ aws-ebs-csi-driver/aws-ebs-csi-driver   =>
helm list -n kube-system   =>
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update   =>
kubectl create namespace monitoring   =>
helm search repo grafana-community/grafana   =>
helm upgrade --install roboshop-grafana grafana-community/grafana --set service.type=LoadBalancer --namespace monitoring   =>
kubectl get secret —namespace monitoring roboshop-grafana -o jsonpath=“{.data.admin-password}” | bnase64 —decode ; echo   =>
kubectl get svc -n monitoring    => Go get “EXTERMAL IP”
Now copy “EXTERMAL IP” & browse in browser with http. Then Grafana login page will be displayed then enter “admin” for “Email or username” field then for password enter roboshop-grafana monitoring secret key.
kubectl get pods   =>
kubectl get pods -n monitoring    => 
helm uninstall roboshop-grafana -n monitoring   =>
helm uninstall catalogue   =>
helm uninstall mongodb   =>
helm uninstall nginx -n default  =>



Diagrams:



Timestamps:
Assignment - roboshop-helm practice for Redis, RabbitMQ User, Cart, Shipping, Payments = 01:14:10
Interview questions = 01:17:00





Mistakes & Learning:




Doubts Link Clarification AI chat link: