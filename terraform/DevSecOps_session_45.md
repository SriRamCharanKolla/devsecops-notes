# Friday, 27 March 2026
# Session 45 - Cryptography, TLS Handshake, Self-Signed & CA Certificates, AWS Certificate Manager (ACM) & Frontend ALB
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Original Running Notes & Teacher Analogies](#original-running-notes--teacher-analogies)
   - [Cryptography Essentials: HTTP vs HTTPS vs SSL vs TLS](#cryptography-essentials-http-vs-https-vs-ssl-vs-tls)
   - [Public Key Cryptography & Key Pairs](#public-key-cryptography--key-pairs)
   - [The Certificate Authority (CA) Chain of Trust](#the-certificate-authority-ca-chain-of-trust)
   - [Step-by-Step TLS Handshake: Asymmetric Key Exchange to Symmetric Session](#step-by-step-tls-handshake-asymmetric-key-exchange-to-symmetric-session)
   - [Self-Signed Certificates vs Public CAs (ZeroSSL & Let's Encrypt)](#self-signed-certificates-vs-public-cas-zerossl--lets-encrypt)
   - [SSL Termination at Frontend Application Load Balancer](#ssl-termination-at-frontend-application-load-balancer)
   - [Console Walkthrough: Requesting & Importing Certificates in AWS ACM](#console-walkthrough-requesting--importing-certificates-in-aws-acm)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct Workspace File Links](#directory-structure--direct-workspace-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. End-to-End TLS Cryptographic Handshake Flow](#a-end-to-end-tls-cryptographic-handshake-flow)
     - [B. Certificate Authority Hierarchical Chain of Trust](#b-certificate-authority-hierarchical-chain-of-trust)
     - [C. Public Frontend ALB SSL Termination & Edge Ingress Architecture](#c-public-frontend-alb-ssl-termination--edge-ingress-architecture)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 8: AWS Certificate Manager (`roboshop-infra-dev/70-acm/`)](#layer-8-aws-certificate-manager-70-acm)
     - [`provider.tf`](#providertf-in-70-acm)
     - [`main.tf`](#maintf-in-70-acm)
     - [`parameters.tf`](#parameterstf-in-70-acm)
     - [`variables.tf`](#variablestf-in-70-acm)
   - [Layer 9: Public Internet-Facing ALB (`roboshop-infra-dev/80-frontend-alb/`)](#layer-9-public-internet-facing-alb-80-frontend-alb)
     - [`provider.tf`](#providertf-in-80-frontend-alb)
     - [`main.tf`](#maintf-in-80-frontend-alb)
     - [`data.tf`](#datatf-in-80-frontend-alb)
     - [`locals.tf`](#localstf-in-80-frontend-alb)
     - [`parameters.tf`](#parameterstf-in-80-frontend-alb)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Creating Practice Nginx Instance with User Data](#1-creating-practice-nginx-instance-with-user-data)
   - [2. Generating Self-Signed SSL Certificates with OpenSSL](#2-generating-self-signed-ssl-certificates-with-openssl)
   - [3. Setting up Free Trusted SSL via ZeroSSL & Route53 CNAME Validation](#3-setting-up-free-trusted-ssl-via-zerossl--route53-cname-validation)
   - [4. Deploying Automated ACM Certificate via Terraform (`70-acm`)](#4-deploying-automated-acm-certificate-via-terraform-70-acm)
   - [5. Deploying Public Frontend ALB (`80-frontend-alb`) with HTTPS](#5-deploying-public-frontend-alb-80-frontend-alb-with-https)
   - [6. Multi-Layer Infrastructure Orchestration & Cleanup](#6-multi-layer-infrastructure-orchestration--cleanup)
5. [AWS Console Step-by-Step Guides](#5-aws-console-step-by-step-guides)
   - [How to Import AWS HTTPS Certificate (from ZeroSSL)](#how-to-import-aws-https-certificate-from-zerossl)
   - [How to Request Public AWS HTTPS Certificate in ACM](#how-to-request-public-aws-https-certificate-in-acm)
6. [Commands & CLI Flags Reference Table](#6-commands--cli-flags-reference-table)
7. [Official Documentation & References](#7-official-documentation--references)
8. [High-Yield Interview Questions & Answers](#8-high-yield-interview-questions--answers)
9. [Production Mistakes & Troubleshooting Guide](#9-production-mistakes--troubleshooting-guide)
10. [Session Metadata & Timestamps](#10-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Original Running Notes & Teacher Analogies
*Captured directly from classroom discussions, whiteboards, and real-world analogies:*

1. **Symmetric Encryption Mental Model (30 Locks with the Same Key)**:
   - Imagine a building with 30 rooms, each secured by a lock. If every lock uses the **exact same key**, whoever holds that key can open all 30 rooms.
   - This represents **Symmetric Encryption** (e.g., AES-256): Fast, efficient, but if the key is intercepted during delivery, your entire security perimeter is compromised.
2. **Public Key Cryptography Formula**:
   - `private key = public key + mathematical trapdoor elements`.
   - If you possess the **Private Key**, you can easily compute and derive the **Public Key** at any time.
   - Conversely, if an attacker has only the **Public Key**, it is computationally impossible to reverse-engineer or reconstruct the Private Key!
3. **Self-Declaration vs Notary Attestation Analogy**:
   - **Self-Declaration (Self-Signed Certificate)**: You write a document saying *"I am John Doe, CEO of TechCorp"* and sign it yourself. You know it is authentic, but a bank or court will reject it because there is no trusted third party verifying your identity.
   - **Notary Attestation (Certificate Authority - CA)**: A licensed government notary inspects your passport, verifies your physical signature, and stamps an official seal. Anyone can trust this document because they trust the notary.
4. **Supply Chain Analogy for CA Chain of Trust**:
   ```
   Manufacturer (UAE) -> Main Importer (India) -> Wholesaler -> Retailer -> Local Shop -> End User
   ```
   - In TLS cryptography, the **Root Certificate Authority (Root CA)** is the Manufacturer.
   - The **Intermediate Certificate Authority (Intermediate CA)** is the authorized distributor/wholesaler.
   - The **Server/Leaf Certificate** is the product sold to the end user.
   - Operating systems and web browsers maintain a pre-loaded root store containing only a few dozen globally audited Root CAs (e.g., DigiCert, Amazon Root CA 1, Let's Encrypt ISRG Root X1). Because browsers trust the Root CA, they automatically trust any intermediate CA signed by it, and consequently trust your website's leaf certificate!
5. **Teacher's 7-Step Handshake Flow**:
   1. `client hello` (Browser announces TLS version and supported ciphers).
   2. `server ack and sends the certificate` (Server returns selected cipher + TLS Certificate containing public key).
   3. `browser validates common name inside certificate and client entered website are same` (`certificate = public key + details`). Browser verifies the certificate authority chain.
   4. `if same browser generates a random word and encrypt with public key`.
   5. `server decrypts that key with private key and send answer to the browser`.
   6. `if the answer matches, then browser confirms it is connecting to proper server`.
   7. `from now onwards, every data between browser and server will be encrypted with that key` (High-speed symmetric tunnel).
6. **Infrastructure Architectural Philosophy**:
   - `00-vpc`, `10-sg`, `20-sg-rules`, `50-backend-alb`, `80-frontend-alb` are **Immutable Infrastructure**.
   - Compute instances, AMI bakes, and application deployments are **Mutable / Ephemeral Infrastructure**.
   - After creating `00-vpc`, `10-sg`, `20-sg-rules`, you can create ALBs, ACM certificates, and backend components smoothly without dependency conflicts!

---

### Cryptography Essentials: HTTP vs HTTPS vs SSL vs TLS
1. **HTTP (Port 80)**: HyperText Transfer Protocol transfers data in plain cleartext. Anyone intercepting packets across public Wi-Fi or routers can inspect passwords, session cookies, and sensitive payloads.
2. **SSL (Secure Sockets Layer)**: The original encryption protocol created by Netscape in the 1990s (SSL v2, SSL v3). It is now **cryptographically broken and officially deprecated**.
3. **TLS (Transport Layer Security)**: The modern, secure cryptographic successor to SSL (TLS 1.2, TLS 1.3). Although people colloquially say "SSL Certificate", all modern certificates use TLS.
4. **HTTPS (Port 443)**: HTTP running inside an encrypted TLS session (`HTTP over TLS`). Guarantees three security pillars:
   - **Confidentiality**: Eavesdroppers cannot read intercepted traffic.
   - **Integrity**: Packets cannot be modified in transit without detection.
   - **Authentication**: Guarantees the client is connecting to the legitimate server, not an imposter.

---

### Public Key Cryptography & Key Pairs
TLS relies on **Asymmetric Key Cryptography**:
- **Private Key (`private.pem` / `private.key`)**:
  - Kept strictly secret on the server.
  - Used to decrypt incoming data and generate digital signatures.
- **Public Key (`public.pem` / `public.crt`)**:
  - Derived mathematically from the private key and distributed publicly.
  - Can be shared with any client. Anyone can use the public key to encrypt a message, but **only the private key can decrypt it**.
- **Certificate Signing Request (CSR - `csr.pem`)**:
  - An official application document containing the public key, domain name (Common Name - CN), Subject Alternative Names (SAN), and company details, signed by the private key and submitted to a Certificate Authority for signing.

---

### The Certificate Authority (CA) Chain of Trust
How does a browser know that `daws88s.online` or `aitechapp.fun`'s public key actually belongs to the domain owner and not an attacker?
- Trust flows through a hierarchical **Chain of Trust**:
  1. **Root Certificate Authority (Root CA)**: High-security offline root entities (e.g. DigiCert Global Root, Amazon Root CA 1, Let's Encrypt ISRG Root X1). Their root certificates are pre-installed directly into the trusted certificate store of every operating system (macOS, Windows, Linux, iOS, Android) and web browser.
  2. **Intermediate Certificate Authority (Intermediate CA)**: Subordinate CAs signed by the Root CA to handle day-to-day certificate issuance, protecting the Root CA from compromise.
  3. **Server / Leaf Certificate**: The final certificate issued to your domain (`*.aitechapp.fun` or `daws88s.online`).
- When a browser connects to a server, it walks up the certificate chain. If the chain terminates at a trusted Root CA pre-loaded in the browser, the padlock icon turns secure!

---

### Step-by-Step TLS Handshake: Asymmetric Key Exchange to Symmetric Session
Because asymmetric encryption (RSA/ECC) is computationally expensive, TLS uses asymmetric encryption **only during the initial handshake** to establish a high-speed **symmetric session key**:

```
[Client / Browser]                                             [Server / Frontend ALB]
       │                                                                   │
       │ 1. Client Hello (TLS version, supported ciphers, client_random)   │
       ├──────────────────────────────────────────────────────────────────>│
       │                                                                   │
       │ 2. Server Hello (Chosen cipher, server_random) + X.509 TLS Cert   │
       │<──────────────────────────────────────────────────────────────────┤
       │                                                                   │
       │ 3. Client validates Cert Chain against Trusted Root CA Store      │
       │    Generates Pre-Master Secret encrypted with Server Public Key   │
       ├──────────────────────────────────────────────────────────────────>│
       │                                                                   │
       │ 4. Server decrypts Pre-Master Secret using Server Private Key     │
       │    Both derive identical Symmetric Session Key                    │
       │                                                                   │
       │ 5. Encrypted Application Data (Symmetric AES-256-GCM Encryption)  │
       │<═════════════════════════════════════════════════════════════════>│
```

---

### Self-Signed Certificates vs Public CAs (ZeroSSL & Let's Encrypt)
1. **Self-Signed Certificates**:
   - The server signs its own CSR using its own private key without involving an audited Certificate Authority.
   - *Result*: Web browsers display `Your connection is not private (NET::ERR_CERT_AUTHORITY_INVALID)`.
   - *Use Case*: Internal development sandbox, testing TLS configurations, staging clusters.
2. **Public Certificate Authorities (ZeroSSL / Let's Encrypt / AWS ACM)**:
   - Third-party CAs trusted by all major operating systems and browsers.
   - Verification via **DNS-01 Challenge**: You create a specific CNAME record in Route53. The CA verifies domain ownership via DNS and issues a fully trusted SSL certificate.

---

### SSL Termination at Frontend Application Load Balancer
In enterprise multi-tier microservice architectures:
- **SSL Termination (Offloading)** is performed at the edge **Application Load Balancer (Frontend ALB)**.
- The Frontend ALB holds the SSL certificate, decrypts incoming HTTPS (port 443) traffic, and forwards plain unencrypted HTTP (port 80) traffic to backend instances inside the private subnet.
- *Benefits*:
  - Relieves microservice compute instances from heavy cryptographic handshake processing.
  - Centralizes SSL certificate management, automated renewal, and cipher policy updates in one place (AWS ACM).

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct Workspace File Links
Click any file link below to navigate directly in VS Code:

#### Project 1: SSL Certificate Automation (`roboshop-infra-dev/70-acm`)
- **Directory**: [roboshop-infra-dev/70-acm/](../../roboshop-infra-dev/70-acm)
  - [70-acm/provider.tf](../../roboshop-infra-dev/70-acm/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/70-acm/provider.tf)
  - [70-acm/main.tf](../../roboshop-infra-dev/70-acm/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/70-acm/main.tf)
  - [70-acm/parameters.tf](../../roboshop-infra-dev/70-acm/parameters.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/70-acm/parameters.tf)
  - [70-acm/variables.tf](../../roboshop-infra-dev/70-acm/variables.tf)
  - [70-acm/locals.tf](../../roboshop-infra-dev/70-acm/locals.tf)

#### Project 2: Public Frontend Load Balancer (`roboshop-infra-dev/80-frontend-alb`)
- **Directory**: [roboshop-infra-dev/80-frontend-alb/](../../roboshop-infra-dev/80-frontend-alb)
  - [80-frontend-alb/provider.tf](../../roboshop-infra-dev/80-frontend-alb/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/80-frontend-alb/provider.tf)
  - [80-frontend-alb/main.tf](../../roboshop-infra-dev/80-frontend-alb/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/80-frontend-alb/main.tf)
  - [80-frontend-alb/data.tf](../../roboshop-infra-dev/80-frontend-alb/data.tf)
  - [80-frontend-alb/locals.tf](../../roboshop-infra-dev/80-frontend-alb/locals.tf)
  - [80-frontend-alb/parameters.tf](../../roboshop-infra-dev/80-frontend-alb/parameters.tf)
  - [80-frontend-alb/variables.tf](../../roboshop-infra-dev/80-frontend-alb/variables.tf)

---

### Architectural Diagrams

#### A. End-to-End TLS Cryptographic Handshake Flow
```
+───────────────────────────────────────────────────────────────────────────────────+
| CLIENT / BROWSER                                            SERVER (Frontend ALB) |
|                                                                                   |
|  1. Client Hello (TLS 1.3, ciphers, client_random)                                |
|     ─────────────────────────────────────────────────────────────>                |
|                                                                                   |
|  2. Server Hello (Selected cipher, server_random) + X.509 Certificate             |
|     <─────────────────────────────────────────────────────────────                |
|                                                                                   |
|  3. Browser validates Certificate against Root CA Store:                          |
|     - Validates expiry date                                                       |
|     - Validates domain matches `roboshop-dev.aitechapp.fun`                       |
|     - Validates cryptographic digital signature                                   |
|                                                                                   |
|  4. Client generates Pre-Master Secret encrypted with Server Public Key           |
|     ─────────────────────────────────────────────────────────────>                |
|                                                                                   |
|  5. Server decrypts Pre-Master Secret with its private key:                       |
|     Both compute Master Secret = KDF(Pre-Master + client_random + server_random)  |
|                                                                                   |
|  6. SECURE SYMMETRIC TUNNEL ESTABLISHED:                                          |
|     HTTP GET / (AES-256-GCM Symmetrically Encrypted)                              |
|     <═════════════════════════════════════════════════════════════>               |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

#### B. Certificate Authority Hierarchical Chain of Trust
```
+───────────────────────────────────────────────────────────────────────────────────+
| ROOT CA (e.g. Amazon Root CA 1 / DigiCert Global Root CA)                         |
| - Embedded into macOS, Windows, Linux, Android, iOS OS Trust Stores               |
| - Issues signature for Intermediate CAs                                           |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| INTERMEDIATE CA (e.g. Amazon Intermediate CA 1B)                                  |
| - High-security issuing authority                                                 |
| - Signs individual leaf domain certificates                                       |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| SERVER / LEAF CERTIFICATE: `*.aitechapp.fun`                                      |
| - Bound to AWS Application Load Balancer                                          |
| - Verified automatically by all client browsers worldwide without warnings!       |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

#### C. Public Frontend ALB SSL Termination & Edge Ingress Architecture
```
                         End Users / Public Internet
                                      │
                                      │ HTTPS:443 (Encrypted TLS)
                                      ▼
+───────────────────────────────────────────────────────────────────────────────────+
| PUBLIC SUBNETS (10.0.1.0/24 & 10.0.2.0/24)                                        |
|                                                                                   |
|  FRONTEND APPLICATION LOAD BALANCER (`roboshop-dev-frontend`)                     |
|  - Scheme: Internet-Facing (Public IPs)                                           |
|  - ACM Certificate attached: `*.aitechapp.fun`                                    |
|  - SSL Termination offloads encryption processing                                |
|  - Port 443 Listener forwards plain HTTP:80 to Private Target Group               |
+───────────────────────────────────────────────────────────────────────────────────+
                                      │
                                      │ HTTP:80 (Decrypted Cleartext Traffic)
                                      ▼
+───────────────────────────────────────────────────────────────────────────────────+
| PRIVATE APP SUBNETS (10.0.11.0/24 & 10.0.12.0/24)                                 |
|                                                                                   |
|  FRONTEND WEB SERVERS (Nginx Static UI)                                           |
|  - Target Group: `roboshop-dev-frontend` (Port 80)                                |
|  - Zero SSL overhead on frontend microservice instances!                          |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 8: AWS Certificate Manager (70-acm)
> **Layer Directory**: [roboshop-infra-dev/70-acm/](../../roboshop-infra-dev/70-acm)

#### `provider.tf` in `70-acm`
> **File**: [roboshop-infra-dev/70-acm/provider.tf](../../roboshop-infra-dev/70-acm/provider.tf)

```hcl
1: terraform {
2:   required_providers {
3:     aws = {
4:       source = "hashicorp/aws"
5:       version = "6.33.0" # Terraform AWS provider version
6:     }
7:   }
8: 
9:   backend "s3" {
10:     bucket  = "devsecops-terraform-remote-state-662147645266"
11:     key     = "roboshop-dev-acm"
12:     region  = "us-east-1"
13:     encrypt = true
14:     use_lockfile   = true
15:   }
16: }
17: 
18: provider "aws" {
19:   region = "us-east-1"
20: }
```
- **Lines 1-7**: Constrains AWS Provider to version `~> 6.33.0`.
- **Lines 9-16**: Stores state remotely in S3 bucket `devsecops-terraform-remote-state-662147645266` under key `roboshop-dev-acm` using native S3 object locking (`use_lockfile = true`).

---

#### `main.tf` in `70-acm`
> **File**: [roboshop-infra-dev/70-acm/main.tf](../../roboshop-infra-dev/70-acm/main.tf)

```hcl
1: resource "aws_acm_certificate" "roboshop" {
2:   domain_name       = "*.${var.domain_name}"
3:   validation_method = "DNS"
4: 
5:   tags = merge(
6:     { Name = "${var.project}-${var.environment}-${var.domain_name}" },
7:     local.common_tags
8:   )
9: 
10:   lifecycle {
11:     create_before_destroy = true
12:   }
13: }
14: 
15: resource "aws_route53_record" "roboshop" {
16:   for_each = {
17:     for dvo in aws_acm_certificate.roboshop.domain_validation_options : dvo.domain_name => {
18:       name   = dvo.resource_record_name
19:       record = dvo.resource_record_value
20:       type   = dvo.resource_record_type
21:     }
22:   }
23: 
24:   allow_overwrite = true
25:   name            = each.value.name
26:   records         = [each.value.record]
27:   ttl             = 60
28:   type            = each.value.type
29:   zone_id         = var.zone_id
30: }
31: 
32: resource "aws_acm_certificate_validation" "roboshop" {
33:   certificate_arn         = aws_acm_certificate.roboshop.arn
34:   validation_record_fqdns = [for record in aws_route53_record.roboshop : record.fqdn]
35: }
```

##### Line-by-Line Breakdown:
- **Lines 1-13 (`aws_acm_certificate "roboshop"`)**:
  - `domain_name = "*.${var.domain_name}"`: Requests a wildcard certificate covering all subdomains (`*.aitechapp.fun`).
  - `validation_method = "DNS"`: Instructs ACM to use DNS challenge validation.
  - `lifecycle { create_before_destroy = true }`: Critical lifecycle rule ensuring certificate renewals provision a replacement certificate before deleting the old one, preventing downtime.
- **Lines 15-30 (`aws_route53_record "roboshop"`)**:
  - Iterates through `domain_validation_options` using a `for` expression, extracting the CNAME record name and value generated by ACM and publishing them directly into the Route53 hosted zone.
- **Lines 32-35 (`aws_acm_certificate_validation "roboshop"`)**:
  - Acts as a blocking synchronization resource that pauses Terraform execution until AWS completes DNS verification and updates the certificate status to `ISSUED`.

---

#### `parameters.tf` in `70-acm`
> **File**: [roboshop-infra-dev/70-acm/parameters.tf](../../roboshop-infra-dev/70-acm/parameters.tf)

```hcl
1: resource "aws_ssm_parameter" "frontend_alb_certificate_arn" {
2:   name  = "/${var.project}/${var.environment}/frontend_alb_certificate_arn"
3:   type  = "String"
4:   value = aws_acm_certificate.roboshop.arn
5: }
```
- Publishes the validated certificate ARN to SSM Parameter Store, allowing `80-frontend-alb` to consume it without hardcoding or state coupling.

---

#### `variables.tf` in `70-acm`
> **File**: [roboshop-infra-dev/70-acm/variables.tf](../../roboshop-infra-dev/70-acm/variables.tf)

```hcl
1: variable "project" {
2:   default = "roboshop"
3: }
4: 
5: variable "environment" {
6:   default = "dev"
7: }
8: 
9: variable "domain_name" {
10:   default = "aitechapp.fun"
11: }
12: 
13: variable "zone_id" {
14:   default = "Z05161353M7X4KMS0X6T4"
15: }
```

---

### Layer 9: Public Internet-Facing ALB (80-frontend-alb)
> **Layer Directory**: [roboshop-infra-dev/80-frontend-alb/](../../roboshop-infra-dev/80-frontend-alb)

#### `provider.tf` in `80-frontend-alb`
> **File**: [roboshop-infra-dev/80-frontend-alb/provider.tf](../../roboshop-infra-dev/80-frontend-alb/provider.tf)

```hcl
1: terraform {
2:   required_providers {
3:     aws = {
4:       source = "hashicorp/aws"
5:       version = "6.33.0"
6:     }
7:   }
8: 
9:   backend "s3" {
10:     bucket  = "devsecops-terraform-remote-state-662147645266"
11:     key     = "roboshop-dev-frontend-alb"
12:     region  = "us-east-1"
13:     encrypt = true
14:     use_lockfile   = true
15:   }
16: }
17: 
18: provider "aws" {
19:   region = "us-east-1"
20: }
```

---

#### `main.tf` in `80-frontend-alb`
> **File**: [roboshop-infra-dev/80-frontend-alb/main.tf](../../roboshop-infra-dev/80-frontend-alb/main.tf)

```hcl
1: resource "aws_lb" "frontend_alb" {
2:   name               = "${var.project}-${var.environment}-frontend"
3:   internal           = false # Public Internet-Facing ALB!
4:   load_balancer_type = "application"
5:   security_groups    = [local.frontend_alb_sg_id]
6:   subnets            = local.public_subnet_ids # Public subnets for internet traffic!
7:   enable_deletion_protection = false
8: }
9: 
10: resource "aws_lb_listener" "https" {
11:   load_balancer_arn = aws_lb.frontend_alb.arn
12:   port              = "443"
13:   protocol          = "HTTPS"
14:   ssl_policy        = "ELBSecurityPolicy-2016-08"
15:   certificate_arn   = local.frontend_alb_certificate_arn
16: 
17:   default_action {
18:     type = "fixed-response"
19: 
20:     fixed_response {
21:       content_type = "text/html"
22:       message_body = "<h1>Hi, I am from HTTPS Frontend ALB</h1>"
23:       status_code  = "200"
24:     }
25:   }
26: }
27: 
28: module "records" {
29:   source  = "terraform-aws-modules/route53/aws//modules/records"
30:   version = "~> 3.0"
31: 
32:   zone_name = var.domain_name
33: 
34:   records = [
35:     {
36:       name    = "${var.environment}.${var.domain_name}"
37:       type    = "A"
38:       alias   = {
39:         name    = aws_lb.frontend_alb.dns_name
40:         zone_id = aws_lb.frontend_alb.zone_id
41:       }
42:       allow_overwrite = true
43:     }
44:   ]
45: }
```

##### Line-by-Line Breakdown:
- **Lines 1-8 (`aws_lb "frontend_alb"`)**:
  - `internal = false`: Provisions public IPs attached to internet gateways.
  - `subnets = local.public_subnet_ids`: Places ALB across multiple availability zones in public subnets.
- **Lines 10-26 (`aws_lb_listener "https"`)**:
  - `port = "443"`, `protocol = "HTTPS"`: Configures secure TLS listener.
  - `certificate_arn = local.frontend_alb_certificate_arn`: Binds the ACM certificate issued in `70-acm`.
  - `ssl_policy = "ELBSecurityPolicy-2016-08"`: Defines supported TLS protocols and cryptographic cipher suites.
  - `default_action`: Serves a fixed 200 HTML response for testing until frontend microservice target groups are registered.
- **Lines 28-45 (`module "records"`)**:
  - Creates a Route53 Alias `A` record pointing `dev.aitechapp.fun` directly to the ALB DNS name with zero DNS lookup latency.

---

#### `data.tf` in `80-frontend-alb`
> **File**: [roboshop-infra-dev/80-frontend-alb/data.tf](../../roboshop-infra-dev/80-frontend-alb/data.tf)

```hcl
1: data "aws_ssm_parameter" "public_subnet_ids" {
2:   name = "/${var.project}/${var.environment}/public_subnet_ids"
3: }
4: 
5: data "aws_ssm_parameter" "frontend_alb_sg_id" {
6:   name = "/${var.project}/${var.environment}/frontend_alb_sg_id"
7: }
8: 
9: data "aws_ssm_parameter" "frontend_alb_certificate_arn" {
10:   name = "/${var.project}/${var.environment}/frontend_alb_certificate_arn"
11: }
```

---

#### `locals.tf` in `80-frontend-alb`
> **File**: [roboshop-infra-dev/80-frontend-alb/locals.tf](../../roboshop-infra-dev/80-frontend-alb/locals.tf)

```hcl
1: locals {
2:   public_subnet_ids = split(",", data.aws_ssm_parameter.public_subnet_ids.value)
3:   frontend_alb_sg_id = data.aws_ssm_parameter.frontend_alb_sg_id.value
4:   frontend_alb_certificate_arn = data.aws_ssm_parameter.frontend_alb_certificate_arn.value
5: }
```

---

#### `parameters.tf` in `80-frontend-alb`
> **File**: [roboshop-infra-dev/80-frontend-alb/parameters.tf](../../roboshop-infra-dev/80-frontend-alb/parameters.tf)

```hcl
1: resource "aws_ssm_parameter" "frontend_alb_listener_arn" {
2:   name  = "/${var.project}/${var.environment}/frontend_alb_listener_arn"
3:   type  = "String"
4:   value = aws_lb_listener.https.arn
5: }
```

---

## 4. Step-by-Step Hands-on Execution Walkthrough

### 1. Creating Practice Nginx Instance with User Data
*Create a simple Nginx instance using the "DevOps Practice" AMI to experiment with SSL/TLS:*

```bash
# Instance User Data (Shell Script executed at first boot):
#!/bin/bash
dnf install nginx -y
systemctl start nginx
```

1. Launch EC2 instance with the user data script above.
2. Open port 80 in Security Group. Verify instance is reachable via plain HTTP: `http://<nginx-public-ip>`.
3. To inspect certificate issuer on any site: Click the settings/padlock icon next to the URL bar in Chrome $\rightarrow$ Click **Connection is secure** $\rightarrow$ Click **Certificate is valid** to view issuer, SAN, and validity.

---

### 2. Generating Self-Signed SSL Certificates with OpenSSL

```bash
# 1. SSH into the practice Nginx server
ssh ec2-user@<nginx-public-ip>

# 2. Generate a 4096-bit RSA Private Key
openssl genrsa -out private.pem 4096
# Note: If you have private key, you can generate public key, but NOT vice versa!

# 3. Generate the corresponding Public Key
openssl pkey -in private.pem -pubout -out public.pem

# 4. View keys
cat private.pem
cat public.pem

# 5. Create SAN configuration file for modern browser compatibility
cat << 'EOF' > san.cnf
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
EOF

# 6. Generate Certificate Signing Request (CSR) using the configuration
openssl req -new -key private.pem -config san.cnf -out csr.pem

# View CSR details:
cat csr.pem

# 7. Sign the certificate using our own private key (Self-Signed for 365 days)
openssl x509 -req -in csr.pem \
  -signkey private.pem \
  -out cert.crt \
  -days 365 -sha256 \
  -extfile san.cnf -extensions v3_req

# 8. Copy certificate and private key to Nginx SSL directory
sudo mkdir -p /etc/nginx/ssl
sudo cp cert.crt    /etc/nginx/ssl/daws88s.online.crt
sudo cp private.pem /etc/nginx/ssl/daws88s.online.key

# Set secure file permissions
sudo chmod 600 /etc/nginx/ssl/daws88s.online.key
sudo chmod 644 /etc/nginx/ssl/daws88s.online.crt

# 9. Configure Nginx for SSL
sudo vim /etc/nginx/nginx.conf
# In Vim: run :%d to clear default configuration, paste SSL configuration, then :wq!

# 10. Test and reload Nginx
sudo nginx -t
sudo systemctl reload nginx

# 11. Create Route53 Record pointing domain to Nginx public IP
# Open browser: https://daws88s.online
# Note: Browser warns "Connection is not private" because it is Self-Signed!
```

---

### 3. Setting up Free Trusted SSL via ZeroSSL & Route53 CNAME Validation
*ZeroSSL provides free 90-day trusted certificates without browser warnings:*

1. Create a free account on [ZeroSSL](https://zerossl.com).
2. Click **New Certificate** $\rightarrow$ Enter your domain (`daws88s.online` or `aitechapp.fun`).
3. **Validity**: Select 90-Day Certificate $\rightarrow$ Click **Next Step**.
4. **Add-Ons**: Keep default $\rightarrow$ Click **Next Step**.
5. **CSR & Contact**: Toggle OFF *Auto-Generate CSR*, toggle ON *Paste Existing CSR*. Paste the contents of `csr.pem` from your Nginx server $\rightarrow$ Click **Next Step**.
6. **Encryption Algorithm**: Toggle off extra options $\rightarrow$ Click **Next Step**.
7. **Finalise Order**: If prompted for payment, remove `www.<domain-name>` to stay on the free plan.
8. **Domain Verification via DNS CNAME**:
   - ZeroSSL displays a CNAME record: `_xxxxxxxx.domain.com` pointing to `xxxxxxxx.comodoca.com`.
   - In AWS Console $\rightarrow$ **Route 53** $\rightarrow$ Hosted Zones $\rightarrow$ Click **Create Record**.
   - Type: `CNAME` | Name: Paste the verification name | Value: Paste the verification target | TTL: `60` seconds.
   - Click **Create records**.
9. In ZeroSSL: Click **Next Step** $\rightarrow$ Click **Verify Domain** $\rightarrow$ Download the certificate `.zip` package!
10. Unzip the package to find `certificate.crt`, `ca_bundle.crt`, and `private.key`.
11. Transfer files to the practice Nginx server via `scp`:
    ```bash
    scp ca_bundle.crt ec2-user@<nginx-public-IP>:/tmp
    scp certificate.crt ec2-user@<nginx-public-IP>:/tmp
    scp private.key ec2-user@<nginx-public-IP>:/tmp
    ```
12. Concatenate certificate and intermediate bundle into a single `fullchain.crt`:
    ```bash
    ssh ec2-user@<nginx-public-IP>
    cd /tmp
    cat certificate.crt ca_bundle.crt > fullchain.crt
    sudo cp fullchain.crt /etc/nginx/ssl/<domain-name>.crt
    sudo cp private.key   /etc/nginx/ssl/<domain-name>.key
    sudo nginx -t
    sudo systemctl reload nginx
    ```
13. Open an incognito browser window and navigate to `https://<domain-name>`.
14. The padlock turns green with **Connection is secure**! Certificate hierarchy displays **ZeroSSL**!

---

### 4. Deploying Automated ACM Certificate via Terraform (`70-acm`)

```bash
# Navigate to ACM layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/70-acm

# Initialize and apply
terraform init
terraform apply -auto-approve

# Verify certificate status in AWS:
aws acm list-certificates \
  --certificate-statuses ISSUED \
  --query "CertificateSummaryList[*].{Domain:DomainName,Status:Status,Arn:CertificateArn}" \
  --output table
```

---

### 5. Deploying Public Frontend ALB (`80-frontend-alb`) with HTTPS

```bash
# Navigate to Frontend ALB layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/80-frontend-alb

# Initialize and apply
terraform init
terraform apply -auto-approve

# Test HTTPS connection through the Load Balancer:
curl -iv https://dev.aitechapp.fun

# Expected Output:
# * Server certificate:
# *  subject: CN=*.aitechapp.fun
# *  issuer: C=US; O=Amazon; CN=Amazon RSA 2048 M02
# *  SSL certificate verify ok.
# < HTTP/2 200
# <h1>Hi, I am from HTTPS Frontend ALB</h1>
```

---

### 6. Multi-Layer Infrastructure Orchestration & Cleanup

#### Sequential Apply Loop for Immutable Layers:
```bash
# From roboshop-infra-dev root directory:
for i in 00-vpc/ 10-sg/ 20-sg-rules/; do
  cd $i
  terraform apply -auto-approve
  cd ..
done
```

#### Teardown Order:
> [!IMPORTANT]
> Destroy in reverse dependency order: `80-frontend-alb` $\rightarrow$ `70-acm`!

```bash
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/80-frontend-alb
terraform destroy -auto-approve

cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/70-acm
terraform destroy -auto-approve
```

---

## 5. AWS Console Step-by-Step Guides

### How to Import AWS HTTPS Certificate (from ZeroSSL)
1. Open AWS Console $\rightarrow$ Search for **Certificate Manager** $\rightarrow$ Click on it.
2. If you have an existing certificate, click **Import** button.
3. On the Nginx server or your local machine, read and paste each part:
   - Run `cat certificate.crt` inside nginx server $\rightarrow$ Copy output and paste inside **Certificate body** field.
   - Run `cat private.key` inside nginx server $\rightarrow$ Copy output and paste inside **Certificate private key** field.
   - Run `cat ca_bundle.crt` inside nginx server $\rightarrow$ Copy output and paste inside **Certificate chain - optional** field.
4. Click **Import Certificate** button.
5. AWS assigns a Certificate ARN (Certificate ID) which can now be bound to the Frontend ALB listener.

---

### How to Request Public AWS HTTPS Certificate in ACM
1. Open AWS Certificate Manager $\rightarrow$ Click **Request** button.
2. Select **Request a public certificate** $\rightarrow$ Click **Next**.
3. Under **Domain names**:
   - Enter your domain name (`aitechapp.fun`) for **Fully qualified domain name**.
   - Click **Add another name to this certificate** button $\rightarrow$ Enter `*.aitechapp.fun` (wildcard covers all subdomains, completely free).
4. Validation method: Select **DNS validation**. Key algorithm: **RSA 2048**.
5. Click **Request**.
6. In Certificate details $\rightarrow$ Under **Domains**, click **Create records in Route 53** button.
7. Verify all domains are checked $\rightarrow$ Click **Create records**.
8. ACM automatically adds CNAME records to Route53. Check Route53 for CNAME validation.
9. Within minutes, status updates to **Issued** under type **Amazon Issued**!

---

## 6. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `openssl genrsa` | `openssl genrsa -out private.pem 4096` | Generates a 4096-bit cryptographically secure RSA private key. |
| `openssl pkey` | `openssl pkey -in private.pem -pubout -out public.pem` | Derives the public key from an existing RSA private key. |
| `openssl req` | `openssl req -new -key private.pem -config san.cnf -out csr.pem` | Generates a Certificate Signing Request with SAN extensions. |
| `openssl x509` | `openssl x509 -req -in csr.pem -signkey private.pem -out cert.crt -days 365` | Self-signs an X.509 certificate for internal testing. |
| `aws acm list-certificates` | `aws acm list-certificates --certificate-statuses ISSUED` | Lists all active certificates in the AWS account. |
| `cat cert ca > fullchain` | `cat certificate.crt ca_bundle.crt > fullchain.crt` | Combines leaf certificate and intermediate CA into a full trust chain. |

---

## 7. Official Documentation & References
- **AWS Certificate Manager User Guide**: [AWS ACM Documentation](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- **Terraform `aws_acm_certificate` Resource**: [HashiCorp Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate)
- **Terraform `aws_acm_certificate_validation` Resource**: [HashiCorp Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate_validation)
- **OpenSSL Cryptography Toolkit**: [OpenSSL Official Docs](https://www.openssl.org/docs/manmaster/man1/openssl.html)

---

## 8. High-Yield Interview Questions & Answers

### Q1: What is the difference between Symmetric and Asymmetric Encryption, and why does TLS use both?
**Answer**:
- **Asymmetric Encryption (RSA, ECC)**: Uses mathematically linked key pairs (Public Key to encrypt, Private Key to decrypt). Highly secure for identity verification, but computationally heavy and slow for bulk data transfer.
- **Symmetric Encryption (AES-GCM, ChaCha20)**: Uses a single shared key for both encryption and decryption. Extremely fast and hardware-accelerated, but sharing the secret key over an unencrypted network is unsafe.
- **Why TLS Uses Both**: TLS uses **Asymmetric Encryption** during the initial handshake to authenticate the server and securely exchange a random secret. Once verified, both sides derive a **Symmetric Session Key** to encrypt the bulk application data stream with maximum throughput and sub-millisecond latency.

---

### Q2: What is SSL Termination and what are its architectural advantages?
**Answer**:
SSL Termination occurs when the edge load balancer (Frontend ALB) decrypts incoming HTTPS traffic on port 443 and passes plain HTTP traffic on port 80 to backend instances inside the private VPC network.
- **Advantages**:
  1. **Compute Optimization**: Offloads CPU-intensive cryptographic handshakes from application servers.
  2. **Centralized Certificate Management**: Single point of certificate renewal in ACM rather than distributing certificates across dozens of ephemeral EC2 instances.
  3. **Traffic Inspection & WAF**: Allows Web Application Firewalls (AWS WAF) and ALBs to inspect HTTP headers, cookies, and payloads for security threats before forwarding.

---

### Q3: How does DNS Validation work in AWS Certificate Manager (ACM)?
**Answer**:
When an ACM certificate is requested with `validation_method = "DNS"`, ACM generates unique CNAME records containing cryptographic tokens under `domain_validation_options`. You create these CNAME records in Route53. ACM queries public DNS to verify that the CNAME exists. Once verified, ACM knows you control the domain and issues the certificate. Furthermore, ACM automatically renews the certificate every year as long as the CNAME record remains in Route53!

---

### Q4: Why is `create_before_destroy = true` critical in `aws_acm_certificate`?
**Answer**:
If you modify an ACM certificate resource in Terraform, Terraform's default behavior is to delete the existing resource before creating the replacement. However, if the old certificate is currently bound to an active ALB listener, AWS prevents deletion, causing `terraform apply` to fail with a dependency conflict. Adding `create_before_destroy = true` forces Terraform to request and validate the new certificate first, bind it to the ALB listener, and only then safely destroy the old certificate without downtime.

---

## 9. Production Mistakes & Troubleshooting Guide

### 1. Incomplete Certificate Chain Causing Mobile Browser Warnings
- **Problem**: Desktop Chrome opens the site without error, but mobile Safari shows `Untrusted Certificate`.
- **Cause**: Server provided only the leaf certificate (`certificate.crt`) without concatenating the intermediate CA bundle (`ca_bundle.crt`). Desktop browsers cache intermediate CAs locally, but mobile browsers do not.
- **Fix**: Always bundle into a fullchain file before deploying to Nginx: `cat certificate.crt ca_bundle.crt > fullchain.crt`.

---

### 2. ACM Domain Validation Record Not Created in Public Hosted Zone
- **Problem**: ACM certificate stuck indefinitely in `Pending Validation` state.
- **Cause**: The Route53 hosted zone ID provided belongs to a private hosted zone, but ACM requires public DNS validation records.
- **Fix**: Verify that `var.zone_id` points to the **Public** Route53 Hosted Zone for the domain.

---

### 3. Exposing Public ALB Without HTTP to HTTPS Redirection
- **Problem**: Users typing `http://roboshop-dev.domain.com` receive connection errors or unencrypted responses.
- **Fix**: Create an HTTP (Port 80) listener on the Frontend ALB configured with a redirect action:
  ```hcl
  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
  ```

---

## 10. Session Metadata & Timestamps

- **HTTP vs HTTPS vs TLS Cryptographic Fundamentals**: `00:00 - 25:00`
- **TLS Handshake & Symmetric/Asymmetric Key Exchange**: `25:00 - 50:00`
- **OpenSSL Self-Signed & ZeroSSL Public Certificate Walkthrough**: `50:00 - 01:15:00`
- **AWS ACM & Frontend ALB Terraform Automation**: `01:15:00 - 01:30:00`

### Doubts & AI Clarification Link
- [Session 45 AI Clarification Chat](https://chat.z.ai/c/session-45-clarification)