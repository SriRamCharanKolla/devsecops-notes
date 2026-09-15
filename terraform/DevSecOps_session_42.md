Friday, 13 March 2026

Session 42 - Load balancer manual approach(Load Balancer, Listener, Rule, Health Check, Catalogue instance and configuration, Route53 record), Load balancer using terraform(Load Balancer, Listener, Route53 record)
Class Notes
LB -> TL/TM
Listener -> 80/HTTP, 443/HTTPS
Rules -> catalogue.* -> catalogue target group
Target group -> a group of instances
Health check -> check instance is available or not

Launch template -> template to launch instance
Autoscaling -> uses launch template as input, can trigger instance if CPU utilisation is more than 70%

http://IP:8080/health

3 success -> healthy
2 failure -> not healthy

*.backend-alb-dev.daws88s.online

catalogue.backend-alb-dev.daws88s.online -> catalogue target group
user.backend-alb-dev.daws88s.online -> user target group


icicibank.com -> main host

retailbanking.icicibanking.com -> retailbanking target group
corporatebanking.icicibanking.com

retailbanking.icicibanking.com -> hostpath

icicibank.com/reatailbanking   -> Context path
icicibank.com/citbanking

10 instances -> v3

Rolling update
===============
4 instances -> v4
4 instances -> v4
2 instances -> v4

create new instance

download v4 and configure it
stop the instance
take AMI

I can use this AMI to create instances
4 instances -> v4
4 old instances -> terminate

at same point of time, our app serves 2 versions

1. load balancer
2. listener
3. r53 record

instance
configure
stop it
create AMI
create target group
add instance to target group
create ALB rule -> catalogue.alb_url

provider.tf
main.tf
variables.tf
locals.tf
data.tf

Commands:
for 
Copy “roboshop-dev-infra” DNS name once status is “Active and run “curl <paste-DNS name link>
ssh ec2-user@<catalogue-server-IP> => Connect to Catalogue EC2 instance server.
git clone <ansibleroboshop-roles-tf-repo-url> => Clone “roboshop-infra-dev” repo.
cd ansibleroboshop-roles-tf =>  
sudo dnf install ansible -y  => Install Ansible.
 ansible-playbook -e component-catalogue -e env-dev roboshop.yaml   =>
curl http://localhost:8080/health =>
Add Rule inside Load Balancer Listener.
curl http://catalogue.backend-alb-dev.daws.online/health => 
Will 503 error because not able to reach load balancer because request traffic forwarding/going to “roboshop-dev-catalogue” here  there no team members so that got “503 Service Temporarily Unavailable” because not able to reach end target, now 2nd time hitting instance is available then this error will be resolved because instance(server) is available.  => 
Create R53 record to redirect requests to load balancer target group like “*.backend-alb-dev” then select Alias toggle on to set load balancer for target group redirect.
curl http://catalogue.backend-alb-dev.<domain-name>  => To check request redirecting to target group through load balancer. 1st request will hit load balancer listener(port number 80 listener) then it check rules at present no rules so it will take default one and give “Hi, I am from Backend HTTP ALB” message. Firstly it checked url that includes “catalogue.backend-alb-dev.<domain-name>” so it allowed to port number 80 listener the did remaining process.
Write VPC, SG-Rules Terraform code to create backend_alb, catalogue_backed, SG-Rule.
Connect to Catalogue instance with Catalogue private IP address from bastion. ssh ec2-user@mongodb-dev.<domain-name> ‘netstat -lntp’  => To check mongodb running on its default port number 27027.
ssh ec2-user@<catalogue-private-IP> => 
git clone <github-ansible-roboshop-roles-tf-link>  => Clone “ansible-roboshop-roles-tf” inside catalogue server.
cd ansible-roboshop-roles-tf  => inside catalogue server.
sudo dnf install ansible -y  =>  inside catalogue server, network requests are serving by NAT Gateway.
ansible-playbook -e component=catalogue -e env=dev roboshop.yaml  => inside catalogue server,  
curl http://localhost:8080/health  => inside catalogue server,
Add load balancer listener rule.
Enter “Host header condition value”(catalogue.backend-alb-dev.<domain-name>) then select “Target group”(roboshop-dev-catalogue) then click on “Next” button then under “Listener rules” enter “Priority”(10), 1st evaluate from priority, if nothing evaluated then then evaluated from default, then click on “Next” button then click on “Add rule” button.
exit => Exit from Catalogue instance/server.
curl http://catalogue.backend-alb-dev.<domain-name>/health => from bastion server, to health check catalogue.
Got “503 Service Temporarily Unavailable” error because Target groups are not registered only registered any target.
Go to “Target group” from side menu then select “roboshop-dev-catalogue” then click on “Register target” button then select “roboshop-dev-catalogue” then click on “Include as pending below” button then click on “Register pending targets” button then check health check by clicking on refresh button under “Register targets” then it will show “Health status” as “Healthy”.
curl http://catalogue.backend-alb-dev.<domain-name>/health => from bastion server, to health check catalogue. Now will get “ok” status means success.
Deleted Manually created resource like robohop-backend-alb-dev, backend-alb load balancer, Target group then write Terraform code to create resource/infra. 
1st create Backend ALB, so create “50-backend-alb” folder then create provider.tf, main.tf file then change s3 key name to “roboshop-dev-backend-alb” in provider.tf file.
Start write complete Terraform code for main.tf, data.tf, locals.tf, variables.tf.
After writing load balancer, load balancer listener, alias Terraform code with VPC and SG groups data taking from 00-vpc, 10-sg, 20-sg-rules then run Terraform code.
Generally Terraform Load Balancer creation takes 2 to 3 minutes of time.
Check Load Balancer and Listener are created in AWS.     


How to create AWS Load Balancer Manually?
Go to AWS EC2 instance then click on “Load Balancers” in side menu then click on “Create load balancer” button then enter “Load balancer name” then select “Internal” option for “Scheme” if load balancer is for backend or any internal purpose else if for frontend or load balancer is accessible by public then choose “Internet-facing” option(must & should implement strict security rule) then select “IPv4” for “Load balancer IP address type” then under “Network mapping” select “roboshop-dev” for “VPC” then should select at least 2 availability zones since we have 2 should select both select “roboshop-dev-private-us-east-1a” subnet for “us-east-1a (use1-az1)” then select “roboshop-dev-private-us-east-1b” subnet for “us-east-1b” then under “Security groups” select “roboshop-dev-backend-alb” for Security groups” then under “Listeners and routing” select “Return fixed response” for “Default action” then enter “Response code”(200) then select “text/html” for “Content type” then write/enter “Response body” like <h1>Hi, I am from Backend HTTP ALB<h1> if any one hits backend load balancer then this success HTML code message will be displayed, then click on “Create load balancer” button.

How to create AWS Target Group for a Load Balancer Manually?
Go to AWS EC2 instance then click on “Target Groups” in side menu then click on “Create target group” button then keep default “Instances” option same for “Target type” then enter “Target group name”(roboshop-dev-catalogue) select “HTTP” for “Protocol”(keep default one same) then enter “Port” number(8080 for catalogue) then keep “IPv4” option same as default one then select “roboshop-dev” VPC option for “VPC” then keep “HTTP1” option same for “Protocol version” then under “Health checks” keep “HTTP” option same then enter “Health check path” whatever given by development team/developer like “/health” then under “Advanced health check settings” keep “Traffic port” option same for “Health check port” then enter “Healthy threshold”(like 2) then enter “Unhealthy threshold”(like 2)	consecutive health check health check like 3 healthy & 2 unhealthy then application is healthy, then enter “Timeout”(like 5 seconds for response) then enter “interval”(like for every 10 seconds load balancer should check application health with health check path given, then enter “Success code”(200-299 since status code starting from 200 are success codes) then finally click on “Next” button. Then for “Register targets” page leave it present not registering no instances then click on “Next” button then click on “Create target group” button “Available instances” page.
Provide 60 seconds for “Deregistration delay” under “Target deregistration management” then click on “Save changes” button. 


Diagrams:
roboshop-infra
load-balancer


Timestamps:
Recently Faced problem(interview question) = 00:21:00
QA = 01:22:07

Mistakes & Learning:




Doubts Link Clarification AI chat link: