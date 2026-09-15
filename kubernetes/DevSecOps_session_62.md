Wednesday, 15 April 2026

Session 62 - Statefulset, Statefulset vs deployment, Headless service, Horizontal vs vertical autoscaling, HorizontalPodAutoscaler, Taints and tolerations.
Class Notes
emptyDir
hostPath
configmap as volume

ebs static
ebs dynamic
efs static
efs dynamic
ebs vs efs
pv, pvc, sc

statefulset -> this is for stateful applications, where data storage is important
deployment -> this is for stateless applications, data storage is not required

1. pod names are standard in statefulset mysql-0, mysql-1, mysql-2, etc. deployment pod names are random
2. pods in statefulset are created in order from 0 to n, deleted in reverse order one by one. pod creation/deletion is random
3. pod in statefulset have its own disk. pods in deployment have shared disk.
4. statefulset should have pv, pvc as mandatory. pv, pvc is not required for deployment
5. statefulset requires headless service to find other nodes in the cluster when data replication is required, deployment does not require headless service..
6. statefulset pods will have same identity

a service with clusterIP none is called headless service

1. drivers install
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58"
2. IAM role for instances
3. storage class create

Autoscaling
============
1. horizontal scaling
2. vertical scaling

vertical scaling
================
1GB RAM
8GB HD
2 CPU

stop the server
increases the resources
restart
Disadvantages:
===========
1. downtime
2. single point of failure

horizontal scaling
============
you dont increases resources in single server, instead we create replicas
Disadvantages:
===========
1. no downtime
2. no single point of failure

Kubernetes Horizontal pod Autoscaling:
1. you must configure resource limits
2. metrics server should be running
3. Horizontal pod Autoscaling attached to deployment.
4. Automatically create a replica.
5. Maximum 10 replicas can be created.


taints and tolerations
===================
taint == paint

if you taint a node, it means kubernetes scheduler will not schedule any pods in that node..

1. we reserver that node to other special workloads
2. may be we can attach GPU node to the cluster, we reserve that for AI/ML workloads
3. DB may accept connections from specific IP, we only send the pods that connects to DB in the tainted node.

kubectl taint nodes node1 key1=value1:NoSchedule

NoSchedule -> dont schedule the new pods
PreferNoSchedule -> try not to schedule the new pods. scheduler may schedule the pods
NoExecute -> dont schedule new pods and evict existing pods

Commands:
git clone “k8-resources-git-link”   =>
In “k8-resources” in “17-deployment.yaml” file add a service.
Now in new tab connect to K9s.
cd k8-resources   =>
kubectl apply -f 17-deployment.yaml  =>  
kubectl apply -f 04-labels.yaml  =>
Go labels pod in K9s terminal tab.
nslookup nginx   => To verify nslookup is installed or not. Run inside K9s terminal tab inside labels pod shell.
cat /etc/*release   => To know OS version of the pod. Run inside K9s terminal tab inside labels pod shell.
apt update  => Run inside K9s terminal tab inside labels pod shell.
apt install dnsutils -y  => Run inside K9s terminal tab inside labels pod shell.
nslookup nginx   => Run inside K9s terminal tab inside labels pod shell.
Now create headless service inside “17-deployment.yaml” file then push changes.
git pull  =>
kubectl apply -f 17-deployment.yaml  =>
kubectl get pods -o wide  =>
nslookup nginx-headless   => Run inside K9s terminal tab inside labels pod shell.
Create “k8-roboshop-volumes” new git repo then copy “01-namespace.yaml” file & also copy “mongodb” folder from “k8-roboshop”.
Then in mongodb’s “manifest.yaml” file write Kubernetes statefulset.
kubectl apply -k “github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58”  =>
Now select “us-east-1c” “roboshop-spot-Node” then click on “Security” tab then click on “IAM role” ID then click on “Add permissions” then search for “ebs” then select “AmazonEBSCSDriverPolicy” then click on “Add permissions”.
Now create storage class.
cd  =>
git clone “k8-roboshop-git-link”   =>
cd k8-roboshop/volumes  =>
kubectl apply -f 01-namespace.yaml   =>
kubectl apply -f 04-ebs-sc.yaml   =>
cd mongodb =>
kubectl apply -f manifest.yaml   =>
exit  => Run inside K9s terminal tab inside labels pod shell.
Now in K9s search for “namespace” then select “roboshop”.
Now “mongodb-0” will be shown.
Now create “catalogue” folder in “k8-roboshop-volumes” then create “manifest.yaml” file then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f manifest.yaml   =>

Now Kubernetes Horizontal pod Autoscaling.
Now inside catalogue manifest.yaml file write Kubernetes Horizontal pod Autoscaling code to create & then push & pull code.
cd ../catalogue/   =>
git pull  =>
kubectl apply -f manifest.yaml   =>
kubens roboshop   =>
kubectl exec -it mongodb-0 —bash  => 
while true; do curl http://catalogue:8080/health; done   => Run after mongodb bash above command.
apt update   => Run after mongodb bash above command. 
apt install curl   => Run after mongodb bash above command.
while true; do curl http://catalogue:8080/health; done   => Run after mongodb bash above command.
Now check in K9s load is increasing on catalogue so that autoscaling is implementing automatically by Horizontal pod Autoscaling.
exit  => Run after mongodb bash above command. 
cd  =>
cd k8-resources   =>
ls -l  =>
kubectl delete -f 17-deployment.yaml  =>
kubens default   =>
kubectl delete -f 17-deployment.yaml  =>
kubectl delete -f 04-labels.yaml  =>
kubens roboshop   =>
cd ../k8-resources-volumes/   =>
kubectl delete -f catalogue/manifest.yaml   =>
kubectl delete -f mongodb/manifest.yaml   =>
kubectl get pv  =>
kubectl get pvc  =>
kubectl delete pvc mongodb-mongodb-0 mongodb-mongodb-1 mongodb-mongodb-2     =>
Go to K9s then search for “pvc” then delete all 3 pvc’s then delete pic volumes in AWS volumes.
cd  =>
Create “k8-selectors” new git repo then clone in your local machine.
kubectl get nodes  =>
kubectl taints nodes <paste-node-id> project=roboshop:NoSchedule   => This a Scheduler listen pods changes.
Create “01-pod.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
git clone “k8-selectors-git-link”   =>
cd k8-selectors  =>
kubens default   =>
kubectl apply -f 01-pod.yaml   =>
kubectl get pods -o wide  =>
Create “02-toleration.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 02-toleration.yaml  =>
Pod not went to expected node, so try with delete and create once with toleration.yaml.
kubectl delete -f 02-toleration.yaml  =>
kubectl get pods -o wide  =>
kubectl apply -f 02-toleration.yaml  =>
Agin pod not went to expected node, so try with delete and create once with toleration.yaml. This is because Scheduler is not garnet to create pod in expected node.
kubectl delete -f 02-toleration.yaml  =>
kubectl apply -f 02-toleration.yaml  =>

Diagrams:
k8-roboshop
statefulset


GitHub Repos:
k8-roboshop-volumes
k8-resources
k8-selectors


Timestamps:
Assignment = Create Stepupset for remaining databases like MySQL, RabbitMQ etc. = 01:05: 20
Interview questions = 





Mistakes & Learning:




Doubts Link Clarification AI chat link: