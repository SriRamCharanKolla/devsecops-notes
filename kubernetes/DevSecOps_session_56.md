Sunday, 5 April 2026

Session 56 - K8 Resources ( Namespace, Pod, Labels, Annotations, Environment, Resources, Service( ClusterIp ).
Class Notes
everything is resource inside k8s
K8s -> PaaS

namespace -> an isolated space inside k8 cluster where can create our project related resources. we can completely control them. it is like our project inside k8s.

1. namespace level -> VPC level
2. cluster level -> global level

kind: <type-of-resource>
apiVersion: 

there are resources categorised based on apiVersion..

pod
===
pod is smallest deployable unit in kubernetes. a pod can have multiple containers in it. containers inside pod share same network and storage

docker run -d -p 80:80 --name frontend nginx:1.24

kubectl describe pod pod-name -n namespace

assign to node
node pulls the image
create container
start container

kubectl exec -it multi-container -c nginx -- bash

labels
======
lables adds metadata to the resource, labels are selectors inside kubernetes..

CrashLoopBackOff -> when container is not able to start
ImagePullBackOff/ErrorImagePull -> when your node is not able to pull the image.. image address may be wrong or authentication problems

annotations:
======
annotations are again like labels adds metadata, but annotations are used to select external resources..
labels have limitations in key value length and size. special charecters are not allowed inside labels

resources
=======
you need to always restrict the resources to the pod..if you get more requests we can autoscaling

1cpu = 1000m

env
====
env variables useful for pod

siva -> saiavaaa

service
=======
pod1 -> pod2

if we want pod to pod communication, ip address of pods are not useful since they are ephemeral. we can use k8s service to acheive
1. pod to pod communication through DNS, service name works as DNS here
2. load balancing

cart -> catalogue

pod1 -> service -> pod2, pod3, pod4
Types of Services
1. ClusterIP
2. NodePort
2. NodePort
3. LoadBalancer

service ip: 10.100.10.134
endpoints(pods): 192.168.5.23:80

svc-test -> service -> labels

url/repo-name/image-name:version

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
curl -sLO “https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz”  =>
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz  =>
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl  =>
eksctl version  =>
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl  =>
chmod +x ./kubectl  =>
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH  =>
kubectl  version —client  =>
aws configure  =>
Then give AWS Configure keys.
kubectl get nodes  =>
kubeclt api-resource  =>
Namespace an isolated space inside k8 cluster where can create our project related resources. we can completely control them. it is like our project inside k8s. 1. namespace level -> VPC level. 2. cluster level -> global level.
Create “k8-resources” git repo and clone it in your local machine. And create “01-namespace.yaml” file ans start writing code.
kubectl get namespace   =>
cd   =>
kubectl create namespace roboshop   =>
kubectl get namespace   =>
kubectl delete namespace roboshop   =>
In Google search for Kubenetes namespace yaml to create namespace through .yaml file. And get required code.
Write yaml code push changes.
Clone “k8-resources” git repo in AWS EC2 Kubernetes “workstation” instance.
git clone “k8-resources-git-repo”   => 
cd k8-resources/   =>
kubectl apply -f 01-namespace.yaml   =>
kubectl get ns   => To get all Kubernetes Namaspaces, ns is shortcut for namespace.
kubectl describe ns roboshop  =>
kubectl describe ns default   => 
kubectl delete -f 01-namespace.yaml   =>
Pod is smallest deployable unit in kubernetes. A pod can have multiple containers in it. Containers inside pod share same network and storage.
Create “02-pod.yaml” file then write pod code then push & pull code.
kubectl apply -f 02-pod.yaml   =>
kubectl get pods   =>
kubectl describe pod nginx   =>
kubectl get pods -o wide   => To get IP address and node data
kubectl delete -f 02-pod.yaml   =>
kubectl apply -f 02-pod.yaml   =>
kubectl create namespace roboshop   =>
kubectl apply -f 02-pod.yaml   =>
kubectl get pods -o wide   => Fetching Pods from default namespace.
kubectl get pods -o wide -n roboshop  => Fetching Pods from default namespace. -n means namespace.
Create “03-multi-container.yaml” file then write pod code then push & pull code.
kubectl apply -f 03-multi-container.yaml   =>
kubectl get pods   => To check status of pods(containers)
kubectl get pods   => To check status of pods(containers). Every time status got change.
kubectl get pods   => To check status of pods(containers). Some time getting errors.
kubectl get pods   =>
kubectl describe pod multi-container  => Almalinux don’t have any command to run nginx sp that container is not running to we need to add command in 03-multi-container.yaml file.
Add sleep code in 03-multi-container.yaml file and push & pull changes =>
git pull  => 
kubectl apply -f 03-multi-container.yaml   =>
If code is existing we can’t apply some code changes like “sleep” so need to delete existing pod and apply again.
kubectl delete -f 03-multi-container.yaml   =>
kubectl apply -f 03-multi-container.yaml   =>
kubectl get pods   =>
kubectl describe pod multi-container  =>
kubectl exec -it multi-container -c nginx — bash => Syntax to login into container.
exit  => Run inside multi-container.
kubectl apply -f 02-pod.yaml   =>
 kubectl exec -it nginx -c roboshop — bash =>
exit  => Run inside nginx.
kubectl exec -it multi-container -c nginx — bash =>
curl http://localhost   => Run inside multi-container.
exit  => Run inside multi-container.
kubectl exec -it multi-container -c almalinux— bash =>
curl http://localhost   => Run inside almalinux.
exit  => Run inside almalinux.
Labels adds metadata to the resource, labels are selectors inside kubernetes. Labels have limitations in key value length and size. special charecters are not allowed inside labels.
Create “04-labels.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl apply -f 04-labels.yaml   =>
kubectl describe pod labels =>
kubectl get pods   =>
kubectl get pods labels -n roboshop   =>
Check labels added in code.
Annotations are again like labels adds metadata, but annotations are used to select external resources. Annotations have no limitations, can define long key value pairs.
Create “05-annotation.yaml” file then write pod code then push & pull code.
Resources should always restrict the resources to the pod. If you get more requests we can autoscaling.
Create “06-resource.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl apply -f 06-resources.yaml   =>
kubectl get pods   =>
kubectl describe pod resources =>
Env variables useful for pod.
Create “07-env.yaml” file then write pod code then push & pull code.
kubectl get pods   =>
kubectl apply -f 07-env.yaml   =>
kubectl describe pod env-demo =>
kubectl exec -it env-demo — bash =>
env => Run inside env-demo. Check environmental variables.
exit => Run inside env-demo.
Create “08-configmap.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl apply -f 08-configmap.yaml   =>
kubectl get cm   => 
kubectl describe cm nginx-config =>
Create “09-pod-configmap.yaml” file to attach configmap to a pod, then write pod code then push & pull code. cm stands for configmap. 
git pull  =>
kubectl apply -f 09-pod-configmap.yaml   =>
kubectl exec -it pod-configmap-demo -- bash =>
echo “admin” | base64  =>
echo “admin12” | base64  => Copy output of this command and add in “10-screts.yaml” file Kubernetes code.
echo “encripted-id-of-above” | base64 —decode  => example echo "YWRtaW4K" | base64 --decode
Create “10-screts.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl apply -f 10-secrets.yaml   =>
Create “11-pod-secrets.yaml” file to attach secrets to a pod, then write pod code then push & pull code.
git pull  =>
kubectl apply -f 11-pod-secrets.yaml   =>
kubectl exec -it pod-secrets-demo — bash =>
env  => Run inside pod-secrets-demo
exit  => Run inside pod-secrets-demo
Create “12-service.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl get pods   =>
kubectl apply -f 04-labels.yaml   => Both Service & Labels are not is same space so comment out roboshop namespace in 04-labels.yaml file.
kubectl get pods   =>
git pull  =>
kubectl delete -f 04-labels.yaml   =>
kubectl get pods   =>
kubectl apply -f 12-service.yaml   =>
kubectl get svc   => svc is short cut for service
kubectl describe svc nginx   =>
kubectl get pods -o wide   => To check labels IP address.
Create “13-service-test.yaml” file then write pod code then push & pull code.
kubectl get pods   =>
kubectl apply -f 13-service-test.yaml   =>
kubectl exec -it svc-test— bash =>
curl nginx  => Run inside svc-test.
 eksctl delete cluster --name=roboshop --region=us-east-1  => Command to delete Kubernetes Cluster.
eksctl delete cluster -f eks.yaml --force   => Command to force delete Kubernetes Cluster.

Diagrams:
pod 


Timestamps:
Interview questions





Mistakes & Learning:




Doubts Link Clarification AI chat link: