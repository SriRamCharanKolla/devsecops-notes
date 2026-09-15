Thursday, 7 May 2026

Session 71 - PDB and TopologySpreadConstraint, K8 Architecture.
Class Notes
InitContainers
How can they fetch secrets

Networking -> VPC CNI

PDG
TopologyConstraints

Deployment, hpa

min -> 2nodes
min replicas -> 2

node-1 in us-east-1a -> 2 pods
node-2 in us-east-1b

spread pods across the nodes in HA manner -> topologySpreadConstraint

upgrade
drain
delete pod
autoscaling down

there should be always a minimum number of replicas running in voluntary disruption cases -> PDB

min available = total pods - maximum pods on one node/total pods

3 nodes
5 pods

2:2:1

5-2/5 = 60%
3 nodes
5 pods
2:2:1 -> 60%

node-1 drain -> success
node-2 drain -> success
node-3 drain -> success

2 pods 2 nodes

node-1 ->1 pod -> 2 pods
node-2 ->2nd pod -> 2 pods

node-1 drain -> fail

min 3 nodes 1:1:1 -> topologySpreadConstraint

2 zones ->
2 pods -> 1 pod 1st zone, 2nd pod another zone
3 pods -> 2 pods 1st zone, 3rd pod 2nd zone -> maxSkew -> 1pod

4 nodes
zone-1 -> 2 nodes
zone-2 -> 2 nodes

4 replicas

2 pods in zone-1 -> 1st pod in node-1 and 2nd pod in node-2
2 pods in zone-2 -> 3rdt pod in node-3 and 4th pod in node-4

7 pods 2 nodes
max pods -> 4 
node-1 -> 4
node-2 -> 3 -> down

4/7 -> 50. != 60

3 nodes 5 pods
2:2:1

k8 architecture
================

control plane
============
1. api server -> entrypoint for everything, checks authentication and authorization, call another components

2. scheduler -> schedules the pod on available node, node resources, node selectors, affinity, anti affinity, taints and toleration, topologySpreadConstraint, etc.. it always checks pending pods can be scheduled or not

3. control manager
	replica/deployment controller -> always checks desired number of replicas running or not
	Node controller:  desired number of nodes are always up
	EndpointSlice controller: always populates pod ips to service endpoints
	ServiceAccount controller: Create default ServiceAccounts for new namespaces.
	
4. etcd -> memory for k8s cluster, manifest files, config maps, secrets, etc... key-value pair database

5. cloud-controller-manager -> cloud specific api handle

node components
===========
kubelet -> always in contact with api-server, it is like agent running in every node. node health status report sends to control plane, responsible to run the pods in node

kube-proxy -> manages networking rules to bring traffic to pods, iptables management

Container runtime -> containerd is the run time...

Addons 
======


namespaces
pod
service
configmap
secrets
pv, pvc, sc
taints and tolerations
affinity and anti affinity
replicaset
deployment
statefulset
deamonset
rbac
ingress
volumes
network policies
pdb and topologySpreadConstraint
upgrade cluster
blue green deployments
hpa
helm charts

blue -> 1.34
green -> 1.35

8 nodes
2 nodes

taints and toleration and affinity -> 2 nodes
topologySpreadConstraint -> topologyKey is host, topologyKey zone and host



GIT
====
1. repo creation
2. adding to staging area, commit to local repo
3. push to central repo

git init
git branch -M main -> create and move to main
git remote add origin <url> -> connect to central repo

git pull


branches
========
dev sit/uat/qa prod

main -> prod ->
create another branch, do the changes, test it and then push to main
PR -> when you bring change from another branch to main branch


merge
=====
1. you got extra commit, it is special commit it has 2 parents, merge commit
2. it is preserving history

rebase
=====

git branch -M main -> creating main and main branch
git checkout -b karam-dosa -> create new branch karam-dosa and move inside to it

it will not preserve any history
it is linear history
seems like main branch moved forward
it is rewriting commits

Commands:
Pod Disruption Budget:     spread pods across the nodes in HA manner -> topologySpreadConstraint. there should be always a minimum number of replicas running in voluntary disruption cases -> PDB. min available = total pods - maximum pods on one node/total pods.
Create “k8-ha” New GitHub Repo.
git clone https://github.com/daws-88s/k8-ha.git     =>  Run on your local system.
Create “manifest.yaml” file in “k8-ha” repo then write required code.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
git clone https://github.com/daws-88s/k8-ha.git     =>
cd k8-ha   =>
kubectl apply -f manifest.yaml    =>
kubectl get pods -o wide     =>
kubectl describe pod <pod-ID>    =>  Command & Syntax, you should place your <pod-ID>.
Pods are in Pending state because we got error due to wrong Topology key for EKS. Fix Error then push code.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
git pull    =>
kubectl apply -f manifest.yaml    =>
kubectl get pods -o wide     =>
kubectl delete -f manifest.yaml    =>
kubectl apply -f manifest.yaml    =>
kubectl get pods -o wide     =>
Now add Pods disruption budget code in “manifest.yaml” file in “k8-ha” repo.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
kubectl apply -f manifest.yaml    =>
kubectl get pdp     =>
kubectl drain <enter-your-node-name> —ignore-demonsets —delete-emptydir-data    => Command & Syntax
kubectl drain ip-10-0-11-77.ec2.internal —ignore-demonsets —delete-emptydir-data    => 
git pull    =>
kubectl apply -f manifest.yaml    =>
kubectl drain ip-10-0-11-77.ec2.internal —ignore-demonsets —delete-emptydir-data    =>
kubectl get pods     =>
kubectl delete pod <ngix-pod-name>     => Command & Syntax and you should place your <ngix-pod-name>
kubectl get pods     =>
kubectl describe pod <pod-ID>    =>  Command & Syntax, you should place your <pod-ID>.
kubectl get nodes     =>
kubectl uncordon <node-name>     => Command & Syntax and you should place your <ngix-pod-name>
kubectl get pods     =>
kubectl get pods -o wide     =>
Kubernets Architecture:    
Control plane :     
api server -> entrypoint for everything, checks authentication and authorization, call another components
scheduler -> schedules the pod on available node, node resources, node selectors, affinity, anti affinity, taints and toleration, topologySpreadConstraint, etc.. it always checks pending pods can be scheduled or not.
Control manager :   
Replica/deployment controller -> always checks desired number of replicas running or not.
Node controller:  desired number of nodes are always up.
EndpointSlice controller: always populates pod ips to service endpoints.
ServiceAccount controller: Create default ServiceAccounts for new namespaces.
etcd -> memory for k8s cluster, manifest files, config maps, secrets, etc... key-value pair database.
cloud-controller-manager -> cloud specific api handle.
Node components :     
kubelet -> always in contact with api-server, it is like agent running in every node. node health status report sends to control plane, responsible to run the pods in node
kube-proxy -> manages networking rules to bring traffic to pods, iptables management
Container runtime -> containerd is the run time…
Addons :    namespaces, pod, service, configmap, secrets, pv, pvc, sc, taints and tolerations, affinity and anti affinity, replicaset, deployment, statefulset, deamonset, rbac, ingress, volumes, network policies, pdb and topologySpreadConstraint, upgrade cluster, blue green deployments, hpa, helm charts.
blue -> 1.34, green -> 1.35, 8 nodes.
2 nodes.
taints and toleration and affinity -> 2 nodes.
topologySpreadConstraint -> topologyKey is host, topologyKey zone and host.
GIT:   repo creation, adding to staging area, commit to local repo, push to central repo
git init
git branch -M main -> create and move to main
git remote add origin <url> -> connect to central repo
git pull
Branches:   dev sit/uat/qa prod
main -> prod ->
create another branch, do the changes, test it and then push to main
PR -> when you bring change from another branch to main branch
Merge:   you got extra commit, it is special commit it has 2 parents, merge commit, it is preserving history
Rebase:     
git branch -M main -> creating main and main branch.
git checkout -b karam-dosa -> create new branch karam-dosa and move inside to it
it will not preserve any history
it is linear history
seems like main branch moved forward
it is rewriting commits
Create “dosa-shop” new git Repo.
git clone https://github.com/daws-88s/dosa-shop.git     =>  Run on your local system.
Create “readme.md” file in “dosa-shop” repo then write required markdown code like “Dosa-Shop”.
git add . ; git commit -m “k8”; git push origin main    =>  Run on your local system.
Then go to GitHub then click on “Settings” then click on “Rules” then click on “Rulessets” then click on “New ruleset” then click on “New branch ruleset” then enter “main” in “Ruleset Name” then select “Active” for “Enforcement status” then click on “Add target” under “Target branches” then select “Include my pattern” then enter “main” in “Branch naming pattern” then click on “Add inclusion pattern” then under “Branch rules” select “Restrict deletion” then select “Block force pushes” then select “Required a pull request before merging” then select “1” for “Required approvals” then save changes.
git branch -M plain-dosa   => Run on your local system. T creat plain-dosa branch.
git add . ; git commit -m “plain dosa started”; git push origin plain-dosa    =>  Run on your local system. Adding necessary file to plain-dosa branch the adding “plain dosa started” commit then pushing code to “plain-dosa” branch.
git log   =>  Run on your local system. To check your Repo logs.
git cat-file -p 39f2863    => Run on your local system.
git add . ; git commit -m “dosa butter added”; git push origin plain-dosa    =>  Run on your local system.  
git add . ; git commit -m “light oil added”; git push origin plain-dosa    =>  Run on your local system.
git log   =>  Run on your local system. To check your Repo logs.
Now again go to GitHub then click on “Settings” then click on “Collaborators and teams” under “Manage access” click on “Add people” then enter GitHub Repo of your team members then choose “Write” then click on “Add selection” then they will receive collaboration request main then they should approve to work that GitHub Repository.
Now click on “Pull requests” tab then clicm on “New pull request” then enter “Plain dosa” in “Add a title” then enter “Hi, I developed plain dosa using better and light oil applied” in “Add a description” then click on “Create pull request” then “Review required” & “Merging is blocked” should display.
Now any other team member will check our PR then review changes then write “Commit” if any anything need to update by submitting review, then PR raised person will reply for the comment then otherwise if everything is fine then they PR reviewer will approve the PR or after convinced with PR raised person point.
Now the PR raised person can merge his Branch to main branch. Now all changes will come to main branch.
After merge then they should inform to every in the Team then every one should pull changes from main branch.
git pull origin main  => Run on your local system. 
git log   =>  Run on your local system. To check your Repo logs.
git cat-file -p 90e3    => Run on your local system. To verify what changes or code in this commit ID.
git checkout main    => Run on your local system.
git log   =>  Run on your local system. To check your Repo logs.
Once our Branch code is successfully merged to Main branch then we can delete our branch.
When create new Branch in any Git Repo then should run “git pull” to get latest updated code from main branch.
git pull   => Run on your local system.
git checkout -b karam-dosa    => To create new “karam-dosa” branch.
Now do development in “karam-dosa” branch by writing required code. 
git add . ; git commit -m “dosa butter added”; git push origin plain-dosa    =>  Run on your local system. 
 git add . ; git commit -m “oil”; git push origin plain-dosa    =>  Run on your local system.
 git add . ; git commit -m “karam”; git push origin plain-dosa    =>  Run on your local system.
 git add . ; git commit -m “erra karam”; git push origin karam-dosa    =>  Run on your local system.
 Now raise PR for “karma-dosa”.
 The someone will comment then replay for their comment then they will approve your PR then now instead of merge choose “Rebase and merge”.
 git log —online  =>  Run on your local system. To check your Repo logs.
 git checkout main    => Run on your local system.
 git pull origin main   =>  Run on your local system.
  git log —online  =>  Run on your local system. To check your Repo logs.
  it cat-file -p 97c0d6d    => Run on your local system. To verify what changes or code in this commit ID.
 


Diagrams:
git-branch


Git Repo Links:



Timestamps:
Assignment -
Interview questions:





Mistakes & Learning:




Doubts Link Clarification AI chat link: