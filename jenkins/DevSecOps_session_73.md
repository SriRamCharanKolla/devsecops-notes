Friday, 8 May 2026

Session 73 - Git cherry-pick, CI, Jenkins installation, Freestyle job, Pipeline, Master node architecture, Pipeline syntax.
Class Notes
merge vs rebase
branching strategy
	* Git flow
	* github flow/feature branching strategy
reset, revert
squash/interactive-rebase
stash

cherry-pick
============
when you want only a certain commit/changes in another branch we can go for cherry-pick. 

C1 C2 C3 C4
C2-C1 = a particular feature

git cherry-pick C2

a hot fix came, fixed it in hot-fix branch...directly deployed into PROD. you can get that code as cherry-pick into release branch

CI(Continous Integration)
=========================
Image/AMI build
Infra creation
commands to deploy k8 applications

application is an artifact
build the application
test the application
scan the code
deploy the application

3 months sprint, once everyone says development completed then they go for deployment in DEV

shift-left
===========
instead of testing and scanning the application after deployment can introduce lot of issues, fixing them takes more cycles again and again. but we can shift this before deployment

functional defects -> functionality is not working as expected
build errors -> you can't build image/ami
deployment errors ->

it is a process of integrating code into artifact. we can trigger this CI after every commit into git...we can also follow shift-left, we can do testing and scanning the code before build..

Jenkins and GitHub actions
===========================
1. Complete control with us, We must depend on GitHub
2. We need to main extra CI server and take care of upgrade, patching, etc...No need of extra server for GitHub. We must use GitHub as coding platform.
Cons/disadvantages of using workstation:
logs, visualisation, rbac, diff platforms testing, etc..
plugins add extra capabilities to jenkins


CI
==
sudo curl -o /etc/yum.repos.d/jenkins.repo     https://pkg.jenkins.io/rpm-stable/jenkins.repo

freestyle job
=============
code directly in jenkins
accidental changes
cant easily track who did that changes
tough to restore
no version control

pipeline
========
we can keep the code in git

declarative vs scripted pipeline
==================================

scripted -> old pipeline, groovy syntax. before pipeline it can't check for errors. at run time if error comes it will exit. we have more control to write scripts
declarative -> latest pipline from jenkins 2.0. it allows groovy syntax, but before execution it check for errors

we use a hybrid approach, we have script block in our declarative pipeline to have more control on the stages

Master and Node architecture
============================

pre-build
build
post-build

TRIGGERS
=========
web-hooks

event driven

if someone commits to git repo, we can trigger this pipeline

Commands:
cherry-pick:   when you want only a certain commit/changes in another branch we can go for cherry-pick.
In “dosa-shop” do some development.
cd /dosa-shop/      => Run on your local system.
git status       => Run on your local system.
git add . ; git commit -m “changes something” ; git push origin panneer-dosa    =>  Run on your local system.  
git checkout main     =>  Run on your local system.
git pull origin main     => Run on your local system.
git checkout -b neyyi-karam-dosa     =>  Run on your local system.
Now write code required neyyi-karam-dosa development.
git add . ; git commit -m “dosa batter added” ; git push origin neyyi-karam-dosa    =>  Run on your local system.
Now to Gee adding development.
git add . ; git commit -m “ghee added” ; git push origin neyyi-karam-dosa    =>  Run on your local system.
git log —online  =>  Run on your local system. To check your Repo logs.
Now copy commit ID to do cherry pick.
git cherry-pick 97c0d6d      => Run on your local system.
Now resolve conflit changes.
git add . ; git commit -m “got erra karma from karma-dosa” ; git push origin neyyi-karam-dosa    =>  Run on your local system.
A hot fix came, fixed it in hot-fix branch. directly deployed into PROD. you can get that code as cherry-pick into release branch. 
CI(Continous Integration):  It is a process of integrating code into artifact. we can trigger this CI after every commit into git. we can also follow shift-left, we can do testing and scanning the code before build.
Jenkins and GitHub actions:     1. Complete control with us, We must depend on GitHub.       2. We need to main extra CI server and take care of upgrade, patching, etc...No need of extra server for GitHub. We must use GitHub as coding platform. 3. plugins add extra capabilities to jenkins.
Now create a “jenkins” AWS EC2 instance with DevOps-Practive RedHat-RHEL-9 community AWS AMI.
Now in Google search for Jenkins installations steps for RedHat Enterprise Linux.
Connect to “jenkins” server.
ssh ec2-user@<jenkins-server>     =>
sudo curl -o /etc/yum.repos.d/jenkins.repo     https://pkg.jenkins.io/rpm-stable/jenkins.repo          =>
sudo yum install fontconfig java-21-openjdk -y      =>
Jenkins is a web-server developed in Java.
sudo yum install jenkins -y     =>
sudo systemctl daemon-reload   =>
sudo systemctl enable jenkins   =>    
sudo systemctl start jenkins   =>
Jenkins runs on port number 8080. Go to google then “http://<jenkins-server-ip>:8080”.
sudo cat /var/lib/jenkins/secrets/initialAdminPassword    =>
Now enter initialAdminPassword in “Administrator password”.
Then click on “Continue”, then in “Custom Jenkins” page click on “Install suggested plugins” then required plugins will be installed in your Jenkins server.
Now in “Create First Admin User” page fill all the form details. Then click on “Save and Continue” then click on “Save and Finish”.
Now Full Jenkins web-server will be loaded. Now click on “New Item” then enter “hello-world” then select “Freestyle project” then click on “OK” now in “Configure page now select “Git” under “Source Code Management”.
Jenkins FreeStyle code stored in Jenkins only and pipeline code we can store in GitHub and we can do required changes and run code as a pipeline.
Now click on “New Item” then enter “hello-pipeline” then select “Pipeline” then click on “OK” now in “Configure page now select “Pipeline script from SCM” in “Definition” under “Pipeline”. 
Now create “jenkins” New Git Repo.
git clone https://github.com/daws-88s/jenkins.git     =>  Run on your local system.
Create “Jenkinsfile” file in “jenkins” repo then write Jenkins Pipeline code.
cd jenkins/    => Run on your local system.
git add . ; git commit -m “jenkins” ; git push origin main    =>  Run on your local system.  
Now go to Jenkins server then under Pipeline select “Git” in “SCM” then enter “https://github.com/daws-88s/jenkins.git"  in “Repository URL” under “Repositories” then replace “master” with “main” in “Branch Specific (blank for ‘any’)” then enter “Jenkinsfile” in “Script Path”(we can also provide other file names rather than “Jenkinsfile” but we must & should provide that file name here) then click on “Apply” then click on “Save” then click on “Build now” to build pipeline in Jenkins server then click on in-progress build showing under “Builds” then click on “Console Output” then verify how Jenkins pipeline building.
Here we have issue that is Stage wise view not coming, so should install required Jenkins plugins. Click on Settings then click on “Plugins” then click on “Available plugins” then in search bar search for “stage view” then select “Pipeline: Stage View” then click on “Install” then go to Jenkins home page then click on “hello-pipeline” then now we can see stage wise view and also can see stage wise logs by clicking on Logs.
Create “Jenkinsfile.dcl” file in “jenkins” repo then write Jenkins Declarative Pipeline code.
git add . ; git commit -m “jenkins” ; git push origin main    =>  Run on your local system.  
Now in “hello-pipeline” click on “Build Now” to build pipeline again.
Now create a “jenkins-agent” AWS EC2 instance with DevOps-Practive RedHat-RHEL-9 community AWS AMI in “Configure storage” enter storage “50” not 20.
ssh ec2-user@<jenkins-agent-server-public-IP>    =>  Run on your local system.
sudo yum install fontconfig java-21-openjdk -y      => Run inside jenkins-agent server.
Now go to Jenkins Web-server then click on “Nodes” then click on “New Node” then enter “roboshop” in “Node name” then select “Permanent Agent” in "Type” then click on “Create” then enter “3” in “Number of executors”(based on server(AWS EC2 instance capacity we should provide this number, if instance capacity is more then we can run multiple nodes parallel, if instance capacity is less than can only run less nodes parley) then enter “/home/ec2-user/jenkins-agent” in “Remote root directory” then enter “ROBOSHOP” in “Labels”(if you want to enter Multiple labels then should provide space between each label name) then select “Only build jobs with label expressions matching this node” in “Usage” then select “Launch agent via SSH” then enter “your-jenkins-agent-private-IP-address” in “Host” then click on “Add” in “Credentials” then select “Username with password” then click on “Next” then enter “ec2-user” in “Username” then enter “DevOps321” in “Password” then enter “ssh-auth” in “ID” then enter “ssh-auth” in “Description” then click on “Create” then select created credentials like “ec2-user/**** (ssh-auth” in “Credentials” then select “Non verifying Verification Strategy” in “Host Key Verification Strategy” then select “Keep this agent online as much as possible” then click on “Save”.
Now in Jenkins Nodes page click on “roboshop” node then click on “Log” then check “/home/ec2-user/jenkins-agent” is connected or not.
cd /home/ec2-user/jenkins-agent   => Run inside jenkins-agent server.
ls -l  => Run inside jenkins-agent server.
remoting.jar is the software which connects Jeskins master and Jenkins-agent.
df -hT  => Run inside jenkins-agent server.
sudo lsblk   =>  Run inside jenkins-agent server.
sudo su -   =>. Run inside jenkins-agent server.
sudo growpart /dev/nvme0n1 4    =>  Run inside jenkins-agent server.
sudo lvextend -L +10G /dev/mapper/RootVG-homeVol  =>  Run inside jenkins-agent server.
sudo lvextend -L +10G /dev/mapper/RootVG-rootVol  =>  Run inside jenkins-agent server.
sudo lvextend -L +10G /dev/mapper/RootVG-varVol  =>   Run inside jenkins-agent server.
sudo xfs_growfs /   =>  Run inside jenkins-agent server.
sudo xfs_growfs /home   =>  Run inside jenkins-agent server.
sudo xfs_growfs /var   =>  Run inside jenkins-agent server.
df -hT  => Run inside jenkins-agent server.
Now go to Jenkins nodes listing page then click on refresh button to refresh nodes to check extended “Jenkins-agent” EC2 server stage is working and making “roboshop” node is keep connect online, click on it then click on “Log” then check logs it is connected or not, if in Logs if you find “Agent successfully connected and online” then your agent successfully connect to Jenkins web-server.
Now if you want to use created Jenkins agent then in “Jenkinsfile” under agent block add node block then add label and labelName of your Jenkins agent.
git add . ; git commit -m “ghee added” ; git push origin main   =>  Run on your local system.
Now execute/build hello-pipeline in Jenkins.
cd /home/ec2-user/jenkins-agent/workspace/hello-pipeline   => Run inside jenkins-agent server.
ls -l  => Run inside jenkins-agent server.
cat Jenkinesfile  => Run inside jenkins-agent server.
exit  => Run inside jenkins-agent server.
Now in “Jenkinsfile” add post build Jenkins code snippet block and add always under post build block with echo print msg.
git add . ; git commit -m “jenkins” ; git push origin main    =>  Run on your local system.
Now click on build now then verify always block echo print msg was printed or not. In our case its successfully printed.
Now in “Jenkinsfile” add success, failure blocks under post build block with echo print msg. Under stages block under scripts block under sh add “exit 1” means intentionally making script failure to print failure msg after taking jenkins build.
git add . ; git commit -m “jenkins” ; git push origin main    =>  Run on your local system.
Now click on build now then verify failure block echo print msg was printed or not. In our case its successfully printed.
Now build hello-pipeline in Jenkins. Check in blogs post block echo message in printed or not.
Now comment “exit 1” under stages scripts block and then push code, 
git add . ; git commit -m “jenkins” ; git push origin main    =>  Run on your local system.
Now build hello-pipeline in Jenkins. Check in blogs success and failure  block echo message in printed or not.
Now add “environment” block to create custom environment variables and we can use environment variables wherever we want. Use it under scripts like “echo $COURSE” as we declared COURSE environment variable.
Now push code changes and take build by clicking on “Build Now” button then verify Jenkins build logs environment variable value is printed or not, in our case printed successfully.
Now in “Jenkinsfile” add options add “disableConcurrentBuilds” option Jenkins code snippet/block. In production we should not trigger 2 builds at a time in this case we can use “disableConcurrentBuilds”.
git add . ; git commit -m “jenkins” ; git push origin main    =>  Run on your local system.
Now click on “Build Now” button 2 times in the Jenkins then verify only one build should be proceed and next build should be in pending state.
Add “sleep 5” under scripts block to make 2nd build in pending state.
Now again on “Build Now” button 2 times in the Jenkins then verify only one build should be trigger and next build should be in pending state.
timeout - timeout block is used to set timeout for a Jenkins build, the build should be completed within time if everything is correct. Add timeout(time: 5, unit: ‘SECONS') under “options” block.
Now push code and then click on “Build Now” button in the Jenkins then verify “timeout” is working fine or not.
parameters - parameters are used to add dynamic parameters for a taking Jenkins build. With the help of parameters we can create UI/form like dropdown, radio buttons, checkbox, text-field etc, . If parameters are added 1st time should take build by clicking on “Build Now” button then only Jenkins notify that the jenkins scripts have parameters then it will display “Build with Parameters” button instead of “Build Now” button and also whatever parameters added will be shown. When user taking build 2nd time onwards should add parameters. So add parameters block and access parameters under scripts with echo print to prints user entered parameters while taking builds.




How to Implement Jenkins Triggers for Git Push?
Jenkins triggers are implemented with the help of web-hooks. Web-hooks are used to trigger events based on actions performed. To implement Git Push Jenkins trigger 1st choose a GitHub repo for which you want to implement git push trigger Jenkins auto build trigger. Then click on “Settings” button of the repo then click on “Webhooks” then add jenkins running url then add “/github-webhooks/“ after jenkins url after slash(/) after “/github-webhooks/“ should not miss slash(/). Then select “application/json” under “Content type * ” then select “Just the push event”(we can also add our custom events with the help of “Let me select individual events” option under “Which events would you like to trigger this webhook?” then click on “Add Webhook” then GitHub verified url then allow webhooks.  Then in the Jenkins settings under “Triggers” select “Build with GitHub” option. Then push something to the repo you implemented Jenkins webhooks. Then go to jenkins then refresh page then verify the webhooks are triggered or not, in our case is triggered successfully. Before webhooks periodic builds are used to trigger jenkins on a time period.



Diagrams:
jenkins 


Git Repo Links:
https://github.com/SriRamCharanKolla/jenkins.git


Timestamps:
Assignment -
Interview questions:
Screen share and write Jenkins pipeline = 38:10
declarative vs scripted pipeline = 40:32





Mistakes & Learning:




Doubts Link Clarification AI chat link: