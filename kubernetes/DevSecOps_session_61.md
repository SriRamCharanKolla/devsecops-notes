Tuesday, 14 April 2026

Session 61 - K8 volumes(EFS static provisioning, EFS dynamic provisioning, EBS vs EFS).
Class Notes
Ephemeral volumes
- Configmap as volume
- emptyDir -> at pod level -> sidecar containers
- hostPath -> underlying host level -> daemonset(a pod replica runs on each and every node) -> collect host level logs and metrics

volumes:
- name: <some-name>
  source-of-volume
  
  volumeMounts:
  - name:
	mountPath:
	
Static provisioning
Dynamic provisioning

EBS and EFS

Static provisioning
===================
PV -> physical representation of the underlying disk, it is equivalant k8 resource to the disk. we can work with pv to manage the disk
PVC -> claiming the disk from PV
SC -> part of dynamic provisioning. it creates disk and pv on behalf of us..

1. install EBS drivers
2. attach EBS IAM role to the instances
3. create disk
4. create pv and pvc
5. mount volume in pod


1. install EBS drivers
2. attach EBS IAM role to the instances
3. create storage class
4. create pvc and pod

if EBS volume should be in same az as server

aws eks update-kubeconfig --region us-east-1 --name roboshop

kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58"

EBS vs EFS
===========
1. EBS is like HD EFS is like drive following NFS protocol
2. EBS should be in same AZ but EFS can be anywhere in network
3. EBS is faster than EFS
4. EBS is suitable for OS and DB, EFS is suitable to store some files
5. EBS is fixed size, EFS can grow automatically
6. EBS we can select fs type, EFS is set to NFS
7. EBS SG is not required, EFS SG is mandatory

EFS static provisioning
=======
1. Install drivers
2. Add IAM role
3. Create filesystem
4. Make sure filesystem SG allows 2049(NFS port)
5. Create PV, PVC and pod

kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-3.0" > private-ecr-driver.yaml

Commands:
Create “roboshop-dev-workstation” AWS EC2 instance then connect with it.
aws eks update-kubeconfig --region us-east-1 --name roboshop  => Whenever authentication is failed then should run this command.
kubectl get nodes   =>
aws eks update-kubeconfig —region us-east-1 —name roboshop  => 
To install drivers in the web search for “ebs csi driver”.  
kubectl apply -k “github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58”   =>
Select any AWS EC2 instance “roboshop-spot-Node” then click on “Security” tab then click on “IAM role” then click on “Add permissions” then select “Attach policies” then search for “ebscs” then tick “AmazonEBSCSDriverPolicy” then click on “Add permissions”.
Now create a Disk, Go AWS EC2 instance in side menu Select “Volumes” under “Elastic Bock Store” then click on “Create volume” then enter “2” for “Size(GiB)” then select “use1-az6(us-east-1d)” for “Availability Zone” then click on “Create volume” button.
Now copy Disk volume ID in “03-ebs-static.yaml” under volumes folder under “k8-resources”. Now Kubernetes will create Persistence Volume. Push changes and then pull changes in the workstation server.
git clone “k8-resources-git-link”   =>
cd k8-resources/volumes  =>
kubectl apply -f 03-ebs-static.yaml   =>
kubectl get pv  => To verify storage disk is bonded to the pod or not. 
kubectl describe pv ebs-static  => If any error occurred, it will be displayed.
kubectl describe pvc ebs-static  => If any error occurred, it will be displayed.
kubectl get pvc  =>
kubectl get pods  =>
kubectl describe pod app =>
If EBS Volume state is In-use then Successful.
kubectl delete pod app => must and should delete the volumes. Maybe in the practice session if we forgot to delete storage volumes, it will incur charges in our AWS account.
kubectl delete -f 03-ebs-static.yaml   => checking pod is deleted or not.
Now delete previously created Disk volume(2Gib Disk).
kubectl apply -f 04-ebs-sc.yaml   => creating storage volume is a administrator job, not a DevOps engineers job, maybe in some companies, DevOps engineer need to be do cooperatives administrator, tasks as well.
kubectl apply -f 05-ebs-daynamic.yaml  =>
kubectl get pods  =>
Now Volume will be created Dynamically.
cd  =>
kubectl kustomize "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-3.0" > private-ecr-driver.yaml   =>
ls -l   =>
kubectl apply -f private-ecr-driver.yaml   =>
kubectl get pods -n kube-system  => 
kubectl describe pod <pod/image-id> -n kube-system   =>
kubectl delete -f private-ecr-driver.yaml   =>
rm private-ecr-driver.yaml   =>
kubectl kustomize \ "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-3.0" > private-ecr-driver.yaml   =>
kubectl apply -f private-ecr-driver.yaml   =>
kubectl get pods -n kube-system  =>
kubectl describe pod <pod/image-id> -n kube-system   =>
Now in new tab connect to K9s then select “kube-system” then select error pod and check what the error occurred.
The error is due to Driver version compatibility issue so will try with one step back version.
kubectl delete -f private-ecr-driver.yaml   =>
rm private-ecr-driver.yaml   =>
kubectl kustomize \ “github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-2.3" > private-ecr-driver.yaml   => 
kubectl apply -f private-ecr-driver.yaml   =>
Now check in K9s, now its working fine.
Select any AWS EC2 instance “roboshop-spot-Node” then click on “Security” tab then click on “IAM role” then click on “Add permissions” then select “Attach policies” then search for “ebscs” then tick “AmazonEBSCSDriverPolicy” then click on “Add permissions”.
Now in AWS search for EFS then click on it then click on “Create file system” then enter “roboshop” for “Name” then Select “eksctl-roboshop-cluster/VPC” for “Virtual private cloud” then click on “Create file system”.
EFS initially have few bits it can grow upto 50GB.
Now Click on “View file system” in EFS then click on “Network” tab then wait upto getting “Security groups” then copy Security group ID and then go to “Security groups” search with id then click on “Security group ID” then click on “Edit inbound rules” then click on “Add rule” then select “NFS” for “Type” then select “eks-security-group-id” then click on “Save rules”.
Now create “06-efs-ststic.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
git pull  =>
cd volumes  =>
ls  =>
kubectl apply -f 06-efs-ststic.yaml  => 
kubectl get pv  =>
git pull  =>
kubectl apply -f 06-efs-ststic.yaml  =>
kubectl get pvc  =>
kubectl delete -f 05-ebs-daynamic.yaml  =>
kubectl get pv  =>
kubectl delete pv <pv-id>  =>
Now delete 4GB Pv volume also.
kubectl get pv  =>
Now add pod.
git pull  =>
kubectl apply -f 06-efs-ststic.yaml  =>
kubectl get pods  =>
kubectl describe pod app  =>
git pull  =>
kubectl apply -f 06-efs-ststic.yaml  =>
kubectl get pods  =>
kubectl describe pv efs-static  =>
kubectl delete pod app  =>
kubectl delete pvc efs-static  =>
kubectl delete pv efs-static  =>
git pull  =>
kubectl apply -f 06-efs-ststic.yaml  =>
kubectl get pods  =>
kubectl describe pod app  =>
Not got Volume log so go to K9s then select “default+” then select “app” shell.
cd /usr/share/nginx/html/   => Run inside K9s inside app shell.
ls -l   => Run inside K9s inside app. 
touch hi.html  => Run inside K9s inside app shell.
exit    => Run inside K9s inside app. 
Now in K9s delete app then recreate by running this command “kubectl apply -f 06-efs-ststic.yaml”.
kubectl apply -f 06-efs-ststic.yaml  =>
Now in K9s new app will be created so select it.
cd /usr/share/nginx/html/   => Run inside K9s inside app shell.
ls -l   => Run inside K9s inside app shell. 
exit    => Run inside K9s inside app shell.
Now volume is mounted properly.
Now create EFS dynamic.
kubectl delete -f 06-efs-ststic.yaml  =>
Now delete EFS volume disk also. Then create new EFS click on “Create file system” then enter “roboshop” for “Name” then Select “eksctl-roboshop-cluster/VPC” for “Virtual private cloud” then click on “Create file system”.
Now create “07-efs-sc.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 07-efs-sc.yaml  =>
kubectl get sc  =>
Now create “08-efs-dynamic.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 08-efs-dynamic.yaml =>
Now check in EFS volume under “Access points” new access point should be created.
 kubectl delete -f 08-efs-dynamic.yaml =>
Now check access point will be deleted.



Diagrams:
k8-roboshop



Timestamps:
Interview questions = 





Mistakes & Learning:




Doubts Link Clarification AI chat link: