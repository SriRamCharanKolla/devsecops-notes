Tuesday, 1 September 2026

Session 76 - Multi branch pipeline, Jenkins shared library, Onboarding process, Updating GIT commit status, nodeJSEKSPipeline.
Class Notes
shared library

DRY
common functions
ansible roles
terraform modules
pipeline as library -> shared library

jenkins will give us default variables in env

call is the default function in groovy, if we call by filename by default it calls call() function

language		Platform	BuildTool
========		========	=========
nodeJS				EKS			npm -> nodeJSEKSPipeline
nodeJS				VM			npm
java				EKS			maven -> javaMavenEKSPipeline
java				EKS			gradle -> javaGradleEKSPipeline
java				EKS			ant    -> javaAntEKSPipeline
java				VM 			maven  -> javaVMPipeline
java				VM			gradle
java				VM			ant

Onboarding -> central devops/platform enginner
===========
project
component
1. make sure they have the namespace created
2. make sure they have folder in the jenkins in their project name
3. sonarqube access
4. do they need seperate agent requirements
5. do they have ECR repo
6. do they need ingress/gateway requirement
7. any databases, do you need any firewalls
8. github settings/tokens

2 prod releases

someone from their team, devops enginner

pipeline{
	agent{node{label 'wipro-finance'}}
	options{
		timeout(5 minutes)
	}
	parameters{
		string param
		boolean param
		choice param
	}
	stages{
		stage('build'){
			steps{
				script{
					sh "mvn build"
				}
			}
		}
		stage('unit test'){
			steps{
				script{
					sh "mvn test"
				}
			}
		}
		stage('scans'){
			steps{
				script{
					sh "mvn test"
				}
			}
		}
		stage('deploy'){
			steps{
				script{
					sh "mvn test"
				}
			}
		}
		
		stage('functional test'){
			steps{
				script{
					sh "mvn test"
				}
			}
		}
	}
	post{
		always{
		
		}
		success{
		
		}
		failure{
		
		}
	}
}

Commands:
Create “shared-library-test” repo in GitHub.
git clone https://github.com/SriRamCharanKolla/shared-library-test.git   => Run in your local system.
Create “vars” folder in “shared-library-test”.
Under “vars” folder in “shared-library-test” create “testPipeline.groovy” file then write jenkins pipeline code with testing stage.
Now in “catalogue” repo folder rename “Jenkinesfile” file into ““Jenkinesfile.bkp” then create new “Jenkinesfile” file then push “shared-library-test” repo code then now go to Jenkins then go to manage jenkins then go to settings then go to System then go to “Global Trusted Pipeline Libraries” then click on “+ Add” button then enter “jenkins-test-library” in “Name” field then enter “main” in “Default version” then select/keep “Modern SCM” for “Retrieval method” then enter “your-shared-library-test-github-repo-link” then click on “Apply” & “Save” buttons.
Now in “catalogue” repo folder in “Jenkinesfile” file add “@Library(‘jenkins-test-library’)” then write configMap, echo, if else code to verify its main branch or not, then go to “testPipeline.groovy” file in ‘testing’ stage echo statement access configMap data then wrap all pipeline code in “call” function then add “Map configMap” argument to the call function so that it will take catalogue Jenkins file configMap data in “testPipeline.groovy” file.
git checkout -b feature-1  =>  Run in your local system. To create “feature-1” branch.
git add . ; git commit -m “feature-1”; git push origin feature-1   =>
Now go to Jenkins “ROBOSHOP” then click on “New Item” then enter “catalogue” in “Name” field then choose “Multibranch Pipeline” then click on “OK” then click on “+ Add source” in “Branch Sources” then select “Git” then enter “your-catalogue-github-repo-link” for “Project Repository” then click on “Save” button.
Now click on “Scan Multibranch Pipeline Now” then check 2 pipelines should be create one with “feature-1” and another with “main” branches.
Now in “catalogue-unit-tests” repo folder rename “Jenkinesfile” file into ““Jenkinesfile.bkp” then create new “Jenkinesfile” file then copy and paste Now in “catalogue” repo folder “Jenkinesfile” file.
Create “jenkins-shared-library” repo in GitHub.
git clone https://github.com/SriRamCharanKolla/jenkins-shared-library.git  => Run in your local system.
Create “vars” folder in “jenkins-shared-library” then write shared library pipeline code for catalogue, nodes etc.
Now go to Jenkins then go to manage jenkins then go to settings then go to System then go to “Global Trusted Pipeline Libraries” then click on “+ Add” button then enter “jenkins-shared-library” in “Name” field then enter “main” in “Default version” then check “Load implicitly” checkbox then select/keep “Modern SCM” for “Retrieval method” then enter “your-jenkins-shared-library-github-repo-link” then click on “Apply” & “Save” buttons.
Now in “catalogue-unit-tests” repo folder rename “Jenkinesfile” file remove “@Library(‘jenkins-test-library’)” to test Jenkins library is loading from “jenkins-shared-library” then in else block change pipeline name to “nodeJSEKSPipeline” then 
Now in Jenkins “ROBOSHOP” workspace delete “catalogue” multipranch pipeline then create new “catalogue” multibranch pipeline with “catalogue-unit-tests” GitHub repo similars like catalogue multibranch pipeline.
Jenkins will verify all branches in the GitHub repo then finds Jenkins file then automatically create a pipeline.
Build pipeline and check pipeline is success or not.
Now go to “catalogue-unit-tests” GitHub repo then go to settings the under “Rules” click on “Rulessets” then click on “New ruleset” then select “New branch ruleset” then enter “main” in “Reuleset Name” field then select “Active” for “Enforcement status” then  under “Target branches” click on “Add target” button then select “Include by pattern” then enter “main” then click on “Add inclusion pattern” button then under “Branch rules” then select “Restrict deletions” then select “Require a pull request before merging” then select “1” for “Required approvals” then select “Require status checks to pass” then click on “+ Add checks” then enter “unit-tests” then select “Add unit-tests” then click on “+ Add checks” then enter “sonar-scan” then select “Add sonar-scan”.
Now do some changes in “catalogue-unit-tests” GitHub repo code.
Now to add Webhooks for nultibranch pipeline we need to install “Multibranch Scan Webhooks Trigger” Plugin in Jenkins.
Now in Jenkins “ROBOSHOP” workspace catalogue multibranch pipeline click on “Configure” then under “Scan Multibranch Pipeline Triggers” select “Scan by webhook” then enter “roboshop-catalogue” in “Trigger token” field then click on “Apply” & “Save” buttons.
Now in “catalogue-unit-tests” GitHub repo settings click on “Webhooks” then click on “Add webhook” button then enter “http://jenkins.<your-domain-name:8080/multibranch-webhook-trigger/invoke?token=roboshop-catalogue” in “Payload URL” field then select “application/json” for “Content type” then under “SSL verification select “Disable (not recommended)” then click on “Add webhook” button. Now if anything is pushed in “catalogue-unit-tests” GitHub repo then GitHub webhook will trigger a push event to Jenkins Server with “roboshop-catalogue” token as given in the webhook url payload then the Jenkins pipeline with “roboshop-catalogue” token will be trigger automatically.
Push the changes did in “catalogue-unit-tests” GitHub repo then now if the developer raise pull request the he can’t merge his code if any pipeline status like sonar scan, unit tests is failed.
Now in “jenkins-shared-library” repo folder in “nodeJSEKSPipeline.groovy” file write remaining sonar-scan, aws ecr etc code then push code and build “catalogue-unit-tests” feature-1 jenkins pipeline. Now check pull request in GitHub the Jenkins pipeline checks we set through GitHub should be success based one each stage is passed.
Similar in GitHub Webhooks we also add “triva-scan”, “library-scan”, “push-scan” checks.
After that we will deploy application to Dev Environment.

Diagrams:
everything


Git Repo Links:
https://github.com/SriRamCharanKolla/jenkins-shared-library.git
https://github.com/SriRamCharanKolla/shared-library-test.git
https://github.com/SriRamCharanKolla/catalogue.git
https://github.com/SriRamCharanKolla/catalogue-unit-tests.git


Timestamps:
Assignment -
Interview questions:






Mistakes & Learning:




Doubts Link Clarification AI chat link: