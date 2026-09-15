Monday, 6 April 2026

Session 57 - K8 Resources, Services ( ClusterIp, NodePort, Load Balancer ), ReplicaSet, Deployment, Deployment rollback.
Class Notes
1. Build the image
2. Run the image

Namespace -> isolated project space where we create our project resources
Pod -> smallest deployable unit in kubernetes, pod contains multiple containers. every container share 	   same network and storage
Labels -> adds metadata to the pods, labels are selectors. we can attach labels to any resource in k8s
env -> can pass env variables to the containers
resources -> we need to restrict pod resources. request and limits soft limit and hard limit
annotations -> agains adds metadata, but used to select external resources like load balancers, IAM roles, etc
config map -> configuration resource, we can attach config map to the pod
secrets -> confidential information, can attach to pod environment, but only encoded information not encrypted
services
1. pod to pod communication since podip is ephemeral. we can communicate with service name
2. load balancing between pods
3. service select pods based on labels as selectors

1. cluster ip -> internally exposing pod with in the cluster
2. node port -> cluster ip is subset of nodeport.. every node inside eks cluster open the nodeport. 30000–32767
3. load balancer -> node port is subset of load balancer. it will create a classic load balancer

Sets
=====
1. ReplicaSet -> create multiple replicas of your pod. always runs desired number of pods
2. Deployment
3. DaemonSet
4. StatefulSet


deployment/release
==========
you have v1 running, v2 is ready

1. stop the application server or component. systemctl stop catalogue
2. remove the v1 code
3. download v2 code
4. restart catalogue. systemctl restart catalogue

since pods are immutable, we can create another set of v2 pods, then delete v1 pods

etcd -> k8 cluster storage...

85  03/04/26 02:55:51 kubectl describe pod nginx-78cc6c5555-b55qs
   86  03/04/26 02:56:01 kubectl rollout history deployment/nginx
   87  03/04/26 02:56:14 kubectl undo deployment nginx
   88  03/04/26 02:56:23 kubectl rollout undo deployment nginx
   89  03/04/26 02:56:34 kubectl rollout history deployment/nginx
   90  03/04/26 03:05:32 cd ..
   91  03/04/26 03:05:40 git clone https://github.com/daws-88s/k8-roboshop.git
   92  03/04/26 03:05:46 cd k8-roboshop/
   93  03/04/26 03:05:47 clear
   94  03/04/26 03:05:54 kubectl apply -f 01-namespace.yaml
   95  03/04/26 03:05:59 cd mongodb/
   96  03/04/26 03:06:04 kubectl apply -f manifest.yaml
   97  03/04/26 03:06:15 kubectl get deployment -n roobshop
   98  03/04/26 03:06:24 kubectl get deployment -n roboshop
   99  03/04/26 03:06:35 kubectl get rs -n roboshop
  100  03/04/26 03:06:46 kubectl get pods -o wide -n roboshop
  101  03/04/26 03:06:56 kubectl get svc -n roboshop
  102  03/04/26 03:07:10 kubectl describe svc mongodb -n rooboshop
  103  03/04/26 03:07:14 kubectl describe svc mongodb -n roboshop
  104  03/04/26 03:12:15 git pull
  105  03/04/26 03:12:21 cd ../catalogue/
  106  03/04/26 03:12:32 kubectl apply -f manifest.yaml
  107  03/04/26 03:12:37 kubectl get pods
  108  03/04/26 03:12:43 kubectl get pods -n roboshop
  109  03/04/26 03:13:08 kubectl exec -it catalogue-7758f68dc7-j46rc -- bash
  110  03/04/26 03:13:12 kubectl exec -it catalogue-7758f68dc7-j46rc -- sh
  111  03/04/26 03:13:18 kubectl get pods -n roboshop
  112  03/04/26 03:13:34 kubectl exec -it catalogue-7758f68dc7-j46rc -n roboshop -- sh
  113  03/04/26 03:18:15 cd ../../eksctl/
  114  03/04/26 03:19:32 kubectl get svc -o wide
  115  03/04/26 03:23:21 history



Commands:
kubectl get nodes   =>
git clone <git-k8-resources-link>  =>
cd k8-resources    =>
kubectl apply -f 04-labels.yaml  =>
kubectl apply -f 12-services.yaml  =>    
kubectl get svc  =>
kubectl describe svc nginx   => 
kubectl delete -f 12-services.yaml  =>
Create “14-service-np.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl apply -f 14-service-np.yaml  => 
kubectl get svc  => Check now CLUSTER-IP should be created for “NodePort”.
Go to AWS EC2 instances list page and select “roboshop-spot-node” then click on networking tab and under Security group click on “eks-cluster-sg-roboshop” then click on “Edit inbond rules” then click on “Add rule” enter then “32141” for “Port range” then select “0.0.0.0” for “Source” then click on “Save rules”.
git pull  =>
Open chrome browser and enter “roboshop-spot-node” public IP address with port number “32141” like <public-IP>:32141.
kubectl get pods -o wide   =>
kubectl delete -f 14-service-np.yaml =>
kubectl apply -f 14-service-np.yaml  =>
kubectl get svc  =>
Create “15-svc-lb.yaml” file then write pod code then push & pull code.
git pull  =>
kubectl delete -f 14-service-np.yaml  =>
kubectl apply -f 15-svc-lb.yaml  =>
kubectl get svc  =>
Now go to AWS EC2 instance under it select Load Balancers then select load balancers then take load balancer URL then enter in the browser then should show nginx response.
Create “16-replicaset.yaml” file then write pod code then push & pull code.  
git pull =>
kubectl apply -f 16-replicaset.yaml  => 
kubectl get pods  =>
kubectl get pods  =>
kubectl get rs  => rs means replica set.
kubectl delete pod nginx-kqhdb  => kqhdb is a k8 resource name. If any pod deleted by mistake or by any issue then replica set will make sure that replica set will create that pod again. Its keeps desired number of pods are per the template and replicas mentioned.
kubectl get pods  =>
Now change replicas to 10 in 16-replicaset.yaml.
git pull =>
kubectl apply -f 16-replicaset.yaml  =>
kubectl get pods  =>
kubectl edit rs nginx  => To edit replicas to 1 and save changes(!wq).
kubectl get pods  => Now check only 1 replica is created. Pods creation will done quickly.
Since pods are immutable, we can create another set of v2 pods, then delete v1 pods.
Now change nginx image then save changes and push code.
git pull =>
kubectl apply -f 16-replicaset.yaml  =>
kubectl get pods  => Now observe pod is not changed because replicaset will not consider image version updated.
Create “17-deployment.yaml” file then write pod code then push & pull code.
git pull =>
kubectl delete -f 16-replicaset.yaml  =>
kubectl apply -f 17-deployment.yaml  =>
kubectl get pods  =>
kubectl get rs  =>
kubectl get deployment  =>
Now edit replicas to 2 in 17-deployment.yaml file.
git pull =>
kubectl apply -f 17-deployment.yaml  =>
kubectl get deployment  =>
kubectl get pods  =>
Now change nginx to latest version the save changes and push code.
git pull => After updating nginx with latest
kubectl apply -f 17-deployment.yaml  =>
kubectl get pods  => 2 pods should be created.
kubectl get rs  => To Check old and newly created replicates. Old one have DESIRED 0 and new one have DESIRED 2.
kubectl get rs  =>
kubectl get deployment  =>
kubectl rollout history deployment/nginx  => To check rollout hostory
kubectl rollout undo deployment/nginx  => To do Rollback of nginx to previous version. This command we used when any issues occurred in the new deployment then business also down due to issues so we should roll back to previous version and then we will start debug/trouble the issues then after fixing all issues then we will release new issues.
kubectl get pods  =>
kubectl get pods  =>
kubectl describe pod <pod-id> => To verify previous nginx version.
We can also do Rollback in any region.
kubectl rollout status deployment/nginx  => To check nginx rollback status success or not.
Now in 17-deployment.yaml file add “annotations” under it add “kubernetes.io/change-cause" then enter change cause message
git pull =>
kubectl apply -f 17-deployment.yaml  =>
kubectl rollout status deployment/nginx  =>
kubectl rollout history deployment/nginx  =>
Now change ‘change cause message’ from “moving to 1.29” to “moving to 1.27” then save changes and push code.
git pull => After adding nginx:1.29.7
kubectl apply -f 17-deployment.yaml  =>
kubectl get pods  =>
kubectl describe pod <pod-id> =>
kubectl rollout history deployment/nginx  =>
kubectl rollout undo deployment nginx  => 
kubectl rollout history deployment/nginx  =>
Create “k8-roboshop” git repo and clone it in your local machine then create “01-namespace.yaml” file this file should be outside of “mongodb” folder then write namespace code and create “mongodb” folder then create “manifest.yaml” file and then write Kubernetes deployment code then write service code for a cluster(for internal use).
Push code into “k8-roboshop” git repo and then pull inside AWS EC2 instance “workstation” server.
cd ..    => 
git clone <git-k8-roboshop-link>  =>
cd k8-roboshop   =>
kubectl apply -f 01-namespace.yaml  =>
cd mongodb  => 
kubectl apply -f manifest.yaml  =>
kubectl get deployment -n roboshop =>
kubectl get rs -n roboshop   =>
kubectl pods -o wide -n roboshop => To see all the information about all pods available in “roboshop” cluster, will get pod NAME, READY, STATUS, RESTARTS, AGE, IP, NODE, NOMINATED NODE, READINESS GATES.
kubectl get svc -n roboshop   =>
kubectl describe svc mongodb -n roboshop   =>
Create “catalogue” folder and then create “manifest.yaml” file then write required catalogue Kubernetes code.
Push code into “k8-roboshop” git repo and then pull inside AWS EC2 instance “workstation” server.
cd ../catalogue/    =>
kubectl apply -f manifest.yaml  => 
kubectl get pods  =>
kubectl get pods -n roboshop   => Run this command to chek catalogue pod status.
kubectl exec -it <catalogue-pod-id> --n roboshop -- sh =>
curl http://localhost:8080/health  => Run inside catalogue, check catalogue status should be “ok”.
exit => Run inside catalogue.
eksctl delete cluster --name=roboshop --region=us-east-1  => Command to delete Kubernetes Cluster.
eksctl delete cluster -f eks.yaml --force   => Command to force delete Kubernetes Cluster.



Diagrams:
k8-svc
k8-deployment


Timestamps:
Interview questions





Mistakes & Learning:




Doubts Link Clarification AI chat link: