# AWS Global Content Delivery with Custom HTTPS Domain, ALB, Geo-Restrictions & Cache Invalidation

This repository documents an end-to-end Cloud & DevOps architecture built on **Amazon Web Services (AWS)** integrating **Amazon CloudFront**, **AWS Certificate Manager (ACM)**, an **Application Load Balancer (ALB)**, **Amazon EC2**, and **DigitalPlat DNS** (`www.priyank.dpdns.org`).

It demonstrates how to deliver a web application globally over **HTTPS** with a free SSL/TLS certificate, Layer 7 load balancing, edge geographic access control (`403` country blocking), and global cache invalidation (`/*`).

---

## Architecture Diagram

![AWS CloudFront + ALB + Custom Domain HTTPS Architecture](images/cloudfront-alb-custom-domain-architecture.png)

*(Tip: Save your architecture diagram inside an `images/` folder in your repository as `cloudfront-alb-custom-domain-architecture.png` so it renders directly on GitHub.)*

---

## Resource & Configuration Inventory

| Component | Configuration & Resource Identifiers |
| :--- | :--- |
| **Custom Domain & DNS** | `www.priyank.dpdns.org` (Managed via DigitalPlat DNS — `CNAME` → `d2nlacc35xztyk.cloudfront.net.`) |
| **SSL/TLS Certificate** | AWS Certificate Manager (`us-east-1`) — `priyank.dpdns.org` & `*.priyank.dpdns.org` (`6af2b3dd-6469-4c4a-a53e-601f256399d1`, DNS Validated, `Issued`) |
| **CloudFront Distribution** | `priya-cfd-alb` (`E1TTLH8LS9VCJR` / `d2nlacc35xztyk.cloudfront.net`) — `Redirect HTTP to HTTPS`, `CachingOptimized`, Auto Gzip/Brotli Compression |
| **CloudFront Origin Policy** | `HTTP only` on `Port 80` pointing to `ALB-DNS` |
| **Application Load Balancer** | `ALB-DNS` (`://amazonaws.com`) — Internet-facing, IPv4, Subnets across `us-east-1a`, `us-east-1b`, `us-east-1d` |
| **Target Group** | `WS-TG` (`HTTP:80`, Instance Target Type, Health Check Path `/`) |
| **Security Group** | `CFD-ALB-SG` — Inbound Rules: `HTTP` (`Port 80`) and `SSH` (`Port 22`) from `0.0.0.0/0` |
| **Backend Compute** | Amazon Linux 2023 EC2 (`i-0412f5032fda596a3`, `us-east-1d`) running Apache HTTP Server (`httpd`) |

---

## End-to-End Implementation Guide

### Phase 1: Issue Free SSL/TLS Certificate in AWS ACM (`us-east-1`)
1. Open **AWS Certificate Manager (ACM)** in **US East (N. Virginia) `us-east-1`** (required for CloudFront SSL certificates).
2. Request a public certificate for **`priyank.dpdns.org`** and **`*.priyank.dpdns.org`** using **DNS validation**.
3. Copy the validation `CNAME` record (`_b55ba3fb6aae9662aad565f61a599720` → `_7bff00f461e8d3278ccb40e99ffef0a9.wzccmgtwzk.acm-validations.aws.`) into **DigitalPlat DNS**.
4. Wait ~2 minutes until ACM validates the record and updates status to **`Issued`**.

---

### Phase 2: Launch Backend EC2 Web Server, Target Group & ALB (`us-east-1`)
1. **Security Group (`CFD-ALB-SG`):** Create a dedicated Security Group permitting inbound **HTTP (`Port 80`)** and **SSH (`Port 22`)** from `0.0.0.0/0`.
2. **EC2 Web Server (`i-0412f5032fda596a3`):** Launch an Amazon Linux 2023 (`t3.micro`) instance with `CFD-ALB-SG` attached and bootstrap Apache via **User Data**:
   ```bash
   #!/bin/bash
   yum install -y httpd
   systemctl enable --now httpd
   echo "<h1>Live on https://dpdns.org via AWS CloudFront + ALB!</h1>" > /var/www/html/index.html
   ```
3. **Target Group (`WS-TG`):** Create an Instance-based `HTTP:80` target group and register the running EC2 instance.
4. **Application Load Balancer (`ALB-DNS`):** Create an Internet-facing ALB mapped across the Availability Zones including the EC2 instance's AZ (`us-east-1d`), attach `CFD-ALB-SG`, and forward `HTTP:80` traffic to `WS-TG`.

---

### Phase 3: Configure Amazon CloudFront & Custom Domain Routing
1. **Create Distribution (`E1TTLH8LS9VCJR`):**
   * **Origin:** Select `ALB-DNS` (`://amazonaws.com`) with **Origin Protocol Policy** set to **`HTTP only` (`Port 80`)**.
   * **Viewer Policy & Caching:** Configure `Redirect HTTP to HTTPS`, automatic Gzip/Brotli compression, and `CachingOptimized`.
   * **Alternate Domain Name (CNAME):** Add **`www.priyank.dpdns.org`** and attach the issued **`priyank.dpdns.org`** ACM certificate.
2. **Configure DigitalPlat DNS Routing:**
   * Add a `CNAME` record mapping **`www`** → **`d2nlacc35xztyk.cloudfront.net.`** (using a trailing dot to denote a FQDN).
3. **Verify Live HTTPS Access:**
   * Open `https://dpdns.org` and confirm valid TLS encryption and edge delivery.

---

### Phase 4: Edge Security (Geo-Restrictions) & Cache Invalidation
1. **Geographic Restrictions (`403` Country Blocking):**
   * Under **CloudFront → `E1TTLH8LS9VCJR` → Security → Geographic restrictions**, configured a **Block list** for **India**, verified CloudFront edge servers blocked access with `403 ERROR: The Amazon CloudFront distribution is configured to block access from your country`, and restored access (`No restrictions`).
2. **Global Cache Invalidation (`/*`):**
   * Executed wildcard invalidation **`/*`** (`I81ISGLN60MKWVM9V1TM4OHYH1`) under **Invalidations** to purge stale edge objects and force immediate origin refresh.

---

## Key Troubleshooting Lessons & Resource Cleanup

* **Resolving `503 Service Temporarily Unavailable` (`Unused` Target):** Ensure the ALB's subnet mapping includes the exact Availability Zone (`us-east-1d`) where the backend EC2 instance resides so target health checks transition from `Unused` to `Healthy`.
* **Resolving `504 Gateway Timeout` on CloudFront:** When an ALB listens only on `HTTP:80`, CloudFront's Origin Protocol Policy must be set to **`HTTP only`** rather than `HTTPS only` (which attempts to connect to `Port 443` on the ALB).
* **DNS Trailing Dot Formatting:** In DNS management panels that auto-append the root zone, include a trailing period (`d2nlacc35xztyk.cloudfront.net.`) on external `CNAME` targets to prevent double-domain concatenation.
* **Post-Lab Resource Cleanup:** After testing, disable and delete the CloudFront distribution (`E1TTLH8LS9VCJR`), delete `ALB-DNS` and `WS-TG`, and terminate the EC2 instance (`i-0412f5032fda596a3`) to stop hourly compute/load-balancer billing while retaining the free domain and ACM certificate.
