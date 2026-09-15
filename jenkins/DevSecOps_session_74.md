Saturday, 15 August 2026

Session 74 - DevSecOps process, Shift left, SonarQube.
Class Notes
CI
====
npm install
docker build

shift-left
=====
testing and scanning

docker build -t catalogue:1.0.0

scanning
========
application security
network security
data security

DevSecOps
==========
SonarQube
----------
Static source code analysis
Static application security testing

Dynamic application security testing
Opensource libraries scanning
Image scanning

1. sonarqube scanner -> agent running in jenkins agent
2. sonarqube server -> central server

Overall code = total code
new code = C2-C1

vulnerabilties = 0
issues in code that can be used by attackers to exploit

Bugs
Bugs that can cause unexpected behaviour in runtime

Code Smells
very hard to understand

Coverage
How many lines of code is tested

Duplications
Code should not be duplicated

Security Hotspots
SAST

code + node_modules -> catalogue.zip

jenkins
jenkins agent

* stage view
* aws creds
* aws steps
* pipeline utility steps
* sonarqube

* master agent
* add creds

* sonarqube tool setup
* sonarqube system setup

Commands:
input - Jenkins Input is used to ask permissions from the user while taking build. We can’t automate everything, we need proper process and permissions by whom the actions was performed like that.
Now in Jenkins repo in Jenkins file write jenkins input code and push & pull code in Jenkins server.
Now create a Jenkins EC2 server with JoinDevOps RedHat RHEL9 Linux OS then install & setup jenkins configuration then install required jenkins plugins in the Jenkins server like “Stage View” and then create a node in Jenkins, here we are going to creat “roboshop” node.
While creating Jenkins “roboshop” node enter “3” in “Number of executions” input field then enter “/home/ec2-user/jenkins-agent” in “Remote root directory” then select “Only build jobs with label expressions matching this node” for “Usage” then select “Launch agents via SSH” for “Launch method” then under “Launch method” enter “Jenkins-agent.<your-domain-name>” then for “Credentials” click on “+Add” button then select “global” then select “Username with password” click on “Next” button then select “Global (Jenkins, nodes, items, all child items, etc)” for “Scope” then enter “ec2-user” for “Username” then enter “DevOps321” for Password(which is Jenkins RHEL AWS IAM image password) then enter “ssh-auth” for “ID” then enter “ssh-auth” for “Description” as well then click on “Create” button.
Now create “hello-pipeline” in the Jenkins select “Pipeline” then click on “OK” button then under pipeline configuration select “Pipeline script from SCM” then select “Git” for “SCM” then enter “your-Jenkins-github-repo-url” for “Repository URL” then replace “/master” with “/main” for “Branch Specifier (blank for any)” then click on “Save” button then click on “Build Now” button.
Now click on “Console Output” button then verify pipeline will be proceed and will wait for input then click on “Deploy” stage in the stage view then select “Yes, we should” then now pipeline will be proceed. This is called input gate for the pipeline.
Now to learn about Jenkins “when” condition we should do changes in “booleanParam” name: ‘DEPLOY’, defaultValue:  false, now add when block under “Deploy” stage blobs then add expression then write expression based on DEPLOY parama.
Now in Jenkins run pipeline once then again run without checking “DEPLOY” checkbox. Now the Deploy stage should be failed based on “DEPLOY” checkbox value and “DEPLOY” parama based Jenkins when condition.

In Chrome browser search for “Jenkins package.json version read” then you will find and then “Install “Pipeline Utility Steps” plugin in Jenkins to read “version” (catalogue app versiosn)  in “package.json” file then in catalogue Jenkins file add package.json read version code copied from chrome.
To access appVersion globally declare it as an ENV variable then in “Read version” stage appVersion value updating with “Pipeline Utility Steps” plugin package.json file variable. To install apm with jenkins the server should have npm so we need to connect to “Jenkins-agent” server then install npm  and also should install Docker as well.
sudo su -  => Run in Jeknins-agent server.
dnf module enable nodejs:20 -y   => Run in Jeknins-agent server.
dnf install nodejs -y    => Run in Jeknins-agent server.
dnf -y install dnf-plugins-core  => Run in Jeknins-agent server.
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo => Run in Jeknins-agent server.
dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y  => Run in Jeknins-agent server.
systemctl start docker  => Run in Jeknins-agent server.
systemctl enable docker  => Run in Jeknins-agent server.
usermod -aG docker ec2-user   => Run in Jeknins-agent server.
docker ps   => Run in Jeknins-agent server.
exit   => Run in Jeknins-agent server. Exiting to get Docker user access.
Now in Catalogue Jenkins file add “Build Image” stage then add “docker build -t catalogue:$appVersion . ” script command.
Now in Jenkins click on “New Item” then enter “ROBOSHOP” in “Enter an item name” filed then select “Folder” then click on “OK” button then click on “Save” button. Now again click on “New Item” then enter “catalogue” in “Enter an item name” filed then select “Pipeline” then click on “OK” then click on “Configure” then under Pipeline select “Pipeline script from SCM” for “Defination” then select “Git” for “SCM” then enter “your-github-repo -url” then change “/master” to “/main” branch then click on “Apply” & then click on “Save” then click on “Build Now” button.
Now in Jenkins disconnect agent and then connect agent because already connected agent right so we not get connection otherwise.
Now create AWS ECR for catalogue check below for the process.
Now install “AWS Credentials” in Jenkins server to create AWS setup & access secret key.
Now go to Manage Jenkins then Credentials then click on “Add Credentials” then select “AWS Credentials” then enter “aws-creds” for ID then enter “aws-creds” for Description as well then enter “you-AWS-CLI-Access-key” in “Access Key ID” field then enter “you-AWS-CLI-Access-key” in “Secret Access Key” field then click on “Create” button.
Now install “AWS Steps” plugin to work with our AWS creeentails while working with AWS in Jenkins.
Now in Catalogue Jenkins file in “Build Image” stage block in script block add “withAWS” Jenkins code then now in “withAWS” inside sh enter “roboshop/catalogue” AWS authentication command in sh block then add ECR docker build & push commands in sh block. Now take Jenkins build & check.
Shiftleft - SonarQube - for SonarQube Scanning in “roboshop-infre-eks” “31-cicd-tools” create infra to provision Jenkins and SonarQube AWS IAM based server we can use that while practicing.
After running “roboshop-infre-eks” “31-cicd-tools” terraform code once infra created successfully then copy SonarQube server public IP then connect to SonarQube server.
ssh -i <ssh-keys-file-name> ubuntu@<sonarqube-server-ip>   => Run in you system.
cat /opt/default-sonar-login.txt   => Run in you system.
Then in the browser enter “sonar.<your-domain>:9000” to open SonarQube. Then enter SonarQube username & password. sonarqube scanner -> agent running in jenkins agent. sonarqube server -> central server.
Now in Jenkins install “SonarQube Scanner” plugin.
Now in Jenkins go to Tools & search for “SonarQube” then you will find “SonarQube Scanner” under it enter “sonar-8” in Name field then choose “SonarQube Scanner 8.1.0.6389” then click on “Apply” & then click on “Save” button.
Now in Jenkins click on “System”(system settings) under “SonarQube servers” click on “+ Add SonarQube” button then enter “sonar-server” for Name then enter “sonar-server.<your-domain-name>” for “Server URL” then to set sonarqube athentication go to SonarQube then go to setting then click on “My Account” then click on “Security” then under “Generate Tokens” enter “jenkins” for Name then select “Global Analysis Token” for Type then select “30 days” for “Expires in” then click on “Generate” button then now copy token then Jenkins under “Server authentication token” click on “+ Add” button then enter “genarated-sonarqube-token” in “Seret” field then enter “sonar-creds” for ID then enter “sonar-creds” for Description” as well then click on “Create” button then select “sonar-creds” for “Server authentication token” then click on “Apply” & then “Save” buttons.
Now to scan catalogue code using SonarQube in Google search for “Jenkins sonarqube scanner” then write SonarQube scanner Jenkins stage code.
Create “sonar-project.properties” file in “catalogue” then write SonarQube Scanner properties configuration code to scan code and include which files/folders SonarQube should not scan then push code.
Once the catalogue is pipeline is executed successfully then go to sonarqube then check for SonarQube results and then its working fine then developer will access the SonarQube then verify security issues, bugs, code issues and fix those.
Now in SonarQube click on “Quality gates” tab then click on “Create” button then enter “roboshop” in Name filed then click on “create” then click on “Add Condition” then select “on Overall Code” for “Where?” then select “Security Rating” for “Quality Gate fails when” then select “A” for “Value* ” then click on “Add Condition” button then click “roboshop” then delete “Coverage” then again click on “Add Condition” then select “On Overall Code” for “Where?” then select “Vulgerbilities” for “Quality Gate fails when” then select “0” for “Value* ” then click on “Add Condition” button. Then again click on “Add Condition” then select “On Overall Code” for “Where?” then select “Maintainsbility Rating” for “Quality Gate fails when” then select “A” for “Value* ” then click on “Add Condition” button. 
Now take Jenkins build after setting Quality gates then verify quality gates are passed or not. Hera the main point is the pipeline will be success but the sonarqube quality gates will be failed.
To enforce SonarQube quality gates we should add “Quality Gate” stage.
To pass information from SonarQube to Jenkins we should add SonarQube Webhooks.
Now in SonarQube click on “Administration” select “Webhooks” in “Configuration” then click on “Create” button then enter “jenkins” in Name filed then enter “http://jenkins.<your-domain:8080/sonarqube-webhook/“ in “URL” then click on “Create” button.
Now in Jenkins take catalogue pipeline build again.
Now the pipeline will be failed due to SonarQube Quality Gate failure.
Now the Developer will Fix SonarQube showing issues.

How to create AWS Elastic Container Registry for Roboshop Catalogue Microservice?
In AWS search for ECR then click on “Elastic Container Registry” then click on “Create” button then enter “roboshop/catalogue” for “Repository name” then under “Image tag settings” keep “Mutable” option same for “Image tag mutability” then click on “Create” button. Now open open the /roboshop/catalogue” image then click on “View push commands” now copy “account ID” and paste in Jenkins ENV variable value like “ACC_ID = “1234567890123” ”.

Diagrams:
scannings


Git Repo Links:
roboshop-infra-eks
https://github.com/SriRamCharanKolla/jenkins.git


Timestamps:
Assignment -
Interview questions:






Mistakes & Learning:




Doubts Link Clarification AI chat link: