Thursday, 16 April 2026

Session 65 - K8 RBAC(Role, Role Binding, ClusterRole, ClusterRoleBinding, AWS auth, Service Account, Secret fetching through ServiceAccount).
Class Notes
Helm Charts
============
1. package manager for k8s
2. templatise manifest files

1. build image
2. run image -> what are the resources and configs

Chart.yaml -> chart information. name, api version, version, app version, description, etc -> metadata about the chart
templates/ -> manifest files with placeholders
values.yaml -> placeholder actual values here
values-dev.yaml, values-prod.yaml -> env specific values

helm upgrade --install <chart-name> --set key=value .
helm uninstall <chart-name>
helm history <chart-name>
helm rollback <chart-name>
helm rollback <chart-name> <revision> 

RBAC
=====
K8s is platform as a service -> it has its own authentication and authorisation mechanism. but we can integrate this with IAM for authentication and authorisation can be managed in EKS

api-groups -> resources -> actions

1. create user and assign cluster describe policy
arn:aws:iam::160885265516:user/suresh
2. create role and rolebinding
3. map IAM user to k8s user

trainee == pod-reader
juniors == deployer -> create, read
senior == checker -> create, update, read
admin == admin -> create, read, update, delete

service account
==============
1. create sa
2. create IAM role and attach permission
3. run pod with sa

OIDC provider
eksctl utils associate-iam-oidc-provider --cluster roboshop --approve

eksctl create iamserviceaccount \
  --name roboshop-secret-reader \
  --namespace roboshop \
  --cluster roboshop \
  --attach-policy-arn arn:aws:iam::160885265516:policy/RoboshopMySQLSecretReader1 \
  --approve

aws secretsmanager get-secret-value --secret-id roboshop/dev/mysql_root_password


Commands:
Create Cluster.
kubectl get nodes  =>
Go to AWS IAM Dashboard then select “Policies” then click on “Create policy” then under “Service” choose “EKS” then under “Actions allowed” under “Read” select “DescribeCluster” then click on “Add ARNs” then enter “us-east-1” for “Resource region” then enter “roboshop” for “Resource cluster name” then enter “roboshop cluster ARN” in “Resource ARN” then click on “Add ARNs” then click on “Next” then enter “RoboshopEKSClusterDescribe” for “Police name” then click on “Create policy”.
Go to AWS IAM Dashboard then under “IAM resources” click on “Users” then click on “Create user” then enter “suresh” for “User name” then click on “Next” button then under “Permissions options” select “Attach policies directly” then under “Permissions policies” search for “RoboshopEKSClusterDescribe” then select it then click on “Next” then click on “Create user”.
Create “k8-rbac” new git repo then clone it in your local machine then create “01-role.yaml” file under it then write Kubernetes code then push & pull code.
Create “02-role-binding.yaml” file under “k8-rbac” then write Kubernetes code then push & pull code.
cd   =>
git clone “k8-rbac-git-repo-link”   =>
cd k8-rbac  =>  
kubectl create namespace roboshop =>
kubectl apply -f 01-role.yaml   =>
kubectl get roles -n roboshop  =>
kubectl apply -f 02-role-binding.yaml   =>
kubectl get rolebindings -n roboshop  =>
kubectl get configmap aws-auth -n kube-system =>
kubectl get configmap aws-auth -n kube-system -o yaml  =>
Create “03-aws-auth.yaml” file under “k8-rbac” then write Kubernetes code then push & pull code.
kubectl apply -f 03-aws-auth.yaml  =>
kubectl get configmap aws-auth -n kube-system -o yaml  =>
Create AWS EC2 instance with “roboshop-elk-client” with RedHat RHEL 9 DevOps AMI image. Then connect with your Local Machine.
Install 
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl   => Run inside roboshop-elk-client.
chmod +x ./kubectl  => Run inside roboshop-elk-client.
sudo cp kubectl /usr/local/bin/kubectl  => Run inside roboshop-elk-client.
kubectl version   => Run inside roboshop-elk-client.
aws configure   => Run inside roboshop-elk-client.
Create Credentials for Suresh.
Go to IAM select “suresh” then select “Security credentials” then click on “Create access key” then under “Use case” select “Command Line Interface” then select “Confirmation” then click on “Next” then click on “Create access key” then copy “Access key” & “Secret key” and then enter in Run inside roboshop-elk-client AWS instance connect terminal. 
Enter suresh “Access key”.
Enter suresh “Secret key”.
region name - us-east-1.
Next one leave empty.
aws eks update-kubeconfig —region us-east-1 —name roboshop   => Run inside roboshop-elk-client.
After running above command in “.kube” folder “config” file will be generated.   
kubectl get pods  => Run inside roboshop-elk-client.
kubectl get pods -n roboshop  => Run inside roboshop-elk-client.
kubectl get deployments -n roboshop  => Run inside roboshop-elk-client.
Create “04-roboshop-admin.yaml” & “05-roboshop-admin-rb.yaml” files under “k8-rbac” then write Kubernetes code then push & pull code.
Create user “ramesh” in AWS IAM and attach “RoboshopEKSClusterDescribe” policy while creating user.
git pull  =>
kubectl apply -f 04-roboshop-admin.yaml    =>
kubectl apply -f 05-roboshop-admin-rb.yaml   =>
Add “roboshop-admin” group code under “mapUsers” in “03-aws-auth.yaml” file and then push & pull code.
git pull  =>
kubectl apply -f 03-aws-auth.yaml  =>
Got error due do “resourceVersion” so remove “resourceVersion” line. Kubernetes will automatically update “resourceVersion” every time run the application.
git pull  =>
kubectl apply -f 03-aws-auth.yaml  =>
kubectl get configmap aws-auth -n kube-system -o yaml   =>
Now create “Security credentials” to “ramesh” same like suresh.
Now configure “ramesh” credentials in “roboshop-elk-client” server.
aws configure   => Run inside roboshop-elk-client.
Enter suresh “Access key”.
Enter suresh “Secret key”.
region name - us-east-1.
Next one leave empty.
aws sts get-caller-identity  =>
aws eks update-kubeconfig —region us-east-1 —name roboshop   => Run inside roboshop-elk-client.
kubectl get deployments -n roboshop  => Run inside roboshop-elk-client.
vim mongodb.yaml   => Run inside roboshop-elk-client. Then type wq!.
kubectl apply -f mongodb.yaml  =>
kubectl get deployments -n roboshop  => Run inside roboshop-elk-client.
kubectl get resources | grep pv  => Run inside roboshop-elk-client. 
kubectl get resources | grep sc  => Run inside roboshop-elk-client.
kubectl get resources | grep role  => Run inside roboshop-elk-client.
Now add “cluster-role” in “04-roboshop-admin.yaml” file and add volumes & storage permissions to user and bind role in “05-roboshop-admin-rb.yaml” file then push & pull code.
git pull  =>
kubectl apply -f 04-roboshop-admin.yaml    =>
kubectl apply -f 05-roboshop-admin-rb.yaml   =>
kubectl get pv  => Run inside roboshop-elk-client. 
kubectl get sc  => Run inside roboshop-elk-client.
Create “John” & “rahim” users in AWS IAM and attach “RoboshopEKSClusterDescribe” policy while creating user and also create “Security credentials” to configure/login with user credentials in “roboshop-elk-client”.
Create“06-trainee-role-binding.yaml” file under “k8-rbac” then write Kubernetes code then push & pull code.
Map “john” & “rahim” users & group users under “roboshop-trainee” group under “mapUsers” block of code in “03-aws-auth.yaml” file and then push & pull code.
kubectl apply -f 06-trainee-role-binding.yaml  =>
kubectl apply -f 03-aws-auth.yaml  =>
Now in “roboshop-elk-client” server login with “john” security credentials.
aws configure   => Run inside roboshop-elk-client.
Enter suresh “Access key”.
Enter suresh “Secret key”.
region name - us-east-1.
Next one leave empty.
aws sts get-caller-identity  =>
kubectl get pods -n roboshop  => Run inside roboshop-elk-client.
kubectl get pods  => Run inside roboshop-elk-client.
aws eks update-kubeconfig —region us-east-1 —name roboshop-prod   => Run inside roboshop-elk-client. Will get error because “John” deascrCluster only have for “roboshop” not for “roboshop-prod”. Means no authentication permission for “roboshop-prod”.
aws eks update-kubeconfig —region us-east-1 —name roboshop   => Run inside roboshop-elk-client.
kubectl get sa -n roboshop  => Run inside roboshop-elk-client. To get Service Account details. sa stands for Service Account.
k9s  =>
Go to roboshop name space in k9s select mongodb then describe it. Check mongodb will be running with default service account. 
eksctl utils associate-iam-oidc-provider --cluster roboshop --approve  =>
Then create service account. So search for “eke service account create” and then copy code from google and use it.
Go to AWS Secrete Manager then get any secrete details example “mysql_root_password”.
Now go to AWS IAM then go to policy click on “Create policy” then select “Secrets Manager” for “Select a service” select “GetSecretValue” under “Read” then click on “Add ARNs” then copy “mysql_root_password” ARN and paste it in “Resource ARN” field then click on “Add ARNs” then click on “Next” then enter “RoboshopMySQLSecretReader1” in “Policy name” under Policy details then click on “Create policy” then in “Policies” listing page search for “RoboshopMySQLSecretReader1” then click on it then copy ARN then add ARN in attach-policy under eksctl create iamsecreteaccount command.
Add all required data in eksctl create iamsecreteaccount command and then run this command.
eksctl create iamserviceaccount \ --name roboshop-secret-reader \ --namespace roboshop \ --cluster roboshop \ --attach-policy-arn arn:aws:iam::160885265516:policy RoboshopMySQLSecretReader1 \  —approve   =>
kubectl get sa -n roboshop  =>
kubectl get sa roboshop-secret-reader -o yaml -n roboshop  =>
Copy syntax to use in “07-sa.yaml” file by creating it to create Service Account with Kubernetes.
Create “07-sa.yaml” file under “k8-rbac” then write Kubernetes code then push & pull code. Paster copied syntax to create Service Account with Kubernetes.
Create “08-pod.yaml” file under “k8-rbac” then write Kubernetes code to check service account then push & pull code.
k9s  => To get syntax of pod for Service account(sa) checking with Pod kubernets code. 
 Select roboshop then select mongodb then type y to get Kubenetes yaml code then search for service account and copy it and use in “08-pod.yaml” file.
 git pull  =>
 kubectl apply -f 08-pod.yaml    =>
 kubectl get pods -n roboshop  =>
 kubens roboshop   => 
 kubectl get pods -n roboshop  =>
 No command given in 08-pod.yaml” file command code. Then push code.
 git pull  =>
kubectl delete -f 08-pod.yaml    =>
 kubectl apply -f 08-pod.yaml    =>
 k9s  =>
 Select roboshop then select aws-cli then take shell then run below command in shell.
 aws secretsmanager get-secret-value --secret-id roboshop/dev/mysql_root_password  => Will get response because we had provided access to mysql_root_password.
 aws secretsmanager get-secret-value --secret-id roboshop/dev/mysql_password  => Not provided access to “mysql_password” so output will be error.


Diagrams:
rbac

GitHub Repos:
k8-rbac


Timestamps:
Assignment -
Interview questions = 





Mistakes & Learning:




Doubts Link Clarification AI chat link: