Wednesday, 15 April 2026

Session 63 - K8S selectors( Node selector, Taints and tolerations, Node affinity anti affinity, Pod affinity and anti affinity, Use case ).
Class Notes
PV
PVC
StorageClass

Statefulset vs Deployment
headless service

selectors -> how you control the pod scheduling. Scheduler in Master node
nodeSelector
taints and tolerations

we can assign labels to the nodes, and ask scheduler to select the node based on this label.

alpha.eksctl.io/cluster-name=roboshop,
alpha.eksctl.io/nodegroup-name=spot,
beta.kubernetes.io/arch=amd64,
beta.kubernetes.io/instance-type=c5.large,
beta.kubernetes.io/os=linux,
eks.amazonaws.com/capacityType=SPOT,
eks.amazonaws.com/nodegroup-image=ami-00b16894e17387eb9,
eks.amazonaws.com/nodegroup=spot,
eks.amazonaws.com/sourceLaunchTemplateId=lt-076a33d8f5ddc65cc,
eks.amazonaws.com/sourceLaunchTemplateVersion=1,
failure-domain.beta.kubernetes.io/region=us-east-1,
failure-domain.beta.kubernetes.io/zone=us-east-1c,
k8s.io/cloud-provider-aws=d573a2d478e01899d878f9c9cc3fcc2a,
kubernetes.io/arch=amd64,
kubernetes.io/hostname=ip-192-168-22-49.ec2.internal,
kubernetes.io/os=linux,
node.kubernetes.io/instance-type=c5.large,
topology.k8s.aws/zone-id=use1-az4,
topology.kubernetes.io/region=us-east-1,
topology.kubernetes.io/zone=us-east-1c

kubectl label nodes ip-192-168-22-49.ec2.internal project=roboshop

when pods will go to pending state? - interview question very important 
1. if there is no resources available on the node
2. if pod is not able to mount the volume

taints and tolerations
=====================
if you taint the node, k8s will not schedule any pod on to that node.
1. project specific hardware. GPU
2. any specific workloads like database accept only from specific ip

if we want our pod on to tainted node, then we should use toleration, toleration will not gaurentee pod scheduling

1. we need to select the node
2. toleration acts as exception/reservation to that node

NoSchedule -> incoming pods no schedule
NoExecute -> running pods should be evicted

kubectl taint nodes ip-192-168-22-49.ec2.internal project=roboshop:NoExecute

running pods in this node will be evicted.

NodeAffinity
=============
NodeSelector does not have much options, just key value pairs. but affinity have more options. we can have operators In, Exist, NotExist, etc...

requiredDuringSchedulingIgnoredDuringExecution -> Hard Rule. labels should match otherwise pod can't be scheduled

preferredDuringSchedulingIgnoredDuringExecution -> Soft rule. if label selectors are not match, k8 considers another node to schedule

1a 1b 1c 1d

1a 1b

what is topology.kubernetes.io/zone=us-east-1c
============
we are telling k8 to consider this zone as border

alpha.eksctl.io/cluster-name=roboshop,alpha.eksctl.io/nodegroup-name=spot,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=c5.large,beta.kubernetes.io/os=linux,eks.amazonaws.com/capacityType=SPOT,eks.amazonaws.com/nodegroup-image=ami-00b16894e17387eb9,eks.amazonaws.com/nodegroup=spot,eks.amazonaws.com/sourceLaunchTemplateId=lt-076a33d8f5ddc65cc,eks.amazonaws.com/sourceLaunchTemplateVersion=1,failure-domain.beta.kubernetes.io/region=us-east-1,failure-domain.beta.kubernetes.io/zone=us-east-1c,k8s.io/cloud-provider-aws=d573a2d478e01899d878f9c9cc3fcc2a,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-22-49.ec2.internal,kubernetes.io/os=linux,node.kubernetes.io/instance-type=c5.large,project=roboshop,topology.k8s.aws/zone-id=use1-az4,topology.kubernetes.io/region=us-east-1,topology.kubernetes.io/zone=us-east-1c

39-136 -> project = amazon
22-49 -> project = roboshop

1d-2
1c-1

taints
tolerations

we select the node using below strategies
node affinity
node anti affinity
requiredDuringSchedulingIgnoredDuringExecution
preferredDuringSchedulingIgnoredDuringExecution

Pod Affinity
============

backend cache
1. backend and cache in same node
2. backend and cache in diff node

backend and cache -> affinity
I want high available, I want 3 replicas

backend -> backend -> anti-affinity
cache -> cache -> anti-affinity

nodeSelector -> simply selecting the node based on labels
nodeAffinity/nodeAntiAffinity -> improved version of nodeSelector
podAffinity -> affinity towards pods
podAntiAffinity -> repel towards pods

requiredDuringSchedulingIgnoredDuringExecution
preferredDuringSchedulingIgnoredDuringExecution

taint and tolerations

scheduler will schedule -> node executes the pod

4 boxes


Commands:
aws eks update-kubeconfig —region us-east-1 —name roboshop   =>
kubectl get nodes —show-lables  =>
kubectl get nodes  =>
git clone “k8-selectors-git-link”   =>
cd k8-selectors  =>
kubectl apply -f 01-pod.yaml   =>
kubectl get pods -o wide  =>
kubectl taint nodes ip-192-168-22-49.ec2.internal project=roboshop:NoExecute   => ip-192-168-22-49.ec2.internal replace with your’s node-ip/id, Applying Taint
kubectl get pods  =>
kubectl apply -f 01-pod.yaml   =>
kubectl get pods  =>
Now add nodeSelector & NoExecute effect code in 02-toleration.yaml file in “k8-selectors”  =>
git pull  =>
kubectl apply -f 01-pod.yaml   =>
kubectl delete -f 01-pod.yaml   =>
kubectl apply -f 02-toleration.yaml  =>
kubectl get pods  =>
kubectl get pods -o wide  =>
Create “03-node-afinity.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
kubectl taint nodes ip-192-168-22-49.ec2.internal project=roboshop:NoExecute-   => ip-192-168-22-49.ec2.internal replace with your’s, Applying Untaint
kubectl delete -f 02-toleration.yaml  =>
git pull  =>
kubectl apply -f 03-node-afinity.yaml  =>
kubectl get pods -o wide  =>
kubectl get nodes —show-lables  =>
kubectl label node <node-ip/id> project=amazon  => ip-192-168-22-49.ec2.internal replace with your’s node-ip/id,, Applying Taint
git pull  =>
kubectl delete -f 03-node-afinity.yaml  =>
kubectl apply -f 03-node-afinity.yaml  =>
kubectl get pods  =>
kubectl get pods -o wide  =>
kubectl delete -f 03-node-afinity.yaml  =>
git pull  =>
kubectl apply -f 03-node-afinity.yaml  =>
kubectl get pods -o wide  =>
kubectl delete -f 03-node-afinity.yaml  =>
Create “04-anti-afinity.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 04-anti-afinity.yaml =>
kubectl get pods -o wide  =>
git pull  =>
kubectl delete -f 04-anti-afinity.yaml =>
kubectl apply -f 04-anti-afinity.yaml =>
kubectl get pods  =>
kubectl get nodes  =>
kubectl label node <node-ip/id> project=amazon  => ip-192-168-22-49.ec2.internal replace with your’s node-ip/id,, Applying Taint
git pull  =>
kubectl delete -f 04-anti-afinity.yaml =>
kubectl apply -f 04-anti-afinity.yaml =>
kubectl delete -f 04-anti-afinity.yaml =>
git pull  => After apply “NotIn”
kubectl apply -f 04-anti-afinity.yaml =>
kubectl get pods -o wide  =>
kubectl taint node <node-ip/id> hardware=gpu:NoExecute  => Applying Taint
kubectl get pods -o wide  =>
git pull  =>
kubectl apply -f 04-anti-afinity.yaml =>
kubectl delete -f 04-anti-afinity.yaml =>
kubectl taint node <node-ip/id> hardware=gpu:NoExecute-  => Applying Untaint
Create “05-pod-afinity.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 05-pod-afinity.yaml =>
kubectl get pods -o wide  =>
Apply Pod Affinity means pod2 should run at location where pod1 is running.
git pull  =>
kubectl apply -f 05-pod-afinity.yaml =>
kubectl get pods -o wide  =>
kubectl delete -f 05-pod-afinity.yaml =>
Create “06-pod-anti-afinity.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 06-pod-anti-afinity.yaml   =>
kubectl get pods -o wide  =>
kubectl delete -f 06-pod-anti-afinity.yaml   =>
Create “07-use-case.yaml” file in “k8-selectors” git project. Then write Kubernetes code then push & pull code.
kubectl apply -f 07-use-case.yaml  =>
kubectl get pods -o wide  =>
Now Add application pod in 07-use-case.yaml.
git pull  =>
kubectl apply -f 07-use-case.yaml  =>
kubectl get pods -o wide  =>
Add another replica.
git pull  =>
kubectl apply -f 07-use-case.yaml  =>
kubectl get pods -o wide  =>
Add 5 replicas then remaining 2 will in pending status due to Anti-Affinity.
git pull  =>
kubectl apply -f 07-use-case.yaml  =>
kubectl get pods -o wide  =>
Schedule remains 2 pending replicas.
git pull  =>
kubectl delete -f 07-use-case.yaml  =>
kubectl apply -f 07-use-case.yaml  =>
kubectl get pods -o wide  =>



Diagrams:
affinity

GitHub Repos:
k8-selectors


Timestamps:
Interview questions = 





Mistakes & Learning:




Doubts Link Clarification AI chat link: