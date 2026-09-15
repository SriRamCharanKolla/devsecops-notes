Tuesday, 1 September 2026

Session 75 - Code coverage, Diff types of testing, Unit, functional, integration, E2E, etc, SonarScan, Library scan, Trivy image scan.
Class Notes
Coverage
=========
brick -> basic building block

code -> function

login(username, password)
singup(details)
forgotpassword(email_id)

what is object/class?

1. knows -> parameters
2. does -> 

1000 functions

catalogue
===========
unit testing -> testing individual function -> developers
functionality testing -> involved multiple functions -> developers/testing team

integration testing
===========
all components integratedly working or not?
cart -> catalogue
shipping -> cart

smoke testing -> after deployment, our system is working or not
regression testing -> after change, check everything. system working as expected or not

login(username, password){
	check_user_in_database()
	if (success){
		getHomePage()
	}
	else{
		sendCredsFailure()
	}
}

10 functions -> 9 functions -> 90%

we dont had scanning in our projects

6 months
=========
sonarqube installation -> dev, prod

2nd month -> onboard every project and include scanning for them
2 months -> high and critical bugs should be fixed
5th month -> we will stop the build if bugs and vuln are found
6th month -> new code, code coverage 80%, overall code 10%

npm test -> run the unit test cases
it will save the report -> .txt, .json, .html
sonarqube -> send this test report, then it shows code coverage report there

quality profiles -> dev team architect

NVD -> national vuln database -> Non profit org -> funded by US gov
google, facebook, amazon, etc...

they continously scan for vuln -> bounty programs

CVE -> Common vuln exposure
which package, version
what is the bug
what is the effect
what is the criticality -> 
7/10 -> critical
6 -> high
1 -> low
which version this bug is fixed
maintainer 
CVSS -> 

scanning software -> -> internal DB -> NVD

library scanning -> nexusiq, blackduck, github dependabot

query dependabot, check for high and critical, if found fail the pipeline

curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <TOKLE>" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  https://api.github.com/repos/daws-88s/catalogue/dependabot/alerts
  

  
Image scanning
==============
Docker -> base images

trivy -> image scanning tool

base os scan, check for high and medium alerts, if found exit, give report in table/html format

trivy image \
                                --scanners vuln \
                                --pkg-types os \
                                --severity HIGH,MEDIUM \
                                --format table \
                                --exit-code 1 \
                                --quiet \
                        160885265516.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:1.0.0
						
docker -> internal repo proxy
whitelisted base images -> 

node:18
RUN apk updte 

custom-node:18.1.0

FROM custom-node:18.1.0


Commands:
Create Jenkins & SonarQube infra with the help of “31-cicd-tools” in “roboshop-infra-eks”.
cat <jenkins-admin-password-path>  => Run in Jenkins server.
Then Create Admin User in Jenkins.
Now install all the plugins as per “31-cicd-tools” in “roboshop-infra-eks” README.md documentation.
Now in Jenkins as per “31-cicd-tools” in “roboshop-infra-eks” README.md documentation create Credentials.
Now create “roboshop” node in Jenkins then enter “3” for “Number of executions” then enter “/home/ec2-user/jenkins-agent”  for “Remote root directory” then enter “roboshop” for “Labels” then select “Only build jobs with latest expressions matching this node” then select “Launch agents via SSH” for “Launch method” then enter “jeknins-agent.<your-domain-name>” for “Host” then select “ec2-user/*****(ssh-creds)” for “Credentials” then select “Non verifying Verification Strategy” for “Host Key Verification Strategy” then click on “Save” button.
Now check “roboshop” jenkins node connected with jenkins or not.
Now setup SonarQube.
Connect to SonarQube server.
ssh -i <sonarqube-ssh-keys-path> ubuntu@<sonarqube-server-IP>  => Run in your system.
cat /opt/default-sonar-login.txt   => Run in SonarQube server.
Then in SonarQube enter password got from above command to Login into SonarQube.
Now generate SonarQube token —> go to SonarQube then go to setting then click on “My Account” then click on “Security” then under “Generate Tokens” enter “jenkins” for Name then select “Global Analysis Token” for Type then select “30 days” for “Expires in” then click on “Generate” button then now copy token then Jenkins under “Server authentication token” click on “+ Add” button then enter “genarated-sonarqube-token” in “Seret” field then enter “sonar-creds” for ID then enter “sonar-creds” for Description” as well then click on “Create” button then select “sonar-creds” for “Server authentication token” then click on “Apply” & then “Save” buttons.
Create SonarQube Webhooks for Jenkins, to create go to SonarQube click on “Administration” select “Webhooks” in “Configuration” then click on “Create” button then enter “jenkins” in Name filed then enter “http://jenkins.<your-domain:8080/sonarqube-webhook/“ in “URL” then click on “Create” button..
Now in Jenkins go to Tools & search for “SonarQube” then you will find “SonarQube Scanner” under it enter “sonar-8” in Name field then choose “SonarQube Scanner 8.1.0.6389” then click on “Apply” & then click on “Save” button.
Now in Jenkins click on “System”(system settings) under “SonarQube servers” click on “+ Add SonarQube” button then enter “sonar-server” for Name then enter “sonar-server.<your-domain-name>” for “Server URL” then enter “sonar-creds” for ID then enter “sonar-creds” for Description” as well then click on “Create” button then select “sonar-creds” for “Server authentication token” then click on “Apply” & then “Save” buttons.
Now in Jenkins click on “New Item” then enter “ROBOSHOP” then click/select “Folder” then click on “Save” button then click on “Save” button.
Now in Jenkins “ROBOSHOP” folder click on “New Item” then enter “catalogue” then select Pipeline then config pipeline script from SCM & from Git then provide git repo then click on “Apply & Save”.
Now in Jenkins click on “Build Now” button and verify pipeline success or not.
Now in SonarQube setup Quality Gates as per requirements.
Now create “catalogue-unit-tests” repo in GitHub. We already have all the test file in this repo as given by JoinDevOps we need to use it in Jenkins pipeline as a Stage to ensure all unit tests are passed in the pipeline.
Now in Jenkins in “ROBOSHOP” create “catalogue-unit-test” pipeline in this pipeline add “catalogue-unit-tests” repo and main brach then save and apply then build the pipeline then check pipeline is success, even if any single unit test case failed then pipeline will be Failed. So all unit test cases should pass.
Now in Catalogue GitHub repo setting enable “Dependency Graph” & “Dependabot” for code security sacnning then click on “Security & Quality” click on “View Dependaboat alerts” then click on the issue then verify its details.
Now in GitHub create a “Personal access token” for “catalogue” repo with “Dependaboat alerts” permissions.
Now run below command with your GitHub token to get catalogue vulnerabilities.
curl -L \ -H "Accept: application/vnd.github+json" \ -H "Authorization: Bearer <TOKLE>" \ -H "X-GitHub-Api-Version: 2026-03-10" \ https://api.github.com/repos/daws-88s/catalogue/dependabot/alerts  => 
Now add your GitHub Toke in Jenkins Credentials.
Now write a Jenkins stage to scan GitHub Dependaboat alerts with the help of Claude code.
Now in Catalogue Jekninesfile comment SonarQube related code because as of now required and its a time taking process.
Now the based of vulnerabilities found through GitHub Dependaboat alerts scan developers fix issues and then push code then pipeline will be trigger again then now no vulnerabilities will be found.
As our team is strong we also include meadium & low vulnerabilities as well.
Now Image scanning - For Image scanning we use Trivy and now in Jenkins-agent server we need to install Trivy 
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.74.0  => Run in Jenkins-agent server.
docker images  => Run in Jenkins-agent server. Copy image name.
trivy image <docker-image>  => Run in Jenkins-agent server.
 trivy image \                                --scanners vuln \                                --pkg-types os \                                --severity HIGH,MEDIUM \                                --format table \                                --exit-code 1 \                                --quiet \                       160885265516.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:1.0.0  => Run in Jenkins-agent server.




Diagrams:



Git Repo Links:
https://github.com/SriRamCharanKolla/catalogue.git
https://github.com/SriRamCharanKolla/catalogue-unit-tests.git


Timestamps:
Assignment -
Interview questions:






Mistakes & Learning:




Doubts Link Clarification AI chat link: