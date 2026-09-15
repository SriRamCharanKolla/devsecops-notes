Tuesday, 7 April 2026

Session 58 - K8 Roboshop(MongoDB, Catalogue, Liveness, Readiness and startup probe, Redis, User, Cart, Shipping), 
Class Notes
1. Build images in docker -> optimise
2. Run inside kubernetes

Liveness, Readiness and Starup probe

every 5 sec, check the server is running or not -> health check

readiness -> is your app ready to handle traffic? usually app takes some time to start/get ready.

startup probe -> usually java apps or high memory apps take some more time to start

startup -> readiness -> liveness

1. if 8080 is opened then app is running
2. /health is giving response then app is running

git clone https://github.com/ahmetb/kubectx.git ~/.kubectx
COMPDIR=$(pkg-config --variable=completionsdir bash-completion)
ln -sf ~/.kubectx/completion/kubens.bash $COMPDIR/kubens
ln -sf ~/.kubectx/completion/kubectx.bash $COMPDIR/kubectx
cat << EOF >> ~/.bashrc


#kubectx and kubens
export PATH=~/.kubectx:$PATH
EOF

curl -sS https://webinstall.dev/k9s | bash
Shift+:


/home/ec2-user/.kubectx:
/usr/local/sbin:
/usr/local/bin: -> local user
/usr/sbin: -> for all users system commands
/usr/bin: -> for all users
/sbin:
/bin

catalogue:1.0.0 -> PROD

new release
============
1.1.0 -> release
rebuild catalouge same version 1.1.0

building image = image develop -> build -> push to hub
running image = manifest ==> image pull and then run the image using manifest config

JDK JRE

JRE -> Java runtime environment
JDK -> Java development kit

JDK = JRE + Development/Build tools

node_js -> npm install -> create node_modules(libraries are here)
java -> maven package -> create target folder and .jar file inside that
.jar = bytecode + dependencies

Commands:
Start writing remaining Kubernetes Catalogue code.
cd  => 
git clone <k8-roboshop>   =>
cd k8-roboshop  =>
kubectl apply -f 01-namespace.yaml   =>
kubectl apply -f mongodb/manifest.yaml   =>  
kubectl get pods -n roboshop  =>
kubectl apply -f catalogue/manifest.yaml   =>
kubectl get pods -n roboshop  => Run this command to verify both Mangodb & Catalogue pods are created successfully or not.
Increase “initialDelaySeconds” from 5 seconds to 15 seconds because once service is schedule then it will take atleast 15 seconds of time to pull docker container and create pods and service.
cd  =>
To make easy and avoid providing “-n roboshop” in the kubectl command “kubectx” tool will should be used.
git clone https://github.com/ahmetb/kubectx.git ~/.kubectx    =>
COMPDIR=$(pkg-config --variable=completionsdir bash-completion)   =>
sudo ln -sf ~/.kubectx/completion/kubens.bash $COMPDIR/kubens   =>

kubens roboshop  => After kubectx installed and configured in the server then this command automatically refer to “roboshop” name space.
kubectl get pods  => After configuring “kubens” for roboshop namespace now this command refer to roboshop name space automatically.
kubens default  => Now “kubens” is configured to default namespace.
kubectl get pods  => Previously “kubens” is configured to default namespace so now no pods will be displayed because no pods under default Kubernetes namespace.
kubens roboshop  => Now “kubens” is configured to “roboshop” namespace.
curl -sS https://webinstall.dev/k9s | bash   => To install k9s 
k9s   => in new terminal tab connect to k8s-workstation then run this after successful installation of k9s in k8s-workstation.
Now in k9s terminal all pods should display after running “k9s” command.
Press “l+n” to see logs.
Press “esc” to come back.
Press “s” to get shell” then enter “exit” to came out from shell.
Press “shift+:” then you will get input fields at top then you can enter “deployments” then you will be redirected to deployments. 
Press “shift+:” then you will get input fields at top then you can enter “namespace” then you will be redirected to namespaces.
Now create “redis” folder then create “manifest.yaml’ file inside “redis” folder then write Kubernetes Deployment code with Template with docker image want to pull and run in the roboshop project for Redis DB setup for the project then write Kubernetes Service below Deployment code with three hyphens. Now push code and run in the k8 workstation server to add Redis to Roboshop project.
git pull  => Run inside workstation server.
kubectl apply -f redis/manifest.yaml  => To pull Redis Docker image and run in k8 cluster for Roboshop project.
cd  =>
git clone “roboshop-docker-url”    =>
cd roboshop-docker   =>
ls -l  =>
cd catalogue   =>
cat Dockerfile   =>
Verify once docker is optimised or not. In our case we already optimised image.
docker build -t <docker-username/catalogue:1.4.0(this version number should by verifying this image version in your docker hub. Must & should not provide already provided version )> .   =>
docker login -u <your-docker-username>   => Then enter your password to login into your docker nub account and then to push this newly build docker image.
docker push <your-docker-hub-username/catalogue:<build-version(previously build version)>  =>
 imagePullPolicy: Always # safe to pull everytime, there may be images with same version rebuilt and pushed.   Add thin in catalogue k8-roboshop in manifest.yaml file.
Now push catalogue k8-roboshop in manifest.yaml file changes.
git pull =>
cd ../../k8-roboshop/ =>
kubectl apply -f catalogue/manifest.yaml  =>
Now go to K9S terminal then verified new catalogue pod is created or not and then select latest created catalogue pod by pressing S button on the keyboard to open shell in K9S.
Now create debug folder under “roboshop-docker” then create Dockerfile then push changes to the GitHub. This for testing docker images which we build for the roboshop project.
cd ../roboshop-docker  =>
git pull =>
cd debug  =>
docker build -t <your-docker-hub-username/debug:<build-version> .  => 
docker push <your-docker-hub-username/debug:<build-version>  =>
Now under k8-oboshop create debug folder, then under the folder, create manifest.yaml file write a debug Pod Kubernetes code then push changes. This pod is to test wether we are able to connect with mongodb & catalogue pods or not.
cd  =>
cd k8-roboshop  =>
git pull =>
Now in K9S select debug pod then press “s” in the keyboard to go to shell and to test can able to connect with mongodb & catalogue pods or not.
telnet mongodb 27017   => After selecting debug pod in the K9S then run this command in K9S connected terminal.
quit  => to quit from telnet.
exit => to get exit from debug pod in K9S. 
kubectl apply -f debug/manifest.yaml  =>
Debug pod able to connect with mongodb pod on port number 27027.
telnet redis 5679   => After selecting debug pod in the K9S then run this command in K9S connected terminal. This command is not connect with redis pod need to debug.
nslookup redis  => To get redis pod server IP and address.
telnet redis 5679   => In this command we wrongly set port number in k8-roboshop redis manifest.yaml file port & targetPort number in the Kubernetes code thats why its not able to connect with the redis pod, so changes port & targetPort in the manifest.yaml file then push changes then apply changes with kubectl.
git pull =>
kubectl apply -f redis/manifest.yaml  =>
Now in K9S debug shell run “telnet redis 6379” command to verify can able to connect with redis pod or not. Now can able to connect with redis pod successfully.
quit  => to quit from telnet.
exit => to get exit from debug pod in K9S.
Now optimise roboshop-docker cart docker image also.
Now edit roboshop-docker cart Dockerfile and optimise with alpine3.23 and docker best practices. In this optimisation we do some changes like add packages in the run time like that and shuffle docker image build process as per the Docker best practices.
As we are providing environment variable with Kubernetes env variables we should comment out cart Dockerfile ENV code remaining things are same.
cd ../roboshop-docker  =>
git pull =>
docker build -t <your-docker-hub-username/cart:<build-version> .  =>
Now under “k8-roboshop” create “cart” folder under it create “manifest.yaml” file start writing cart Kubernetes code.
Now go to “robodhop-documentatiom” verify cart service file & configuration setup.
cd k8-roboshop  =>
git pull =>
kubectl apply -f cart/manifest.yaml   =>
Now go to K9S and verify cart image is building by Kubernetes or not. Here we forgot to pushed cart docker image with latest image optimised changes.
docker push <your-docker-hub-username/cart:<build-version>  =>
Now go to K9S and verify cart image will be building by Kubernetes automatically as we pushed our image into DockerHub.
Now in K9S select debug pod the press “s” in your keyboard to go to debug shell and to test cart is working fine and can able to connect with cart with health check URL.
curl http://cart:8080/health  => Run inside K9S debug shell.
exit => Run inside K9S debug shell.
Now optimise roboshop-docker user docker image also.
Now edit roboshop-docker user Dockerfile and optimise with alpine3.23 and docker best practices. In this optimisation we do some changes like add packages in the run time like that and shuffle docker image build process as per the Docker best practices.
cd ../roboshop-docker/user/  =>
git pull =>
docker build -t <your-docker-hub-username/user:<build-version> .  =>
docker push <your-docker-hub-username/user:<build-version>  =>
Now under “k8-roboshop” create “user” folder under it create “manifest.yaml” file start writing cart Kubernetes code.
Now go to “robodhop-documentatiom” verify user service file & configuration setup.
cd k8-roboshop  =>
git pull =>
kubectl apply -f user/manifest.yaml   =>
 As observed user deployment restarted 2 times so need to check logs. So select user pod in K9S then check logs then found the issue, issue is invalid url now press “w” key in the keyboard actually in redis manifest file under configMap “REDIS_URL” was only given as “redis” its actpally “redis://redis:6379". Kubernetes restart user pod until url is fixed it will restart almost 6 times.
 git pull =>
 kubectl apply -f user/manifest.yaml   => After running this command in the out put the configMap of user component will be changed and deployment and service will be unchanged due to this if we verify user pod in K9S the user pod will be still running with invalid user deployment process so to fix this issue we should delete present user pod by pressing “ctrl+d” in the keyboard then immediately new pod created by the Kubernetes, in this now pod correct url will be set in by the updated configMap to the deployment & service.
 Now in k9s select debug pod then go to its shell by pressing “s” in the keyboard then run “curl http://user:8080/health" in the debug shell if we got “{“app:”OK”, “mongo”:true}” then roboshop user service is deployment successfully and can able to connect to user app.
 We should not delete “Deployment” because all “ReplicaSets” will be deleted so we can’t be rollback.
 cd ../roboshop-docker  =>
 cd shipping/  =>
 docker build -it shipping .   => Building shipping Docker image to check shipping size.
 docker images  => To check all images and its size.
 Now optimise roboshop-docker shipping docker image also.
 Now edit roboshop-docker shipping Dockerfile and optimise with alpine3.23 and docker best practices. In this optimisation we do some changes like add packages in the run time like that and shuffle docker image build process as per the Docker best practices.
 git pull =>
 docker build -t <your-docker-hub-username/shipping:<build-version> .  =>
 docker push <your-docker-hub-username/shipping:<build-version>   =>
 docker images  => To check all images and its size.
 Now under “k8-roboshop” create “shipping” folder under it create “manifest.yaml” file start writing cart Kubernetes code. For “shipping” need to add “startupprobes” because 
 Now go to “robodhop-documentatiom” verify shipping service file & configuration setup.
 cd  =>
 cd k8-roboshop  =>
 git pull =>
 kubectl apply -f shipping/manifest.yaml   =>
 Shipping is failed because not run “MySQL” but we are running shipping.
 eksctl delete cluster --name=roboshop --region=us-east-1  => Command to delete Kubernetes Cluster.
 eksctl delete cluster -f eks.yaml --force   => Command to force delete Kubernetes Cluster.



Diagrams:
k8-setup



GitHub Repos:
k8-roboshop
roboshop-docker


Timestamps:
Interview questions





Mistakes & Learning:




Doubts Link Clarification AI chat link: