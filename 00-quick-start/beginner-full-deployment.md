# Beginner's Complete AWS Deployment Guide: From Zero to Production

**Target Audience:** Complete beginners, non-IT professionals, first-time AWS users  
**Time Required:** 3-4 hours first deployment  
**Difficulty Level:** Beginner (no prior cloud/DevOps experience needed)  
**Goal:** Deploy a production-ready Node.js application with HTTPS on a custom domain

---

## 📚 What You Will Accomplish

By following this guide **exactly as written**, you will have:

✅ Created and secured an AWS account with best practices  
✅ Set up IAM users with minimum required permissions  
✅ Created a private VPC for network isolation  
✅ Launched an EC2 instance (t3.micro or t3.small)  
✅ Connected securely via SSH with key pairs  
✅ Deployed a Node.js application using PM2  
✅ Configured Nginx as a reverse proxy (port 80/443)  
✅ Set up a custom domain with Route53  
✅ Enabled HTTPS using Let's Encrypt (free SSL)  
✅ Validated end-to-end functionality  
✅ Understood troubleshooting procedures  

**After this guide, your setup will look like:**

```
Internet Users
    ↓ (HTTPS on custom domain)
Route53 (DNS)
    ↓ (resolves example.com → EC2 IP)
Nginx on Port 443
    ↓ (SSL/TLS termination)
Node.js Application on Port 3000
    ↓ (PM2 manages process)
Your Business Logic
```

---

## ⚠️ Safety First: Critical Before You Start

**DO NOT skip this section.** These steps prevent permanent damage:

### 1. Root User Security (IMMEDIATE)

You will NEVER use root user after initial setup. Root account has unrestricted access and cannot be easily audited.

**Why this matters:** If someone steals root credentials, they can delete everything, rack up $100k bills, or steal your data.

### 2. Create Separate IAM User for Daily Work

AWS best practice: Root user = emergency only. Daily work = limited IAM user.

### 3. Enable Billing Alerts

AWS sends alerts if you exceed predicted costs. Prevents bill shock.

### 4. Backup SSH Keys

Losing SSH keys = locked out of server forever. Cannot be recovered. Store backups securely.

### 5. No Credentials in Code

Never put AWS credentials (access keys, passwords) in GitHub or email. Use IAM roles instead.

---

## 🎓 Beginner Concepts (Read This First)

Before diving into steps, understand what these terms mean:

### What is AWS?

AWS (Amazon Web Services) = **cloud rental service**. Instead of buying physical servers, you rent computers by the hour from AWS data centers worldwide.

### Key Components You'll Use

| Component | What It Is | Why You Need It |
|-----------|-----------|-----------------|
| **Account** | Your AWS login + billing | Container for everything you create |
| **IAM User** | Limited-permission login | Safer than root user for daily work |
| **VPC** | Virtual Private Network | Isolates your infrastructure |
| **EC2** | Rented computer (server) | Runs your application |
| **Elastic IP** | Static IP address | Your server's permanent address |
| **Security Group** | Firewall rules | Controls who can access your server |
| **Route53** | DNS service | Maps example.com → server IP |
| **SSH** | Secure shell | Encrypted tunnel to your server |
| **SSL/HTTPS** | Encryption certificate | Makes connections secure (lock icon) |
| **PM2** | Process manager | Keeps app running 24/7 |
| **Nginx** | Web server/reverse proxy | Handles HTTP/HTTPS traffic |

### Simple Flow: How Users Access Your App

```
1. User types: https://example.com
2. Browser asks Route53: "What IP is example.com?"
3. Route53 responds: "52.123.456.789" (your server)
4. Browser connects to that IP on port 443 (HTTPS)
5. Nginx receives request on port 443
6. Nginx decrypts HTTPS (SSL/TLS)
7. Nginx sends to Node.js on port 3000 (localhost)
8. Node.js processes request
9. Response flows back: Node.js → Nginx → Browser
10. User sees your app
```

---

## 📋 Pre-Deployment Checklist

Before starting, gather these items:

- [ ] AWS account created (with credit card)
- [ ] Terminal/Command Prompt access on your laptop
- [ ] A domain name (or purchase one at Route53 for $10-15/year)
- [ ] Git installed on laptop (`git --version` to check)
- [ ] Node.js installed on laptop (`node --version` to check)
- [ ] SSH client available (built-in on Mac/Linux; PuTTY on Windows)
- [ ] A text editor or IDE
- [ ] 3-4 hours uninterrupted time

---

## 🚀 Step 1: Secure AWS Account

### 1.1: Create AWS Account

1. Go to: https://aws.amazon.com/
2. Click "Create an AWS Account"
3. Enter email, password, AWS account name
4. Enter billing information (required; free tier covers costs for learning)
5. Verify phone number (AWS will call you)
6. Choose support plan: **Basic (free)**
7. Account creation complete

**You now have:** Root user credentials (never use for daily work)

### 1.2: Enable MFA on Root User (5 minutes)

**Why:** Prevents account compromise even if password is stolen.

**Steps:**

1. Log in to AWS Console: https://console.aws.amazon.com
2. Click your account name (top right) → "My Security Credentials"
3. Click "MFA" → "Assign MFA Device"
4. Choose "Virtual MFA device" (use authenticator app on phone)
5. Scan QR code with authenticator (Google Authenticator, Microsoft Authenticator, or Authy)
6. Enter 6-digit code from app
7. Complete setup

**After this:** Every root user login requires 6-digit code from phone.

### 1.3: Enable Billing Alerts (10 minutes)

Prevent surprise $1000+ bills.

**Steps:**

1. Log in to AWS Console
2. Search for "Billing" → go to Billing Dashboard
3. Click "Billing Preferences"
4. Check: "Receive Billing Alerts"
5. Save
6. Go to CloudWatch (search for it)
7. Alarms → Create Alarm
8. Select metric: "EstimatedCharges"
9. Set threshold: $50 (adjust to your comfort level)
10. Create SNS topic: "BillingAlerts"
11. Enter your email
12. Confirm email subscription

**After this:** You'll get email alerts if charges exceed $50.

---

## 🔐 Step 2: Create IAM User (Daily Driver)

### Why Not Use Root User?

- Root user cannot be monitored or limited
- Credentials cannot be rotated
- If compromised, attacker has full access
- Best practice: root user = emergency access only

### 2.1: Create IAM User

**Steps:**

1. Log in to AWS Console (as root)
2. Search for "IAM"
3. Click "Users" (left menu)
4. Click "Create User"
5. Username: `devops-user` (or your name)
6. Click "Create User"

### 2.2: Attach Administrator Policy (for now)

⚠️ **Temporary:** We'll restrict permissions later.

**Steps:**

1. Click the user you just created
2. Click "Add permissions"
3. Click "Attach policies directly"
4. Search for: `AdministratorAccess`
5. Check the checkbox
6. Click "Next" → "Add permissions"

**After this:** User has admin permissions (temporary).

### 2.3: Create Access Keys

IAM users need "access keys" to use AWS CLI.

**Steps:**

1. Click the user again
2. Click "Security credentials" tab
3. Under "Access keys": Click "Create access key"
4. Select "Command Line Interface (CLI)"
5. Check "I understand..." checkbox
6. Click "Create access key"
7. **SAVE these immediately:**
   - Access Key ID (looks like: AKIA...)
   - Secret Access Key (looks like: wJalrXUt...)
8. **Store in password manager or secure location**
9. Click "Done"

⚠️ **CRITICAL:** You will never see the Secret Access Key again. If you lose it, delete and create a new one.

### 2.4: Create Login Password

Allow this user to log in to AWS Console.

**Steps:**

1. User page → "Security credentials" tab
2. Under "Console password": Click "Set user password"
3. Choose "I want to create a Console password"
4. Set password (strong, 15+ characters, mix of upper/lower/numbers/symbols)
5. **SAVE password** in password manager
6. Click "Set password"

**After this:** User can log in to AWS Console

### 2.5: Enable MFA for IAM User

Same as root user but for this user.

**Steps:**

1. User page → "Security credentials"
2. Click "Assign MFA device"
3. Virtual MFA device → Scan QR code → Enter 6-digit code
4. Complete

**After this:** Every login requires phone confirmation.

### 2.6: Create Login Alias (Optional but Recommended)

Instead of logging in with 12-digit account ID, use a friendly URL.

**Steps:**

1. IAM Dashboard
2. "AWS Account Alias" (right side)
3. Click "Create"
4. Enter: `yourcompany-aws` (must be globally unique)
5. Create alias

**After this:** You can log in at: `https://yourcompany-aws.signin.aws.amazon.com/console`

**Bookmark this URL.**

---

## 🌐 Step 3: Create VPC (Virtual Private Network)

**What is a VPC?** Think of it as a private office building where only you can go. EC2 instances live inside a VPC.

### 3.1: Create VPC

**Steps:**

1. Log in as IAM user
2. Search for "VPC"
3. Click "Create VPC"
4. Name: `myapp-vpc`
5. IPv4 CIDR: `10.0.0.0/16` (gives you 65,536 IP addresses)
6. Click "Create VPC"

**What happened:** Created a network that can hold your server.

### 3.2: Create Subnet (Public)

A VPC needs subnets (subdivisions). We'll create one public subnet where the EC2 instance lives.

**Steps:**

1. Left menu → "Subnets"
2. Click "Create subnet"
3. Select VPC: `myapp-vpc`
4. Subnet name: `public-subnet-1a`
5. Availability Zone: `us-east-1a` (or your region's first AZ)
6. IPv4 CIDR: `10.0.1.0/24` (256 IPs)
7. Click "Create subnet"

**What happened:** Created a subdivision with 256 IP addresses.

### 3.3: Create Internet Gateway

An Internet Gateway allows traffic from the internet to reach your VPC.

**Steps:**

1. Left menu → "Internet Gateways"
2. Click "Create internet gateway"
3. Name: `myapp-igw`
4. Click "Create internet gateway"
5. Click "Attach to VPC"
6. Select: `myapp-vpc`
7. Click "Attach internet gateway"

**What happened:** Connected your VPC to the internet.

### 3.4: Create Route Table

A route table tells traffic how to flow.

**Steps:**

1. Left menu → "Route Tables"
2. Click "Create route table"
3. Name: `public-routes`
4. Select VPC: `myapp-vpc`
5. Click "Create route table"
6. Select the route table you just created
7. Click "Edit routes"
8. Click "Add route"
   - Destination: `0.0.0.0/0` (all internet traffic)
   - Target: Select Internet Gateway → `myapp-igw`
9. Click "Save routes"
10. Go to "Subnet associations"
11. Click "Edit subnet associations"
12. Select: `public-subnet-1a`
13. Click "Save associations"

**What happened:** Set up traffic rules so requests from internet reach your subnet.

---

## 💻 Step 4: Create EC2 Instance

### 4.1: Generate SSH Key Pair (CRITICAL)

SSH keys are how you securely log into your server. **Losing keys = locked out forever.**

**Steps:**

1. Search for "EC2"
2. Left menu → "Key Pairs"
3. Click "Create key pair"
4. Name: `myapp-key`
5. Key pair type: `RSA`
6. Private key format: `.pem` (for Mac/Linux) or `.ppk` (for Windows PuTTY)
7. Click "Create key pair"
8. **Browser downloads: `myapp-key.pem`**
9. **BACKUP THIS FILE IMMEDIATELY:**
   - Save to password manager
   - Print a copy (locked in drawer)
   - Save to encrypted USB drive
   - DO NOT save to public cloud storage

⚠️ **If you lose this file, you cannot access the server. No recovery possible.**

### 4.2: Create Security Group (Firewall)

Security groups are firewalls controlling which traffic reaches your server.

**Steps:**

1. EC2 Dashboard
2. Left menu → "Security Groups"
3. Click "Create security group"
4. Name: `myapp-sg`
5. Description: `Firewall for myapp EC2 instance`
6. Select VPC: `myapp-vpc`
7. Under "Inbound rules": Click "Add rule"
   - Type: SSH
   - Protocol: TCP
   - Port: 22
   - Source: 0.0.0.0/0 (or your specific IP for security)
8. Click "Add rule" again
   - Type: HTTP
   - Protocol: TCP
   - Port: 80
   - Source: 0.0.0.0/0
9. Click "Add rule" again
   - Type: HTTPS
   - Protocol: TCP
   - Port: 443
   - Source: 0.0.0.0/0
10. Click "Create security group"

**What these do:**
- SSH (port 22): Allows you to log in
- HTTP (port 80): Allows web traffic
- HTTPS (port 443): Allows secure web traffic

### 4.3: Create EC2 Instance

**Steps:**

1. EC2 Dashboard
2. Click "Launch Instance"
3. Name: `myapp-server`
4. OS Image: `Amazon Linux 2` (free tier eligible, simple, similar to RHEL)
5. Instance type: `t3.micro` (free tier) or `t3.small` (if expecting traffic)
6. Key pair: Select `myapp-key`
7. Network settings:
   - VPC: `myapp-vpc`
   - Subnet: `public-subnet-1a`
   - Auto-assign Elastic IP: **YES** (fixed IP address)
   - Security group: Select `myapp-sg`
8. Storage: 
   - 20 GB (free tier allows 30GB)
   - Encrypted: YES
9. Advanced details:
   - IAM instance profile: None (for now)
10. Click "Launch instance"

**Status:** Instance is launching (takes 1-2 minutes)

### 4.4: Allocate Elastic IP (Static IP Address)

An Elastic IP ensures your server's IP never changes (important for DNS).

**Steps:**

1. EC2 Dashboard
2. Left menu → "Elastic IPs"
3. Click "Allocate Elastic IP address"
4. Click "Allocate"
5. Select the Elastic IP you created
6. Click "Associate Elastic IP address"
7. Instance: Select `myapp-server`
8. Network interface: (auto-selected)
9. Click "Associate"

**What happened:** Your server now has a permanent IP address (like 52.123.456.789).

**Write down this IP address.** You'll need it for DNS later.

---

## 🔑 Step 5: Connect to Server via SSH

### 5.1: Find Your Server's IP

**Steps:**

1. EC2 Dashboard
2. Click "Instances"
3. Select `myapp-server`
4. Copy "Public IPv4 address" (should match your Elastic IP)

### 5.2: Set SSH Key Permissions (Important)

SSH keys must have restricted permissions or AWS will refuse them.

**On Mac/Linux:**

```bash
# Navigate to where myapp-key.pem is stored
cd ~/Downloads  # or wherever you saved it

# Restrict permissions (SSH requirement)
chmod 400 myapp-key.pem

# Verify (should show: -r---------)
ls -la myapp-key.pem
```

**On Windows (PuTTY):**

1. Download PuTTYgen
2. Open PuTTYgen
3. File → Load Private Key → select `myapp-key.ppk`
4. (no extra steps needed if already in .ppk format)

### 5.3: SSH into Server

**On Mac/Linux:**

```bash
ssh -i myapp-key.pem ec2-user@52.123.456.789
# Replace 52.123.456.789 with your actual Elastic IP

# First time: Press "yes" to accept server fingerprint
# Expected prompt: "Are you sure you want to continue connecting?"
```

**On Windows (PuTTY):**

1. Open PuTTY
2. Host Name: `52.123.456.789` (your Elastic IP)
3. Under "SSH" → "Auth": Select your `.ppk` file
4. Click "Open"
5. When prompted for username: type `ec2-user`

**Success if you see:**

```
[ec2-user@myapp-server ~]$
```

**Congratulations! You're inside your server.**

---

## 📦 Step 6: Install Dependencies

### 6.1: Update System

```bash
sudo yum update -y
# This takes 2-3 minutes
# "sudo" = run as administrator
# "yum update -y" = update all packages, don't ask for confirmation
```

### 6.2: Install Node.js (v18 LTS)

```bash
# Install Node.js 18
curl -sL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install nodejs -y

# Verify installation
node --version
npm --version
# Should show v18.x.x and v9.x.x
```

### 6.3: Install Nginx

```bash
sudo yum install nginx -y

# Start Nginx
sudo systemctl start nginx

# Enable Nginx to start on reboot
sudo systemctl enable nginx

# Verify status
sudo systemctl status nginx
# Should show: active (running)
```

### 6.4: Install PM2 (Node Process Manager)

```bash
# Install PM2 globally
sudo npm install -g pm2

# Verify
pm2 --version
# Should show version like 5.x.x
```

### 6.5: Install Git

```bash
sudo yum install git -y

# Verify
git --version
```

---

## 🚀 Step 7: Deploy Your Application

### 7.1: Clone Your Repository

```bash
# Create app directory
mkdir -p ~/myapp
cd ~/myapp

# Clone your application (replace with your repo URL)
git clone https://github.com/yourusername/your-app-repo.git .

# If private repo, you'll need to set up GitHub SSH keys
# For now, assume it's public or you've configured Git
```

### 7.2: Install Dependencies

```bash
cd ~/myapp

# Install Node dependencies
npm install

# This creates node_modules/ directory (~100-500 MB depending on app)
```

### 7.3: Build Application (if needed)

If your app has a build step:

```bash
npm run build
# Creates dist/ or build/ directory
```

### 7.4: Test Locally

```bash
# Start app on port 3000
node index.js
# or
npm start

# You should see: "Server running on port 3000"

# Stop it (Ctrl+C)
```

### 7.5: Start with PM2

```bash
# Start app with PM2
pm2 start npm --name "myapp" -- start

# Or if your start command is different:
# pm2 start "node index.js" --name "myapp"

# View running apps
pm2 list
# Should show: myapp (one row, status "online")

# Save PM2 config to auto-restart on reboot
pm2 startup
# Follow instructions on screen

pm2 save
```

**What PM2 does:**
- Keeps app running even if it crashes
- Auto-restarts app on server reboot
- Provides monitoring and logging

### 7.6: Verify App is Running

```bash
# Check if port 3000 is listening
curl http://localhost:3000

# Or use lsof to see listening ports
sudo lsof -i -P -n | grep LISTEN
# Should show node on port 3000
```

---

## 🌐 Step 8: Configure Nginx (Reverse Proxy)

Nginx acts as a "traffic controller" that accepts internet requests and forwards them to Node.js.

### 8.1: Create Nginx Configuration

```bash
# Create config file
sudo nano /etc/nginx/conf.d/myapp.conf
```

**Paste this configuration:**

```nginx
upstream myapp_backend {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    server_name _;  # Will change to your domain later

    # Log file locations
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    # Proxy to Node.js
    location / {
        proxy_pass http://myapp_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Save:** Ctrl+X → Y → Enter

### 8.2: Test Nginx Configuration

```bash
sudo nginx -t
# Should show: "syntax is ok" and "test is successful"
```

### 8.3: Reload Nginx

```bash
sudo systemctl reload nginx

# Verify
sudo systemctl status nginx
```

### 8.4: Test Connection

From your laptop:

```bash
# Replace with your Elastic IP
curl http://52.123.456.789

# Should receive your app's response (HTML, JSON, etc.)
```

Or open browser: `http://52.123.456.789`

---

## 🌍 Step 9: Set Up Domain & DNS

### 9.1: Register or Transfer Domain

**Option A: Buy from Route53**

1. Route53 Dashboard
2. Click "Registered domains"
3. Click "Register domains"
4. Search for domain (e.g., `myapp.com`)
5. Check availability
6. Add to cart
7. Continue through checkout
8. Takes 15 minutes to 24 hours

**Option B: Domain from another registrar**

If you already have a domain from GoDaddy, Namecheap, etc., you can transfer it to Route53 or manage DNS separately.

### 9.2: Create Route53 Hosted Zone

1. Route53 Dashboard
2. Click "Hosted zones"
3. Click "Create hosted zone"
4. Domain name: `myapp.com`
5. Type: Public
6. Click "Create hosted zone"

**You now have nameservers** (looks like):
```
ns-123.awsdns-45.com
ns-456.awsdns-78.net
ns-789.awsdns-01.co.uk
ns-234.awsdns-56.org
```

If using external registrar, update nameservers there to point to these.

### 9.3: Create Route53 A Record

This maps your domain to your server's Elastic IP.

**Steps:**

1. Inside hosted zone: `myapp.com`
2. Click "Create record"
3. Record name: Leave blank (or type `www` for `www.myapp.com`)
4. Record type: `A`
5. Value: `52.123.456.789` (your Elastic IP)
6. TTL: 300 (seconds)
7. Click "Create records"

**What happens:**
- When users visit `myapp.com`, DNS returns your Elastic IP
- Their browser connects to `52.123.456.789`
- Nginx receives the request
- Your app responds

### 9.4: Test DNS Resolution

Wait 1-5 minutes for DNS to propagate, then:

```bash
# From your laptop, test DNS
nslookup myapp.com
# Should return: 52.123.456.789

# Or
dig myapp.com
# Should show your Elastic IP in answer section
```

### 9.5: Test from Browser

Open browser: `http://myapp.com`

**Should work!** (but not secure yet - no HTTPS)

---

## 🔒 Step 10: Enable HTTPS with Let's Encrypt

### 10.1: Install Certbot

Certbot automatically creates and renews SSL certificates.

```bash
sudo yum install certbot python3-certbot-nginx -y
```

### 10.2: Generate Certificate

```bash
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Prompts:
# 1. Email: (enter your email)
# 2. Agree to terms: (y)
# 3. Marketing emails: (n or y)
# 4. Share IP: (n)
```

**What certbot does:**
- Creates SSL certificate for your domain
- Automatically updates Nginx configuration
- Sets up HTTPS (port 443)
- Redirects HTTP → HTTPS

### 10.3: Verify Certificate

```bash
sudo certbot certificates
# Shows certificate details, expiration date
```

### 10.4: Test HTTPS

Open browser: `https://myapp.com`

**Should show:**
- Green lock icon ✅
- "Connection is secure"
- Your app content

### 10.5: Certificate Auto-Renewal

Certbot automatically renews certificates before expiration.

**Verify:**

```bash
sudo systemctl list-timers | grep certbot

# Should show certbot.timer running
```

**Manual renewal (if needed):**

```bash
sudo certbot renew --dry-run
# --dry-run = test without actually renewing

sudo certbot renew
# Actually renew
```

---

## ✅ Step 11: Validation Checklist

Go through these checks to ensure everything works:

### 11.1: DNS Resolution

```bash
# Should return your Elastic IP
nslookup myapp.com
dig myapp.com

# Check both A record and any aliases
```

### 11.2: HTTP Redirect

Open browser: `http://myapp.com`

**Should automatically redirect to `https://myapp.com`**

### 11.3: HTTPS Connection

Open browser: `https://myapp.com`

**Check:**
- [ ] Green lock icon visible
- [ ] "Connection is secure"
- [ ] Certificate shows correct domain
- [ ] No SSL errors

To verify certificate details:
1. Click lock icon
2. Click "Connection is secure"
3. Click "Certificate is valid"
4. Verify issuer: Let's Encrypt
5. Verify domain matches

### 11.4: Application Response

```bash
# Test API endpoint
curl https://myapp.com/api/health
# Should return your app's response

# Or open in browser and check app functions
```

### 11.5: PM2 Status

From server:

```bash
pm2 list
# Should show: myapp → online → active
```

### 11.6: Nginx Status

```bash
sudo systemctl status nginx
# Should show: active (running)
```

### 11.7: Security Group Rules

In AWS Console:

1. EC2 → Security Groups → `myapp-sg`
2. Inbound rules should show:
   - SSH (22) from your IP or 0.0.0.0/0
   - HTTP (80) from 0.0.0.0/0
   - HTTPS (443) from 0.0.0.0/0

### 11.8: Elastic IP Persistent

Check that Elastic IP hasn't changed:

```bash
# From server
curl http://169.254.169.254/latest/meta-data/public-ipv4
# Should match your Elastic IP from AWS Console
```

### 11.9: SSL Certificate Expiration

```bash
openssl s_client -connect myapp.com:443 -showcerts | grep -i "not after"

# Or
sudo certbot certificates
# Shows expiration date (~90 days from creation)
```

### 11.10: Overall System Health

```bash
# SSH into server and run:
free -h
# Check memory usage

df -h
# Check disk usage

top -b -n 1 | head -20
# Check CPU and processes

ps aux | grep node
# Verify Node.js is running
```

**All checks passing?** Congratulations! 🎉

---

## 🔧 Troubleshooting Guide

### Issue: "Connection refused" when accessing domain

**Causes:**
1. Domain hasn't propagated yet (wait 1-5 minutes)
2. Security group missing HTTP/HTTPS rules
3. Nginx not running
4. Node.js app not running

**Solutions:**

```bash
# Check if Nginx running
sudo systemctl status nginx

# Check if Node.js running
pm2 list

# Check if port 3000 listening
curl http://localhost:3000

# Check Security Group rules in AWS Console
# (verify port 80, 443 allow from 0.0.0.0/0)

# Check Nginx configuration
sudo nginx -t
```

### Issue: SSL Certificate Error

**Causes:**
1. Certificate not yet created
2. Domain name doesn't match certificate
3. Certbot failed to validate domain

**Solutions:**

```bash
# Check certificate status
sudo certbot certificates

# If not present, run certbot again
sudo certbot --nginx -d myapp.com

# Check Nginx config includes SSL
grep -i "ssl" /etc/nginx/conf.d/myapp.conf

# Reload Nginx
sudo systemctl reload nginx
```

### Issue: App not responding on 3000

**Causes:**
1. Node.js crashed
2. npm start command failed
3. Port 3000 already in use

**Solutions:**

```bash
# Check PM2 status
pm2 list

# View recent logs
pm2 logs myapp

# Restart PM2
pm2 restart myapp

# Check if port in use
sudo lsof -i :3000

# Kill process on 3000 if needed
sudo kill -9 <PID>

# Restart app
pm2 start npm --name "myapp" -- start
```

### Issue: Can't SSH into Server

**Causes:**
1. SSH key permissions wrong
2. Wrong key file
3. Wrong Elastic IP
4. Security group missing SSH rule

**Solutions:**

```bash
# On laptop:
# 1. Check key permissions
ls -la myapp-key.pem
# Should show: -r-------- (chmod 400)

# 2. Fix if needed
chmod 400 myapp-key.pem

# 3. Verify Elastic IP
# (check AWS Console EC2 dashboard)

# 4. Verify SSH key is correct
ssh-keygen -y -f myapp-key.pem
# (should output public key, not error)

# 5. Try again with verbose output
ssh -vvv -i myapp-key.pem ec2-user@<your-elastic-ip>
# (shows detailed connection info)
```

### Issue: High Disk Usage or Memory Usage

```bash
# Check what's using disk space
du -sh /home/* /var/* /opt/*
# Identify largest directories

# Check memory usage per process
ps aux --sort=-%mem | head

# Clean up unused packages
sudo yum autoremove -y

# View PM2 memory usage
pm2 monit
# (real-time monitoring)
```

### Issue: Application Crashes Frequently

```bash
# View PM2 error logs
pm2 logs myapp --err

# View system logs
tail -50 /var/log/messages

# Check disk space (out of space = crashes)
df -h

# Increase PM2 restart limit
pm2 set PM2_MAX_INSTANCES 5
pm2 restart myapp

# Monitor in real-time
pm2 monit
```

---

## 📈 Next Steps & Scaling

After this basic deployment works, consider:

### Phase 2: Hardening

1. **Database:** Add RDS PostgreSQL (not storing data on server)
2. **Backups:** Enable automated EC2 snapshots
3. **Monitoring:** Set up CloudWatch alarms (CPU, disk, memory)
4. **Security:** Restrict SSH to specific IP, add WAF

### Phase 3: High Availability

1. **Multi-AZ:** Launch replica instance in different AZ
2. **Load Balancer:** Add ALB to distribute traffic
3. **Auto Scaling:** Create ASG to scale automatically
4. **Database Failover:** Enable RDS Multi-AZ

### Phase 4: CI/CD

1. **GitHub Actions:** Automate deployments
2. **Docker:** Containerize application
3. **Artifact Repository:** Store Docker images
4. **Rolling Deployments:** Zero-downtime updates

### Phase 5: Advanced Monitoring

1. **CloudWatch Dashboards:** Visualize metrics
2. **Application Logging:** ELK Stack or CloudWatch Logs
3. **Distributed Tracing:** X-Ray for debugging
4. **Cost Analysis:** Identify optimization opportunities

---

## 📚 Additional Resources

After completing this guide:

1. Read: `01-foundations/aws-fundamentals.md` (understand regions, pricing)
2. Read: `02-networking/vpc-complete.md` (deeper VPC knowledge)
3. Read: `03-compute/ec2-instances.md` (instance types, scaling)
4. Read: `05-databases/rds-postgres.md` (add production database)
5. Read: `06-deployment/nginx-webserver.md` (advanced Nginx config)
6. Read: `08-ci-cd/github-actions.md` (automate deployments)

---

## ⚠️ Common Mistakes to Avoid

1. **Losing SSH Key** → Server locked forever
   - **Fix:** Backup immediately to password manager + printed copy

2. **Using Root User Daily** → Security risk
   - **Fix:** Always use IAM user with limited permissions

3. **Public Database** → Data theft risk
   - **Fix:** Keep database private, only EC2 can access

4. **Hardcoding Secrets** → Credential leak on GitHub
   - **Fix:** Use AWS Secrets Manager or environment variables

5. **No Backups** → Data loss if server dies
   - **Fix:** Enable automated snapshots, use RDS for databases

6. **Ignoring SSL Warnings** → User distrust
   - **Fix:** Always enable HTTPS (Let's Encrypt is free)

7. **Single Server Only** → Single point of failure
   - **Fix:** After scaling, use ASG + ALB for redundancy

8. **Not Monitoring Costs** → Bill shock
   - **Fix:** Set CloudWatch alarms, check billing weekly

---

## 🎓 Congratulations!

You've successfully:

✅ Created a production-ready AWS setup  
✅ Deployed a Node.js application  
✅ Configured Nginx and SSL  
✅ Set up a custom domain with HTTPS  
✅ Understood the complete stack  

**Your application is now:**
- Accessible worldwide on your domain
- Encrypted with HTTPS
- Managed by PM2 (auto-restart)
- Behind Nginx (handles traffic efficiently)
- Running on an Elastic IP (permanent address)
- In a private VPC (isolated network)
- Monitored for uptime

This is a production-grade setup suitable for small-to-medium workloads. As your application grows, refer to the advanced guides for scaling strategies, high availability, and cost optimization.

**Questions or issues?** Refer to the troubleshooting section or consult the specialized guides in the knowledge base.
