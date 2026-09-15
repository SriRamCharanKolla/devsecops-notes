Saturday, 28 March 2026

Session 47 - CDN Manual way, CDN through terraform.
Class Notes
1. Check SG rules
2. Ansible Roles service files. check port number and host_url/ip
3. Ansible Roles host URL's
4. nginx config

CDN -> Cloudfront

US -> India
latency increase, un necessary traffic/bandwidth

application -> DB
application -> Cache -> DB

browser -> origin server
CDN => Content Delivery Network
CDN == AWS Cloudfront

a network of storage servers(AWS Edge locations) across the globe used to cache the content. There will be an origin server, first application hits edge location, if data is not there then it gets data from origin and save it in edge location so that subsequent requests get data from edge location

GET POST UPDATE DELETE

GET -> can be cached -> product details(images, description, faq, rating, reviews)
TTL -> Time to live in cache
POST UPDATE DELETE -> generally Shouldn't be cached
.js .css images

invalidation
===========
when new deployment/release happens usually we should invalidate all the caching content, it deletes the data from all the edge servers. content will be downloaded again from origin to edge locations

https://frontend-dev.daws88s.online/media/stan.png
https://frontend-dev.daws88s.online/media/graph.png

/media/* -> we can cache

US Domain provider -> CDN -> ALB/Reverse proxy -> Actual web server

laptop -> Company VPN(Forward) -> Bastion Host
* client aware
* content filtering
* content monitoring
* anonymous access




How to Configure AWS CloudFront Manually?
In AWS search bar search for “CloudFront” then click on “CloudFront” then click on “Create distribution” button then select “Pay as you go” option if showing plans along with “Pay as you go” option else if not showing plans and “Pay as you go” option then in “Get started” page under “Distribution options” enter “Distribution name”(roboshop-dev) then for “Distribution type” keep “Single website or app” option same then under “Domain” enter “Route 53 managed domain - optional”(your domain name like aitechapp.fun) then click on “Check domain” button then click on “Next” button then in “Specify origin” page then under “Origin type” select “Elastic Load Balancer” option for “Origin type” then under “Origin” enter “Elastic Load Balancing origin”(frontend-dev.<domain-name>) then under “Settings” select “Customize origin settings” option for “Origin settings” then select “Customize cache settings” option for “Cache settings” then select “HTTPS only” option for “Viewer protocol policy” then select “GET, HEAD, OPTIONS” option for “Allowed HTTP methods” then select “CachingOptimized” option for “Cache policy” then click on “Next” button then in “Enable security” page select “Do not enable security protections” option for “Web Application Firewall (WAF)” then click on “Next” button then in “Get TLS certificate” page then under “TLS certificate” select “ *.<domian-name> “ for “Available certificates” then click on “Next” button then click on “Create distribution” button. It will take 5 minutes and create CDN with AWS CloudFront.

How to Configure AWS CloudFront Behaviour Manually?
Select a distribution then click on “Create behavior” then under “Settings” enter “Path pattern”(/media/*) then for “Origin and origin groups” select your “frontend-dev.<domain-name-aws-generated-text” then select “GET, HEAD, OPTIONS” for “Allowed HTTP methods” then select “CachingOptimized” option for “Cache policy” then click on “Create behaviour” button. Here creating cache policy for Media for “Roboshop”. Similarly create behaviour for Images also. Here creating cache policy for Videos for “Roboshop”. Similarly create behaviour for Images also. Finally for “Default” for “Cache policy” select “CachingDisabled”, this means if /media/* , /images/* , /videos/* are not matching in user request URL then remaining thing no need to cache.

Create Invalidation -> it is used when we did changes in resources like images, videos or all resources then with “Create Invalidation” path we can clear all cache in application. /* to clear all cache and /images/* to clear images cache.  Note: Allowed HTTP methods, Cache policy, Origin request policy, Response headers policy should be defined by developers. Cache policy should be discussed with Developer and need to setup Cache policy.

How to Create Route53 Record for AWS CloudFront URL Manually?
Go to AWS Route53 then click on DNS then select	DNS then enter “Record name”(roboshop-dev) then toggle on “Alias” button then select “Alias to CloudFront distribution” option then select “AWS CloudFront URL” then click on “Create record” button. Then copy created DNS record URL then enter in the Browser then click on chrome inspection by mouse right click then click on any .png image in the “Network” tab then in Status code 200  you can find message like “(from memory cache)” then at “X-Amz-Cf-Id” value you will also see “CloudFront”.

Commands:
Configure AWS CloudFront Manually.
Configure AWS CloudFront Behaviour Manually for Media, Images, Videos.
If any changes did in the application resources then Create Invalidation.
Once CloudFront is deployed/Ready by AWS provide one URL, take that URL and create a AWS Route53 record.
Then copy created DNS record URL then enter in the Browser then click on chrome inspection by mouse right click then click on any .png image in the “Network” tab then in Status code 200  you can find message like “(from memory cache)” then at “X-Amz-Cf-Id” value you will also see “CloudFront”.
Now test Roboshop Application with Registration with this URL “roboshop-dev.<domain-name>” then go to login page enter details then click on “Register” button, Getting error.
Now test Roboshop Application with Load Balancer with Registration with this URL “roboshop-dev.<domain-name>” then go to login page enter details then click on “Register” button. With Load Balancer its working fine no issue.
Debug issue with public endpoint URL “roboshop-dev.<domain-name>”.
When accessing Roboshop application with AWS CloudFront then getting issue.
Copy error & search for html to page the click on a website then paste this error html code then read the output error message then here the issue with CloudFront Behaviour “Default(*)” should select “GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE” for “Allow HTTP methods” so in AWS CoudFront select Roboshop distribution then click on “Behaviours” then select “Default(*)” then select “GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE” for “Allow HTTP methods” then click on “Save changes”.
AWS will take time to deploy changes.
Now implement with Terraform Code.
Create “95-cdn” folder for AWS CloudFront CDN implementation.
ls -d *     =>
for I in $(ls -d *); terraform apply -auto approve.
For CDN we are not name folder name with 100 because shell command takes 100 after 10 so the flow to infra creation missed and for issue while we are using for loop based command scripts. So that folder naming as “95-cdn”.
Create provider.ts, data.tf etc to write Terraform AWS CloudFront code.
Now once check Roboshop CloudFront distribution “Default(*)” behaviour deployment will be completed, check with previously registered login details then check with clicking on “Login” button, now the login is successful so CloudFront “Default(*)” behaviour issues resolved.
Now go to Route53 & delete CloudFront DNS record to create it with Terraform code.
Now disable Roboshop CloudFront distribution & then delete it, after disabling only we can able to delete Roboshop CloudFront distribution.
cd 95-cdn/   =>
terraform init   =>
terraform plan  =>
terraform apply -auto-approve  =>
Generally for developers we create one VPN and VPN instance in AWS and provide that VPN access to the Developers so that Developers can access Backend Components and Databases Securely and also we can track network and network required coming to     Backend Components and Databases of our company. We have complete secure control.
In interviews we can tell like we used Open VPN for our development environment.
From VPN we can access Bastion and internal URLs.
 



Diagrams:




Timestamps:
Interview questions:
What production issue you faced recently in CI/CD?
Developer forgot to update application version got production approvals from his team did deployment but pipeline failed then Client asked we managed client as client asked why pipeline failed then we replied like version is previous version developer missed manual application version so we will implement automation for version update also to avoid this problem repeatedly.




Mistakes & Learning:




Doubts Link Clarification AI chat link: