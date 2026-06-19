# Nginx Web Server: Enterprise Configuration & Deployment

## Table of Contents
1. [Introduction and Architecture](#introduction-and-architecture)
2. [Installation and Setup](#installation-and-setup)
3. [Core Nginx Concepts](#core-nginx-concepts)
4. [Virtual Hosts and Server Blocks](#virtual-hosts-and-server-blocks)
5. [SSL/TLS Configuration](#ssltls-configuration)
6. [Load Balancing](#load-balancing)
7. [Performance Optimization](#performance-optimization)
8. [Caching Strategies](#caching-strategies)
9. [Security Hardening](#security-hardening)
10. [Monitoring and Logging](#monitoring-and-logging)
11. [Production Deployment Patterns](#production-deployment-patterns)
12. [Troubleshooting Guide](#troubleshooting-guide)
13. [Cost Analysis](#cost-analysis)

---

## Introduction and Architecture

Nginx (pronounced "engine-ex") is a high-performance, open-source web server and reverse proxy that powers approximately 35% of the world's websites. Unlike Apache, which uses a process-per-connection model, Nginx uses an event-driven, asynchronous architecture that makes it exceptionally efficient for handling thousands of concurrent connections with minimal resource consumption.

### Why Nginx on AWS?

When deploying applications on AWS EC2, Nginx serves multiple critical roles:

1. **Web Server**: Directly serves static content (HTML, CSS, JavaScript, images) with minimal overhead
2. **Reverse Proxy**: Forwards requests to application servers (Node.js, Python Flask/Django, Java Spring) running on localhost
3. **Load Balancer**: Distributes traffic across multiple application instances within a single server or across servers
4. **SSL/TLS Termination**: Handles encryption/decryption, offloading CPU-intensive cryptographic operations from application servers
5. **Caching Layer**: Reduces load on application servers by caching responses for configurable durations
6. **Rate Limiting**: Protects backend services from overwhelming traffic
7. **Compression**: Reduces bandwidth usage by compressing responses on-the-fly

### Nginx vs Other Solutions

**vs Apache**: Nginx uses fewer resources, handles concurrent connections more efficiently, and has simpler configuration syntax. Apache's module-based architecture allows for more extensibility but consumes more memory per connection.

**vs Application-level Load Balancing**: Running Nginx on the same EC2 instance as your application allows you to handle traffic distribution and SSL termination without the cost of AWS Application Load Balancer (ALB), though ALB is still needed for multi-instance distribution.

**vs CloudFront**: Nginx is deployed within your VPC for origin-facing acceleration, while CloudFront is AWS's global CDN. Both can be used together: CloudFront caches at edge locations worldwide, Nginx caches at your origin.

### Typical Architecture

```
Internet Traffic (Port 443 HTTPS)
         ↓
AWS Security Group (Allow 443, 80)
         ↓
Nginx on EC2 (Port 443 - SSL Termination)
         ↓
    Port 3000, 8080, 5000 (App Processes)
         ↓
    Application (Node.js, Python, etc.)
         ↓
    RDS Database
```

In this pattern:
- External clients connect to port 443 (HTTPS) on the EC2 instance
- Nginx terminates SSL/TLS encryption
- Nginx proxies requests to the application running on an internal port
- Application processes don't handle encryption, reducing CPU usage
- Multiple application instances can run on different ports, with Nginx load-balancing between them

---

## Installation and Setup

### Prerequisites

Before installing Nginx, ensure your EC2 instance has:
- Internet connectivity or access to package repositories
- Sufficient disk space (Nginx itself is ~5MB, plus space for logs and cache)
- Appropriate Security Group rules allowing inbound traffic on ports 80 and 443

### Installation on Amazon Linux 2 and Ubuntu

**Amazon Linux 2:**

```bash
# Update package manager
sudo yum update -y

# Install Nginx from Amazon Linux Extras
sudo amazon-linux-extras install nginx1 -y

# Start Nginx service
sudo systemctl start nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx

# Verify installation
nginx -v
# Output: nginx version: nginx/1.18.0
```

**Ubuntu 20.04 and later:**

```bash
# Update package manager
sudo apt-get update

# Install Nginx
sudo apt-get install nginx -y

# Start Nginx service
sudo systemctl start nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx

# Verify installation
nginx -v
# Output: nginx version: nginx/1.18.0
```

### Verifying Installation

After installation, verify that Nginx is running:

```bash
# Check service status
sudo systemctl status nginx

# Expected output:
# ● nginx.service - The NGINX HTTP and Reverse Proxy Server
#      Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
#      Active: active (running) since Mon 2026-05-14 10:30:00 UTC; 5s ago
```

### Directory Structure

Understanding Nginx's directory layout is crucial for effective configuration:

```
/etc/nginx/                          # Main configuration directory
├── nginx.conf                        # Primary configuration file
├── conf.d/                           # Additional configuration files
│   └── default.conf                  # Default server configuration
├── sites-available/                  # Available site configurations
├── sites-enabled/                    # Enabled site configurations (symlinks)
└── modules-enabled/                  # Loaded modules

/var/log/nginx/                       # Log directory
├── access.log                        # Request logs
└── error.log                         # Error and diagnostic logs

/var/www/html/                        # Default document root for static files

/var/cache/nginx/                     # Cache storage directory

/var/run/nginx.pid                    # Process ID file
```

On Amazon Linux 2, the structure is slightly different:
- `/etc/nginx/conf.d/` is used instead of `/etc/nginx/sites-available/`
- There is no `/etc/nginx/sites-enabled/` directory by default

### Key Configuration Principles

1. **Separation of Concerns**: Place different configurations in separate files for maintainability
2. **Symbolic Links**: Use symlinks in `sites-enabled/` to activate configurations without duplicating files
3. **Testing**: Always test configuration syntax before reloading Nginx
4. **Backup**: Always backup your configuration before making changes

---

## Core Nginx Concepts

### Configuration File Syntax

Nginx configuration files use a hierarchical structure with contexts and directives. Understanding this syntax is fundamental to effective Nginx administration.

```nginx
# Directive: key-value pair
worker_processes auto;

# Context: block containing directives
http {
    # Nested directive within http context
    server_tokens off;
    
    # Nested context: server block
    server {
        listen 80;
        server_name example.com;
        
        # Location context for matching URI patterns
        location / {
            proxy_pass http://localhost:3000;
        }
    }
}
```

### Main Contexts

**1. Main Context (Global Settings)**

The highest level of configuration affecting all connections:

```nginx
# Number of worker processes to spawn
# 'auto' detects available CPU cores
worker_processes auto;

# Maximum file descriptors per worker process
worker_rlimit_nofile 65535;

# System user for running worker processes
user nginx;

# PID file location
pid /var/run/nginx.pid;
```

**2. Events Context (Connection Processing)**

Configures how Nginx handles connections:

```nginx
events {
    # Maximum concurrent connections per worker process
    # Total connections = worker_connections × worker_processes
    # For 4 workers with 1024 connections each = 4096 total
    worker_connections 1024;
    
    # Method for multiplexing connections (epoll on Linux, kqueue on BSD)
    use epoll;
    
    # Accept multiple connections from queue at once
    multi_accept on;
}
```

**3. HTTP Context (HTTP Protocol Settings)**

Applies to all HTTP traffic:

```nginx
http {
    # Include MIME types for file responses
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';
    
    # Access log location
    access_log /var/log/nginx/access.log main;
    
    # Buffer and timeout settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
}
```

### Configuration Reload Without Downtime

One of Nginx's key advantages is the ability to reload configuration without dropping connections:

```bash
# Test configuration syntax without reloading
sudo nginx -t

# Output should be:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Reload configuration (SIGUP signal)
# This gracefully reloads configuration without stopping existing connections
sudo systemctl reload nginx

# Alternative method (direct signal)
sudo kill -HUP $(cat /var/run/nginx.pid)

# Verify reload was successful
sudo tail -f /var/log/nginx/error.log | grep -i "signal"
```

The reload process:
1. Master process reads new configuration file
2. Master spawns new worker processes with new configuration
3. Old worker processes finish existing connections
4. Old worker processes terminate once connections complete
5. No client connections are dropped (graceful reload)

---

## Virtual Hosts and Server Blocks

Virtual hosts allow a single Nginx instance to serve multiple websites or applications. Each virtual host is defined in a separate `server` block.

### Basic Server Block Structure

```nginx
server {
    # Listen on port 80 for HTTP
    # You can specify: listen 80; listen [::]:80; (IPv4 and IPv6)
    listen 80;
    
    # Server name (domain name or names this block serves)
    # Can use wildcards: *.example.com or regular expressions: ~^www\.example\.com$
    server_name example.com www.example.com;
    
    # Document root for static files
    root /var/www/example.com;
    
    # Default file to serve if directory is requested
    index index.html index.htm;
    
    # Access and error logs for this virtual host
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;
    
    # URI matching and request handling
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Server Block Matching Order

Nginx matches incoming requests against `server_name` directives in this order:

1. **Exact match**: `server_name example.com;`
2. **Wildcard match** (leftmost): `server_name *.example.com;`
3. **Wildcard match** (rightmost): `server_name example.*;`
4. **Regex match**: `server_name ~^(?<subdomain>.+)\.example\.com$;`
5. **Default server**: First server block or one marked with `default_server`

Example with multiple matching rules:

```nginx
# First server block (implicit default if no default_server is set)
server {
    listen 80;
    server_name example.com;
    # Handles: example.com
}

# Exact match takes priority
server {
    listen 80;
    server_name www.example.com;
    # Handles: www.example.com
}

# Catch-all with wildcard
server {
    listen 80;
    server_name *.example.com;
    # Handles: api.example.com, app.example.com, etc.
}

# Default for unmatched domains (low priority)
server {
    listen 80 default_server;
    server_name _;
    # Handles: any request not matched above
}
```

### Location Blocks for URI Routing

Location blocks match request URIs and determine how to handle them:

```nginx
server {
    listen 80;
    server_name api.example.com;
    
    # Exact match (highest priority)
    # Only matches URI exactly as "/api"
    location = /api {
        return 200 "API endpoint";
    }
    
    # Case-sensitive prefix match
    # Matches "/api/users", "/api/users/123", etc.
    location /api/ {
        proxy_pass http://localhost:3000;
    }
    
    # Case-insensitive prefix match
    # ~ = case-sensitive regex
    # ~* = case-insensitive regex
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        # Serve static files from disk
        root /var/www/static;
        expires 30d;
    }
    
    # Named location (used with try_files or error_page)
    location @app {
        proxy_pass http://localhost:3000;
    }
    
    # Catch-all (lowest priority)
    location / {
        try_files $uri @app;
    }
}
```

Location matching priority (highest to lowest):
1. `=` (exact match)
2. `^~` (case-sensitive prefix, stops checking)
3. `~` or `~*` (regex, first matching wins)
4. Default prefix match (longest prefix wins)

### Organizational Structure for Multiple Sites

For servers hosting many virtual hosts, organize configurations separately:

**Approach 1: Using sites-available and sites-enabled**

```bash
# Create configuration for new site
sudo nano /etc/nginx/sites-available/example.com

# Create symlink to enable it
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/example.com

# Test and reload
sudo nginx -t && sudo systemctl reload nginx
```

**Approach 2: Using conf.d directory (Amazon Linux 2)**

```bash
# Create configuration file directly
sudo nano /etc/nginx/conf.d/example.com.conf

# Test and reload
sudo nginx -t && sudo systemctl reload nginx
```

**Approach 3: Single file with multiple server blocks**

```bash
# Edit main configuration
sudo nano /etc/nginx/nginx.conf

# Add multiple server blocks in http context
http {
    include /etc/nginx/conf.d/*.conf;
}
```

### Example: Multi-Tenant Application

A common scenario where one Nginx instance serves multiple applications:

```nginx
# /etc/nginx/sites-available/multi-tenant

# Tenant 1: API
server {
    listen 80;
    server_name api.customer1.example.com;
    
    location / {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Tenant 2: Web App
server {
    listen 80;
    server_name app.customer1.example.com;
    
    location / {
        proxy_pass http://localhost:3002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Tenant 3: Static Site
server {
    listen 80;
    server_name customer2.example.com;
    
    root /var/www/customer2;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## SSL/TLS Configuration

SSL/TLS encryption is mandatory for modern web applications. Nginx efficiently terminates SSL connections, decrypting incoming HTTPS traffic and proxying to backend servers over HTTP.

### Obtaining Certificates

**Using Let's Encrypt with Certbot (Recommended)**

Let's Encrypt provides free SSL certificates that are automatically renewed:

```bash
# Install Certbot
sudo apt-get install certbot python3-certbot-nginx -y  # Ubuntu
# or
sudo yum install certbot python3-certbot-nginx -y      # Amazon Linux 2

# Obtain certificate (automatic Nginx configuration)
sudo certbot --nginx -d example.com -d www.example.com

# Non-interactive (for automation):
sudo certbot certonly --standalone -d example.com -d www.example.com --non-interactive --agree-tos -m admin@example.com

# Renew certificates (runs automatically via cron)
sudo certbot renew

# List all certificates
sudo certbot certificates

# Renew specific certificate
sudo certbot renew --cert-name example.com
```

**Using AWS Certificate Manager (ACM)**

If using AWS load balancers with Nginx behind them, use AWS-managed certificates:

```bash
# Certificates are managed in AWS Console or via CLI
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" \
  --domain-validation-options DomainName=example.com,ValidationDomain=example.com \
  --region us-east-1
```

**Self-Signed Certificates (Development Only)**

```bash
# Generate private key and certificate (valid for 365 days)
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/private.key \
  -out /etc/nginx/ssl/certificate.crt \
  -subj "/C=US/ST=State/L=City/O=Company/CN=example.com"

# Verify certificate
openssl x509 -in /etc/nginx/ssl/certificate.crt -text -noout
```

### SSL Configuration Basics

**Minimal HTTPS Server Block**

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL certificate and private key paths
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP redirect to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

### SSL Security Configuration

Modern SSL/TLS configuration balancing security with browser compatibility:

```nginx
# File: /etc/nginx/ssl.conf (included in server blocks)

# SSL protocol versions (modern: TLS 1.2+)
ssl_protocols TLSv1.2 TLSv1.3;

# Preferred cipher suites (priority order)
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';

# Prefer server ciphers over client preferences
ssl_prefer_server_ciphers on;

# Session cache for SSL session reuse (improves performance)
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;

# Diffie-Hellman parameter for forward secrecy
# Generate once: openssl dhparam -out /etc/nginx/dhparam.pem 2048
ssl_dhparam /etc/nginx/dhparam.pem;

# Enable SSL session tickets (allows resumption across connections)
ssl_session_tickets on;

# HTTP Strict Transport Security (tell browsers to always use HTTPS)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

# Certificate Authority Authorization (prevent unauthorized certificate issuance)
add_header CAA "0 issue \"letsencrypt.org\"" always;
```

In your server block:

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # Include shared SSL configuration
    include /etc/nginx/ssl.conf;
    
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

### Certificate Renewal Automation

Let's Encrypt certificates expire every 90 days and must be renewed. Certbot handles this automatically:

```bash
# Verify auto-renewal is configured
sudo systemctl list-timers | grep certbot

# Expected output:
# NEXT                        LEFT     LAST                        PASSED UNIT              
# Thu 2026-05-15 03:12:00 UTC 17h left Wed 2026-05-14 15:30:15 UTC 5h ago certbot.timer

# Manual renewal (runs automatically daily)
sudo certbot renew --dry-run

# Force renewal of specific certificate
sudo certbot renew --cert-name example.com --force-renewal

# Setup email notifications for renewal failures
sudo certbot update_account --email admin@example.com
```

### Testing SSL Configuration

```bash
# Test SSL with OpenSSL
openssl s_client -connect example.com:443 -tls1_2

# Check certificate expiration with online tools
curl -sI https://example.com | grep -i "date\|expires"

# Grade your SSL configuration (A+, A, B, C)
# Use: https://www.ssllabs.com/ssltest/

# Command-line SSL testing
testssl() {
    docker run --rm -it nmap/nmap:latest nmap --script ssl-enum-ciphers -p 443 "$1"
}
testssl example.com
```

---

## Load Balancing

Nginx can distribute traffic across multiple backend servers (application instances, databases, caches) using various load-balancing algorithms. This is useful for:
1. Distributing traffic to multiple application processes on the same server
2. Distributing traffic across multiple servers in an Auto Scaling Group
3. Distributing traffic across multiple database replicas

### Upstream Blocks (Backend Groups)

Define groups of backend servers:

```nginx
# Simple upstream with multiple servers
upstream app_servers {
    server localhost:3000;
    server localhost:3001;
    server localhost:3002;
}

# Usage in server block
server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://app_servers;
    }
}
```

### Load Balancing Algorithms

**1. Round Robin (Default)**

Distributes requests equally across all servers:

```nginx
upstream app_servers {
    server app1.example.com:3000;
    server app2.example.com:3000;
    server app3.example.com:3000;
}

# Request distribution:
# Request 1 → app1
# Request 2 → app2
# Request 3 → app3
# Request 4 → app1 (cycle repeats)
```

**2. Least Connections**

Routes requests to server with fewest active connections (better for long-lived connections):

```nginx
upstream app_servers {
    least_conn;
    
    server app1.example.com:3000;
    server app2.example.com:3000;
    server app3.example.com:3000;
}

# Useful for: WebSockets, long-polling, streaming connections
```

**3. IP Hash**

Routes requests from same IP to same server (maintains session affinity):

```nginx
upstream app_servers {
    ip_hash;
    
    server app1.example.com:3000;
    server app2.example.com:3000;
    server app3.example.com:3000;
}

# All requests from 192.168.1.100 → same server
# Useful for session affinity without shared session storage
```

**4. Hash (Custom)**

Routes based on custom key (e.g., user ID, cookie value):

```nginx
upstream app_servers {
    hash $cookie_userid;
    
    server app1.example.com:3000;
    server app2.example.com:3000;
    server app3.example.com:3000;
}

# All requests with same userid cookie → same server
# Maintains session affinity without IP tracking
```

**5. Random**

Distributes requests randomly (useful for load balancing across many servers):

```nginx
upstream app_servers {
    random;
    
    server app1.example.com:3000;
    server app2.example.com:3000;
    server app3.example.com:3000;
}
```

### Server Weights

Distribute unequal load based on server capacity:

```nginx
upstream app_servers {
    server app1.example.com:3000 weight=1;      # Gets 25% of requests
    server app2.example.com:3000 weight=2;      # Gets 50% of requests
    server app3.example.com:3000 weight=1;      # Gets 25% of requests
}

# Total weight = 1 + 2 + 1 = 4
# app2 gets 2/4 = 50% of requests
```

### Server States

Control server availability in load-balancing group:

```nginx
upstream app_servers {
    server app1.example.com:3000;               # Active (default)
    server app2.example.com:3000 backup;        # Used only if others down
    server app3.example.com:3000 down;          # Permanently disabled
    server app4.example.com:3000 max_fails=3 fail_timeout=30s;
}

# max_fails: Mark down after N failures
# fail_timeout: Wait this long before retrying after marking down
```

### Health Checks

Monitor backend server health and automatically remove unhealthy servers:

```nginx
upstream app_servers {
    # Active health check (Nginx Plus feature, not in open source)
    # Check /health endpoint every 5 seconds
    server app1.example.com:3000;
    server app2.example.com:3000;
    server app3.example.com:3000;
    
    # Open source fallback: passive health check
    server app1.example.com:3000 max_fails=3 fail_timeout=30s;
}

# Passive health check:
# - After 3 failed requests, mark server as down
# - After 30 seconds, try again
# - If requests succeed, mark as up

# Simple health check endpoint your app should provide:
# GET /health → HTTP 200 OK
# Response body: { "status": "healthy" }
```

For more sophisticated health checking, implement passive checks in your application:

```nginx
# Application-level health check
server {
    listen 80;
    server_name example.com;
    
    location /health {
        access_log off;  # Don't log health checks
        
        # Check if app is responsive
        proxy_pass http://app_servers;
        proxy_connect_timeout 1s;
        proxy_read_timeout 1s;
    }
}
```

### Example: Multi-Server Load Balancing

Distributing traffic across multiple EC2 instances in an Auto Scaling Group:

```nginx
# /etc/nginx/conf.d/load-balance.conf

upstream app_cluster {
    # Instance 1
    server 10.0.1.50:3000 max_fails=3 fail_timeout=30s;
    # Instance 2
    server 10.0.1.51:3000 max_fails=3 fail_timeout=30s;
    # Instance 3
    server 10.0.1.52:3000 max_fails=3 fail_timeout=30s;
    # Instance 4 (backup - used only if others fail)
    server 10.0.1.53:3000 backup;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    
    # Load balance to application cluster
    location / {
        proxy_pass http://app_cluster;
        
        # Pass original client information to backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Connection pooling and keepalive
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Timeout settings
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

### Dynamic Load Balancing

As servers scale up/down in an Auto Scaling Group, the IP addresses in upstream blocks must be updated. Use one of these approaches:

**Approach 1: Periodic Configuration Updates (Manual)**

```bash
#!/bin/bash
# update-upstream.sh - Run via cron every minute

# Get IP addresses from Auto Scaling Group
IPS=$(aws ec2 describe-instances \
  --filters "Name=tag:aws:autoscaling:groupName,Values=my-app-asg" \
  --query "Reservations[0].Instances[*].PrivateIpAddress" \
  --region us-east-1 \
  --output text)

# Generate upstream configuration
cat > /tmp/upstream.conf << EOF
upstream app_cluster {
EOF

for ip in $IPS; do
    echo "    server $ip:3000 max_fails=3 fail_timeout=30s;" >> /tmp/upstream.conf
done

echo "}" >> /tmp/upstream.conf

# Replace if changed
if ! diff -q /tmp/upstream.conf /etc/nginx/conf.d/upstream.conf >/dev/null 2>&1; then
    sudo cp /tmp/upstream.conf /etc/nginx/conf.d/upstream.conf
    sudo nginx -s reload
    echo "Upstream configuration updated at $(date)"
fi

# Add to cron for automatic updates
# */1 * * * * /usr/local/bin/update-upstream.sh
```

**Approach 2: DNS-Based Load Balancing**

Create a DNS name that resolves to all servers in the ASG:

```nginx
upstream app_cluster {
    # DNS name of AWS ELB or Route53 service discovery endpoint
    server app-cluster.internal:3000;
}

# Reload every 60 seconds to pick up new DNS entries
# (requires resolver configuration)
```

---

## Performance Optimization

### Buffer Management

Nginx uses buffers for reading client requests and backend responses. Improper buffer sizing impacts both memory usage and performance:

```nginx
http {
    # Client request body buffer
    # Small files are buffered in memory; larger files to disk
    client_body_buffer_size 128k;
    
    # Client request headers buffer (usually doesn't need tuning)
    client_header_buffer_size 1k;
    client_max_header_size 4k;
    
    # Backend response buffering
    # Responses larger than buffer_size use backend_buffering_number_of_pages
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;              # 8 buffers of 4KB each
    proxy_busy_buffers_size 8k;      # Start flushing when this is used
    
    # Large file uploads/downloads
    client_body_timeout 20s;
    client_header_timeout 20s;
    send_timeout 20s;
}
```

### Connection Keepalive

Reusing connections reduces overhead:

```nginx
http {
    # Backend keepalive connections
    upstream app_servers {
        keepalive 32;  # Reuse up to 32 connections to backend
        server localhost:3000;
        server localhost:3001;
    }
    
    server {
        location / {
            proxy_pass http://app_servers;
            proxy_http_version 1.1;    # Required for keepalive
            proxy_set_header Connection "";  # Don't close connection
        }
    }
    
    # Client keepalive timeout (how long to keep idle connections alive)
    keepalive_timeout 65s;
    
    # Maximum number of requests on single connection
    # Limits resource per connection to prevent abuse
    keepalive_requests 100;
}
```

### Compression

Reduce bandwidth by compressing responses:

```nginx
http {
    # Enable gzip compression
    gzip on;
    
    # Compression level: 1 (fast) to 9 (best compression)
    # Usually 5-6 provides good balance
    gzip_comp_level 6;
    
    # Minimum response size to compress
    # Don't compress tiny responses (overhead > savings)
    gzip_min_length 1000;
    
    # Content types to compress
    gzip_types text/plain text/css text/javascript 
               text/xml text/x-component text/x-cross-domain-policy
               application/javascript application/json application/xml 
               application/rss+xml application/atom+xml 
               font/truetype font/opentype;
    
    # Don't compress already-compressed content
    # (images, videos, archives are pre-compressed)
    gzip_disable "MSIE [1-6]\.";  # Disable for old IE browsers
    
    # Vary header for caches (tell proxies/CDN cached content varies by encoding)
    gzip_vary on;
}
```

### Static File Serving Optimization

Cache static files aggressively:

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    # Cache for 1 year (longest practical duration)
    expires 1y;
    add_header Cache-Control "public, immutable";
    
    # Allow conditional requests (If-Modified-Since, ETag)
    add_header ETag W/"$filemd5$http_if_modified_since";
    
    # Disable access logging for static files (reduces disk I/O)
    access_log off;
    
    # Serve from disk with sendfile (zero-copy)
    sendfile on;
    sendfile_max_chunk 1m;  # Prevent starving other requests
}

# Dynamic content (HTML, API responses) - don't cache
location / {
    expires -1;  # Disable caching
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    proxy_pass http://app_servers;
}
```

### Worker Process Configuration

Optimize process count and resource limits:

```nginx
# Use one worker per CPU core for optimal performance
worker_processes auto;

# Detect automatically (works on Linux)
worker_processes 4;  # For 4-core instance

# Open files limit per worker
worker_rlimit_nofile 65535;

# CPU affinity - bind workers to specific cores
worker_cpu_affinity 0001 0010 0100 1000;

events {
    # Max connections per worker
    # Total max connections = worker_processes × worker_connections
    worker_connections 1024;      # For general workloads
    worker_connections 2048;      # For high-traffic sites
    worker_connections 4096;      # For very high traffic
}
```

### Kernel Parameter Tuning

System-level tuning for high-traffic scenarios:

```bash
# File: /etc/sysctl.d/99-nginx.conf

# TCP backlog (queue of incoming connections)
net.core.somaxconn = 65535

# Maximum half-open connections (SYN queue)
net.ipv4.tcp_max_syn_backlog = 65535

# TCP connection reuse (for rapid reconnections)
net.ipv4.tcp_tw_reuse = 1

# File descriptor limits
fs.file-max = 2097152

# Apply settings
sudo sysctl -p /etc/sysctl.d/99-nginx.conf
```

### Performance Monitoring Variables

Track performance with Nginx built-in variables:

```nginx
# Log format with performance metrics
log_format performance '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '$request_time $upstream_response_time';

# Variables explained:
# $request_time: Time from receiving request to sending last byte
# $upstream_response_time: Time waiting for backend response
# $pipe: Whether connection was pipelined
# $connection: Connection number
# $connection_requests: Number of requests on this connection
```

---

## Caching Strategies

Nginx can cache responses from backend servers, dramatically reducing load and improving response times. Unlike traditional HTTP caching headers, Nginx's cache can operate independently of server-provided cache directives.

### Proxy Cache Zones

Define cache storage locations:

```nginx
http {
    # Define cache zone named 'cache_zone'
    # Location: /var/cache/nginx/app_cache
    # Size: 1GB of cache data
    # Inactive entries removed after 60 minutes of no access
    proxy_cache_path /var/cache/nginx/app_cache 
        levels=1:2                    # Directory structure: 1/2 (2 levels deep)
        keys_zone=cache_zone:10m      # 10MB shared memory for cache keys
        max_size=1g                   # Maximum cache size
        inactive=60m                  # Remove unused entries after 60 minutes
        use_temp_path=off;            # Don't use temp directory
    
    # Separate cache zone for static assets (longer retention)
    proxy_cache_path /var/cache/nginx/static_cache
        levels=1:2
        keys_zone=static_cache:50m
        max_size=5g
        inactive=1d;
}
```

### Basic Caching Configuration

```nginx
server {
    listen 80;
    server_name api.example.com;
    
    # Enable proxy caching using 'cache_zone'
    proxy_cache cache_zone;
    
    # Cache successful responses (200, 301, 302, 304, 307)
    proxy_cache_valid 200 302 10m;        # 10 minutes
    proxy_cache_valid 304 1h;             # 1 hour
    proxy_cache_valid 404 1m;             # 1 minute (error caching)
    
    # Cache responses regardless of backend headers
    # (overrides Cache-Control: no-cache from backend)
    proxy_ignore_headers Cache-Control Set-Cookie;
    
    # Use stale cache if backend is down
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
    
    # Cache key (what makes entries unique)
    proxy_cache_key "$scheme$request_method$host$request_uri";
    
    # Add cache status header to responses
    add_header X-Cache-Status $upstream_cache_status always;
    # Response values: HIT (from cache), MISS (request backend), BYPASS, etc.
    
    location / {
        proxy_pass http://app_servers;
    }
}
```

### Cache Purging

Remove cached entries manually:

```nginx
# Enable cache purging for specific paths
map $request_method $purge_method {
    PURGE 1;
    default 0;
}

server {
    listen 80;
    server_name api.example.com;
    
    # Configure caching
    proxy_cache cache_zone;
    proxy_cache_valid 200 10m;
    
    # Purge cache on request
    location ~ /purge(/.*) {
        # Restrict to localhost (for security)
        allow 127.0.0.1;
        allow 10.0.0.0/8;  # Private network
        deny all;
        
        proxy_cache_purge cache_zone "$scheme$request_method$host$1";
    }
    
    # Example purge commands:
    # curl http://api.example.com/purge/
    # curl http://api.example.com/purge/api/users/123
}
```

Manual cache clearing:

```bash
# Clear entire cache zone
sudo rm -rf /var/cache/nginx/app_cache/*

# Clear specific path
sudo find /var/cache/nginx/app_cache -name "*users*" -delete

# Reload Nginx to flush cache
sudo systemctl reload nginx
```

### Conditional Caching

Cache selectively based on request properties:

```nginx
# Don't cache authenticated requests
map $http_cookie $cache_bypass {
    ~*auth 1;  # If cookie contains 'auth', bypass cache
    default 0;
}

server {
    proxy_cache cache_zone;
    proxy_cache_bypass $cache_bypass;  # Skip cache for matching requests
    
    location ~ ^/api/ {
        proxy_pass http://app_servers;
        
        # Don't cache if authorization header present
        proxy_cache_bypass $http_authorization;
    }
    
    location ~ ^/admin/ {
        proxy_pass http://app_servers;
        
        # Don't cache POST/PUT/DELETE requests (only GET)
        proxy_cache_bypass $request_method;
    }
}
```

### Cache Statistics and Debugging

Monitor cache effectiveness:

```bash
# View cache directory size
du -sh /var/cache/nginx/

# List cached files
find /var/cache/nginx -type f | head -20

# Real-time cache hit rate
tail -f /var/log/nginx/access.log | grep -c "X-Cache-Status: HIT"
```

Add cache status to logs:

```nginx
log_format cache_log '$remote_addr [$time_local] '
                     '"$request" $status '
                     'Cache: $upstream_cache_status '
                     'Time: ${request_time}s '
                     'Upstream: ${upstream_response_time}s';

access_log /var/log/nginx/cache.log cache_log;

# Expected output with cache hits:
# 192.168.1.100 [14/May/2026:10:30:45 +0000] "GET /api/users HTTP/1.1" 200 Cache: HIT Time: 0.001s Upstream: -
```

---

## Security Hardening

### Hiding Nginx Version

Prevent attackers from exploiting known Nginx vulnerabilities:

```nginx
http {
    # Hide Nginx version in Server header and error pages
    server_tokens off;
    
    # Hide X-Powered-By header if backend sends it
    proxy_set_header X-Powered-By "";
}
```

### Rate Limiting

Prevent abuse and DDoS attacks:

```nginx
http {
    # Define rate limit zone
    # "addr" = limit per IP address
    # rate = 10 requests per second
    # zone name = name_of_zone
    # shared memory size = 10m (can store ~100k IPs)
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    
    # Define stricter limit for login attempts
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
}

server {
    listen 80;
    server_name example.com;
    
    location /api/ {
        # Apply rate limit: 10 requests/sec per IP
        # burst=20: Allow up to 20 requests if over limit (traffic spike)
        # nodelay: Don't delay requests, return 429 immediately if over limit
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://app_servers;
    }
    
    location /login {
        # Stricter rate limit: 5 requests per minute per IP
        limit_req zone=login_limit burst=3;
        
        proxy_pass http://app_servers;
    }
    
    # Return 429 status for rate limit exceeded
    error_page 429 =429 @rate_limit_exceeded;
    
    location @rate_limit_exceeded {
        return 429 '{"error":"Rate limit exceeded"}';
        add_header Content-Type application/json;
    }
}
```

### Request Size Limits

Prevent memory exhaustion from large requests:

```nginx
http {
    # Maximum allowed client request body size (POST data, file uploads)
    client_max_body_size 100m;
    
    # Timeout for reading client request body
    client_body_timeout 20s;
    
    # Timeout for reading client request headers
    client_header_timeout 20s;
    
    # Timeout for backend response
    proxy_read_timeout 30s;
}

server {
    location /upload {
        # Allow large file uploads
        client_max_body_size 500m;
        
        proxy_pass http://app_servers;
    }
    
    location /api {
        # Strict limit for API requests
        client_max_body_size 1m;
        
        proxy_pass http://app_servers;
    }
}
```

### Security Headers

Add headers that instruct browsers to enforce security policies:

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    # Strict Transport Security (force HTTPS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Content Security Policy (prevent XSS and clickjacking)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
    
    # X-Frame-Options (prevent clickjacking)
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # X-Content-Type-Options (prevent MIME sniffing)
    add_header X-Content-Type-Options "nosniff" always;
    
    # X-XSS-Protection (legacy XSS protection)
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Referrer Policy (control what referrer info is sent)
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    location / {
        proxy_pass http://app_servers;
    }
}
```

### Access Control

Restrict access by IP address:

```nginx
server {
    location /admin {
        # Deny access from everyone by default
        deny all;
        
        # Allow specific IPs
        allow 192.168.1.0/24;  # Office network
        allow 10.0.0.0/8;      # VPC
        allow 203.0.113.42;     # Specific IP
        
        proxy_pass http://app_servers;
    }
}
```

### CORS Headers

Enable cross-origin requests when needed:

```nginx
server {
    location /api {
        # Allow requests from specified origins
        if ($http_origin ~* ^https?://(example\.com|api\.example\.com)$) {
            add_header 'Access-Control-Allow-Origin' $http_origin always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
        }
        
        # Handle preflight requests
        if ($request_method = 'OPTIONS') {
            return 204;
        }
        
        proxy_pass http://app_servers;
    }
}
```

---

## Monitoring and Logging

### Access Logs

Detailed request logging for troubleshooting and analytics:

```nginx
http {
    # Default log format
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
    
    # Detailed format with timing information
    log_format detailed '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent $request_time '
                        'upstream: $upstream_addr $upstream_status $upstream_response_time';
    
    # JSON format for machine parsing
    log_format json escape=json
    '{'
        '"time_local":"$time_local",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"bytes_sent":$bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time"'
    '}';
    
    access_log /var/log/nginx/access.log main;
}

server {
    # Per-server specific logging
    access_log /var/log/nginx/example.com.access.log detailed;
    error_log /var/log/nginx/example.com.error.log warn;
    
    # Disable logging for specific requests (e.g., health checks)
    location /health {
        access_log off;
        proxy_pass http://app_servers;
    }
}
```

### Log Rotation

Prevent log files from consuming all disk space:

```bash
# File: /etc/logrotate.d/nginx

/var/log/nginx/*.log {
    daily                 # Rotate daily
    missingok             # Don't error if file missing
    rotate 14             # Keep 14 days of logs
    compress              # Gzip old logs
    delaycompress         # Don't compress yesterday's log
    notifempty            # Don't rotate empty files
    create 0640 nginx nginx  # Create new log file
    sharedscripts         # Run postrotate once, not per file
    postrotate
        [ ! -f /var/run/nginx.pid ] || kill -USR1 `cat /var/run/nginx.pid`
    endscript
}

# Test logrotate (without actually rotating)
sudo logrotate -d /etc/logrotate.d/nginx

# Force rotation
sudo logrotate -f /etc/logrotate.d/nginx
```

### Performance Metrics

Extract performance insights from logs:

```bash
# Average response time
tail -100000 /var/log/nginx/access.log | awk '{print $(NF-2)}' | awk '{sum+=$1; count++} END {print "Average:", sum/count}'

# 95th percentile response time
tail -100000 /var/log/nginx/access.log | awk '{print $(NF-2)}' | sort -n | awk '{a[NR]=$1} END {print a[int(NR*0.95)]}'

# Requests per second
tail -10000 /var/log/nginx/access.log | wc -l / 1000

# Top 404 errors
grep ' 404 ' /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head

# Requests by status code
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c

# Busiest hours
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1 | sort | uniq -c | sort -rn
```

### CloudWatch Integration

Send Nginx logs to AWS CloudWatch:

```bash
# Install CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Configure agent to collect Nginx logs
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null << EOF
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/aws/ec2/nginx/access",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%d/%b/%Y:%H:%M:%S %z"
          },
          {
            "file_path": "/var/log/nginx/error.log",
            "log_group_name": "/aws/ec2/nginx/error",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
EOF

# Start CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### Monitoring via Nginx Status Module

Enable the stub_status module for basic metrics:

```nginx
server {
    listen 127.0.0.1:8080;  # Only listen on localhost
    server_name _;
    
    location /nginx_status {
        stub_status on;
        access_log off;
    }
}

# Query metrics
curl http://127.0.0.1:8080/nginx_status

# Output:
# Active connections: 42
# server accepts handled requests
#  1234567 1234567 2468000
# Reading: 5 Writing: 10 Waiting: 27
#
# Active connections: Current open connections
# accepts: Total connections accepted since startup
# handled: Total connections handled (should equal accepts)
# requests: Total requests processed
# Reading: Connections currently reading request header
# Writing: Connections currently writing response
# Waiting: Idle connections waiting for request
```

---

## Production Deployment Patterns

### Zero-Downtime Reloads

Reload configuration without dropping connections:

```bash
# Verify new configuration
sudo nginx -t

# Graceful reload (sends SIGHUP to master process)
sudo systemctl reload nginx

# Monitor graceful reload in logs
tail -f /var/log/nginx/error.log | grep -i "graceful\|reopen\|signal"

# Verify with connection monitoring
watch -n 1 'ss -tn | grep ESTABLISHED | wc -l'
```

The reload process:
1. Nginx master reads new configuration file
2. Master spawns new worker processes with new config
3. Old workers finish existing connections gracefully
4. Old workers terminate when all connections close
5. No downtime for clients

### Rolling Application Updates

Update backend application while maintaining traffic:

```bash
#!/bin/bash
# rolling-update.sh - Update application with Nginx load balancing

APP_PORTS=(3000 3001 3002)
UPSTREAM_CONFIG="/etc/nginx/conf.d/upstream.conf"

for port in "${APP_PORTS[@]}"; do
    echo "Updating application on port $port..."
    
    # Remove from load balancer temporarily
    sed -i "/$port/d" "$UPSTREAM_CONFIG"
    sudo nginx -s reload
    
    # Wait for connections to drain (max 30 seconds)
    sleep 5
    
    # Stop old application
    kill $(lsof -ti :$port)
    sleep 2
    
    # Deploy new version
    cd /var/www/myapp
    git pull origin main
    npm install
    npm run build
    
    # Start new application
    pm2 start ecosystem.config.js --port $port
    
    # Wait for application to be ready
    sleep 5
    
    # Add back to load balancer
    echo "    server localhost:$port;" >> "$UPSTREAM_CONFIG"
    sudo nginx -s reload
    
    # Verify no errors
    curl -f http://localhost:$port/health || exit 1
done

echo "Rolling update complete"
```

### Health Check Pattern

Implement responsive health checks:

```bash
# In your application (example: Node.js Express)
app.get('/health', (req, res) => {
  const healthy = {
    status: 'UP',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    pid: process.pid
  };
  
  res.json(healthy);
});

# Nginx configuration
server {
    location /health {
        proxy_pass http://app_servers;
        proxy_connect_timeout 2s;
        proxy_read_timeout 2s;
        access_log off;
    }
}

# Manual health check from EC2 instance
curl -v http://localhost:3000/health
```

### Graceful Shutdown Pattern

Ensure connections drain before terminating:

```bash
#!/bin/bash
# graceful-shutdown.sh - Run before terminating EC2 instance

# Remove from AWS load balancer (connection draining)
aws elb deregister-instances-from-load-balancer \
  --load-balancer-name my-app-lb \
  --instances i-1234567890abcdef0

# Wait for connection draining (default 300 seconds)
sleep 30

# Reload Nginx to stop accepting new connections
# while allowing existing connections to complete
sudo systemctl reload nginx

# Wait for graceful shutdown
sleep 60

# Terminate application processes
pkill -TERM pm2
pkill -TERM node

# Wait 30 seconds for graceful termination
sleep 30

# Force kill if still running
pkill -9 node

echo "Graceful shutdown complete"
```

### Backup Configuration

Maintain configuration version history:

```bash
#!/bin/bash
# backup-nginx-config.sh - Run daily via cron

BACKUP_DIR="/var/backups/nginx-config"
mkdir -p "$BACKUP_DIR"

# Backup configuration with timestamp
tar czf "$BACKUP_DIR/nginx-config-$(date +%Y%m%d_%H%M%S).tar.gz" \
    /etc/nginx

# Keep only last 30 days of backups
find "$BACKUP_DIR" -type f -mtime +30 -delete

# Upload to S3 for disaster recovery
aws s3 cp "$BACKUP_DIR/" s3://my-backups/nginx-config/ --recursive
```

---

## Troubleshooting Guide

### Common Issues and Solutions

**Issue 1: "502 Bad Gateway" - Backend Unavailable**

Occurs when Nginx cannot connect to backend servers:

```bash
# Check backend servers are running
ps aux | grep node
ps aux | grep python

# Check if port is listening
netstat -tlnp | grep 3000
netstat -tlnp | grep 8000

# Test connection to backend
nc -zv localhost 3000
# or
curl -v http://localhost:3000

# Check Nginx error log
sudo tail -50 /var/log/nginx/error.log | grep upstream

# If backend is running but Nginx can't connect:
# - Check firewall/security groups
# - Check proxy_pass target is correct
# - Check upstream server list is correct
```

Configuration check:

```nginx
# Verify upstream block
upstream app_servers {
    server localhost:3000;  # Correct
    # server localhost:3000/;  # WRONG - don't add slash after port
}

server {
    location / {
        proxy_pass http://app_servers;  # Use http://
        # NOT: proxy_pass http://app_servers/  # Trailing slash changes URI
    }
}
```

**Issue 2: "413 Request Entity Too Large"**

Client request exceeds size limit:

```bash
# Error in nginx error.log
# "client intended to send too large body"

# Check current limit
grep client_max_body_size /etc/nginx/nginx.conf

# Fix by increasing limit
sudo nano /etc/nginx/nginx.conf
# Find: client_max_body_size 1m;
# Change to: client_max_body_size 100m;

# Reload
sudo systemctl reload nginx
```

**Issue 3: "Too Many Open Files" Error**

System running out of file descriptors:

```bash
# Check current limit
ulimit -n
# Output: 1024 (too low)

# Check Nginx worker limit
ps aux | grep nginx | grep worker | wc -l

# Calculate needed descriptors
# Each connection needs 1-2 file descriptors
# Formula: (worker_processes × worker_connections) × 2 + system_overhead

# Increase limits in /etc/security/limits.conf
sudo bash -c 'echo "nginx soft nofile 65535" >> /etc/security/limits.conf'
sudo bash -c 'echo "nginx hard nofile 65535" >> /etc/security/limits.conf'

# Apply to running processes
sudo systemctl restart nginx

# Verify
ps -ef | grep "[n]ginx" | head -1 | awk '{print $2}' | xargs -I {} cat /proc/{}/limits | grep "open files"
```

**Issue 4: SSL Certificate Issues**

Common SSL/TLS problems:

```bash
# Certificate not found error
# Solution: Verify certificate path exists
ls -la /etc/letsencrypt/live/example.com/
# Expected files: fullchain.pem, privkey.pem, cert.pem, chain.pem

# Certificate permissions issue
sudo chown -R root:root /etc/letsencrypt/
sudo chmod -R 755 /etc/letsencrypt/

# Check certificate expiration
openssl x509 -in /etc/letsencrypt/live/example.com/cert.pem -noout -dates

# Manually renew certificate
sudo certbot renew --cert-name example.com --force-renewal --quiet

# Test SSL configuration
sudo nginx -t

# Check for SSL errors in logs
grep -i "ssl\|certificate" /var/log/nginx/error.log
```

**Issue 5: Slow Response Times**

Diagnosis and remediation:

```bash
# Check backend response times in access log
awk '{print $NF}' /var/log/nginx/access.log | sort -n | tail -20

# Identify slowest requests
awk '$NF > 1 {print $(NF-6), $NF}' /var/log/nginx/access.log | sort -k2 -rn | head

# Check backend server resources
ssh ubuntu@10.0.1.50 'top -b -n 1 | head -10'
ssh ubuntu@10.0.1.50 'free -h'
ssh ubuntu@10.0.1.50 'df -h'

# Check Nginx worker status
watch -n 1 'ps aux | grep nginx'

# Enable cache to reduce backend load
# See Caching Strategies section

# Check for connection pooling
grep proxy_http_version /etc/nginx/nginx.conf
# Should be: proxy_http_version 1.1;
```

**Issue 6: Memory Leaks or High Memory Usage**

```bash
# Monitor Nginx memory usage
watch -n 5 'ps aux | grep "[n]ginx"'

# Check for cache taking excessive memory
du -sh /var/cache/nginx/

# Restart Nginx if memory continuously grows
sudo systemctl restart nginx

# Monitor immediately after restart
ps aux | grep "[n]ginx worker" | awk '{print $6}' | sort -n
```

### Debugging Configuration

Enable debug logging for troubleshooting:

```nginx
# Temporarily enable debug logging
error_log /var/log/nginx/error.log debug;

events {
    worker_connections 1024;
    debug_connection 192.168.1.100;  # Debug only requests from this IP
}

# Reload
sudo systemctl reload nginx

# Monitor debug logs (very verbose)
sudo tail -f /var/log/nginx/error.log | grep -i debug
```

### Testing Changes Safely

Always test before deploying changes:

```bash
# Create test configuration
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

# Make changes
sudo nano /etc/nginx/nginx.conf

# Test syntax
sudo nginx -t

# If syntax OK but reload fails:
# 1. Restore backup
sudo cp /etc/nginx/nginx.conf.backup /etc/nginx/nginx.conf

# 2. Try reload
sudo systemctl reload nginx

# 3. If reload succeeds, re-apply changes more carefully
sudo nano /etc/nginx/nginx.conf

# Compare configuration differences
diff /etc/nginx/nginx.conf.backup /etc/nginx/nginx.conf
```

---

## Cost Analysis

### Direct Costs

**1. EC2 Instance Running Nginx**

Nginx has minimal resource requirements:

```
Instance Type: t3.micro (eligible for free tier)
vCPU: 1
Memory: 1 GB
Estimated monthly cost: $8-$10

For production:
Instance Type: t3.medium
vCPU: 2
Memory: 4 GB
Estimated monthly cost: $35-$40
```

Nginx doesn't significantly increase costs compared to running just the application alone.

**2. Data Transfer Costs**

Nginx's compression reduces outbound data:

```
Without compression:
- Average response: 500 KB
- Requests per day: 100,000
- Daily transfer: 50 GB
- Monthly: 1,500 GB
- Monthly cost: $135 (at $0.09/GB outbound)

With Nginx compression (70% reduction):
- Compressed response: 150 KB
- Daily transfer: 15 GB
- Monthly: 450 GB
- Monthly cost: $40.50
- Savings: $94.50/month
```

**3. SSL/TLS Costs**

```
Let's Encrypt certificates: FREE
AWS Certificate Manager (ACM): FREE
Self-signed certificates: FREE
Commercial SSL certificates: $100-$500/year (not recommended)
```

### Indirect Cost Savings

**1. Reduced Backend Instance Costs**

By handling load balancing, caching, and compression at the Nginx layer:

```
Without Nginx load balancing:
- Need: 10 t3.medium instances ($35/month each)
- Monthly cost: $350

With Nginx + 5 t3.medium instances:
- Nginx instance: $40/month
- Backend instances: $175/month
- Total: $215/month
- Monthly savings: $135

Annual savings: $1,620
```

**2. Reduced Data Transfer via Caching**

Caching reduces backend query load and data transfer:

```
Example: E-commerce site with product pages

Without caching:
- DB hits per second: 1,000
- RDS instance needed: db.r5.xlarge ($2.094/hour)
- Monthly cost: ~$1,500

With Nginx caching (80% cache hit rate):
- DB hits per second: 200
- RDS instance needed: db.t3.small ($0.25/hour)
- Monthly cost: ~$180
- Monthly savings: $1,320

Annual savings: $15,840
```

**3. Reduced CloudFront Costs**

Nginx caching at origin reduces CloudFront requests:

```
Without origin caching:
- All requests to CloudFront: 1,000,000/month
- Requests to origin: 1,000,000
- CloudFront cost: $85/month
- Origin bandwidth: $90/month (1TB)
- Total: $175/month

With Nginx caching (90% cache hit rate):
- Requests to CloudFront: 1,000,000/month
- Requests to origin: 100,000 (90% served from CF)
- CloudFront cost: $85/month
- Origin bandwidth: $9/month (100GB)
- Nginx instance: $40/month
- Total: $134/month
- Monthly savings: $41

Annual savings: $492
```

### Cost Optimization Strategies

**1. Use Compression Aggressively**

Even with minimal CPU impact, compression reduces data transfer 60-80%:

```nginx
gzip on;
gzip_comp_level 6;
gzip_min_length 1000;
gzip_types text/plain text/css application/javascript application/json;
```

**2. Cache Selectively**

Cache high-traffic, low-change content:

```nginx
# Cache expensive queries
proxy_cache_valid 200 10m;

# Don't cache user-specific data
proxy_cache_bypass $http_authorization;
```

**3. Use Spot Instances for Nginx**

Nginx is stateless and can be replaced quickly:

```bash
# Request Spot Instance for 90% cost savings
aws ec2 request-spot-instances \
  --spot-price "0.005" \
  --instance-count 1 \
  --type "one-time" \
  --launch-specification '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.medium",
    "KeyName": "my-key"
  }'
```

**4. Monitor and Right-Size**

Nginx rarely needs large instances:

```bash
# Monitor actual CPU and memory usage
watch -n 1 'ps aux | grep nginx | grep worker'

# Most setups work fine on t3.small or t3.medium
# Vertical scaling (t3.large) provides minimal benefit
```

### Break-Even Analysis

Comparing different deployment architectures:

**Scenario: Small SaaS Application (100 concurrent users)**

```
Architecture 1: Application only on t3.medium
- Cost: $35/month
- Can handle: 50 concurrent users
- Need 2 instances: $70/month

Architecture 2: Nginx (t3.micro) + 1 App (t3.medium)
- Nginx: $8/month
- App: $35/month
- Total: $43/month
- Can handle: 100+ concurrent users
- Benefit: Load balancing, caching, SSL termination

Savings by adding Nginx:
- Can handle 2x load on nearly same cost
- Or reduce app instances from 2 to 1
- Annual savings: $108
```

**Scenario: Large Application (1000+ concurrent users)**

```
Architecture 1: AWS Application Load Balancer + 5 App Instances
- ALB: $25/month (fixed) + $0.0075/hour = $30/month
- App instances (5x t3.medium): $175/month
- Total: $205/month

Architecture 2: Nginx (t3.medium) + 3 App Instances
- Nginx: $40/month
- App instances (3x t3.medium): $105/month
- Total: $145/month
- Savings: $60/month ($720/year)

Trade-offs:
- ALB provides multi-AZ redundancy (Nginx is single instance)
- Nginx provides more control over routing
- ALB is simpler operationally
```

---

## Summary and Best Practices

### Configuration Checklist

Before deploying Nginx to production:

- [ ] Install latest Nginx version
- [ ] Configure SSL/TLS with Let's Encrypt or ACM
- [ ] Enable HTTP/2 for improved performance
- [ ] Configure security headers (CSP, X-Frame-Options, etc.)
- [ ] Set up rate limiting for sensitive endpoints
- [ ] Enable gzip compression
- [ ] Configure proper logging (separate by application)
- [ ] Set up log rotation
- [ ] Configure health check endpoint
- [ ] Test configuration syntax (`nginx -t`)
- [ ] Configure monitoring/alerting
- [ ] Set up access controls (security groups, IP whitelisting)
- [ ] Document upstream servers and routing logic
- [ ] Create backup of configuration
- [ ] Test zero-downtime reload capability
- [ ] Configure graceful shutdown pattern

### Performance Targets

Expect these performance characteristics on a t3.medium instance:

```
Requests per second: 1,000-5,000 (depends on backend)
Response time: 50-100ms P99 (mostly backend time)
Memory usage: 50-100MB for Nginx process
CPU usage: 10-20% under moderate load
Connection limit: 4,096 concurrent (worker_connections 1024 × 4 workers)
```

### When to Use Nginx vs AWS ALB

**Use Nginx when:**
- Cost optimization is critical
- You need fine-grained routing control
- You want to run on a single instance
- You need additional functionality (caching, compression)
- You have stateless applications

**Use AWS ALB when:**
- You need multi-AZ redundancy
- You have hundreds of thousands of requests/second
- You want managed service with AWS integration
- You need advanced routing (hostname, path, query string)
- Operations simplicity is priority

### Key Takeaways

1. **Nginx is incredibly efficient**: Serves thousands of concurrent connections with minimal resources
2. **Configuration is powerful**: Routing, caching, security can all be controlled via simple config files
3. **Zero-downtime reloads**: Configuration can be updated without dropping connections
4. **SSL/TLS termination saves money**: Offloads crypto operations from application servers
5. **Caching reduces infrastructure costs**: 80% cache hit rate can cut backend infrastructure by 80%
6. **Rate limiting protects your application**: Easy to implement, prevents abuse
7. **Monitoring is essential**: Track response times, cache hit rates, and backend health
8. **Simple is better**: Don't add features you don't need; focus on core web server functionality
