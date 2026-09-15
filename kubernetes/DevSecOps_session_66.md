Sunday, 19 April 2026

Session 66 - K8 Ingress controller.
Class Notes
RBAC
=====
Role -> authorisation service in k8s which can control permissions. which api group, which resource inside api group, what actions user can perform on resources
Role Binding -> map user to role
aws auth -> map IAM users to K8 users
Cluster Role -> Cluster level resources permissions control
Cluster Role Binding -> map user to cluster role
Service Account -> an identity to pod, pod can have service account that maps with IAM role

Ingress controller
==================
type: LoadBalancer
classic LoadBalancer -> legacy, not intelligent

hostpath based routing
=================
amazon.joindevops.com -> amazon
roboshop.joindevops.com -> roboshop

ALB -> Listener -> evaluate rule -> target group -> VM
ALB -> Listener -> evaluate rule -> target group -> Pod

AWS load balancer controller

1. we need single controller
2. https://app1.daws88s.online -> app1 pods
3. https://app2.daws88s.online -> app2 pods

1. OIDC provider installation -> external resources authentication
2. create IAM policy and service account -> maps IAM role with permission that can create AWS resources
3. We need to install ingress controller drivers
4. We need ingress resource to provide networking rules to the applications -> it can expose applications inside k8 to the outside world, it can define the routing rules, it can create AWS resources using annotations


eksctl create iamserviceaccount \
--cluster=roboshop \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve

What is Ingress? - Kubernetes interview question.
if we want application running inside k8 to be exposed to internet, we need ingress resource that can route http/s requests to pods, through annotations we can select external resources like alb, listener, target group, certificate arn, etc.

ingress talks to ingress controller drivers, that will have service account mapped to IAM role and permission to create the resources..

10.0.0.0/16 -> VPC CIDR range
EKS_NODE_SG should allow traffic from 10.0.0.0/16


Commands:
Create “k8s-ingress” new git repo.
Clone in your local machine.
Create “app1” folder under “k8-ingress” then create “Dockerfile” write Docker of nginx code then push code.
Similarly create “app2” folder under “k8s-ingress” then create “Dockerfile” write Docker of nginx code then push code.
Create “roboshop-dev-workstation” then connect to it and then create 2 “roboshop-spot-Node”.
git clone <k8s-ingress-git-repo-link>   =>
cd k8s-ingress/app1   =>
docker login -u <your-docker-user-name>   =>
docker build -t <your-docker-user-name>/app1:1.0.0 .   =>   
docker push <your-docker-user-name>/app1:1.0.0 .   =>
cd ../app2/   =>
docker build -t <your-docker-user-name>/app2:1.0.0 .   =>
docker push <your-docker-user-name>/app2:1.0.0 .   =>
kubectl get nodes  =>
aws eks update-kubeconfig —region us-east-1 —name roboshop   =>
kubectl get nodes  =>
eksctl utils associate-iam-oidc-provider \ --region us-east-1 \ --cluster roboshop \ —approve   =>
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.2.1/docs/install/iam_policy.json   =>
aws iam create-policy \ --policy-name AWSLoadBalancerControllerIAMPolicy \ --policy-document file://iam-policy.json   =>
eksctl create iamserviceaccount \ --cluster=roboshop \ --namespace=kube-system \ --name=aws-load-balancer-controller \ --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \ --override-existing-serviceaccounts \ --region us-east-1 \ —approve   => in this command <AWS_ACCOUNT_ID> should be replace with your’s AWS account ID.
helm repo add eks https://aws.github.io/eks-charts  =>
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=roboshop --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller   =>
Now duplicate terminal tab and then connect to “roboshop-dev-workstation” server then connect k9s by typing k9s an then hit enter. Then search for namespace in k9s then select “kube-system”.
Create “manifest.yaml” file in “app1” folder of “k8s-ingress” Kubernetes project. Then write Kubernetes deployment and service code by coping from “k8-resources” project files and then alter code as per Kubernetes ingress requirements.
cd k8s-ingress/app1   =>
Create “manifest.yaml” file in app1 folder & write Kubernetes code then push code.
git pull  =>
kubectl apply -f manifest.yaml   =>
In new tab connect to “roboshop-dev-workstation” server then collect to “k9s”.
Search for namespace in k9s server terminal. Then select default then 
Now add listen-ports, certificate-arn, scheme, tags, target-type & group.name annotations in “manifest.yaml” file in app1 folder.
Go to AWS Certificate Manager then click on “Request” then click on “Next” button then enter “ *.<your-domain-name>” in “Fully qualified domain name” field under “Domain names” then click on “Request”  then click on “Create DNS records in Amazon Route 53” then select “ *.<your-domain-name>” then click on “Create records” then copy ARN then paste in “manifest.yaml” file certificate-arn annotation.
Ingress Controller resources are created by Ingress Controller drivers we installed in the server.
Go to k9s roboshop-dev-workstation” server tab and search for namespace then select “cube-system” then select “was-load-balancer-controller” to check logs so that we come to know if any errors occurred while creating resources.
git pull  =>
kubectl apply -f manifest.yaml   =>
Now check logs in k9s terminal tab, Kubectl ingress controller driver is creating resources.
Now go to AWS EC2 then go to “Load Balancers” and check load balancer is created or not. Then click on it then click on “2 Rules” then click on target group in “Actions(Then)” tab check in target group IP Address are registered those are pods 
To access created Load Balancer through AWS Route 53 then should create route 53 record. So go to AWS Route 53 then clicl on “Hosted zone” then in “Hosted zones” listing page click on “Hosted zone name” you want to create hosted zone then click on “Create record” then enter “app1”(if you want only app1 specific) or “*.”(if you want to allow all) in “Record name” field then toggle on “Alias” toggle then select “Alias to Application and Classic Load Balancer” for “Choose endpoint” dropdown then select “dual stack.k8s-default-app1” load balance for “Choose load balancer” dropdown then click on “Create record” then in browser search bar enter “https://app1.<your-domain-name>” then check whatever HTML page code we have provided that should be visible.
Now delete a pod(app1) in k9s then new pod will be created automatically and immediately, if we use VM then it will take time to create new one.
Create “manifest.yaml” file in app2 folder & write Kubernetes code then push code.
git pull  =>
cd ../app2/   =>
kubectl apply -f manifest.yaml   =>
Now check in Load balancers listing page in AWS a new load balancer will be created with app2 manifest.yaml execution but we only need one load balancer because if we create a load balancer for each application then AWS cost will be increased, single load balancer is enough, if we want single load balancer for app1 & app2 then we need to group under a project for example for roboshop project group all load balancers under roboshop group.
kubectl delete -f manifest.yaml   =>
To group add group.name annotation in app1 manifest.yaml file then push code changes.
Ingress controller means how we will get traffic into our applications running in our Kubernets.
cd ../app1/   =>
git pull  =>
kubectl apply -f manifest.yaml   =>
We need update AWS Route 53 record what we previously created because new load balancer in create and no old load balancer. Go to AWS Route 53 app1 record then click on edit then under Alias “Alias to Application and Classic Load Balancer” & “dual stack.k8s-default-app1” then click on “Save”, this records will be automatically registered once new load balance & pods are created successfully.
Now go to browser search bar refresh “https://app1.<your-domain-name>” then check whatever HTML page code we have provided that should be visible.
Similarly in app2 manifest.yaml add group.name annotation then push code.
cd ../app2/   =>
git pull  =>
kubectl apply -f manifest.yaml   =>
Then create AWS Route 53 record with Alias with kubernetes load balancer, should only create DNS create with app2 and do not use star dot(*.) because use star dot(*.) is not secure because any one can enter into our applications with domain name endpoint.
Now go to browser search bar refresh “https://app2.<your-domain-name>” then check whatever HTML page code we have provided that should be visible like Hi, I am from APP-2 this is the code what we provided for nginx.
kubectl delete -f manifest.yaml   =>
cd ../app1/   =>
kubectl delete -f manifest.yaml   =>
Create “roboshop-infra-eks” new git repo then clone it in your local machine.
Now copy “00-vpc” & “10-sg” from “roboshop-infra-dev” then delete “.terraform” folders & “.terraform.lock.hcl” files in both then do required changes as per the new requirement then push code.
cd roboshop-infra-eks/00-vpc/  => Run inside you local machine terminal likely VS code terminal.
terraform init   => Run inside you local machine terminal likely VS code terminal.
terraform plan  => Run inside you local machine terminal likely VS code terminal.
terraform apply -auto-approve  => Run inside you local machine terminal likely VS code terminal.
Now copy “20-sg-rules” from “roboshop-infra-dev” and then paste in “roboshop-infra-eks”.
Do required Terraform code changes in “20-sg-rules” terraform code files.     



Diagrams:
ingress-controller
roboshop-infra-updated
terraform-eks


Git Repo Links:
https://github.com/daws-88s/k8s-ingress.git
https://github.com/daws-88s/roboshop-infra-eks.git


Timestamps:
Assignment -
Interview questions =>
1.  Kubernets Ingress Controller





Mistakes & Learning:




Doubts Link Clarification AI chat link: