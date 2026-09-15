Friday, 27 March 2026

Session 45 - HTTP vs HTTPS vs SSL vs TLS, How TLS works, Self signed certificate, CA, ZeroSSL CA, AWS ACM, AWS Frontend ALB.
Class Notes
HTTPS SSL TLS

30 locks with same key..

AWS certification -> certificate -> ID and website to validate

private key = public key + other elements

declaration -> self declaration

attestation notary

CA(Certificate Authority) -> joindevops.com -> provide TLS certificate

Manfacture -> UAE -> India(Main distributor) -> Wholesale -> Retail/Local Seller -> End user

Certificate Authority -> only few CA browsers and OS trust

Intermediate Certificate Authorities (Reseller of Root CA)

Root CA -> Intermediate CAS -> Certificate

Self signed certificates
=======================
We will create public key and private key for our website. We generate CSR
public key signed by private key + company info + validity + domain information

openssl req -new -key private.pem -config san.cnf -out csr.pem

1. client hello
2. server ack and sends the certificate
3. browser validates common name inside certificate and client entered website are same
certificate = public key + details
4. if same browser generates a random word and encrypt with public key
5. server decrypts that key with private key and send answer to the browser
6. if the answer matches, then browser confirms it is connecting to proper server.
7. from now onwards, every data between browser and server will be encrypted with that key.

Nginx -> website
Nginx -> Certificate+PublicKey

SSL termination

daws88s.online
*.daws88s.online
www.daws88s.online


https://frontend-dev.daws88s.online


Commands:
Frontend ALB is public so should attach SSL certificate and url should run on https.
Before to know about HTTPS should learn about HTTPs terminology refer “tls.md” file available in daws-88s github “concepts” repo.
To know who issued SSL certificate to an website click on settings icon location after refresh button in chrome then click on “Connection is secure” option then click on “Certificate is valid” then you can files all details.
Create simple “nginx” instance with “DevOps Practice” AMI with below “User data”(shell script - linux commands)
#!/bin/bash
dnf install nginx -y
systemctl start nginx  
5.  Now we can access nginx instance with http, our goal to to convert into HTTPS.
6.  ssh ec2-user@<nginx-public-ip>   =>
7.  openssl genrsa -out private.pem 4096  => Generate private key
8.  If we have private key then we can generate public key but if we have public key then we     can’t create private key.
9.  openssl pkey -in private.pem -pubout -out public.pem   => Generate the corresponding public key.
10. cat private.pem   => 
11. cat public.pem.   =>
12. openssl req -new -key private.pem -config san.cnf -out csr.pem  => Generate CSR — signed by the private key.
13. After running above command system will ask some details.
14. Enter country name.
15. Enter State or Province name. 
16. Enter City name.
17. Enter Organisation name.
18. Enter Email Address.
19. Leave “A challenge password” empty.
20. Enter “An optional company name.
21. ls -l   =>
22. cat csr.pem  => 
23. vim san.cnf  =>   
24. Enter below one and before pasting below content do edits as per your data and requirements.

[req]
distinguished_name = req_distinguished_name
x509_extensions    = v3_req
prompt             = no

[req_distinguished_name]
C  = AE
ST = Dubai
L  = Dubai
O  = joindevops
CN = daws88s.online

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = daws88s.online
DNS.2 = www.daws88s.online
DNS.3 = api.daws88s.online
DNS.4 = app.daws88s.online

25. Save changes.
26. openssl req -new -key private.pem -config san.cnf -out csr.pem  => This command to create with prompt.
27.openssl x509 -req -in csr.pem \
  -signkey private.pem \
  -out cert.crt \
  -days 365 -sha256 \
  -extfile san.cnf -extensions v3_req
28. Sign it using our own private key — hence the term self-signed
29. ls -l =>
30. cat cert.crt   => “cert.crt” is a self signed certificate.
31. Copy files to the Nginx SSL directory  =>
sudo mkdir -p /etc/nginx/ssl
sudo cp cert.crt    /etc/nginx/ssl/daws88s.online.crt
sudo cp private.pem /etc/nginx/ssl/daws88s.online.key

sudo chmod 600 /etc/nginx/ssl/daws88s.online.key
sudo chmod 644 /etc/nginx/ssl/daws88s.online.crt  

32. vim /etc/nginx/nginx.conf  => Edit the Nginx config
33.  :%d => to clear default “/etc/nginx/nginx.conf” file data
24. Copy “nginx.conf” file content from “Concepts” and paste at the end in Vim.
35.  :wq!  => To Save changes with Vim in “/etc/nginx/nginx.conf” file.
36. sudo nginx -t  => Test
37. Copy “nginx” public IP and inter in browser search the check client. NGIX should be displayed and served on HTTP.
38. sudo systemctl reload nginx  => Reload.
39. Now create R53 create for certification attached domain with “nginx” instance public IP address.
40. Now enter your domain name then click on “Connection is secure” option then click on “Certificate is valid” then you can files all details.
41. This Certification generation process only helpful for developing phase testing purpose, internal testing purpose, to test SSL certificates.
42. In Frontend ALB SSL termination is implemented to provide decrypted HTTP access to internal network to process & service client requests.
43. Create Account in “ZeroSSL” for practice purpose SSL Certificate creation. 90 days free offer is available.
44. Login into “ZeroSSL” website then click on “New Certificate” button then enter domain name then for “Validity” then click on “Next Step” button then click “Next Step” button for “Add-Ons” then toggle-off “Auto-Generate CSR” then toggle-on “Paste Existing CSR” then enter CSR key(get from nginx instance) or enter all required data like Email, country etc., for “CSR & Contact” then click on “Next Step” button then click on “Next Step” button for “Encryption Algorithm” then all toggles should be off click on “Next Step” button for “Finalise Your Order”. If asking for payment then remove “www.<domain-name>” then can able to create SSL certificate. Then Create a AWS R53 DNS Record with “CNAME” by selecting “CNAME” option in “Record type” filed copy the name given by “ZeroSSL” then enter in “Record name” filed the copy value from “ZeroSSL” and paste in AWS R52 DNS Record “value” filed then change “TTL” to 1 then click on “Create records” button. Then Click on “Next Step” button in “ZeroSSL” website then click on “Verify Domain” button then download certificate.
45. scp ca_bundle.crt ec2-user@<nginx-public-IP-Address>:/tmp   => scp means secured copy. Run from Terminal.
46. scp certificate.crt ec2-user@<nginx-public-IP-Address>:/tmp   => scp means secured copy. Run from Terminal.
47. scp private.key ec2-user@<nginx-public-IP-Address>:/tmp   => scp means secured copy. Run from Terminal.
48. cd /tmp/  =>
49. ls -l   =>
50. cat certificate.crt ca_bundle.crt > fullchain.crt   => Copy ca_bundle.crt, certificate.crt and private.key from ZeroSSL onto the server, then combine the certificate and bundle into a single fullchain file.
51. sudo cp fullchain.crt /etc/nginx/ssl/<domain-name>.crt   =>
52. sudo cp private.key   /etc/nginx/ssl/<domain-name>.key  =>
53. sudo nginx -t   =>
54. sudo systemctl reload nginx   =>
55. Now open incognito window then enter your domain in browser then you can find your website is serving on HTTPS and now click on settings icon location after refresh button in chrome then click on “Connection is secure” option then click on “Certificate is valid” then you can files all details. You will find “ZeroSSL” in Certificate Hierarchy.
56. Import “ZeroSSL” Certificate(HTTPS) in AWS.
57. Create HTTPS Certificate in AWS.
58. Create “70-acm” folder and create main.tf, provider.tf, variables.tf  files to write Terraform code for HTTPS Certificate creation with AWS Certificate Manager.
59. cd 70-acm/   => 70-acm is not dependent on anything so can execute this Terraform code directly.
60. terraform init  =>
61. terraform plan  =>
62. terraform apply -auto-approve   =>
63. for I in 00-vpc/ 10-sg/ 20-sg-rules/; do cd $i; terraform apply -auto-approve; cd ..;done   =>
64. Create “80-frontend-alb” folder and create same files like “50-backend-alb” because both are almost same.
65. Write required “80-frontend-alb” Terraform code.
66.  cd ../80-frontend-alb/   =>
67. terraform init  => 
68. terraform plan  =>
69. terraform apply -auto-approve   =>
70. 00-vpc, 20-sg, 30-sg-rules, 50-backend-alb, 80-frontend-alb are immutable infra. Remaining infra is mutable. 
71. After creating 00-vpc, 20-sg, 30-sg-rules we can create ALBs, ACM then backend components, there is no issue with this process.
72. curl https://frontend-dev.<domain-name>   =>
73.        


How to Import AWS HTTPS Certificate?    
In AWS search for “Certificate Manager” then click onit then click on “Import button if you have existing certificate else click on “Request” button to create new if no existing certificate. I am clicking on “Import” button because already created certificate with “ZeroSSL” website.
cat certificate.crt => Run this command inside nginx server then copy output and paste inside “Certificate body” field.
cat private.key  => Run this command inside nginx server then copy output and paste inside “Certificate private key” field. 
cat ca_bundle.crt  => Run this command inside nginx server then copy output and paste inside “Certificate chain - optinal” field. 
Then click on “Import Certificate” button.
After importing certificate then we will get Certificate ID and this ID we will attach to “Frontend ALB”.

How to Create AWS HTTPS Certificate?
Click on “Request” button then click on “Next” button in “Request certificate” page then under “Request public certificate page under “Domain names” enter “domain name”(aitechapp.fun) for “Fully qualified domain name” field then click on “Add another name to this certificate” button then enter “ *.<domain-name> “(this AWS service is completely free)  remaining things keep as default one then click on “Request” button then under “Domains” click on “Create records in Route 53” button then check once previously entered domains are checked or not once checked all then click on “Create records” button with this domains validation will be completed by AWS for your created certification, then go to Route 53 and check for CNAME, Records, validation. Now go to AWS Certificate Manager then under Certificates you will find a Certificate with Type “Amazon Issued”. If you don’t need created and important Certificates then delete those then will be less charged.



Diagrams:




Timestamps:




Mistakes & Learning:




Doubts Link Clarification AI chat link: