# High Availability Web Infrastructure with Load Balancing & Reverse Proxy

**Author:** Qais  

## Table of Contents
1. [Introduction](#1-introduction)
2. [Infrastructure Overview](#2-infrastructure-overview)
3. [Setup Steps](#3-setup-steps)
   - [Deploying Backend Servers](#step-1-deploy-backend-servers)
   - [Configuring Reverse Proxies](#step-2-configure-reverse-proxies)
   - [Setting Up Load Balancer (ALB)](#step-3-set-up-load-balancer-alb)
   - [Configuring Route 53 for Domain Resolution](#step-4-configure-route-53)
4. [Failover and High Availability](#4-failover-and-high-availability)
5. [Security Considerations](#5-security-considerations)
6. [Testing and Validation](#6-testing-and-validation)
7. [Final Thoughts and Improvements](#7-final-thoughts-and-improvements)

---

## 1. Introduction
This documentation describes the end-to-end deployment of a highly available, load-balanced web infrastructure consisting of:

- AWS ALB (Application Load Balancer) for traffic distribution.
- Two reverse proxy instances (primary and secondary) running NGINX.
- Two backend servers serving application pages.
- SSL/TLS encryption using AWS ACM (for ALB) and Let's Encrypt (for Reverse Proxies).
- Health monitoring using a status page.
- Failover mechanism with a secondary reverse proxy.

---

## 2. Infrastructure Overview
### Architecture Diagram


```
                        ┌────────────────────────┐
                        │      Route 53 DNS      │
                        │  app.qaisnavaei.com    │
                        └──────────┬────────────┘
                                   │
                      ┌────────────▼────────────┐
                      │  AWS Application LB (ALB) │
                      │  HTTPS (ACM Managed)       │
                      └──────────┬────────────┘
                                   │
      ┌────────────────────────────┴──────────────────────────┐
      │                                                       │
      ▼                                                       ▼
Primary Reverse Proxy (NGINX)                      Secondary Reverse Proxy (NGINX)
172.31.27.144                                      172.31.31.182
(Handles normal traffic)                           (Failover instance)
      │                                                       │
      └────────────────────────────┬──────────────────────────┘
                                   │
      ┌────────────────────────────┴──────────────────────────┐
      │                                                       │
      ▼                                                       ▼
Backend Server 1 (NGINX)                          Backend Server 2 (NGINX)
172.31.24.130                                     172.31.24.203
(Serves application pages)                        (Serves application pages)
```

---

## 3. Setup Steps
### Step 1: Deploy Backend Servers


- Launched two EC2 instances in AWS (Ubuntu 22.04).
- Installed NGINX and configured simple HTML pages for identification:

```bash
sudo apt update && sudo apt install -y nginx
```
- Created `/var/www/html/index.html` with unique text on each backend.

### Step 2: Configure Reverse Proxies
![Reverse Proxy Setup](file-3HbQnF9kgS2Urknm55Br9x)

Deployed two Nginx reverse proxies to handle traffic between ALB and Backend Servers.

#### Primary Reverse Proxy Configuration (`/etc/nginx/nginx.conf`):
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}

http {
    upstream backend_servers {
        server 172.31.24.130:80;
        server 172.31.24.203:80;
    }

    server {
        listen 80;
        server_name app.qaisnavaei.com;

        location / {
            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```
Restart NGINX:
```bash
sudo systemctl restart nginx
```

### Step 3: Set Up Load Balancer (ALB)


- Created an Application Load Balancer (ALB).
- Configured target groups with the two reverse proxies.
- Assigned security groups allowing:
  - Port 80 (HTTP)
  - Port 443 (HTTPS)
- Associated ACM SSL Certificate for HTTPS.

### Step 4: Configure Route 53


- Created an A record in Route 53:
  - Name: `app.qaisnavaei.com`
  - Alias: `Yes`
  - Target: `ALB DNS Name`
- Waited for DNS propagation.

---

## 4. Failover and High Availability
- **Primary Reverse Proxy**: Handles normal traffic.
- **Secondary Reverse Proxy**: Configured as a failover instance.
- **ALB health checks** ensure only healthy instances receive traffic.

#### Health Check Configuration
- **ALB health check path**: `/health`
- **Expected response**: `200 OK`

Configured in `/etc/nginx/nginx.conf`:
```nginx
server {
    listen 80 default_server;
    server_name _;
    return 200 "OK - Reverse Proxy is running";
}
```

---

## 5. Security Considerations
### SSL/TLS
- **ACM for ALB-managed HTTPS**.
- **Let's Encrypt for reverse proxy encryption**.

### Firewalls & Security Groups
- Allowed only HTTP/HTTPS traffic.
- Limited SSH access to trusted IPs.

### DDoS Mitigation
- AWS Shield Standard for ALB.
- Rate limiting in NGINX (if needed).

---

## 6. Testing and Validation
Performed thorough testing:

### Backend Health Check
```bash
curl -I http://172.31.24.130
curl -I http://172.31.24.203
```

### Reverse Proxy Forwarding
```bash
curl -I http://app.qaisnavaei.com
```

### Load Balancer Connectivity
```bash
curl -I http://nginx-app-loadbalancer-xxxxxxxx.eu-west-2.elb.amazonaws.com
```

### Failover Test
- Stopped the primary reverse proxy.
- Verified traffic shifted to the secondary reverse proxy.

---

## 7. Final Thoughts and Improvements
- Implement auto-scaling for backends.
- Use CloudFront + WAF for better security & performance.
- Automate deployment with Terraform/Ansible.

---

## Conclusion
This documentation outlines the full end-to-end deployment of a highly available web infrastructure using AWS ALB, Reverse Proxies, and Backend Servers.

