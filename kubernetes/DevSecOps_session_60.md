Tuesday, 14 April 2026

Session 60 - K8 volumes(emptyDir, hostPath, Persistent volume, Persistent volume claim, Storage Class, EBS static provisioning, EBS dynamic provisioning).
Class Notes
Volumes
========

storage admins
--------------
1. keep the storage secure
2. daily backup
3. restore
4. restore testing
5. replication -> another copy
6. archieval -> secondary

eks version upgrade

1. send a email to storage team
2. give them servers address
3. tell them to take the backup before upgrade
4. once they take backup, they respond it is completed
5. then we start upgrade

1. ephemeral volumes -> inside pods and hosts
2. eternal volumes

emptyDir
--------
1. containers in pod can share same storage..
2. create volume at the pod level
3. mount to the containers you want inside pod

in case of sidecar containers to read the storage and push the logs outside of cluster, we can use emptyDir

hostPath
-------
1. hostPath is not secure, we cant allow containers/pods to access host data.
2. but in rare cases we allow pods/conatiners to access hostPath in read-only mode
3. Daemonset makes sure of a pod replica runs on each and every worker node. while workernodes are adding to the cluster daemonset make sure a replica runs on new node also...
4. daemonset pod replicas access the underlying host information through hostpath volumes, and can ship it to the external clusters

Persistant volumes
Persistant volume claim
Storage class

1. send email to storage about disk creation. 100GB
2. disk id and how to connect

1. static provisioning
2. dynamic provisioning

EBS(Elastic Block Storage)
=======================
1. HD should be as near as possible to server
2. EBS can be mounted to only one server at a time
3. EBS is faster than  EFS
4. EBS is suitable for OS and databases because of low latency

we can use below access modes for EBS
1. ReadWriteOnce 
2. ReadWriteOncePod 

Reclaim policies
===========
Retain -> keep the disk and data even the node is deleted
Delete -> Delete the disk
Recycle -> Delete the data in disk, but keep the disk

vol-0b9fa8fe3f22c6bbe

drivers installtion

1. create the disk in the same az as your servers are in
2. install the ebs drivers for k8 to understand ebs
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58"
3. your worker nodes should ebs permission to mount the disk. attach AmazonEBSCSIDriverPolicy to the workernode IAM role to provide authorization

EFS
=====
1. 

PV(Persistant Volume)
=====================
1. Physical representation of the disk
2. Equivalant object/resource for the disk inside k8S cluster

PVC
====
it is claiming the PV that it needs storage

kid -> mother -> father -> wallet
Pod ->	PVC	  ->	PV	-> EBS

kid -> mother -> UPI wallet

Storage admins
==============
1. creating the disks

K8s admins
========
1. create PV

project/namespace devops engineer
========
1. create pvc and pod

StorageClass -> one time admin task
============
1. You can create disks and PV using storage class

Commands:
Create “volumes” folder under “k8-resources” git repo project. Then create “01-empty-dir.yaml” file and write Kubernetes code then push and pull code.
aws eks update-kubeconfig --region us-east-1 --name roboshop  => Whenever authentication is failed then should run this command.
ls -la  =>
cd .kube/   =>
ls -l  => 
cat config  =>
cd  =>
kubectl get nodes   =>
git clone “k8-resources-git-link”   => 
cd k8-resources/volumes  =>
kubectl apply -f 01-empty-dir.yaml   =>   
Now Start K9s and K9s commands should remember for interview purpose, must be asked in interviews.
curl -sS https://webinstall.dev/k9s | bash   => This command should run with New Terminal tab with After connecting with “k8-workstation” AWS EC2 instance server. This package/software is used to monitor k8-clusters deployed and its status monitoring.
k9s  => Run inside K9s server terminal tab.
git pull  =>
kubectl apply -f 01-empty-dir.yaml   =>
Select “test-pd” pod after its status is Running  => Run inside K9s server terminal tab.
Select “almalinux” pod after its status is Running  => Run inside K9s server terminal tab.
cd /mnt/   => Run inside K9s server terminal tab. Inside almalinux shell.
ls -l    => Run inside K9s server terminal tab. Inside almalinux.
cd nginx-logs/   => Run inside K9s server terminal tab. Inside almalinux shell.
ls -l    => Run inside K9s server terminal tab. Inside almalinux.
cat error.log    => Run inside K9s server terminal tab. Inside almalinux shell.
Now create “02-host-path.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 02-host-path.yaml   =>
If you went to AWS EC2 Auto Scaling Group then edit Desired capacity to 3 then Kubernetes Demonset will create new worker node(AWS EC2 instance).
Go AWS EC2 instance in side menu Select “Volumes” under “Elastic Bock Store” then click on “Create volume” then enter “5” for “Size(GiB)” then select “use1-az6(us-east-1d)” for “Availability Zone” then click on “Create volume” button.
Now create “03-ebs-static.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
Attach previously created EBS volume dis ID in “volumeHandle” of 03-ebs-staticyaml file.
kubectl apply -k “github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58”   =>
git pull  =>
kubectl apply -f 03-ebs-static.yaml   =>
kubectl get pv  =>
After bonding
git pull  =>
kubectl apply -f 03-ebs-staticyaml   =>
kubectl get pvc  =>
kubectl api-resources  =>
kubectl api-resources | grep pv  =>
kubectl delete pod app =>
kubectl delete pvc ebs =>
kubectl delete pv ebs =>
Now create “04-ebs-sc.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
kubectl api-resources | grep sc  =>
git pull  =>
kubectl apply -f 04-ebs-sc.yaml   =>
kubectl get sc  =>
kubectl get sc roboshop-ebs -o yaml  =>
After applying Retain.
git pull  =>
kubectl apply -f 04-ebs-sc.yaml   =>
kubectl delete sc roboshop-ebs  =>
kubectl apply -f 04-ebs-sc.yaml   =>
kubectl get sc  =>
Now create “05-ebs-daynamic.yaml” file under “volumes” folder then write Kubernetes code then push & pull code.
git pull  =>
kubectl apply -f 05-ebs-daynamic.yaml  =>



Diagrams:
k8-volumes

Diagrams:
k8-resources
terraform-aws-workstation
k8-roboshop
roboshop-docker



Timestamps:
Interview questions = 56:00





Mistakes & Learning:




Doubts Link Clarification AI chat link: