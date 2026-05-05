# Enterprise AWS Infrastructure & Deployment Playbook

---

How to use this playbook:

- If you are new, start with **Beginner Mode: Foolproof A to Z** below.
- If you are experienced, use the Advanced Reference sections (0-15).
- Every step includes a Validation or Expected Result. If you cannot confirm it, stop and resolve before moving on.

## Table of Contents

| #    | Section                                                                                   | What You Will Learn                                                     |
| --- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Start | [Beginner Mode: Foolproof A to Z](#beginner-mode-foolproof-a-to-z)                       | Step-by-step safe deployment without breaking existing systems          |
| 0.0 | [AWS Console Orientation](#00-aws-console-orientation)                                    | Console layout, regions, account context                                |
| 0.1 | [Creating an AWS Account](#01-creating-an-aws-account-first-time-setup)                   | Root account setup and safety                                           |
| 0.2 | [Installing the AWS CLI](#02-installing-the-aws-cli)                                      | CLI installation on Windows, macOS, and Linux                           |
| 0.3 | [Configuring AWS CLI Credentials](#03-configuring-aws-cli-credentials)                    | Access keys, profiles, validation                                       |
| 0.4 | [Creating an SSH Key Pair](#04-creating-an-ssh-key-pair)                                  | SSH keys, permissions, key safety                                       |
| 0.5 | [Do Not Touch Existing Resources](#05-do-not-touch-existing-resources)                    | Safe-change protocol for shared accounts                                |
| 0.6 | [Pre-Flight Audit](#06-pre-flight-audit)                                                  | Baseline inventory before creating anything                             |
| 0.7 | [Quick Deployment Runbook (A → Z)](#07-quick-deployment-runbook-a--z)                     | End-to-end deployment flow                                              |
| 0.8 | [New Project Isolation Protocol](#08-new-project-isolation-protocol)                      | VPC/SG isolation, naming, port strategy                                 |
| 0.9 | [Billing Protection](#09-billing-protection)                                              | Budgets, Free Tier, anomaly alerts                                      |
| 0.10 | [New Developer Onboarding](#010-new-developer-onboarding)                                | Join an existing account safely                                         |
| 0.11 | [Installing Essential Tools](#011-installing-essential-tools)                            | jq, git, curl                                                           |
| 0.12 | [Pro Tips for Beginners](#012-pro-tips-for-beginners)                                    | Early guardrails and habits                                             |
| 1   | [Introduction and Architectural Philosophy](#1-introduction-and-architectural-philosophy) | Why this playbook exists, who should use it, scope and philosophy       |
| 2   | [Scenario-Based Entry & Decision Logic](#2-scenario-based-entry--decision-logic)          | How to classify your engagement: Greenfield, Brownfield, or Black Box   |
| 3   | [AWS Account & Security (IAM)](#3-aws-account--security-iam)                              | Root account lockdown, IAM Users/Roles/Policies, MFA, secrets           |
| 4   | [AWS Networking (VPC, Subnets, Routing)](#4-aws-networking-vpc-subnets-routing)           | VPC creation, public/private subnets, IGW, NAT Gateway, routing         |
| 5   | [EC2 Setup & Compute Strategy](#5-ec2-setup--compute-strategy)                            | Instance types, AMI selection, SSH, Node.js, Nginx setup                |
| 6   | [S3 Storage & Bucket Management](#6-s3-storage--bucket-management)                        | Bucket creation, permissions, signed URLs, static hosting               |
| 7   | [Database Strategy (RDS & Alternatives)](#7-database-strategy-rds--alternatives)          | RDS provisioning, backups, read replicas, scaling                       |
| 8   | [Project Deployment (Backend + Frontend)](#8-project-deployment-backend--frontend)        | Node.js/PM2, React/Nginx/S3, secrets, deployment runbook                |
| 9   | [Scaling Strategies (Vertical & Horizontal)](#9-scaling-strategies-vertical--horizontal)  | Auto Scaling Groups, ALB, target tracking policies                      |
| 10  | [CI/CD Pipeline](#10-cicd-pipeline)                                                       | GitHub Actions, Build→Test→Deploy, Rolling & Blue-Green deployment      |
| 11  | [Monitoring & Logging](#11-monitoring--logging)                                           | CloudWatch metrics, logs, alarms, dashboards, structured logging        |
| 12  | [Advanced Security Practices](#12-advanced-security-practices)                            | Least privilege, Secrets Manager lifecycle, WAF, GuardDuty, breaches    |
| 13  | [Troubleshooting Guide](#13-troubleshooting-guide)                                        | SSH issues, port diagnostics, PM2 debugging, 60-second triage           |
| 14  | [Running Node.js on AWS at Scale](#chapter-14-running-nodejs-on-aws-at-scale)             | Stateless services, session storage, scaling patterns, cost controls    |
| 15  | [Production Readiness Checklist](#production-readiness-checklist)                         | Final go-live checklist before production launch                        |

---

## Beginner Mode: Foolproof A to Z

This is the primary path for freshers, developers, and junior DevOps engineers. Follow each step in order. Do not skip validation checks.

### Step 0: Safety Gate (Read This First)

⚠️ **DO NOT TOUCH EXISTING RESOURCES**

If this AWS account already has any resources (EC2, RDS, S3, VPCs), you must treat it as a shared production account.

Rules:

1. Never modify resources you did not create.
2. Always create new VPCs, security groups, EC2 instances, IAM roles, and S3 buckets.
3. Never use the same VPC or security group as an existing project.
4. Never touch anything labeled `prod`, `production`, or `live`.

✅ **Expected result:** You understand whether the account is new (empty) or existing (shared).

How to check if resources already exist (AWS Console):

1. Open https://console.aws.amazon.com/
2. In the top search bar, type **EC2** and click **EC2**.
3. In the left menu, click **Instances**.
4. If you see any running or stopped instances, the account already has resources.
5. Repeat for **RDS** (Databases) and **S3** (Buckets).

❌ **If you see resources you did not create:** Stop and complete the Pre-Flight Audit in Section 0.6 before doing anything else.

---

### Step 1: Choose Project Name, Environment, and Region

Naming convention (mandatory):

```
project-name-environment-resource
example: acme-dev-ec2
```

Micro-steps:

1. Pick a short project name (lowercase, no spaces). Example: `acme`.
2. Choose an environment: `dev`, `staging`, or `prod`.
3. Choose one AWS region and stick to it (example: `us-east-1`).

✅ **Expected result:** You have three values you will use everywhere:

- Project name: `acme`
- Environment: `dev`
- Region: `us-east-1`

💡 Tip: Keep a small note with these values while you work.

---

### Step 2: Create an AWS Account (Skip if you already have one)

Where: **AWS Console (browser)**

Micro-steps:

1. Open Chrome or Firefox.
2. Go to https://aws.amazon.com/
3. Click **Create an AWS Account** (top-right).
4. Enter a dedicated email (example: `aws-root@yourcompany.com`).
5. Enter an AWS account name (example: `acme-dev`).
6. Verify the email with the 6-digit code.
7. Create a strong password (16+ characters).
8. Fill in contact details.
9. Add a valid credit card.
10. Verify identity via SMS or call.
11. Choose **Basic support plan (Free)**.
12. Click **Complete sign up**.

✅ **Expected result:** You can log in at https://console.aws.amazon.com/ and see the AWS Console dashboard.

❌ **Common error:** "This email address is already registered".
- **Fix:** Use **Sign in to existing account** instead of creating a new one.

---

### Step 3: Lock Down the Root Account (Mandatory)

Where: **AWS Console**

Micro-steps:

1. Log in as **Root user**.
2. Click your account name (top-right) -> **Security credentials**.
3. Under **Multi-factor authentication (MFA)**, click **Assign MFA device**.
4. Choose **Authenticator app**.
5. Scan the QR code in Google Authenticator or Authy.
6. Enter two consecutive codes and click **Add MFA**.

✅ **Expected result:** The MFA device shows as **Assigned**.

If any **Root access keys** exist:

1. In **Security credentials**, scroll to **Access keys**.
2. Click **Delete** on each key.

✅ **Expected result:** No Root access keys exist.

---

### Step 4: Create an IAM Admin User (Daily Use)

Where: **AWS Console**

Micro-steps:

1. In the search bar, type **IAM** and open it.
2. Click **Users** -> **Create user**.
3. Username: `yourname-admin` (example: `ravi-admin`).
4. Check **Provide user access to the AWS Management Console**.
5. Choose **I want to create an IAM user**.
6. Set a custom password and uncheck "Users must create a new password".
7. Click **Next**.
8. Permissions: **Attach policies directly**.
9. Search and select **AdministratorAccess**.
10. Click **Next** -> **Create user**.

✅ **Expected result:** IAM user exists and can sign in.

---

### Step 5: Enable Billing Protection

Where: **AWS Console**

Micro-steps:

1. Click your account name -> **Account**.
2. Scroll to **IAM user and role access to Billing information**.
3. Click **Edit** -> enable the checkbox -> **Update**.
4. Go to **Billing and Cost Management**.
5. In left menu, click **Budgets** -> **Create budget**.
6. Choose **Use a template (simplified)** -> **Zero spend budget**.
7. Enter budget name: `acme-zero-spend`.
8. Add your email address.
9. Click **Create budget**.

✅ **Expected result:** Budget appears in the Budgets list.

❌ **Common error:** Budget page is missing.
- **Fix:** Use https://console.aws.amazon.com/billing/ and ensure billing access is enabled.

---

### Step 6: Install the AWS CLI (Local Machine)

Where: **Your laptop**

Windows:

1. Download: https://awscli.amazonaws.com/AWSCLIV2.msi
2. Run the installer and click **Next** until **Finish**.
3. Close and reopen PowerShell.
4. Run:

```powershell
aws --version
```

macOS:

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

Linux:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

✅ **Expected result:** `aws-cli/2.x.x` appears.

❌ **Common error:** `aws: command not found`.
- **Fix:** Close and reopen your terminal, then retry.

---

### Step 7: Configure AWS CLI Credentials

Where: **AWS Console + your laptop**

Micro-steps (Console):

1. IAM -> Users -> click your IAM user.
2. Open **Security credentials** tab.
3. Under **Access keys**, click **Create access key**.
4. Choose **Command Line Interface (CLI)** and confirm.
5. Click **Create access key**.
6. Download the CSV file immediately.

Micro-steps (Terminal):

```bash
aws configure
```

Enter:

- AWS Access Key ID: (from CSV)
- AWS Secret Access Key: (from CSV)
- Default region: `us-east-1` (or your chosen region)
- Default output format: `json`

Verify:

```bash
aws sts get-caller-identity
```

✅ **Expected result:** Your Account ID and IAM user ARN are shown.

❌ **Common error:** `InvalidClientTokenId`.
- **Fix:** Re-run `aws configure` and paste keys carefully.

---

### Step 8: Create an SSH Key Pair

Where: **AWS Console**

Micro-steps:

1. Search **EC2** -> open EC2.
2. In left menu, click **Key Pairs**.
3. Click **Create key pair**.
4. Name: `acme-dev-keypair`.
5. Type: **ED25519** (recommended).
6. File format: **.pem**.
7. Click **Create key pair**.

✅ **Expected result:** A `.pem` file downloads to your computer.

Secure the key file:

Windows (PowerShell):

Replace `YOUR_NAME` with your Windows username (the folder name under `C:\Users`).

```powershell
icacls "C:\Users\YOUR_NAME\Downloads\acme-dev-keypair.pem"
icacls "C:\Users\YOUR_NAME\Downloads\acme-dev-keypair.pem" /inheritance:r /grant:r "%username%:(R)"
```

macOS/Linux:

```bash
chmod 400 ~/Downloads/acme-dev-keypair.pem
```

❌ **Common error:** `Permissions are too open`.
- **Fix:** Run the permission command above.

---

### Step 9: Pre-Flight Audit (Required for Existing Accounts)

Where: **Your laptop terminal**

Run this read-only audit and save the output before creating anything:

```bash
echo "=== IDENTITY ===" && aws sts get-caller-identity
echo "=== REGION ===" && aws configure get region
echo "=== EC2 ===" && aws ec2 describe-instances --output table
echo "=== VPCS ===" && aws ec2 describe-vpcs --output table
echo "=== S3 ===" && aws s3 ls
echo "=== RDS ===" && aws rds describe-db-instances --output table
```

✅ **Expected result:** You have a baseline of what already exists.

---

### Step 10: Create a Dedicated VPC (New Project Isolation)

Where: **AWS Console**

Micro-steps:

1. Search **VPC** -> open **VPC** service.
2. Click **Create VPC**.
3. Choose **VPC only**.
4. Name: `acme-dev-vpc`.
5. IPv4 CIDR: `10.0.0.0/16`.
6. Click **Create VPC**.

✅ **Expected result:** VPC appears in the VPC list with your name.

Create a public subnet:

1. In the left menu, click **Subnets** -> **Create subnet**.
2. Choose your VPC: `acme-dev-vpc`.
3. Subnet name: `acme-dev-public-1a`.
4. Availability Zone: pick the first in the list.
5. IPv4 CIDR: `10.0.1.0/24`.
6. Click **Create subnet**.
7. Select the subnet -> click **Actions** -> **Edit subnet settings**.
8. Enable **Auto-assign public IPv4 address** -> **Save**.

✅ **Expected result:** Subnet is created and auto-assign public IP is enabled.

Create Internet Gateway:

1. Left menu -> **Internet Gateways** -> **Create internet gateway**.
2. Name: `acme-dev-igw`.
3. Click **Create internet gateway**.
4. Click **Actions** -> **Attach to VPC** -> select `acme-dev-vpc`.

✅ **Expected result:** Internet Gateway is attached.

Create Route Table:

1. Left menu -> **Route Tables** -> select the main route table for your VPC.
2. Click **Routes** tab -> **Edit routes** -> **Add route**.
3. Destination: `0.0.0.0/0`.
4. Target: select your Internet Gateway `acme-dev-igw`.
5. Save changes.
6. Click **Subnet associations** -> **Edit subnet associations**.
7. Select `acme-dev-public-1a` -> **Save**.

✅ **Expected result:** Subnet is associated and has internet access.

---

### Step 11: Create a Security Group (Firewall)

Where: **AWS Console**

Micro-steps:

1. In EC2, click **Security Groups** (left menu).
2. Click **Create security group**.
3. Name: `acme-dev-sg`.
4. Description: `Security group for acme dev app`.
5. VPC: select `acme-dev-vpc`.
6. Add inbound rules:
   - SSH: TCP 22, Source: **My IP**
   - HTTP: TCP 80, Source: **0.0.0.0/0**
   - HTTPS: TCP 443, Source: **0.0.0.0/0**
   - App port: TCP 3000, Source: **0.0.0.0/0**
7. Click **Create security group**.

✅ **Expected result:** Security group exists with four inbound rules.

❌ **Common error:** SSH from anywhere (`0.0.0.0/0`).
- **Fix:** Restrict SSH to **My IP** only.

---

### Step 12: Launch an EC2 Instance

Where: **AWS Console**

Micro-steps:

1. In EC2, click **Instances** -> **Launch instances**.
2. Name: `acme-dev-ec2`.
3. AMI: **Amazon Linux 2023** (Free tier eligible).
4. Instance type: **t3.micro**.
5. Key pair: select `acme-dev-keypair`.
6. Network settings -> **Edit**:
   - VPC: `acme-dev-vpc`
   - Subnet: `acme-dev-public-1a`
   - Auto-assign public IP: **Enable**
   - Security group: select `acme-dev-sg`
7. Storage: set to **20 GiB gp3**.
8. Click **Launch instance**.

✅ **Expected result:** Instance state becomes **running** and shows a public IPv4 address.

---

### Step 13: Connect to the Server (SSH)

Where: **Your laptop terminal**

Micro-steps:

1. In EC2, select your instance.
2. Copy the **Public IPv4 address**.
3. Run the command below (replace the IP with your actual IP).

macOS/Linux:

```bash
ssh -i "~/Downloads/acme-dev-keypair.pem" ec2-user@12.34.56.78
```

Windows PowerShell:

Replace `YOUR_NAME` with your Windows username.

```powershell
ssh -i "C:\Users\YOUR_NAME\Downloads\acme-dev-keypair.pem" ec2-user@12.34.56.78
```

If asked **"Are you sure you want to continue connecting?"** type `yes` and press Enter.

✅ **Expected result:** You see a prompt like `[ec2-user@ip-10-0-1-42 ~]$`.

❌ **Common error:** `Permission denied (publickey)`.
- **Fix:** Ensure the key path is correct and permissions are set to 400.

---

### Step 14: Install Dependencies (Node.js + Nginx)

Where: **EC2 server (SSH session)**

Amazon Linux 2023:

```bash
sudo dnf update -y
sudo dnf install -y git curl unzip jq nginx
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
npm install -g pm2
sudo systemctl start nginx
sudo systemctl enable nginx
```

✅ **Expected result:** `node -v` shows v20.x.x and `nginx` is running.

---

### Step 15: Deploy a Sample App (Safe Test)

Where: **EC2 server (SSH session)**

```bash
mkdir -p ~/apps/acme
cd ~/apps/acme
npm init -y
npm install express

cat > app.js <<'EOF'
const express = require("express");
const app = express();
app.get("/health", (req, res) => res.status(200).send("OK"));
app.get("/", (req, res) => res.send("Hello from acme-dev"));
app.listen(3000, "0.0.0.0", () => console.log("App listening on 3000"));
EOF

pm2 start app.js --name "acme-dev"
pm2 save
```

✅ **Expected result:** `pm2 status` shows `acme-dev` as **online**.

Test locally on the server:

```bash
curl http://localhost:3000/health
```

✅ **Expected result:** `OK`

---

### Step 16: Configure Nginx Reverse Proxy

Where: **EC2 server (SSH session)**

```bash
sudo tee /etc/nginx/conf.d/acme-dev.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

sudo nginx -t
sudo systemctl reload nginx
```

✅ **Expected result:** `nginx -t` shows syntax OK and Nginx reloads successfully.

Test from your laptop:

Replace the IP with your EC2 public IP.

```bash
curl http://12.34.56.78/
```

✅ **Expected result:** `Hello from acme-dev`

---

### Step 17: Domain + HTTPS (Optional but Recommended)

If you have a domain, point it to your EC2 public IP using Route 53 or your DNS provider.

Route 53 (Console):

1. Open **Route 53** -> **Hosted zones** -> **Create hosted zone**.
2. Enter your domain (example: `example.com`).
3. Create an **A record** pointing to your EC2 public IP.

HTTPS with Certbot (EC2 server):

Replace `example.com` with your domain name.

```bash
sudo dnf install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
sudo systemctl status certbot.timer
```

✅ **Expected result:** `certbot` reports certificate success and auto-renewal timer is active.

❌ **Common error:** DNS not propagated.
- **Fix:** Wait 5-30 minutes and retry `certbot`.

---

### Step 18: Basic CI/CD (Minimal GitHub Actions)

Where: **Your GitHub repository**

Micro-steps:

1. Create a file at `.github/workflows/deploy.yml`.
2. Paste the workflow below.
3. In GitHub -> **Settings** -> **Secrets and variables** -> **Actions**, add:
   - `EC2_HOST` = your EC2 public IP
   - `EC2_USER` = `ec2-user`
   - `EC2_KEY` = your private key contents

Workflow file (copy-paste):

```yaml
name: Deploy
on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install deps
        run: npm ci
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_KEY }}
          script: |
            cd ~/apps/acme
            git pull
            npm ci --production
            pm2 restart acme-dev
```

✅ **Expected result:** A push to `main` triggers deployment and PM2 restarts the app.

---

### Step 19: Monitoring and Logs (Basic)

Where: **EC2 server + AWS Console**

EC2 server quick checks:

```bash
pm2 status
pm2 logs --lines 20
sudo systemctl status nginx
```

CloudWatch CPU alarm (Console):

1. Open **CloudWatch** -> **Alarms** -> **Create alarm**.
2. Select **EC2** metric -> **CPUUtilization** for your instance.
3. Threshold: `70%` for 5 minutes.
4. Add your email as notification.

✅ **Expected result:** Alarm shows as **OK** and you receive emails if CPU spikes.

---

### Step 20: Zero-Downtime Rule (Shared Accounts)

If this account hosts other projects, follow these rules:

- Use a new port (3001, 3002, 4000, etc.).
- Create a new Nginx config file (do not edit existing files).
- Use `sudo systemctl reload nginx` (never `restart`).
- Never run `pm2 restart all` or `pm2 delete all`.

✅ **Expected result:** Your changes do not affect other apps.

---

### Final Validation Checklist

Use this checklist before you declare success:

- [ ] EC2 instance is running
- [ ] Security group allows only required ports
- [ ] App responds to `/health`
- [ ] Nginx proxy works over HTTP
- [ ] HTTPS works (if domain configured)
- [ ] PM2 shows app online
- [ ] CloudWatch alarm configured
- [ ] Billing alerts configured
- [ ] No existing resources were modified

---

Advanced reference and deep dives begin below.

## 0. Prerequisites & Environment Setup

### Purpose

This section ensures that you have a working AWS account, a configured local terminal, and all the tools needed to follow every command in this playbook. **If you skip this section, nothing else in this document will work.**

### What You Will Achieve

By the end of this section, you will have:

- A clear mental model of the AWS Console layout and region/account scope
- A live AWS account with billing configured
- The AWS CLI installed and authenticated on your local machine
- An SSH key pair for connecting to EC2 instances
- A baseline safety protocol for shared accounts (do-not-touch rules and pre-flight audit)
- All essential CLI tools installed (`jq`, `git`, `curl`)

### Step-by-Step Implementation

1. Review the AWS Console orientation and region/account context (Section 0.0).
2. Create or access your AWS account (Section 0.1).
3. Install the AWS CLI on your local machine (Section 0.2).
4. Configure credentials so the CLI can authenticate (Section 0.3).
5. Create and secure your SSH key pair (Section 0.4).
6. If the account already has resources, follow the do-not-touch protocol (Section 0.5) and run the pre-flight audit (Section 0.6).
7. Install baseline tools used later in the playbook (Section 0.11).

### Commands

All required commands are included inline in Sections 0.2 through 0.11. Copy them exactly as shown.

### Validation

Run the validation snippets after each step. At minimum, confirm:

- `aws --version` works
- `aws sts get-caller-identity` returns your account
- Your `.pem` file exists and has restricted permissions

### Common Errors

- CLI not on PATH after install (fix: close and reopen terminal).
- Invalid Access Key or Secret (fix: re-run `aws configure` and paste carefully).
- SSH key permissions too open (fix: `chmod 400 key.pem`).

### Pro Tips

- Set billing protection early (see Section 0.9).
- Choose a primary region and stick to it.
- Store your `.pem` file in a secure password manager.

---

### 0.0 AWS Console Orientation

If you are new to AWS, learn the console layout before creating resources. Most early mistakes happen because of the wrong region or the wrong account.

Console map (approx):

```
+-----------------------------------------------------------------------+
| [menu] AWS  Search bar                 Region v  Account v  Bell       |
|-----------------------------------------------------------------------|
| Recently Visited Services                                             |
| [EC2] [IAM] [S3] [RDS]                                                |
|                                                                       |
| All Services v                                                        |
+-----------------------------------------------------------------------+
```

Key elements to locate:

- Search bar (top center): fastest way to open services
- Region selector (top right): resources are region-scoped; always confirm
- Account menu (top right): shows the 12-digit Account ID and active role
- Services menu (top left): full list of services; use search instead
- Notifications (bell icon): billing and security alerts

Single most important rule: Always confirm the current region and account before creating or modifying resources. Resources are not visible across regions.

Quick glossary (for first-time users):

- Instance: a virtual computer (EC2 instance)
- AMI: the operating system image used to create an instance
- Security group: instance firewall rules
- VPC: your private network inside AWS
- Subnet: a smaller network inside a VPC
- IAM: identity and access management
- S3 bucket: a named container for object storage

Validation: You can identify the region and Account ID in the top-right menu and confirm they match your project.

---

### 0.1 Creating an AWS Account (First-Time Setup)

> **If you already have an AWS account, skip to Section 0.2.**

#### Step-by-Step (AWS Console)

1. Open your browser and go to **https://aws.amazon.com/**
2. Click **"Create an AWS Account"** (top-right corner).
3. Enter your **email address** (this becomes the Root User email — use a team-shared alias like `aws-root@yourcompany.com`, not a personal email).
4. Choose an **Account Name** (e.g., `acme-corp-production`).
5. Enter your **credit card** details (AWS requires a valid payment method. You will not be charged unless you exceed the Free Tier limits).
6. Complete the **identity verification** (phone call or SMS).
7. Select the **Basic Support Plan** (free). You can upgrade later.
8. Click **"Complete Sign Up."**

#### Validation

```bash
# After signup, log in to the AWS Console:
# https://console.aws.amazon.com/
# You should see the AWS Management Console dashboard.
# Your Account ID is shown in the top-right dropdown next to your account name.
```

#### Common Errors

- _Error: "This email is already associated with an AWS account."_
  - **Fix:** You already have an account. Click "Sign in to an existing account" instead.
- _Error: Credit card declined._
  - **Fix:** AWS places a temporary $1 hold to verify the card. Ensure the card allows international transactions (AWS bills from the US).

#### ⚠️ Real-World Warning

The Root User email and password grant **absolute, irrevocable control** over the entire AWS account. If someone gains access to this email inbox, they can reset the Root password and take over the account. Use a dedicated, MFA-protected email — never your personal Gmail.

---

### 0.2 Installing the AWS CLI

The AWS Command Line Interface (CLI) is the tool that lets you manage AWS resources from your terminal. Every `aws` command in this playbook requires it.

#### What is it?

A program you install on your local computer (Windows, Mac, or Linux) that translates your terminal commands into AWS API calls. When you type `aws ec2 describe-instances`, the CLI authenticates with your AWS credentials, sends an HTTPS request to the EC2 API, and returns the result as JSON or a table.

#### Step-by-Step Installation

##### Windows

```powershell
# Download and run the official MSI installer
# Option 1: Download from browser
# Go to: https://awscli.amazonaws.com/AWSCLIV2.msi
# Double-click the downloaded file and follow the installer wizard.

# Option 2: Install via command line (PowerShell as Administrator)
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /quiet

# After installation, CLOSE and REOPEN your terminal, then verify:
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x Windows/10 exe/AMD64
```

##### macOS

```bash
# Download and install the official package
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify the installation
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x Darwin/23.x.x
```

##### Linux (Ubuntu / Amazon Linux / CentOS)

```bash
# Download, unzip, and install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify the installation
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x Linux/x86_64
```

#### Validation

```bash
aws --version
# If this prints a version number, the CLI is installed correctly.
# If you get "command not found", close and reopen your terminal.
```

#### Common Errors

- _Error: `aws: command not found`_
  - **Fix (Windows):** Close ALL terminal windows and reopen. The PATH variable is updated by the installer but only applies to new terminals.
  - **Fix (Linux/Mac):** Run `export PATH=$PATH:/usr/local/bin` and add it to your `~/.bashrc` or `~/.zshrc`.

---

### 0.3 Configuring AWS CLI Credentials

The CLI needs credentials to authenticate with your AWS account. There are two methods:

#### Method 1: IAM Access Keys (For Initial Setup / Learning)

> **⚠️ Warning:** This creates long-lived credentials. In production, use IAM Identity Center (SSO) with short-lived tokens as described in Section 3. For initial setup and learning, Access Keys are acceptable.

##### Step 1: Create an IAM User with Access Keys (AWS Console)

1. Log in to the AWS Console as the Root User.
2. In the search bar at the top, type **"IAM"** and click on the IAM service.
3. In the left sidebar, click **"Users"** → **"Create user"**.
4. Enter a username: `your-name-admin` (e.g., `john-admin`).
5. Click **"Next"**.
6. Select **"Attach policies directly"** → Search for and check **"AdministratorAccess"**.
   - _Why AdministratorAccess?_ For initial setup, you need full permissions. After setup, you will create scoped roles with restricted access (Section 3).
7. Click **"Next"** → **"Create user"**.
8. Click on the created user → **"Security credentials"** tab → **"Create access key"**.
9. Select **"Command Line Interface (CLI)"** → Check the acknowledgment → **"Create access key"**.
10. **SAVE BOTH KEYS IMMEDIATELY.** The Secret Access Key is shown only once. If you lose it, you must create a new key pair.
    - `Access Key ID`: Looks like `AKIAIOSFODNN7EXAMPLE`
    - `Secret Access Key`: Looks like `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`

##### Step 2: Configure the CLI

```bash
aws configure
# You will be prompted for four values:

# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json
```

_Which region to choose?_

- `us-east-1` (N. Virginia): The largest and most feature-complete AWS region. Best default choice.
- `ap-south-1` (Mumbai): Best for India-based applications.
- `eu-west-1` (Ireland): Best for EU-based applications with GDPR requirements.

##### Validation

```bash
# Test that your credentials work
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAEXAMPLEID",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/john-admin"
# }
```

#### Common Errors

- _Error: `An error occurred (InvalidClientTokenId): The security token included in the request is invalid.`_
  - **Fix:** You entered the Access Key ID or Secret incorrectly. Run `aws configure` again and paste the keys carefully.
- _Error: `An error occurred (SignatureDoesNotMatch)`_
  - **Fix:** The Secret Access Key has extra spaces or was truncated during copy-paste. Run `aws configure` again.

#### ⚠️ Real-World Warning

Your credentials are stored in plaintext at `~/.aws/credentials` (Linux/Mac) or `C:\Users\YourName\.aws\credentials` (Windows). **Never share this file**, commit it to Git, or upload it anywhere. If your laptop is stolen, immediately delete the Access Key from the IAM Console.

---

### 0.4 Creating an SSH Key Pair

SSH key pairs are required to connect to EC2 instances. The key pair consists of:

- **Public key:** Stored by AWS, injected into the EC2 instance at launch.
- **Private key (`.pem` file):** Stored on YOUR machine. This is your "password" to the server. If you lose it, you lose access.

#### Step-by-Step (CLI)

```bash
# Create a key pair and save the private key to a file
aws ec2 create-key-pair \
    --key-name prod-ssh-keypair \
    --key-type ed25519 \
    --query 'KeyMaterial' \
    --output text > prod-ssh-keypair.pem

# CRITICAL: Restrict the file permissions (Linux/Mac)
chmod 400 prod-ssh-keypair.pem

# Windows (PowerShell): Restrict permissions
# Right-click the file → Properties → Security → Advanced
# Remove all users except your own, set to "Read" only.
```

_Why `ed25519`?_ It is faster and more secure than the older `rsa` key type. Use `rsa` only if connecting from very old SSH clients that do not support ed25519.

#### Step-by-Step (AWS Console)

1. Go to the AWS Console → Search **"EC2"** → Click **EC2**.
2. In the left sidebar, scroll down to **"Network & Security"** → Click **"Key Pairs"**.
3. Click **"Create key pair"**.
4. Name: `prod-ssh-keypair`
5. Key pair type: **ED25519**
6. Private key file format: **.pem** (for Linux/Mac/WSL) or **.ppk** (for PuTTY on Windows)
7. Click **"Create key pair"** — the `.pem` file automatically downloads.
8. Move it to a safe location and set permissions: `chmod 400 prod-ssh-keypair.pem`

#### Validation

```bash
# Verify the key pair exists in AWS
aws ec2 describe-key-pairs --key-names prod-ssh-keypair --output table

# Verify the local file exists and has correct permissions
ls -la prod-ssh-keypair.pem
# Expected: -r-------- 1 youruser yourgroup ... prod-ssh-keypair.pem
```

#### Common Errors

- _Error: SSH says `Permissions 0644 for 'key.pem' are too open. This private key will be ignored.`_
  - **Fix:** `chmod 400 prod-ssh-keypair.pem` — SSH refuses to use key files that are readable by other users.
- _Error: "Key pair already exists"_
  - **Fix:** Choose a different name, or delete the existing one: `aws ec2 delete-key-pair --key-name prod-ssh-keypair`

#### ⚠️ Real-World Warning

If you lose the `.pem` file, **there is no way to recover it from AWS.** AWS does not store the private key. You must create a new key pair and update all instances. Store backups in a password manager (1Password, HashiCorp Vault), never in Git or Slack.

---

### 0.5 Do Not Touch Existing Resources

If this AWS account already contains workloads, follow this protocol before you create or modify anything.

Golden rule: If you did not create it, do not touch it. Create new, isolated resources for every project.

Rules:

1. Identify before touching. Read the name and tags; stop if it is not your project.
2. Never modify existing VPCs, security groups, EC2 instances, RDS, or S3 buckets. Create new ones.
3. Use the naming convention in Section 0.8 for every resource you create.
4. Capture a baseline (screenshots or CLI output) before you start.
5. Ask when in doubt. A five-minute question prevents a five-hour outage.

Red flags (stop and ask):

- Names containing `prod`, `production`, `live`, or `main`
- Tag `Environment=Production`
- Untagged resources or unknown owners
- Security groups with many inbound rules
- RDS with Multi-AZ enabled
- S3 buckets with heavy activity or recent writes

---

### 0.6 Pre-Flight Audit

Run this read-only audit before creating anything in an existing account. Save the output and share it with your team lead.

```bash
# Save output to a file for later comparison
# Example: audit-results-YYYY-MM-DD.txt

echo "=== 1. IDENTITY CHECK ==="
aws sts get-caller-identity

echo "=== 2. REGION CHECK ==="
aws configure get region

echo "=== 3. EC2 INSTANCES ==="
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[Tags[?Key==`Name`].Value|[0],InstanceId,InstanceType,State.Name,PrivateIpAddress]' \
  --output table

echo "=== 4. VPCS ==="
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0],IsDefault]' \
  --output table

echo "=== 5. SECURITY GROUPS ==="
aws ec2 describe-security-groups \
  --query 'SecurityGroups[*].[GroupId,GroupName,VpcId]' \
  --output table

echo "=== 6. S3 BUCKETS ==="
aws s3 ls

echo "=== 7. RDS INSTANCES ==="
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceClass,Engine,DBInstanceStatus]' \
  --output table

echo "=== 8. IAM USERS ==="
aws iam list-users \
  --query 'Users[*].[UserName,CreateDate,PasswordLastUsed]' \
  --output table

echo "=== 9. BILLING ALERTS (OPTIONAL) ==="
aws cloudwatch describe-alarms --output table
```

If any command fails with `AccessDenied`, request the `SecurityAudit` and `ViewBilling` managed policies before proceeding.

---

### 0.7 Quick Deployment Runbook (A → Z)

Use this only for a new project where you have explicit permission to create new resources. If the account already hosts workloads, complete Section 0.5 and Section 0.6 first, and follow the isolation protocol in Section 0.8.

This is the fastest safe path from a blank AWS account to a live HTTPS Node.js app. It is intentionally explicit and beginner-friendly. Every step is executable.

### Step 1: Create the AWS Account

1. Go to **https://aws.amazon.com/** → **Create an AWS Account**.
2. Use a dedicated root email (not personal), set a strong password, and add a credit card.
3. Choose the **Basic Support Plan**.
4. Log in at **https://console.aws.amazon.com/** and confirm you can see the AWS Console.

_Validation:_ You can open the AWS Console and see your account ID in the top-right dropdown.

### Step 2: Setup IAM + AWS CLI + MFA

1. Enable MFA on the Root account (IAM → Security credentials → MFA).
2. Create an IAM Admin user (IAM → Users → Create user → attach `AdministratorAccess`).
3. Create Access Keys for the IAM Admin user (Security credentials → Create access key).
4. Install AWS CLI and configure credentials:

```bash
aws configure
aws sts get-caller-identity
```

_Validation:_ `aws sts get-caller-identity` returns your account and IAM user ARN.

### Step 3: Create the VPC (Networking Baseline)

**Beginner path (Console):**

1. AWS Console → **VPC** → **Create VPC** → **VPC and more**.
2. Name: `prod-vpc`, IPv4 CIDR: `10.0.0.0/16`.
3. AZs: 2, Public subnets: 2, Private subnets: 2.
4. NAT Gateway: **1 per AZ** (or 1 total for cost control).
5. Click **Create VPC**.

> ⚠️ Cost Warning: NAT Gateways are billed per hour and per GB. For low-traffic dev environments, avoid NAT or use VPC Endpoints for S3/DynamoDB to reduce spend.

_Validation:_ You can see public and private subnets in the VPC Console.

### Step 4: Launch EC2 (Key Pair + Security Group + Instance)

Create the SSH key pair and security group (CLI):

```bash
# Key pair
aws ec2 create-key-pair \
  --key-name prod-ssh-keypair \
  --query 'KeyMaterial' \
  --output text > prod-ssh-keypair.pem
chmod 400 prod-ssh-keypair.pem

# Security group
aws ec2 create-security-group \
  --group-name prod-app-sg \
  --description "Prod app SG" \
  --vpc-id vpc-0abcd1234

# SSH (22) from your IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234def56789 \
  --protocol tcp \
  --port 22 \
  --cidr YOUR_PUBLIC_IP/32

# HTTP (80) and HTTPS (443) from the internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234def56789 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234def56789 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

Launch the instance (CLI):

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name prod-ssh-keypair \
  --subnet-id subnet-public-1a \
  --security-group-ids sg-0abc1234def56789 \
  --associate-public-ip-address \
  --count 1

# Allocate an Elastic IP (stable public IP for DNS)
aws ec2 allocate-address --domain vpc

# Associate the Elastic IP to the instance
# Replace INSTANCE_ID and EIP_ALLOCATION_ID with real values
aws ec2 associate-address \
  --instance-id i-0123456789abcdef0 \
  --allocation-id eipalloc-0123456789abcdef0
```

_Validation:_ `aws ec2 describe-instances` shows the instance in `running` state with a public IP.

### Step 5: SSH into the Server

```bash
ssh -i prod-ssh-keypair.pem ec2-user@PUBLIC_IP
```

_Validation:_ You see a shell prompt on the instance and `hostname` returns the EC2 host.

### Step 6: Install Dependencies (OS + Node + Nginx)

Amazon Linux 2023:

```bash
sudo dnf update -y
sudo dnf install -y git curl unzip jq nginx

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts --latest-npm
node -v
npm -v

# Optional: update npm to the latest stable version
npm install -g npm@latest
npm install -g pm2
pm2 -v
```

Ubuntu 24.04:

```bash
sudo apt-get update -y
sudo apt-get install -y git curl unzip jq nginx

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts --latest-npm
node -v
npm -v
npm install -g pm2
pm2 -v
```

### Step 7: Deploy the Backend

```bash
cd /opt
sudo mkdir -p apps
sudo chown $USER:$USER apps
cd apps
git clone https://github.com/your-org/your-app.git
cd your-app
npm ci --production
npm run build
```

### Step 8: Configure Nginx (Reverse Proxy)

```bash
sudo tee /etc/nginx/conf.d/app.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
    }
}
EOF

sudo nginx -t
sudo systemctl reload nginx
```

Note: Use `reload` to avoid downtime on shared hosts. Avoid `restart` unless you are the only tenant on the server.

### Step 9: Setup Domain + DNS (Route 53)

1. Route 53 → **Hosted zones** → **Create hosted zone** for `yourdomain.com`.
2. If your domain is registered elsewhere, update the domain registrar to use the Route 53 name servers.
3. Add an **A record** pointing to your EC2 **Elastic IP**.
4. If you use an external DNS provider instead of Route 53, create the same **A record** there.

CLI alternative:

```bash
aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference "prod-$(date +%Y%m%d%H%M%S)"

cat > /tmp/route53-record.json <<'EOF'
{
  "Comment": "A record for EC2",
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "yourdomain.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [{"Value": "YOUR_ELASTIC_IP"}]
    }
  }]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID_HERE \
  --change-batch file:///tmp/route53-record.json
```

_Validation:_ `dig +short yourdomain.com` returns your public IP.

### Step 10: Enable HTTPS (SSL)

```bash
# Ubuntu/Debian
sudo apt install certbot python3-certbot-nginx -y

# Amazon Linux 2023
sudo dnf install certbot python3-certbot-nginx -y

sudo certbot --nginx -d yourdomain.com

# Ensure auto-renewal is active (systemd timer)
sudo systemctl status certbot.timer
```

_Validation:_ `sudo certbot renew --dry-run` succeeds.

### Step 11: Start the Application (PM2)

```bash
pm2 start app.js --name my-app
pm2 save
pm2 startup
pm2 logs
```

### Step 12: Verify the Live System

```bash
curl -I https://yourdomain.com
pm2 status
sudo systemctl status nginx
```

_Validation:_ `curl` returns HTTP 200 or 301/302 to HTTPS and `pm2 status` shows `online`.

---

### 0.8 New Project Isolation Protocol

Every new project must be isolated from existing workloads to prevent cross-impact and simplify ownership.

Isolation checklist:

- Dedicated VPC and subnets
- Dedicated security groups
- Dedicated compute resources (EC2, ECS, or Lambda)
- Dedicated IAM role and policies
- Dedicated S3 buckets and databases

#### Naming Convention

Pattern: `{project}-{environment}-{resource}`

Examples:

- `blog-dev-vpc`
- `payments-staging-sg`
- `api-prod-ec2`

Rules: lowercase only, hyphens only, and always include the environment.

#### VPC CIDR Allocation

| Slot | CIDR Block | Intended Use |
| --- | --- | --- |
| 1 | `10.0.0.0/16` | First project |
| 2 | `10.1.0.0/16` | Second project |
| 3 | `10.2.0.0/16` | Third project |
| 4 | `10.3.0.0/16` | Fourth project |

#### Port Isolation (Shared Hosts)

Reserved ports: 22 (SSH), 80 (HTTP), 443 (HTTPS)

Suggested app ports: 3000, 3001, 3002, 4000, 4001, 8080

Port allocation reference:

```
Standard ports (do not use for apps):
22   -> SSH
80   -> Nginx HTTP
443  -> Nginx HTTPS

Suggested app ports:
3000 -> Project 1
3001 -> Project 2
3002 -> Project 3
4000 -> Project 1 API
4001 -> Project 2 API
8080 -> Admin dashboard
```

Check existing listeners before choosing a port:

```bash
sudo ss -tlnp | grep LISTEN
# or
sudo netstat -tlnp | grep LISTEN
```

#### Zero-Downtime Changes on Shared Servers

- Create a new Nginx conf file in `/etc/nginx/conf.d/`; do not edit existing conf files.
- Test and reload configuration changes:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

- Start your app with a unique PM2 name and avoid global operations:

```bash
pm2 start app.js --name "myapp-dev"
pm2 status
```

Avoid `pm2 restart all`, `pm2 stop all`, and `pm2 delete all` on shared servers.

#### Environment Isolation (Dev / Staging / Production)

Always maintain three separate environments, each isolated from the others.

| Environment | Purpose | Scale | Data | Access |
| --- | --- | --- | --- | --- |
| Dev | Experiment freely, break things | Small (t3.micro) | Fake test data only | All developers |
| Staging | Final QA before production | Same as production | Anonymized copy of prod | QA + senior devs |
| Production | Real users and real data | Production size | Real customer data | Senior DevOps only |

Golden rule:

- Code flows up: Dev -> Staging -> Production
- Data flows down: Production -> (anonymized copy) -> Staging
- Production data never goes to Dev

Environment-specific configuration:

```bash
.env.dev       # Development settings
.env.staging   # Staging settings
.env.prod      # Production settings (store in Secrets Manager, not Git)
```

Never put production credentials in `.env.dev`. A single mistake can cause irreversible data damage.

---

### 0.9 Billing Protection

Do this immediately after account creation or before provisioning non-free resources.

Free Tier quick limits (first 12 months):

| Service | Free Tier Limit | Exceeds When |
| --- | --- | --- |
| EC2 | 750 hours/month of t2.micro or t3.micro | More than 750 hours total across instances |
| S3 | 5 GB storage, 20,000 GET requests | Storage > 5 GB or high request volume |
| RDS | 750 hours/month of db.t2.micro or db.t3.micro | Larger instance sizes or Multi-AZ |
| Lambda | 1 million requests/month | More than 1 million requests |
| Data transfer | 1 GB out per month | More than 1 GB outbound data |

Note: Free Tier applies per account, not per project.

1. Enable IAM access to billing information:
   - AWS Console -> Account -> "IAM user and role access to Billing information" -> Edit -> Enable
2. Create a budget:
   - Billing -> Budgets -> Create budget -> choose "Zero spend" or "Monthly cost"
3. Add email recipients for alerts.

Optional: Configure Cost Anomaly Detection under Billing to alert on unusual spend spikes.

Validation: You can see the budget in the Billing console and receive alert emails.

Free Tier usage: https://console.aws.amazon.com/billing/home#/freetier

---

### 0.10 New Developer Onboarding

Before you touch an existing AWS account, your team lead must provide:

- IAM username and temporary password
- Account ID and approved region
- Naming convention for resources
- Approved VPCs/subnets for new projects
- A list of known "do-not-touch" resources
- The escalation contact for approvals

First steps:

1. Sign in and change your temporary password.
2. Enable MFA on your IAM user.
3. Configure the AWS CLI (Section 0.3).
4. Run the read-only pre-flight audit (Section 0.6).
5. Confirm which resources you are allowed to create before starting.

---

### 0.11 Installing Essential Tools

Several tools are referenced throughout this playbook. Install them now to avoid interruptions later.

```bash
# jq — JSON processor (used to parse AWS CLI output)
# macOS:
brew install jq
# Ubuntu/Debian:
sudo apt-get install jq -y
# Amazon Linux / CentOS:
sudo dnf install jq -y
# Windows (via Chocolatey):
choco install jq -y

# git — Version control (used for all code deployments)
# macOS: (pre-installed, or)
brew install git
# Ubuntu/Debian:
sudo apt-get install git -y
# Amazon Linux:
sudo dnf install git -y

# Verify all tools
aws --version && jq --version && git --version && curl --version | head -1
```

---

### 0.12 Pro Tips for Beginners

1. **AWS Free Tier:** New accounts get 12 months of Free Tier, including 750 hours/month of `t2.micro` or `t3.micro` EC2 instances, 5 GB of S3 storage, and 750 hours of RDS `db.t3.micro`. Stay within these limits to avoid charges during learning.
2. **Set Billing Protection Immediately:** Before doing anything else, set billing protection (covered in Section 0.9) so you are alerted if costs exceed $5/day. This is your safety net against accidentally leaving expensive resources running.
3. **Region Matters:** All resources are region-specific. If you create an EC2 instance in `us-east-1` but your Console is set to `us-west-2`, you will not see it. Always check the region selector (top-right of the Console).
4. **When in Doubt, Don't Delete:** If you are unsure whether a resource is important, **stop it** (to halt billing) rather than **terminate it** (which is irreversible). Stopped EC2 instances do not incur compute charges (but EBS volumes still do).

## 1. Introduction and Architectural Philosophy

### Purpose

Explain why this playbook exists, how it should be used, and the consequences of deviating from it.

### What You Will Achieve

- Understand the enterprise rationale behind each control and practice
- Learn when to use (and not use) this playbook
- Build the mental model needed to follow later technical sections safely

### Step-by-Step Implementation

1. Read Sections 1.1 through 1.6 in order.
2. Do not skip the real-world failure examples; they explain why the rules exist.

### Commands

This section is conceptual; no CLI commands are required.

### Validation

You can summarize the core mandate in 2-3 sentences and explain the risks of ignoring it.

### Common Errors

- Skipping this section and treating the playbook as optional guidance.
- Assuming AWS defaults are safe without understanding why they are not.

### Pro Tips

- Revisit this section after your first real incident; it will read differently.
- Use the failure stories as training examples for new team members.

### 1.1 The Prime Directive: Why AWS Setup Documentation is the Lifeblood of the Enterprise

#### What is this Playbook?

This Enterprise AWS Setup and Deployment Playbook is the definitive, authoritative source of truth for all cloud operations within our organization. It is a highly opinionated, rigidly structured architectural constitution. It dictates the exact patterns, guardrails, and mechanical processes required to design, provision, deploy, secure, and monitor infrastructure across our Amazon Web Services (AWS) ecosystem.

Unlike a generic AWS tutorial or the official AWS documentation—which tells you _everything_ that is possible—this playbook explicitly dictates _our specific way_ of utilizing the cloud. It defines the constraints that keep our organization safe, performant, and profitable.

#### Why is this Documentation Critical in a Real Company?

In an enterprise technology organization, cloud infrastructure cannot rely on the fleeting memory, tribal knowledge, or personal preferences of individual senior engineers. The transition from an ad-hoc, startup-minded "click-ops" methodology (manually creating resources via the AWS Management Console) to a formalized, playbook-driven approach is the defining characteristic of a mature, hyper-scaling engineering culture.

1. **Eradication of Configuration Drift:** Without a rigidly enforced playbook, environments inevitably diverge. A developer tweaks a Security Group in the Staging environment to test a feature but forgets to apply that same tweak to Production. Suddenly, a deployment that passed all automated tests in Staging fails catastrophically in Production. This playbook, combined with Infrastructure as Code (IaC), enforces absolute parity. It ensures that infrastructure is deterministic and immutable.
2. **Accelerated Incident Response and Reduced MTTR:** During a Severity-1 (Sev-1) outage at 3:00 AM, cognitive bandwidth is severely limited. Responding engineers do not have the luxury of guessing how the networking topology is configured or where the database read replicas are situated. This playbook provides deterministic, known-good states. It allows incident commanders to rapidly identify deviations from the baseline, isolate the fault domain, and restore services efficiently.
3. **The Financial and Fiduciary Imperative:** Cloud computing operates on a utility model; you pay for what you provision, not necessarily what you use. Without strict, documented standards, cloud costs spiral out of control logarithmically. This playbook establishes mandatory resource tagging, auto-scaling thresholds, and lifecycle management policies that align our cloud spend directly with business revenue, preventing financial hemorrhaging.
4. **Compliance, Security, and Auditing:** Enterprise clients, SOC2 auditors, and strict compliance frameworks (such as HIPAA, PCI-DSS, or FedRAMP) require absolute, documented proof of infrastructure controls. It is never enough to claim that "we are secure." We must prove that our security controls are operationalized, audited, and strictly adhered to. This playbook acts as the verifiable baseline that our security posture is engineered by default, not bolted on as an afterthought.

#### When to use this Playbook

- **Architecting New Microservices:** Before writing a single line of application code, engineers must consult this playbook to align with established networking tiers, IAM boundary definitions, and database selection criteria.
- **Executing Production Deployments:** During both routine feature releases and emergency zero-day hotfixes, the deployment strategies and rollback mechanisms outlined here must be followed to guarantee zero-downtime rollouts.
- **Onboarding and Skill Acceleration:** During the first weeks of an engineer's tenure, this document serves as the primary syllabus for understanding how our specific enterprise harnesses the cloud.
- **Incident Post-Mortems:** When conducting Blameless Root Cause Analyses (RCAs), this playbook is used to determine if a failure occurred due to a deviation from the standard, or if the standard itself needs to be updated to prevent future occurrences.

#### When NOT to use this Playbook

- **Local Sandbox Experimentation:** While the principles herein are beneficial, strict adherence is not mandated for isolated, local-only development using mock frameworks (like LocalStack), provided absolutely no live AWS credentials or enterprise data are utilized.
- **Non-AWS Environments:** This playbook is explicitly and exclusively tailored for Amazon Web Services. Azure, Google Cloud Platform (GCP), or on-premises physical data center operations have fundamentally different operational paradigms and require their own distinct architectural governance.

---

### 1.2 Real-World Pathologies: The True Cost of Poor Standardization

The rules within this playbook are not arbitrary bureaucratic hurdles. They are written in the blood of past outages, severe data breaches, and massive financial missteps. A poorly configured AWS environment is a silent liability that can destroy a startup overnight or cause irreparable reputational damage to an established enterprise. Understanding these real-world pathologies is critical to respecting the engineering rigor we enforce.

#### 1.2.1 Security Tragedies: The Expanding Blast Radius

- **The IAM Wildcard Epidemic**
  - _What it is:_ A pervasive anti-pattern in immature engineering teams is utilizing overly permissive Identity and Access Management (IAM) policies. Developers facing "Access Denied" errors often resort to assigning `"Action": "*", "Resource": "*"` to an IAM role to temporarily bypass the issue, intending to fix it later. "Later" never comes.
  - _Real-World Impact:_ A seemingly harmless application vulnerability (like a Server-Side Request Forgery - SSRF) allows an external threat actor to access the EC2 instance metadata service. Because the instance role has wildcard permissions, the attacker pivots laterally, deletes KMS encryption keys, drops RDS production databases, and spins up hundreds of GPU instances for illicit crypto-mining.
  - _The Playbook Mandate:_ This playbook enforces the absolute Principle of Least Privilege (PoLP). Every IAM role must have explicit boundaries. We do not allow wildcards in production. Ever.

- **The Public S3 Bucket Catastrophe**
  - _What it is:_ Misconfigured Amazon S3 bucket policies or outdated Access Control Lists (ACLs) that unintentionally expose object storage to the public internet.
  - _Real-World Impact:_ A developer temporarily makes an S3 bucket public to easily share a log file with a vendor, forgetting to revert the policy. Months later, automated internet scanners index the bucket. Competitors or malicious actors download terabytes of proprietary datasets, source code, and Personally Identifiable Information (PII), resulting in severe GDPR/CCPA fines, class-action lawsuits, and total loss of customer trust.
  - _The Playbook Mandate:_ Explicit "Block Public Access" at the AWS Account level is mandated. Customer Managed Keys (CMKs) must encrypt all objects, ensuring that even if a bucket policy is breached, the data remains cryptographically secure without the corresponding KMS access.

- **Hardcoded Credentials and Access Key Sprawl**
  - _What it is:_ Leaving long-lived AWS IAM Access Keys and Secret Keys hardcoded in source code, committed to Git repositories, or embedded in unencrypted `.env` files on developer machines.
  - _Real-World Impact:_ Automated scraping bots monitor public and private GitHub repositories 24/7. Within mere seconds of a developer accidentally committing an AWS key, bots authenticate via the AWS CLI and launch thousands of maximum-size compute instances across all global regions. Enterprises have incurred $100,000+ bills over a single weekend due to this mistake.
  - _The Playbook Mandate:_ We strictly forbid the generation of long-lived IAM user keys for humans. All access must be brokered via AWS IAM Identity Center (SSO) with short-lived session tokens and enforced Multi-Factor Authentication (MFA).

#### 1.2.2 Billing Hemorrhages: Death by a Thousand Cuts

- **Orphaned and "Invisible" Resources**
  - _What it is:_ Failing to manage the lifecycle of infrastructure. This occurs when EC2 instances are terminated but their attached Elastic Block Store (EBS) volumes are left behind, or when Elastic IPs are unassigned but never returned to the AWS pool.
  - _Real-World Impact:_ An organization can bleed tens of thousands of dollars monthly on resources that provide absolutely zero business value. Because these resources are not actively attached to a running application, they fly under the radar of engineering teams but heavily impact the bottom line.
  - _The Playbook Mandate:_ Strict Infrastructure as Code practices ensure that when a stack is destroyed, all associated dependencies are cleanly terminated.

- **The NAT Gateway Data Transfer Trap**
  - _What it is:_ Placing high-bandwidth workloads (such as data lakes, continuous CI/CD runners, or heavy internal microservice-to-microservice chatter) in private subnets that route all outbound traffic through a single NAT Gateway, without implementing VPC Endpoints.
  - _Real-World Impact:_ AWS charges heavily for data processed through a NAT Gateway. A company might provision a $50/month EC2 instance but generate a $50,000/month bill purely in NAT Gateway data transfer fees because that instance is constantly pulling multi-gigabyte files from Amazon S3.
  - _The Playbook Mandate:_ This playbook explicitly dictates network routing strategies, mandating the use of Gateway VPC Endpoints for S3 and DynamoDB to bypass NAT charges entirely and keep traffic on the internal AWS backbone.

- **Un-attributable Cloud Spend**
  - _What it is:_ Provisioning AWS resources without a strict, enforced metadata tagging strategy.
  - _Real-World Impact:_ When the monthly AWS bill spikes by 40%, the Finance department demands answers. If resources lack tags (e.g., `Environment: Prod`, `CostCenter: DataScience`, `Service: AuthAPI`), engineering leadership cannot determine which department, team, or specific microservice caused the spike. This makes budget forecasting and departmental chargebacks impossible, leading to a freeze on all cloud spending.
  - _The Playbook Mandate:_ Mandatory tagging policies enforced by AWS Organizations Service Control Policies (SCPs). Any resource deployed without the required enterprise tags will be automatically rejected by the AWS API.

#### 1.2.3 Downtime, Latency, and Operational Paralysis

- **Fake High Availability (HA) and Fault Domain Ignorance**
  - _What it is:_ Deploying critical databases, caching layers, or application servers in a single AWS Availability Zone (AZ) to save money, or simply out of ignorance of AWS global infrastructure design.
  - _Real-World Impact:_ While AWS is highly reliable, individual data centers (AZs) do experience power losses, cooling failures, or network fiber cuts. If an application is pinned to `us-east-1a`, and `us-east-1a` goes offline, the entire application stack crashes. True High Availability requires cross-AZ redundancy.
  - _The Playbook Mandate:_ Multi-AZ deployments are non-negotiable for all staging and production workloads. This document outlines the exact architectures required to survive the complete loss of an Availability Zone with zero data loss (RPO=0).

- **Manual "Click-Ops" Deployments and Fear of Releasing**
  - _What it is:_ Engineers logging into the AWS web console to manually update Lambda functions, upload zip files, change security group routing rules, or modify auto-scaling group capacities.
  - _Real-World Impact:_ Undocumented, untrackable changes accumulate over months (Configuration Drift). Eventually, the environment becomes a fragile "black box." The team becomes terrified to deploy new code because no one truly understands the current state of the infrastructure. When the system inevitably breaks, rollback is impossible because there is no version history of the infrastructure changes.
  - _The Playbook Mandate:_ Immutability. We do not patch or manually mutate running infrastructure. If a change is needed, it is committed to version control, tested, and deployed via a continuous integration/continuous deployment (CI/CD) pipeline.

---

### 1.3 Audience Definition and Playbook Consumption Strategy

An enterprise consists of individuals with vastly different roles, skill sets, and objectives. This playbook is designed to be the universal translation layer between high-level architectural theory and low-level engineering execution. Here is how different personas must interact with this document.

#### 1.3.1 Platform and DevOps Engineers (The Architects and Enforcers)

- **Who they are:** The custodians of the cloud. The engineers who build the platforms that other engineers use.
- **How to use it:** This is your technical constitution. DevOps engineers are responsible for evolving this playbook, debating its contents, and ultimately enforcing its standards programmatically via CI/CD pipelines, Open Policy Agent (OPA), and infrastructure linters (like Checkov, TFSec, or AWS CloudFormation Guard).
- **Focus Areas:** You must deeply study and internalize every section. Pay particular attention to the complex networking topologies, cross-account IAM role assumption mechanics, centralized logging infrastructure, and disaster recovery automation. Use this document as your primary weapon to justify architectural pushback when application development teams request non-standard, insecure infrastructure bypasses.

#### 1.3.2 Application and Backend Developers (The Consumers)

- **Who they are:** The engineers writing the business logic (Node.js, Python, Java, Go) that runs on top of the infrastructure.
- **How to use it:** As a comprehensive integration manual and operational constraint guide. Developers do not need to build the foundation, but they must understand the foundation to build stable houses. You need to understand how the platform will host your containerized code, how secrets are securely injected into your application at runtime, how logs must be formatted to be properly aggregated, and what the network perimeter boundaries look like.
- **Focus Areas:** Focus intensely on the sections detailing CI/CD integration, containerization and Serverless standards (EKS, ECS, Lambda), Database connection pooling, and observability requirements. You do not need to memorize how to provision a Transit Gateway, but you must understand how your microservice's traffic is routed through it to reach the internet.

#### 1.3.3 Freshers and Junior Engineers (The Apprentices)

- **Who they are:** New hires, recent graduates, or engineers transitioning into cloud-native development for the first time.
- **How to use it:** As a structured, step-by-step survival guide and safety net. The sheer scale and complexity of an enterprise AWS environment can induce paralyzing impostor syndrome in freshers. This playbook demystifies the magic and removes the fear of "breaking production."
- **Focus Areas:** Read sequentially. Rely heavily on the step-by-step implementation guides. After executing any step, immediately perform the documented validation checks to confirm success. Pay close attention to the "Why" and "When NOT to use" subsections. This document will train your intuition, allowing you to build a mental model of enterprise trade-offs and safely contribute to production systems within your first month.

#### 1.3.4 Leadership, Clients, and Compliance Auditors (The Validators)

- **Who they are:** Engineering Managers, CTOs, B2B Enterprise Clients, and third-party security auditors (SOC2, ISO 27001).
- **How to use it:** As immutable evidence of operational maturity, risk mitigation, and security compliance.
- **Focus Areas:** The Introduction, Architectural Philosophy, Security Posture, Identity Management, and Disaster Recovery sections. This document proves to stakeholders that the organization relies on disciplined engineering rigor—not luck or heroism—to protect sensitive data, ensure aggressive Service Level Agreements (SLAs), and maintain system uptime.

---

### 1.4 Scope: What This Playbook Covers

This playbook provides an exhaustive, end-to-end blueprint for managing our enterprise-grade AWS environment. Subsequent sections will dive deep into step-by-step implementations of the following domains:

- **Foundational Infrastructure & Networking:**
  - Multi-account strategies utilizing AWS Organizations and AWS Control Tower.
  - Virtual Private Cloud (VPC) design, Subnet isolation (Public, Private, and strictly Isolated database tiers).
  - AWS Transit Gateways for cross-VPC communication.
  - Route53 advanced DNS management and traffic policies.
- **Compute, Container Orchestration, and Serverless:**
  - Autoscaling EC2 fleets and immutable Golden AMI build pipelines.
  - Elastic Kubernetes Service (EKS) and Elastic Container Service (ECS) architectures.
  - Serverless deployment paradigms utilizing AWS Lambda, API Gateway, and AWS Fargate.
- **State, Storage, and Data Persistence:**
  - Relational data management via Amazon RDS and Amazon Aurora (multi-AZ deployments, read replica scaling).
  - High-throughput NoSQL modeling with Amazon DynamoDB.
  - In-memory caching strategies with Amazon ElastiCache (Redis).
  - Object storage standards, robust S3 lifecycle policies, and intelligent data tiering for cost reduction.
- **Deployment Mechanics & CI/CD Tooling:**
  - Immutable infrastructure principles and GitOps workflows.
  - Advanced release strategies: Blue/Green deployments, Canary rollouts, and automated health-check rollbacks.
  - Deep integration with our designated CI/CD runners (e.g., GitHub Actions, GitLab CI, or AWS CodePipeline).
- **Security Posture, Identity, and Perimeter Defense:**
  - Identity and Access Management (IAM) role assumption, least privilege boundary limits, and OIDC federation.
  - Cryptography: Encryption at rest and in transit via AWS Key Management Service (KMS) and AWS Certificate Manager (ACM).
  - Perimeter protection via AWS Web Application Firewall (WAF), AWS Shield Advanced for DDoS mitigation, and strictly enforced Security Groups.
- **Observability, Scaling, and Resilience Engineering:**
  - Centralized logging, distributed request tracing, and telemetry aggregation (CloudWatch, OpenTelemetry).
  - Autoscaling triggers based on custom metrics and predictive scaling algorithms.
  - Disaster Recovery execution plans (defining strict RTO/RPO limits) and automated backup lifecycle management (AWS Backup).

---

### 1.5 Out of Scope: What This Document Will NOT Cover

To maintain strict focus on enterprise infrastructure, security, and deployment excellence, the following topics are explicitly out of scope for this playbook. Diluting this document with unrelated topics diminishes its value as an authoritative infrastructure guide.

1. **Application Business Logic and Source Code Structure:** This playbook dictates _how_ to deploy a Node.js or Python microservice securely and reliably. It does _not_ dictate how to write the specific business logic, design the REST API endpoints, format the JSON payloads, or optimize internal database SQL queries within those applications.
2. **Local Developer Workstation Setup:** Instructions for installing Docker, configuring IDEs (VSCode/IntelliJ), setting up local git hooks, or installing language runtimes on a developer's physical laptop belong in the entirely separate "Local Development Onboarding Wiki."
3. **Basic Programming Language Tutorials:** Readers are expected to possess a baseline working knowledge of the programming and scripting languages (e.g., Python, Go, Bash, HCL) utilized in our automation and infrastructure code. We do not teach syntax here; we teach architectural application.
4. **Third-Party SaaS Tools (Non-Infrastructure):** While we may outline the architectural patterns for integrating logs with external vendors (like Datadog, New Relic, or Splunk) or routing alerts to incident management platforms (like PagerDuty), exhaustive, step-by-step configuration manuals for those external SaaS UI dashboards are excluded.
5. **Physical On-Premises Data Center Management:** This is a cloud-native AWS document. While hybrid network routing (such as configuring AWS Direct Connect or Site-to-Site VPNs back to a corporate office) is covered, managing physical hardware, racking servers, or provisioning local hypervisors is strictly out of scope.

---

### 1.6 The Final Mandate

The cloud infrastructure defined and governed by this playbook is treated as a first-class software product. It requires the exact same level of rigor—mandatory peer code reviews, automated unit testing, static analysis, and strict release management—as our core customer-facing applications.

By aggressively adhering to the guidelines documented in the subsequent sections, we ensure that our AWS ecosystem remains a strategic business asset—infinitely scalable, aggressively secure, and highly resilient—rather than an unpredictable, costly liability.

## 2. Scenario-Based Entry & Decision Logic

### Purpose

Help you classify the environment you are entering so you follow the correct operational path.

### What You Will Achieve

- Determine whether you are in Greenfield, Brownfield, or Black Box mode
- Run the correct pre-flight audits before touching production systems
- Know when to proceed and when to reject the engagement

### Step-by-Step Implementation

1. Read the IF/ELSE decision logic in Section 2.1.
2. Follow the exact scenario path (2.2, 2.3, or 2.4) without mixing steps.
3. Complete the validation checks before provisioning any new resources.

### Commands

All required audit commands are included inline in Sections 2.3 and 2.4.

### Validation

Confirm the environment classification and document the pre-flight audit results.

### Common Errors

- Starting deployments before the audit is complete.
- Accepting insufficient IAM permissions and proceeding anyway.

### Pro Tips

- Save audit outputs (CSV, screenshots) in your project documentation.
- If any audit step raises suspicion, pause and escalate before continuing.

### 2.1 The Engagement Triad: Defining the Operational Baseline

Before deploying a single resource or executing a single script, the Platform Engineering team must accurately classify the environment they are entering. In enterprise and consulting engagements, infrastructure is handed over in vastly different states of maturity and decay.

This section defines the "IF/ELSE" decision tree for infrastructure ingestion. It dictates our Standard Operating Procedures (SOPs) based on the exact scenario we inherit. Failing to correctly identify the scenario leads to assuming risk that belongs to the client or inheriting a compromised system that we are now liable for.

#### IF/ELSE Engagement Decision Logic

- **IF** the engagement requires complete control, greenfield architecture, and absolute isolation from legacy debt:
  - **THEN** Execute **Scenario A (New AWS Account)**.
- **ELSE IF** the client mandates the use of their existing AWS Organization due to billing constraints or pre-existing dependencies:
  - **THEN** Execute **Scenario B (Client Provides AWS Account)**. Proceed with extreme caution and mandatory pre-flight audits.
- **ELSE IF** the client refuses cloud access and only provides root/SSH access to a raw compute instance (EC2 or On-Premises):
  - **THEN** Execute **Scenario C (Client Provides Server Only)**.
- **ELSE** the client provides restricted/read-only access but demands production-level deployments:
  - **THEN REJECT.** Operational authority must match operational responsibility.

---

### 2.2 Scenario A: New AWS Account (Greenfield Deployment)

#### What is it?

The generation of a completely virgin AWS Account, usually provisioned via our internal AWS Organizations Management Account or AWS Control Tower.

#### Why prefer this?

This is the gold standard. A greenfield account guarantees zero configuration drift, zero hidden IAM backdoors, and absolute alignment with this playbook from Day 1. There is no technical debt to untangle.

#### When to use:

- For entirely new products or microservices.
- When migrating a client from on-premises to the cloud (lift-and-shift or refactor).
- When building dedicated Staging/Production environments that must be isolated from Dev sandbox accounts.

#### When NOT to use:

- If the new application must tightly couple with a legacy VPC in a different account where Transit Gateway or VPC Peering is strictly prohibited by client compliance (rare, but possible).

#### Real-World Mistakes:

- **The Root User Trap:** Using the Root User email for daily operations. _Impact:_ If the root user MFA is lost, the account is effectively bricked until a lengthy AWS support process is completed.
- **Ignoring the Default VPC:** Failing to delete the default AWS VPC. _Impact:_ Developers will inevitably deploy resources into the default public subnets instead of the Terraform-managed private subnets, leading to security breaches.

#### Step-by-Step Implementation Flow

**Step 1: Account Provisioning**
Provision the account strictly via AWS Organizations (never standalone).

**Step 2: Lock Down the Root Account**

1. Log in via the Root User.
2. Enable a hardware MFA token (YubiKey) immediately.
3. Delete any pre-existing Root Access Keys.
4. Generate an exceptionally long, randomly generated password and store it in the Enterprise Password Vault (e.g., HashiCorp Vault, 1Password).
5. Never log in as Root again.

**Step 3: Establish Identity Federation (IAM Identity Center)**
Do not create IAM Users. Instead, link the account to our central Identity Provider (Okta/Azure AD) via AWS IAM Identity Center.

**Step 4: Nuke the Defaults**
Execute a cleanup script (e.g., using `aws-nuke` in dry-run, or manually via CLI) to destroy the default VPCs in all regions to force the use of IaC.

```bash
# Delete the default VPC in the primary region
aws ec2 delete-vpc --vpc-id $(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
```

**Validation & Troubleshooting:**

- _Validation:_ Attempt to assume a federated role via the CLI: `aws sts get-caller-identity`. Verify the `Arn` reflects an SSO role, not a static IAM user.
- _Troubleshooting:_ If the VPC deletion fails, it is usually because default subnets or internet gateways are still attached. You must script the deletion of subnets, IGWs, and route tables prior to dropping the VPC.

#### Risks and Responsibilities

In Scenario A, **we own 100% of the risk.** If the account is compromised, if the billing spikes, or if data is lost, there is no client legacy debt to blame. The responsibility for securing the perimeter, setting up AWS Budgets, and enabling CloudTrail falls entirely on the DevOps engineer executing the setup.

---

### 2.3 Scenario B: Client Provides AWS Account (Brownfield Engagement)

#### What is it?

The client hands over credentials (or cross-account role assumption) to an AWS Account that already contains infrastructure, users, and history.

#### Why is this dangerous?

Brownfield accounts are minefields. You are inheriting a house built by unknown contractors. There may be hardcoded keys floating around, wild-carded IAM roles, public S3 buckets, and forgotten EC2 instances quietly mining cryptocurrency.

#### When to use:

- When the client demands we integrate our new system with their existing massive databases or legacy monolithic applications housed in that account.
- When the client's corporate governance prohibits vendor-owned AWS accounts.

#### When to REJECT:

- **REJECT** if the client only provides an IAM User with `AdministratorAccess` without MFA enforcement.
- **REJECT** if the client refuses to grant us permission to enable CloudTrail or AWS Config. We cannot be held liable for an environment we cannot audit.
- **REJECT** if the account shows signs of active compromise during the pre-audit (e.g., unrecognized IAM roles created yesterday).

#### Real-World Mistakes:

- **The Silent Inheritance:** Accepting the account without a pre-audit. _Impact:_ A month later, a cryptojacking script on a legacy EC2 instance causes a $20,000 bill. Because we didn't audit the account upon entry, the client assumes _our_ deployment caused the spike and holds us financially liable.
- **Blindly Running IaC:** Running Terraform without running `terraform import` or understanding existing state. _Impact:_ Overwriting or destroying the client's production legacy databases.

#### Pre-Flight Audit & Validation Steps (BEFORE Starting Work)

Before writing any code or deploying resources, the following audit must be completed, documented, and signed off by the client.

**Step 1: IAM Posture Validation**
We must know exactly who and what has access to this environment.

```bash
# 1. List all IAM Users and look for stale or suspicious accounts
aws iam list-users --output table

# 2. Check for Wildcard Administrator Roles (The biggest threat)
aws iam list-roles --query 'Roles[*].[RoleName, Arn]' --output table

# 3. Generate a Credential Report to find keys older than 90 days
aws iam generate-credential-report
# Wait 10 seconds...
aws iam get-credential-report --query 'Content' --output text | base64 -d > credential_audit.csv
```

_Validation:_ Open `credential_audit.csv`. If any `access_key_1_active` is `true` and older than 90 days, mandate that the client rotates it before we commence work.

**Step 2: Billing and Cost Anomaly Check**
Establish a baseline of what the account currently costs.

```bash
# Check current month's estimated charges to establish a baseline
aws ce get-cost-and-usage \
    --time-period Start=$(date -d "$(date +%Y-%m-01)" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics "UnblendedCost"
```

**AWS Console Walkthrough (Cost Explorer):**

1. Go to **AWS Console** → click your account name (top-right) → **Billing and Cost Management**.
2. In the left menu, click **Cost Explorer**.
3. If prompted, click **Enable Cost Explorer** (one-time setup).
4. In the Cost Explorer dashboard:
   - Set **Time range** to **This month**.
   - Set **Granularity** to **Monthly**.
   - Confirm the **Total** line shows current spend.
5. Take a screenshot for your baseline record.

_Validation:_ Take a screenshot of the AWS Cost Explorer. If the account historically spends $500/mo, and we see it is currently tracking at $5,000/mo, halt the engagement and notify client leadership immediately.

**Step 3: Regional Sprawl Audit**
Clients often deploy resources in `us-east-1` but get hacked in `ap-northeast-1` because they never look there.

```bash
# Check for running EC2 instances across ALL regions
for region in $(aws ec2 describe-regions --query "Regions[*].RegionName" --output text); do
  echo "Checking Region: $region"
  aws ec2 describe-instances --region $region --query 'Reservations[*].Instances[*].[InstanceId, State.Name, InstanceType]' --output table
done
```

_Validation:_ If you find massive GPU instances (`p3.8xlarge`) running in a region the client doesn't operate in, assume the account is compromised.

**Troubleshooting Scenario B:**

- _Issue:_ You receive `AccessDeniedException` when running the audit commands.
- _Resolution:_ The client provided insufficient privileges. Reply immediately: _"To assume operational responsibility for this environment, our playbook mandates a full pre-flight security audit. The provided IAM role lacks `SecurityAudit` and `ViewBilling` managed policies. Please attach these immediately so we may proceed."_

---

### 2.4 Scenario C: Client Provides Server Only (The "Black Box" Server)

#### What is it?

The client refuses cloud control-plane access (no AWS console, no AWS CLI). They hand you an IP address and an SSH key (`.pem` file), telling you: _"Deploy the application here."_ This could be a static EC2 instance, a DigitalOcean droplet, or a physical server in their office basement.

#### Why is this the most dangerous scenario?

You are blind. You have no control over the network perimeter (Security Groups, Firewalls). You cannot auto-scale. You cannot utilize managed services (RDS, ElastiCache). If the underlying hardware fails, you cannot simply terminate and replace it via an API. You are inheriting an operating system that has potentially been mutated by dozens of developers over the years.

#### When to use:

- High-security, on-premises environments (Banking, Defense) where cloud adoption is legally restricted.
- When executing a highly localized proof-of-concept (POC) where the application is completely stateless and self-contained (e.g., a single Docker Compose stack).

#### When to REJECT:

- **REJECT** if the client expects "High Availability" or "Zero-Downtime Deployments". You cannot achieve true HA on a single, unmanaged pet server.
- **REJECT** if the client provides SSH access using password authentication instead of key-based authentication.
- **REJECT** if the server is running an End-of-Life (EOL) operating system (e.g., CentOS 6, Ubuntu 14.04).

#### Real-World Mistakes:

- **The Blind Port Bind:** Deploying a Docker container binding to port 8080, only to discover a legacy Jenkins server is already running on that port, crashing your deployment.
- **The Full Disk Catastrophe:** Deploying the application without checking disk space. The application runs for 2 hours, fills up the remaining 500MB of `/var/log`, and completely bricks the server, taking down the client's other legacy apps.

#### Pre-Flight Audit & Validation Steps (SSH Level)

The moment you SSH into the box, you are a forensic investigator. You must map the territory.

**Step 1: System Identification and Integrity**

```bash
# Check OS version and Kernel
uname -a
cat /etc/os-release

# Check system uptime to gauge stability
uptime
```

_Validation:_ If the uptime is 1200 days, the server has missed years of critical kernel security patches (like Meltdown/Spectre). Flag this as a critical security risk to the client.

**Step 2: Resource Availability (Compute & Storage)**
You must verify the server has the capacity to host your workload.

```bash
# Check CPU architecture and core count
lscpu

# Real-time memory and CPU audit
# Look for zombie processes or crypto-miners hogging the CPU
top -b -n 1 | head -n 20

# Check Memory (RAM) availability
free -m

# CRITICAL: Check disk space.
df -h
```

_Validation:_ If `df -h` shows the root partition `/` is at 95% usage, **STOP**. Do not deploy. Tell the client to expand the EBS volume or clean up their logs. Deploying will crash the server.

**Step 3: Network and Port Collision Audit**
You must know what is already listening on the network to avoid port collisions and understand the attack surface.

```bash
# List all active listening TCP/UDP ports and the processes owning them
sudo netstat -tulnp
# Or if netstat is deprecated on the OS:
sudo ss -tulnp
```

_Validation:_ Document every open port. If port 3306 (MySQL) is bound to `0.0.0.0` (open to the world) instead of `127.0.0.1`, flag it immediately. The server is severely vulnerable.

**Step 4: Security and History Check**
Find out what previous developers were doing.

```bash
# Check the shell history for hardcoded passwords or reckless commands
history | grep -i pass
cat ~/.bash_history

# Check who else has SSH keys authorized
cat ~/.ssh/authorized_keys
```

_Validation:_ If there are 15 different keys in `authorized_keys`, this server is a massive security liability. Anyone who ever worked at the company might still have access.

**Troubleshooting Scenario C:**

- _Issue:_ You cannot run `sudo netstat -tulnp` because the user the client provided does not have `sudo` privileges.
- _Resolution:_ Halt the engagement. You cannot deploy production enterprise software (which often requires installing packages, managing systemd services, or binding to lower ports) without root/sudo escalation privileges. Send the rejection notice citing operational impossibility.

## 3. AWS Account & Security (IAM)

### Purpose

Lock down the AWS account and establish secure identity, access, and secret-handling practices.

### What You Will Achieve

- Secure the Root account and enforce MFA
- Build a clear IAM model (users, roles, policies)
- Implement safe credential and secret management

### Step-by-Step Implementation

1. Secure the Root account (Section 3.1).
2. Learn IAM primitives and RBAC patterns (Sections 3.2 and 3.3).
3. Create a break-glass path and MFA setup (Section 3.4).
4. Implement secrets and billing controls (Sections 3.5 and 3.6).

### Commands

CLI commands are provided inline for IAM, MFA, and billing operations.

### Validation

Verify MFA is enabled, IAM policies are attached as intended, and billing alarms are active.

### Common Errors

- Using the Root user for daily work.
- Creating long-lived access keys for humans.

### Pro Tips

- Use IAM Identity Center (SSO) for day-to-day access.
- Keep a documented, audited break-glass path only.

### 3.1 The Root Account: The Keys to the Kingdom

#### What is it?

The AWS Root User is the overarching identity created when an AWS account is first registered. It possesses absolute, immutable administrative privileges over every single resource, billing component, and configuration within the account.

#### Why is it dangerous?

The Root User bypasses all IAM policies. Even if you write an explicit `Deny` policy to block all users from deleting an S3 bucket, the Root User can still delete it. If a malicious actor gains access to the Root User, they effectively own the company. They can delete all backups, lock out all other administrators, and hold the infrastructure hostage.

#### 3.1.1 Enable Root MFA (AWS Console Walkthrough)

1. Log in as the **Root user** at https://console.aws.amazon.com/.
2. Click your **account name** (top-right) → **Security credentials**.
3. Under **Multi-factor authentication (MFA)**, click **Assign MFA device**.
4. Choose **Authenticator app** (recommended) or **Hardware MFA**.
5. Follow the on-screen instructions:

- Scan the QR code with your authenticator app.
- Enter two consecutive 6-digit codes.

6. Click **Assign MFA**.

_Validation:_ Sign out and sign back in as Root. You should be prompted for the MFA code.

_Common Errors:_

- **Code rejected:** The device time is out of sync. Fix the phone time settings and retry.
- **Lost MFA device:** You must open an AWS Support case to recover Root access.

#### When to use:

- **Never** for daily operations.
- You should only log in as Root to:
  1. Change the AWS Support Plan.
  2. Modify payment methods or billing details.
  3. Close the AWS account.
  4. Restore access if all other IAM Administrators have been locked out.

#### When NOT to use:

- For provisioning EC2 instances, reading S3 buckets, or performing any engineering task.

#### Real Security Failure: The Death of a Company

- _The Scenario:_ A prominent cloud-hosting company suffered a devastating attack when a former employee or hacker gained access to their AWS console via a compromised control panel that had access to root-level API capabilities or the Root User itself.
- _The Mistake:_ The attacker demanded a ransom. When the company refused and tried to lock the attacker out, the attacker retaliated by using their absolute privileges to delete the company's entire AWS infrastructure—including all EC2 instances, EBS volumes, S3 buckets, and, crucially, all database backups.
- _The Impact:_ The company literally ceased to exist within 12 hours. They were forced to publicly announce they could not recover their clients' data and shut down operations permanently.

---

### 3.2 Core IAM Architecture: Users vs. Roles vs. Policies

Understanding the distinction between these three primitives is the foundation of AWS security.

#### 3.2.1 IAM Policies (The Rules)

- **What:** JSON documents that explicitly list what actions are allowed or denied on which resources.
- **Deep Explanation:** Policies do not do anything on their own; they must be attached to an identity. In an enterprise, we strictly use _Identity-Based Policies_ (attached to users/roles) and _Resource-Based Policies_ (attached directly to S3 buckets or KMS keys).
- **Good Practice:** `{"Effect": "Allow", "Action": ["s3:GetObject"], "Resource": ["arn:aws:s3:::prod-bucket/data/*"]}`
- **Bad Practice:** `{"Effect": "Allow", "Action": "*", "Resource": "*"}` (The Wildcard Epidemic).

#### 3.2.2 IAM Users (Long-Term Identity)

- **What:** A distinct identity with long-term credentials (a password for the console, and Access Keys for the CLI).
- **When NOT to use:** In a mature enterprise, **IAM Users are heavily discouraged** for human access. Long-term Access Keys live on developer laptops, get accidentally pushed to GitHub, and are frequently stolen by malware.
- **When to use:** Service Accounts for legacy third-party applications that cannot assume a role via OIDC (OpenID Connect).

#### 3.2.3 IAM Roles (Short-Term, Assumable Identity)

- **What:** An identity that has specific permissions, but no long-term credentials (no passwords, no access keys). It relies on temporary, short-lived session tokens generated by AWS STS (Security Token Service).
- **Why this is the Enterprise Standard:** Roles are inherently secure because the credentials expire automatically (e.g., after 1 hour). If an attacker steals a session token, it becomes useless quickly.
- **When to use:**
  - **Human Access:** Developers log into AWS IAM Identity Center (SSO) via Okta/Google, which temporarily assumes a Role based on their AD group.
  - **Machine Access:** An EC2 instance or Lambda function assumes a Role to access an S3 bucket.

---

### 3.3 Role-Based Access Control (RBAC) Design for Teams

When onboarding teams, we must enforce strict separation of duties. Below is the standard Role taxonomy we implement via IAM Identity Center:

1. **Platform/DevOps Admins (`AWSReservedSSO_AdministratorAccess`)**
   - _Scope:_ Full infrastructure provisioning.
   - _Risk Mitigation:_ Mandated hardware MFA (YubiKey) to assume this role. Alerts triggered on all destructive actions (e.g., `ec2:TerminateInstances`).
2. **Backend Developers (`AWSReservedSSO_PowerUserAccess` + Custom Deny boundaries)**
   - _Scope:_ Can create Lambdas, update ECS services, and view CloudWatch logs.
   - _Restrictions:_ Explicitly denied from creating IAM Roles (to prevent privilege escalation), modifying VPC routing, or altering CloudTrail logs.
3. **Data Science / Analysts (`AWSReservedSSO_ReadOnlyAccess`)**
   - _Scope:_ Can view metrics and read data from specific analytical S3 buckets.
   - _Restrictions:_ Zero write access. Cannot modify or spin up expensive SageMaker instances without CI/CD approval.

#### Real-World Mistake: The "DevOps User" Shared Account

- _Bad Practice:_ Creating a single IAM User named `devops-admin` and sharing the password/MFA token among five engineers.
- _Impact:_ When a production database is dropped, CloudTrail shows `devops-admin` did it. You have zero non-repudiation. You cannot prove _which_ engineer made the mistake, making accountability and auditing impossible.

---

### 3.4 Operationalizing IAM: Commands and Validation

While enterprise environments use Terraform/IaC to manage IAM, engineers must understand the underlying APIs. Below is the mechanical flow for creating a break-glass IAM user (if absolutely necessary).

#### 3.4.1 Create an IAM User via AWS Console (Beginner Path)

1. Go to **AWS Console** → search **IAM** → open **IAM**.
2. Left menu → **Users** → **Create user**.
3. **User name:** `breakglass-admin`.
4. Click **Next**.
5. **Permissions options:** Choose **Attach policies directly**.
6. Search and select **ReadOnlyAccess** (start minimal), or **AdministratorAccess** only if you have written approval.
7. Click **Next** → **Create user**.

_Validation:_

- Click the user → **Permissions** tab → confirm the policy is attached.

_Common Errors:_

- **User created without permissions:** Attach the policy after creation.
- **Too much access:** Use least privilege; upgrade only with approval.

#### 3.4.2 Enable MFA for the IAM User (Console Walkthrough)

1. In IAM → **Users**, click `breakglass-admin`.
2. Open the **Security credentials** tab.
3. Under **Multi-factor authentication (MFA)**, click **Assign MFA device**.
4. Choose **Authenticator app** and follow the QR code setup.
5. Enter two consecutive codes → **Assign MFA**.

_Validation:_

- You should see the MFA device listed as **Assigned**.

_Common Errors:_

- **Authentication codes invalid:** Ensure phone time sync is enabled.
- **Device lost:** Remove and reassign MFA using an admin account.

#### Step 1: Create the IAM User

```bash
# Create the user entity
aws iam create-user --user-name breakglass-admin
```

#### Step 2: Attach a Policy

Instead of attaching policies directly to users, always attach them to a Group, and add the user to the Group. For this direct example, attaching a managed policy:

```bash
# Attach the ReadOnlyAccess managed policy for safety initially
aws iam attach-user-policy \
    --user-name breakglass-admin \
    --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

_Validation:_ Run `aws iam list-attached-user-policies --user-name breakglass-admin` to confirm the policy is bound.

#### Step 3: Enable Multi-Factor Authentication (MFA)

AWS CLI does not seamlessly generate the QR code for virtual MFA (like Google Authenticator) natively in a single step without custom scripts to render the seed. However, the workflow is:

1. Create the virtual MFA device.
2. The output provides a `Base32StringSeed`. You manually enter this seed into your Authenticator app.
3. Your app generates two consecutive 6-digit codes.
4. You bind the device to the user using those codes.

```bash
# 1. Create the virtual MFA device and capture the ARN
aws iam create-virtual-mfa-device --virtual-mfa-device-name breakglass-mfa --outfile mfa-qr.png --bootstrap-method QRCodePNG

# 2. Bind the MFA device to the user (Requires two consecutive codes from the app)
# Replace ARN and codes with actual values
aws iam enable-mfa-device \
    --user-name breakglass-admin \
    --serial-number arn:aws:iam::123456789012:mfa/breakglass-mfa \
    --authentication-code1 123456 \
    --authentication-code2 789012
```

---

### 3.5 Secrets Management & Credential Rotation

Hardcoded secrets in configuration files are the fastest way to suffer a breach.

#### Credential Rotation Strategy

- **The 90-Day Rule:** Any long-term IAM Access Key must be rotated every 90 days. AWS Config rules should be enabled to automatically flag keys older than 90 days as `NON_COMPLIANT`.
- **The Flow:**
  1. Generate Key B.
  2. Update the application to use Key B.
  3. Mark Key A as `Inactive`.
  4. Wait 7 days.
  5. Delete Key A.

#### AWS Secrets Manager vs. Parameter Store

- **AWS Systems Manager (SSM) Parameter Store:**
  - _What:_ A key-value store for configuration data (e.g., `API_URL=https://prod.api.com`). Can encrypt strings via KMS using `SecureString`.
  - _When to use:_ Standard environment variables, non-critical tokens, or when budget is extremely tight (Standard parameters are free).
- **AWS Secrets Manager:**
  - _What:_ A purpose-built secrets vault designed to securely store and automatically rotate database credentials.
  - _When to use:_ RDS database passwords, cross-account API tokens.
  - _Why:_ Secrets Manager can run a Lambda function to connect to RDS, change the database password every 30 days, and update the vault seamlessly without any downtime to the application.

---

### 3.6 Billing as a Security Metric

In the cloud, an operational failure or a security breach almost always manifests as a financial anomaly. **Billing alerts are your first line of defense against compromise.**

#### The Threat Vector

If an attacker compromises an IAM key, their primary goal is usually not to steal your data, but to steal your compute power. They will spin up maximum-capacity instances (`p4d.24xlarge` costing $30+/hour) to mine cryptocurrency.

#### Setting up the Anomaly Defense

You must configure AWS Budgets to alert the DevOps team via Slack/Email the moment costs exceed predicted daily run-rates.

```bash
# Example: Create a budget that alerts if costs exceed $1000
aws budgets create-budget \
    --account-id 123456789012 \
    --budget file://budget.json \
    --notifications-with-subscribers file://notifications.json
```

_(Note: The JSON payloads define the threshold limits and SNS topics for alerting)._

- **Good Practice:** Set up a "Zero Spend" budget on development accounts. If a dev account spends more than $5 in a day, an alert fires.
- **Bad Practice:** Waiting for the monthly invoice to review costs. By the end of the month, a compromised account could have accumulated hundreds of thousands of dollars in illicit charges.

## 4. AWS Networking (VPC, Subnets, Routing)

### Purpose

Build a secure, multi-AZ VPC foundation with public, private, and isolated tiers.

### What You Will Achieve

- A production-ready VPC with correct routing
- Public, private, and isolated subnets across two AZs
- NAT Gateway for private egress and IGW for public ingress

### Step-by-Step Implementation

1. Create the VPC and subnets (Sections 4.3 Steps 1-7).
2. Attach IGW and configure route tables (Steps 3-5).
3. Create NAT Gateway and private routes (Steps 8-9).
4. Validate connectivity (Section 4.5).

### Commands

All CLI commands are provided in Section 4.3. A full AWS Console workflow is in Step 10.

### Validation

Run the public and private subnet connectivity tests in Section 4.5.

### Common Errors

- NAT Gateway created in a private subnet.
- Private subnets pointing to IGW instead of NAT.

### Pro Tips

- Use S3/DynamoDB VPC Endpoints to reduce NAT costs.
- Tag all networking resources to avoid confusion later.

### 4.1 The Virtual Private Cloud (VPC): The Internal Mechanics

#### What is it?

A Virtual Private Cloud (VPC) is your logically isolated slice of the AWS public cloud. Internally, a VPC is not a physical network or a VLAN. AWS utilizes a proprietary Software-Defined Networking (SDN) protocol built on top of hardware acceleration cards (the AWS Nitro System) called the Mapping Service. When an EC2 instance sends a packet, the Nitro hypervisor intercepts it, queries the Mapping Service to find the destination MAC/IP across the global AWS backbone, encapsulates the packet, and routes it directly.

#### Why is this understanding critical?

Because a VPC is entirely software-defined, certain traditional networking paradigms (like ARP spoofing, promiscuous mode, or broadcast packets) **do not work in AWS**. You cannot run a standard Layer 2 firewall or a gratuitous ARP failover script because AWS drops broadcast and multicast traffic at the hypervisor level.

#### When to use:

- Every single infrastructure deployment. The "EC2 Classic" (non-VPC) network was retired years ago. All modern AWS compute lives inside a VPC.

#### Traffic Flow Inside AWS

The traffic flow is deterministic and heavily policed. When a packet leaves an instance:

1. It hits the **Security Group** (Stateful, instance-level firewall).
2. It hits the **Network ACL** (NACL - Stateless, subnet-level firewall).
3. It hits the **Subnet Route Table**, which determines if the packet goes to:
   - The `local` VPC router (to talk to another instance in the same VPC).
   - An Internet Gateway (IGW) for public internet access.
   - A NAT Gateway (to masquerade a private IP to a public IP).
   - A Transit Gateway or VPC Peering connection (to talk to another VPC).
   - A VPC Endpoint (to talk securely to AWS services like S3 or DynamoDB without internet exposure).

---

### 4.2 The Enterprise Multi-Tier Architecture

In production, we strictly enforce a **3-Tier Subnet Architecture** across at least two Availability Zones (AZs) for High Availability (HA).

1. **Public Tier (DMZ):**
   - _What lives here:_ Application Load Balancers (ALB), NAT Gateways, Bastion Hosts.
   - _Routing:_ Route `0.0.0.0/0` points directly to the Internet Gateway (IGW).
2. **Private Application Tier:**
   - _What lives here:_ EKS worker nodes, ECS Fargate containers, backend EC2 web servers, Lambda functions.
   - _Routing:_ Route `0.0.0.0/0` points to the NAT Gateway living in the Public Subnet. These instances can reach the internet (to download patches/APIs) but the internet cannot initiate a connection to them.
3. **Isolated Database Tier:**
   - _What lives here:_ RDS Databases, Amazon Aurora clusters, ElastiCache (Redis).
   - _Routing:_ Strictly no route to `0.0.0.0/0`. These subnets cannot reach the internet, and the internet cannot reach them. They only have the default `local` route.

#### Public vs. Private Subnet (The Golden Rule)

- **A subnet is Public ONLY IF its route table has a route to an Internet Gateway (IGW).**
- If an instance has a Public IP address assigned, but its subnet does not have an IGW route, it still cannot access the internet. Both conditions must be met.
- **NAT Gateway:** A managed service that allows instances in a private subnet to initiate outbound IPv4 traffic to the internet, while blocking inbound traffic initiated by the internet. It acts as a highly available proxy.

---

### 4.3 Operationalizing the Network: CLI Implementation

Below is the step-by-step mechanical execution of building a VPC from scratch. In enterprise environments, this is executed via Terraform or AWS CDK, but understanding the underlying API calls is mandatory for debugging.

#### Step 1: Create the VPC

```bash
# Create a VPC with a /16 CIDR block (providing 65,536 IPs)
aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'

# Output provides the VpcId (e.g., vpc-0abcd1234)
```

#### Step 2: Create Subnets (Public and Private)

```bash
# Create a Public Subnet in Availability Zone A (/24 provides 256 IPs)
aws ec2 create-subnet \
    --vpc-id vpc-0abcd1234 \
    --cidr-block 10.0.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-public-1a}]'

# Create a Private Subnet in Availability Zone A
aws ec2 create-subnet \
    --vpc-id vpc-0abcd1234 \
    --cidr-block 10.0.10.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-1a}]'
```

#### Step 3: Create and Attach the Internet Gateway (IGW)

```bash
# Create the IGW
aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]'
# Output provides IgwId (e.g., igw-0wxyz9876)

# Attach the IGW to the VPC
aws ec2 attach-internet-gateway \
    --vpc-id vpc-0abcd1234 \
    --internet-gateway-id igw-0wxyz9876
```

#### Step 4: Create Route Tables and Configure Routing

Every VPC comes with a Main Route Table, but we explicitly create custom route tables for granular control.

```bash
# Create a Custom Public Route Table
aws ec2 create-route-table \
    --vpc-id vpc-0abcd1234 \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-public-rt}]'
# Output provides RouteTableId (e.g., rtb-11112222)

# Add the default route (0.0.0.0/0) pointing to the IGW
aws ec2 create-route \
    --route-table-id rtb-11112222 \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id igw-0wxyz9876
```

#### Step 5: Associate Subnets to Route Tables

A subnet is only "Public" once this association is made.

```bash
# Associate the Public Subnet with the Public Route Table
aws ec2 associate-route-table \
    --subnet-id subnet-public123 \
    --route-table-id rtb-11112222
```

#### Step 6: Create the AZ-B Subnets (High Availability)

> **Why?** A production VPC requires subnets in at least two Availability Zones. If AZ-A suffers an outage, your application continues running in AZ-B. Without multi-AZ subnets, you cannot create an ALB, RDS Multi-AZ, or a proper Auto Scaling Group.

```bash
# Public Subnet in AZ-B
aws ec2 create-subnet \
    --vpc-id vpc-0abcd1234 \
    --cidr-block 10.0.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-public-1b}]'

# Private Application Subnet in AZ-B
aws ec2 create-subnet \
    --vpc-id vpc-0abcd1234 \
    --cidr-block 10.0.11.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-1b}]'

# Associate the AZ-B Public Subnet with the Public Route Table
aws ec2 associate-route-table \
    --subnet-id subnet-public-1b \
    --route-table-id rtb-11112222
```

#### Step 7: Create Isolated Database Subnets

> **Why?** The database tier must be completely isolated — no internet access whatsoever. These subnets only have the default `local` route (communication within the VPC). This is where RDS, Aurora, ElastiCache, and other data stores live.

```bash
# Isolated Database Subnet in AZ-A
aws ec2 create-subnet \
    --vpc-id vpc-0abcd1234 \
    --cidr-block 10.0.20.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-isolated-1a}]'

# Isolated Database Subnet in AZ-B
aws ec2 create-subnet \
    --vpc-id vpc-0abcd1234 \
    --cidr-block 10.0.21.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-isolated-1b}]'
```

_These subnets intentionally have NO route to 0.0.0.0/0. They use only the default VPC `local` route, making them truly isolated._

#### Step 8: Create the NAT Gateway (For Private Subnet Internet Access)

> **What is a NAT Gateway?** A managed AWS service that allows instances in **private** subnets to reach the internet (to download packages, call external APIs) without being directly reachable from the internet. It translates the private IP to a public Elastic IP on outbound connections.

> **Why do you need it?** Without a NAT Gateway, your application servers in the private subnet cannot run `apt update`, `npm install`, call third-party APIs, or download anything from the internet. They are completely internet-blind.

> ⚠️ Cost Warning: NAT Gateways are billed per hour and per GB of data processed. A lightly used dev NAT still costs money, and high-throughput workloads can generate large data charges. Use VPC Endpoints for S3/DynamoDB and avoid NAT in low-traffic environments.

```bash
# Step 8a: Allocate an Elastic IP for the NAT Gateway
# The NAT Gateway needs a fixed public IP address.
aws ec2 allocate-address \
    --domain vpc \
    --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=prod-nat-eip}]'
# Output provides AllocationId (e.g., eipalloc-0abcdef1234567890)

# Step 8b: Create the NAT Gateway in the PUBLIC subnet
# CRITICAL: The NAT Gateway itself must live in a PUBLIC subnet (with IGW route),
# even though it serves PRIVATE subnets. This is the most common mistake.
aws ec2 create-nat-gateway \
    --subnet-id subnet-public123 \
    --allocation-id eipalloc-0abcdef1234567890 \
    --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-nat-gw}]'
# Output provides NatGatewayId (e.g., nat-0abcdef1234567890)
# Wait 2-3 minutes for the NAT Gateway to become "available"
```

_Validation:_

```bash
# Wait for the NAT Gateway to become available
aws ec2 describe-nat-gateways \
    --nat-gateway-ids nat-0abcdef1234567890 \
    --query 'NatGateways[0].[State, NatGatewayAddresses[0].PublicIp]' \
    --output table
# Expected: available | 54.xxx.xxx.xxx
```

#### Step 9: Create Private Route Table (Route to NAT Gateway)

```bash
# Create a Custom Private Route Table
aws ec2 create-route-table \
    --vpc-id vpc-0abcd1234 \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-private-rt}]'
# Output provides RouteTableId (e.g., rtb-33334444)

# Add the default route pointing to the NAT Gateway (NOT the IGW)
aws ec2 create-route \
    --route-table-id rtb-33334444 \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id nat-0abcdef1234567890

# Associate BOTH private subnets with this route table
aws ec2 associate-route-table \
    --subnet-id subnet-private-1a \
    --route-table-id rtb-33334444

aws ec2 associate-route-table \
    --subnet-id subnet-private-1b \
    --route-table-id rtb-33334444
```

_Note: The Isolated Database subnets are intentionally NOT associated with any custom route table. They use the VPC's default Main Route Table, which only has the `local` route — no internet access._

#### Step 10: Create the VPC via AWS Console (Visual Alternative)

> For readers who prefer the visual approach, here is the Console walkthrough:

1. Go to **AWS Console** → Search **"VPC"** → Click **"VPC"**.
2. Click **"Create VPC"** in the top-right.
3. Select **"VPC and more"** (this creates VPC + subnets + route tables + NAT Gateway in one step).
4. Configure:
   - **Name tag auto-generation:** `prod`
   - **IPv4 CIDR block:** `10.0.0.0/16`
   - **Number of Availability Zones:** `2`
   - **Number of public subnets:** `2`
   - **Number of private subnets:** `2`
   - **NAT gateways:** `In 1 AZ` (for cost savings) or `1 per AZ` (for production HA)
   - **VPC endpoints:** `S3 Gateway` (select this — it is free and avoids NAT charges for S3 traffic)
5. Click **"Create VPC"** — AWS provisions everything automatically.
6. Verify in the VPC dashboard that all subnets, route tables, IGW, and NAT Gateway are created.

#### Complete VPC Architecture Summary

After completing Steps 1-9, your network topology looks like this:

```
┌───────────────────────────────────────────────────────────────┐
│                        VPC: 10.0.0.0/16                       │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐    │
│  │   PUBLIC TIER (DMZ)      │  │   PUBLIC TIER (DMZ)      │    │
│  │   10.0.1.0/24 (AZ-A)    │  │   10.0.2.0/24 (AZ-B)    │    │
│  │   [ALB] [NAT Gateway]   │  │   [ALB]                  │    │
│  │   Route → IGW            │  │   Route → IGW            │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐    │
│  │  PRIVATE APP TIER        │  │  PRIVATE APP TIER        │    │
│  │  10.0.10.0/24 (AZ-A)    │  │  10.0.11.0/24 (AZ-B)    │    │
│  │  [EC2] [Node.js] [PM2]  │  │  [EC2] [Node.js] [PM2]  │    │
│  │  Route → NAT Gateway    │  │  Route → NAT Gateway    │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐    │
│  │  ISOLATED DB TIER        │  │  ISOLATED DB TIER        │    │
│  │  10.0.20.0/24 (AZ-A)    │  │  10.0.21.0/24 (AZ-B)    │    │
│  │  [RDS] [ElastiCache]    │  │  [RDS Standby]           │    │
│  │  Route → local ONLY     │  │  Route → local ONLY     │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

```mermaid
flowchart TB
  subgraph VPC["VPC 10.0.0.0/16"]
    subgraph PublicA["Public Subnet AZ-A 10.0.1.0/24"]
      ALB1[ALB]
      NAT1[NAT Gateway]
    end
    subgraph PublicB["Public Subnet AZ-B 10.0.2.0/24"]
      ALB2[ALB]
    end
    subgraph PrivateA["Private App Subnet AZ-A 10.0.10.0/24"]
      AppA[App EC2]
    end
    subgraph PrivateB["Private App Subnet AZ-B 10.0.11.0/24"]
      AppB[App EC2]
    end
    subgraph IsolatedA["Isolated DB Subnet AZ-A 10.0.20.0/24"]
      RDSA[(RDS Primary)]
    end
    subgraph IsolatedB["Isolated DB Subnet AZ-B 10.0.21.0/24"]
      RDSB[(RDS Standby)]
    end
  end
  Internet((Internet)) --> ALB1
  Internet --> ALB2
  AppA --> NAT1
  AppB --> NAT1
  AppA --> RDSA
  AppB --> RDSA
```

#### ⚠️ Cost Warning: NAT Gateway

NAT Gateways charge **$0.045/hour** (~$32/month) plus **$0.045/GB** of data processed. For high-bandwidth workloads (e.g., pulling Docker images, downloading large datasets from S3), costs can spike significantly. Use **VPC Endpoints** for S3 and DynamoDB (free Gateway Endpoints) to bypass NAT charges. See Section 12 for VPC Endpoint setup.

---

### 4.4 Real-World Architectural Mistakes

#### Mistake 1: The Database in the Public Subnet

- _What happens:_ A fresher spins up an RDS PostgreSQL database and places it in the public subnet to make it "easier" to connect to from DBeaver on their local laptop. They assign it a public IP and open port 5432 to `0.0.0.0/0`.
- _The Impact:_ Within 45 minutes, automated botnets will scan the IP, brute-force the default `postgres` password, and drop all tables, leaving a ransom note.
- _The Mandate:_ Databases **must never** have public IP addresses or sit in public subnets. Developers must use AWS Systems Manager (SSM) Session Manager to tunnel into a private bastion host to access the database.

#### Mistake 2: IP Address Exhaustion (CIDR Overlap)

- _What happens:_ An engineer creates a new VPC using the default `172.31.0.0/16` CIDR block. Six months later, the company acquires a startup and needs to peer the new VPC with the startup's VPC to share internal APIs. The startup also used `172.31.0.0/16`.
- _The Impact:_ VPC Peering and Transit Gateways absolutely cannot route traffic between overlapping IP ranges. The entire infrastructure must be destroyed and rebuilt with non-overlapping IPs, causing massive downtime.
- _The Mandate:_ Always consult the Enterprise IP Address Management (IPAM) registry before selecting a CIDR block. Never use default CIDR blocks.

---

### 4.5 Validation & Troubleshooting

Once infrastructure is provisioned, you must validate network flows using strict diagnostic procedures. Do not guess; prove it works.

#### Test 1: Public Subnet Internet Connectivity

From an EC2 instance launched in the **Public Subnet** (with an assigned Public IP):

```bash
# Test DNS resolution and HTTP/HTTPS egress
curl -I https://www.google.com

# Expected Output: HTTP/2 200

# Ping an external IP to test raw ICMP egress
ping -c 4 8.8.8.8
```

_Troubleshooting if it fails:_

1. Check the Route Table: Does `0.0.0.0/0` point to the `igw-xxx`?
2. Check the Security Group: Is Outbound traffic allowed on port 443 (for curl) and ICMP (for ping)?
3. Does the instance actually have a Public IPv4 address assigned?

#### Test 2: Private Subnet NAT Translation

From an EC2 instance launched in the **Private Application Subnet**:

```bash
# Test connection out to the internet
curl -s http://checkip.amazonaws.com
```

_Validation:_ The IP address returned by this command must be the **Elastic IP attached to your NAT Gateway**, not the private instance's IP. If it returns the NAT IP, your routing is successful.

_Troubleshooting if it hangs:_

1. Did you attach the NAT Gateway to the _Public_ subnet? (Common mistake: placing the NAT gateway in the private subnet).
2. Does the Private Subnet Route Table have a route for `0.0.0.0/0` pointing to `nat-xxx`?
3. Is the NACL blocking ephemeral return ports (1024-65535)?

## 5. EC2 Setup & Compute Strategy

### Purpose

Choose the right EC2 resources and provision instances safely for production workloads.

### What You Will Achieve

- Select an AMI and instance type that fits your workload
- Launch EC2 in the correct subnets with secure access
- Install Node.js and Nginx and harden SSH

### Step-by-Step Implementation

1. Understand instance types and AMI strategy (Sections 5.2-5.3).
2. Create Security Groups (Section 5.4).
3. Launch EC2 and connect securely (Section 5.5).
4. Install runtime dependencies and Nginx (Section 5.5).
5. Harden SSH if required (Section 5.6).

### Commands

All CLI commands are provided inline under each step.

### Validation

Confirm the instance is running, reachable, and serving Nginx locally.

### Common Errors

- Launching in the wrong subnet (public vs. private).
- Using an AMI from another region (IDs are region-specific).

### Pro Tips

- Prefer SSM Session Manager to avoid opening SSH.
- Use Golden AMIs for repeatable, secure builds.

### 5.1 What is EC2?

Amazon Elastic Compute Cloud (EC2) is the foundational compute primitive of AWS. At its core, EC2 provides resizable virtual machines (referred to as "instances") running on the AWS Nitro hypervisor. Each instance is allocated a deterministic slice of the underlying physical host's CPU, memory, storage, and network bandwidth. You pay for the compute capacity you provision, billed per-second (with a one-minute minimum) for Linux instances.

#### Why does EC2 still matter?

Despite the rapid rise of serverless (Lambda) and container orchestration (EKS/ECS Fargate), EC2 remains the backbone of enterprise computing for workloads that require:

- Full operating system control (kernel tuning, custom drivers, GPU passthrough).
- Persistent, long-running processes that exceed Lambda's 15-minute execution limit.
- Legacy application hosting where containerization is not yet feasible.
- High-performance computing (HPC) clusters requiring bare-metal or placement group affinity.

#### When to use EC2:

- Running self-managed databases where RDS managed service limitations are unacceptable (e.g., Oracle RAC, custom PostgreSQL extensions like Citus).
- Hosting long-running worker processes or queue consumers that must maintain persistent TCP connections.
- GPU-accelerated machine learning training jobs on `p4d` or `g5` instance families.
- Legacy monolithic applications (Java EARs, .NET Framework apps on Windows Server) that cannot be refactored into containers within the project timeline.

#### When NOT to use EC2:

- **Stateless HTTP APIs:** If your microservice is a stateless REST API that responds to HTTP requests and scales horizontally, **use ECS Fargate or Lambda**. EC2 forces you to manage OS patching, AMI updates, and auto-scaling group lifecycle hooks—all of which Fargate and Lambda abstract away entirely.
- **Batch Jobs Under 15 Minutes:** If your workload is a short-lived data transformation, image resize, or webhook handler, **use AWS Lambda**. You pay nothing when the function is idle. An EC2 instance costs money 24/7 whether it is processing requests or sitting completely idle.
- **Simple Static Websites:** If you are serving static HTML/CSS/JS, **use S3 + CloudFront**. Provisioning an entire EC2 instance to serve static files is the equivalent of renting a warehouse to store a single shoebox.

---

### 5.2 Instance Type Taxonomy: Choosing the Right Metal

AWS offers hundreds of instance types. Choosing the wrong one is a direct path to either wasting money or suffering catastrophic performance degradation under load. Below is the enterprise decision matrix for the most commonly deployed families.

#### 5.2.1 T-Family (Burstable Performance)

- **What:** `t2`, `t3`, `t3a` instances provide a baseline CPU performance with the ability to "burst" above the baseline using CPU Credits. When idle, the instance accumulates credits. When under load, it spends them.
- **Why it exists:** Most web applications spend 95% of their time idle or under minimal load (serving cached pages, waiting for database queries). Paying for a dedicated 8-core CPU that sits at 5% utilization is wasteful.
- **When to use:**
  - Development and staging environments.
  - Low-traffic internal tools (admin dashboards, CI/CD runners during off-peak).
  - Bastion/Jump hosts that are accessed sporadically.
- **When NOT to use:**
  - **Production workloads with sustained CPU usage.** If your application consistently runs above 30% CPU for extended periods, the instance will exhaust its CPU credits, and performance will be throttled to the baseline (often 20-40% of a full core). Your API latency will spike from 50ms to 2000ms+ with zero warning in application logs. The only indicator is the `CPUCreditBalance` CloudWatch metric dropping to zero.
- **Real-World Mistake:** A startup launches their production Node.js API on a `t3.medium`. During a marketing campaign, traffic spikes 10x. The instance burns through all CPU credits in 4 minutes. The API becomes completely unresponsive. The on-call engineer sees low CPU usage in `top` (because it is being _throttled_, not because it is idle) and misdiagnoses the issue as a database problem, wasting 2 hours.

| Instance    | vCPUs | RAM (GiB) | Baseline CPU | Best For                       |
| ----------- | ----- | --------- | ------------ | ------------------------------ |
| `t3.micro`  | 2     | 1         | 10%          | Bastion hosts, dev sandboxes   |
| `t3.small`  | 2     | 2         | 20%          | Low-traffic internal tools     |
| `t3.medium` | 2     | 4         | 20%          | Staging APIs, light CI runners |
| `t3.large`  | 2     | 8         | 30%          | Staging databases, medium apps |

#### 5.2.2 M-Family (General Purpose)

- **What:** `m5`, `m6i`, `m7g` instances provide a balanced ratio of compute, memory, and networking. No burstable behavior—you get the full CPU allocation at all times.
- **When to use:**
  - **Production application servers** handling sustained, predictable traffic.
  - Backend microservices, middleware, and message broker hosts (RabbitMQ, Kafka).
- **When NOT to use:**
  - Workloads that are heavily CPU-bound (use C-family) or heavily memory-bound (use R-family).

| Instance     | vCPUs | RAM (GiB) | Best For                            |
| ------------ | ----- | --------- | ----------------------------------- |
| `m5.large`   | 2     | 8         | Prod APIs, small Kafka brokers      |
| `m5.xlarge`  | 4     | 16        | Multi-threaded app servers, Jenkins |
| `m5.2xlarge` | 8     | 32        | Medium databases, ELK stack nodes   |

#### 5.2.3 C-Family (Compute Optimized)

- **What:** `c5`, `c6i`, `c7g` instances provide the highest ratio of vCPU performance to cost. They use the latest generation Intel/AMD/Graviton processors.
- **When to use:**
  - Video encoding/transcoding pipelines.
  - Scientific modeling and simulation.
  - High-performance web servers handling extreme request concurrency (e.g., 50,000+ RPS).
  - Machine learning inference (not training—use P-family for training).
- **When NOT to use:**
  - Applications that are memory-hungry but CPU-light (e.g., in-memory caches, large JVM heaps). You will be paying a premium for CPU cores you do not need.

| Instance     | vCPUs | RAM (GiB) | Best For                           |
| ------------ | ----- | --------- | ---------------------------------- |
| `c5.large`   | 2     | 4         | Lightweight batch processing       |
| `c5.xlarge`  | 4     | 8         | Video encoding, CI/CD build agents |
| `c5.2xlarge` | 8     | 16        | High-concurrency API gateways      |

---

### 5.3 AMI Selection: The Operating System Foundation

#### What is an AMI?

An Amazon Machine Image (AMI) is a pre-configured, immutable snapshot of an operating system volume. It is the template from which every EC2 instance is born.

#### The Enterprise AMI Strategy

1. **Start with the Official AWS AMI:** Always begin from an AWS-published, hardened base image (e.g., `Amazon Linux 2023`, `Ubuntu 24.04 LTS`). Never use community or marketplace AMIs from unknown publishers in production—they may contain backdoors or crypto-mining software.
2. **Bake Golden AMIs:** Use a tool like HashiCorp Packer to build a "Golden AMI" pipeline. This pipeline takes the base AWS AMI, installs all required packages (Node.js, Nginx, monitoring agents), hardens the SSH configuration, and outputs a new, versioned AMI. All production instances are launched from this Golden AMI.
3. **Never Patch Live Instances:** If a security patch is released, do not SSH into 50 running instances and run `apt upgrade`. Instead, rebuild the Golden AMI with the patch, update the Auto Scaling Group Launch Template to reference the new AMI ID, and perform a rolling replacement of all instances.

#### When to use Amazon Linux 2023 vs. Ubuntu:

- **Amazon Linux 2023:** Tightly integrated with AWS services (`aws-cli` pre-installed, optimized kernel for Nitro). Best for pure AWS workloads. Uses `dnf` package manager.
- **Ubuntu 24.04 LTS:** Larger community ecosystem, wider third-party software compatibility. Best when the team has deep Debian/Ubuntu expertise. Uses `apt` package manager.

---

### 5.4 Creating Security Groups (Before Launching EC2)

> **Why is this here?** The EC2 launch command in the next step requires a Security Group ID (`sg-xxx`). You must create Security Groups first. These are the virtual firewalls that control which traffic can reach your instance.

Security Groups are **stateful** firewalls. Inbound rules allow traffic to the instance; outbound rules allow traffic from the instance. By default, inbound is denied and outbound is allowed. You must explicitly open only the ports you need.

#### Minimal Public Web Security Group (Beginner)

This is the smallest safe rule set for a single public web server: SSH from your IP only, and HTTP/HTTPS from the internet.

```bash
# Create the security group
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Public web server SG" \
  --vpc-id vpc-0abcd1234

# SSH (22) from your public IP only
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 22 \
  --cidr YOUR_PUBLIC_IP/32

# HTTP (80) from the internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# HTTPS (443) from the internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

#### Step-by-Step: Create the Application Security Group (CLI)

```bash
# Create the Security Group for application servers
aws ec2 create-security-group \
    --group-name prod-app-sg \
    --description "Security group for production application servers" \
    --vpc-id vpc-0abcd1234 \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=prod-app-sg}]'
# Output provides GroupId (e.g., sg-app-0abc1234)

# Allow SSH from your IP only (for initial setup)
# Replace YOUR_PUBLIC_IP with your actual IP (find it at https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
    --group-id sg-app-0abc1234 \
    --protocol tcp \
    --port 22 \
    --cidr YOUR_PUBLIC_IP/32

# Allow HTTP traffic from the ALB Security Group (or temporarily from anywhere)
aws ec2 authorize-security-group-ingress \
    --group-id sg-app-0abc1234 \
    --protocol tcp \
    --port 3000 \
    --cidr 10.0.0.0/16

# Allow HTTP on port 80 (Nginx)
aws ec2 authorize-security-group-ingress \
    --group-id sg-app-0abc1234 \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

# Allow HTTPS on port 443 (TLS)
aws ec2 authorize-security-group-ingress \
  --group-id sg-app-0abc1234 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

#### Create the Database Security Group

```bash
aws ec2 create-security-group \
    --group-name prod-db-sg \
    --description "Security group for production databases" \
    --vpc-id vpc-0abcd1234 \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=prod-db-sg}]'
# Output provides GroupId (e.g., sg-db-0xyz5678)

# Allow PostgreSQL ONLY from the application Security Group (SG chaining)
aws ec2 authorize-security-group-ingress \
    --group-id sg-db-0xyz5678 \
    --protocol tcp \
    --port 5432 \
    --source-group sg-app-0abc1234
```

_Validation:_

```bash
# List all Security Groups in the VPC
aws ec2 describe-security-groups \
    --filters "Name=vpc-id,Values=vpc-0abcd1234" \
    --query 'SecurityGroups[*].[GroupId, GroupName, Description]' \
    --output table

# Verify the inbound rules for the app SG
aws ec2 describe-security-groups \
    --group-ids sg-app-0abc1234 \
    --query 'SecurityGroups[0].IpPermissions' \
    --output json
```

#### Step-by-Step: Create Security Group via AWS Console

1. Go to **AWS Console** → Search **"EC2"** → Click **EC2**.
2. In the left sidebar, scroll to **"Network & Security"** → Click **"Security Groups"**.
3. Click **"Create security group"**.
4. **Security group name:** `prod-app-sg`
5. **Description:** `Security group for production application servers`
6. **VPC:** Select your `prod-vpc`
7. **Inbound rules:** Click **"Add rule"**:
   - **Rule 1:** Type: `SSH`, Source: `My IP` (auto-fills your IP)
   - **Rule 2:** Type: `HTTP`, Source: `Anywhere-IPv4` (`0.0.0.0/0`)
  - **Rule 3:** Type: `HTTPS`, Source: `Anywhere-IPv4` (`0.0.0.0/0`)
  - **Rule 4:** Type: `Custom TCP`, Port: `3000`, Source: `10.0.0.0/16`
8. **Outbound rules:** Leave as default (Allow all).
9. Click **"Create security group"**.

#### ⚠️ Real-World Warning

Opening SSH (port 22) to `0.0.0.0/0` means **anyone on the internet** can attempt to brute-force your server. Always restrict SSH to your specific IP (`YOUR_IP/32`) or, better yet, disable SSH entirely and use SSM Session Manager (see Section 5.5 Step 2).

---

### 5.5 Operationalizing EC2: Step-by-Step Deployment

#### Step 0: Create the IAM Role for SSM (Required for Session Manager)

This role allows the instance to register with AWS Systems Manager so you can access it without opening port 22.

**AWS Console Walkthrough:**

1. Go to **AWS Console** → search **IAM** → open **IAM**.
2. Left menu → **Roles** → **Create role**.
3. **Trusted entity type:** AWS service.
4. **Use case:** **EC2** → **Next**.
5. Attach the policy **AmazonSSMManagedInstanceCore**.
6. (Optional) Attach **CloudWatchAgentServerPolicy** if you plan to ship logs/metrics.
7. **Role name:** `EC2-SSM-Role` → **Create role**.

**CLI Alternative:**

```bash
# Create the trust policy
cat > /tmp/ssm-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role and attach the managed policy
aws iam create-role --role-name EC2-SSM-Role --assume-role-policy-document file:///tmp/ssm-trust.json
aws iam attach-role-policy --role-name EC2-SSM-Role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create an instance profile and add the role
aws iam create-instance-profile --instance-profile-name EC2-SSM-Profile
aws iam add-role-to-instance-profile --instance-profile-name EC2-SSM-Profile --role-name EC2-SSM-Role
```

_Validation:_

- In IAM → Roles, confirm `EC2-SSM-Role` exists and has **AmazonSSMManagedInstanceCore**.

_Common Errors:_

- **SSM session fails:** The instance profile was not attached at launch.
- **SSM shows "Not Managed":** SSM agent is not installed or not running.

#### Step 1: Launch the EC2 Instance (CLI)

```bash
# Launch a t3.medium instance in a private subnet with the Golden AMI
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --instance-type t3.medium \
    --key-name prod-ssh-keypair \
    --subnet-id subnet-private123 \
    --security-group-ids sg-0abc1234def56789 \
    --iam-instance-profile Name=EC2-SSM-Role \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=api-server-01},{Key=Environment,Value=Production},{Key=Service,Value=AuthAPI}]' \
    --count 1
```

> **How to find the AMI ID:** The `--image-id` value is region-specific. To find the latest Amazon Linux 2023 AMI:

```bash
aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=al2023-ami-2023*-x86_64" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].[ImageId, Name]' \
    --output text
# Example output: ami-0abcdef1234567890  al2023-ami-2023.6.20260501.0-kernel-6.1-x86_64
```

#### Launch EC2 via AWS Console (Visual Alternative)

1. Go to **AWS Console** → Search **"EC2"** → Click **EC2**.
2. Click **"Launch instance"** (orange button, top-right).
3. **Name:** `api-server-01`
4. **Application and OS Images (AMI):**
   - Click **"Amazon Linux"** → Select **"Amazon Linux 2023 AMI"** (Free tier eligible).
   - _Why Amazon Linux?_ Pre-installed AWS CLI, optimized for Nitro, `dnf` package manager.
5. **Instance type:** Select **`t3.medium`** (2 vCPUs, 4 GiB RAM).
   - For learning/Free Tier: Use `t3.micro` (Free Tier eligible).
6. **Key pair:** Select **`prod-ssh-keypair`** (created in Section 0.4).
   - If you haven't created one, click **"Create new key pair"** here.
7. **Network settings:** Click **"Edit"**:
   - **VPC:** Select `prod-vpc`
   - **Subnet:** Select `prod-private-1a` (your private application subnet)
   - **Auto-assign public IP:** **Disable** (private subnet — no public IP needed)
   - **Firewall (Security groups):** Select **"Select existing security group"** → Choose `prod-app-sg`
8. **Configure storage:**
   - **30 GiB, gp3** (General Purpose SSD). The default 8 GiB is too small for most applications.
9. **Advanced details** (expand this section):
   - **IAM instance profile:** Select your SSM role (e.g., `EC2-SSM-Role`) to enable Session Manager.
10. Click **"Launch instance"**.
11. Wait 30-60 seconds, then click the Instance ID link to view your running instance.

_Validation:_

```bash
# Confirm the instance is running
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=api-server-01" \
    --query 'Reservations[*].Instances[*].[InstanceId, State.Name, PrivateIpAddress]' \
    --output table
```

#### Step 2: SSH into the Instance

For instances in a **Private Subnet** (which is the standard for all application servers), you cannot SSH directly from the public internet. Use AWS Systems Manager Session Manager (preferred) or an SSH tunnel through a bastion host.

```bash
# PREFERRED: Use SSM Session Manager (no SSH key management, no open ports)
aws ssm start-session --target i-0abc1234def56789

# ALTERNATIVE: SSH via Bastion Host (if SSM is not configured)
# First, SSH to the Bastion in the Public Subnet
ssh -i prod-bastion-key.pem ec2-user@bastion-public-ip

# Then, from the Bastion, SSH to the private instance
ssh -i prod-app-key.pem ec2-user@10.0.10.45
```

_Validation:_

```bash
# Once connected, verify you are on the correct machine
hostname
uname -a
curl -s http://169.254.169.254/latest/meta-data/instance-id
```

#### Step 2.5: Update the OS and Install Base Packages

Keep the instance patched and install baseline tools used for deployments.

**Amazon Linux 2023:**

```bash
sudo dnf update -y
sudo dnf install -y git curl unzip jq
```

**Ubuntu 22.04/24.04:**

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install -y git curl unzip jq
```

_Validation:_

```bash
git --version
curl --version | head -1
jq --version
```

_Common Errors:_

- **Package manager locked:** Another process is running updates. Wait and retry.
- **No internet access:** Private subnet missing NAT Gateway or route.

**SSM Agent Check (if using Session Manager):**

```bash
# Amazon Linux 2023
sudo systemctl enable --now amazon-ssm-agent
sudo systemctl status amazon-ssm-agent

# Ubuntu 22.04/24.04
sudo apt-get install -y amazon-ssm-agent
sudo systemctl enable --now amazon-ssm-agent
sudo systemctl status amazon-ssm-agent
```

#### Step 3: Install Node.js (Using NVM for Version Management)

Do not install Node.js from the default OS package manager (`apt install nodejs`). Those repositories are frequently outdated and ship ancient versions. Use Node Version Manager (NVM) for deterministic version control.

```bash
# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Reload the shell profile
source ~/.bashrc

# Install the LTS version of Node.js
nvm install --lts --latest-npm

# Verify the installation
node -v
npm -v
```

_Validation:_

```bash
# Run a quick smoke test
node -e "console.log('Node.js is operational. Version:', process.version)"
# Expected: Node.js is operational. Version: v22.x.x
```

#### Step 4: Install and Configure Nginx as a Reverse Proxy

Nginx should sit in front of your Node.js application to handle TLS termination, static file serving, request buffering, and connection keep-alive management. Node.js should never be directly exposed to the internet.

```bash
# Install Nginx (Amazon Linux 2023)
sudo dnf install nginx -y

# Start and enable Nginx to survive reboots
sudo systemctl start nginx
sudo systemctl enable nginx
```

Configure Nginx as a reverse proxy to forward traffic to the Node.js app running on `localhost:3000`:

```bash
# Create the reverse proxy configuration
sudo tee /etc/nginx/conf.d/app.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
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
EOF

# Test the configuration syntax
sudo nginx -t
# Expected: nginx: configuration file /etc/nginx/nginx.conf test is successful

# Reload Nginx to apply
sudo systemctl reload nginx
```

_Validation:_

```bash
# Test from the instance itself
curl -I http://localhost
# Expected: HTTP/1.1 502 Bad Gateway (because Node.js app is not running yet)
# A 502 from Nginx PROVES Nginx is correctly proxying to port 3000.
# A "Connection Refused" would mean Nginx itself is not running.
```

#### Step 4b: Minimal Nginx Reverse Proxy (Required Example)

If you need the simplest possible working config, use this exact template and replace `yourdomain.com`.

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
    }
}
```

**Ubuntu/Debian (sites-available/sites-enabled):**

```bash
sudo tee /etc/nginx/sites-available/my-app.conf > /dev/null <<'EOF'
server {
  listen 80;
  server_name yourdomain.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
  }
}
EOF

sudo ln -s /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/my-app.conf
sudo nginx -t
sudo systemctl restart nginx
```

**Amazon Linux (conf.d):**

```bash
sudo tee /etc/nginx/conf.d/my-app.conf > /dev/null <<'EOF'
server {
  listen 80;
  server_name yourdomain.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
  }
}
EOF

sudo nginx -t
sudo systemctl restart nginx
```

---

### 5.5 Security Groups: The Stateful Instance-Level Firewall

#### What is it?

A Security Group (SG) is a virtual stateful firewall attached at the Elastic Network Interface (ENI) level of an instance. "Stateful" means that if you allow an inbound request on port 443, the response is automatically allowed out without requiring an explicit outbound rule.

#### Why is deep understanding critical?

Security Groups are the most commonly misconfigured component in AWS. A single misconfigured inbound rule can expose a database to the entire internet.

#### The Enterprise Rules:

1. **Default Deny Inbound:** A Security Group has no inbound rules by default. This is secure by design. You must explicitly open only the ports you need.
2. **Default Allow All Outbound:** By default, all outbound traffic is allowed. In high-security environments, we restrict outbound rules to only the specific destinations and ports required (e.g., allow port 443 outbound to `0.0.0.0/0` for HTTPS, but deny everything else to prevent data exfiltration).
3. **Reference Other Security Groups, Not IPs:** Instead of allowing `10.0.10.0/24` to access your database, allow the Security Group `sg-app-servers` to access port 5432. This is self-healing: if app servers are added or removed, the rule automatically applies.
4. **One SG Per Logical Function:** Create `sg-alb`, `sg-app`, `sg-db`, `sg-redis` separately. Never create a single "allow-everything" SG and attach it to all resources.

#### Bad vs. Good Practice:

| Aspect          | ❌ Bad Practice                            | ✅ Good Practice                                                 |
| --------------- | ------------------------------------------ | ---------------------------------------------------------------- |
| SSH Access      | Port 22 open to `0.0.0.0/0`                | Port 22 open only to `sg-bastion` or disabled entirely (use SSM) |
| Database Access | Port 5432 open to `0.0.0.0/0`              | Port 5432 open only to `sg-app-servers`                          |
| Outbound Rules  | Allow All to `0.0.0.0/0` on all ports      | Allow 443 to `0.0.0.0/0`, 5432 to `sg-db` only                   |
| Naming          | `sg-0abc123` (default auto-generated name) | `prod-auth-api-sg` with description tags                         |

---

### 5.6 SSH Hardening: Locking Down Remote Access

If SSH must remain enabled (i.e., SSM Session Manager is not available), the following hardening steps are non-negotiable.

#### Step 1: Disable Password Authentication

```bash
# Edit the SSH daemon configuration
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
```

#### Step 2: Disable Root Login

```bash
sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

#### Step 3: Change the Default SSH Port (Optional but Recommended)

Moving SSH off port 22 eliminates 99% of automated brute-force bot traffic. It is security through obscurity—not a replacement for key-based auth—but it drastically reduces log noise.

```bash
# Change the port to a non-standard high port
sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config
```

#### Step 4: Restart the SSH Daemon

```bash
sudo systemctl restart sshd

# CRITICAL: Before closing your current SSH session, open a NEW terminal
# and verify you can connect on the new port. Otherwise you will lock yourself out.
ssh -i key.pem -p 2222 ec2-user@instance-ip
```

_Validation:_

```bash
# Verify the SSH configuration is applied
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|port'
# Expected Output:
# port 2222
# permitrootlogin no
# passwordauthentication no
```

---

### 5.7 Real-World EC2 Mistakes

#### Mistake 1: The "Pets vs. Cattle" Fallacy

- _What happens:_ An engineer lovingly configures a single EC2 instance over weeks—installing custom packages, tweaking kernel parameters, hand-editing config files. This instance becomes a "pet" with a name like "Apollo." When Apollo crashes at 2 AM, no one knows how to recreate it because the setup was never documented or automated.
- _The Impact:_ 8 hours of frantic manual reconstruction, data loss, and a post-mortem revealing that the company's entire revenue-generating API was running on a single, un-reproducible snowflake server.
- _The Mandate:_ Instances are cattle, not pets. They are numbered, not named. They are provisioned from Golden AMIs via Auto Scaling Groups and Launch Templates. If one dies, another identical one is automatically spawned.

#### Mistake 2: Ignoring Instance Store Volumes

- _What happens:_ A developer chooses an `i3.xlarge` instance because of its high I/O performance, stores critical application data on the NVMe instance store volume, and never backs it up.
- _The Impact:_ Instance store volumes are ephemeral. When the instance is stopped or terminated, **all data on the instance store is permanently destroyed**. There is no recovery. The developer loses weeks of processing results.
- _The Mandate:_ Application state and persistent data must always reside on EBS volumes or external storage (S3, EFS, RDS). Instance store volumes are only for temporary scratch space, caches, or swap.

#### Mistake 3: Running Without an Auto Scaling Group

- _What happens:_ A single standalone EC2 instance runs the production API. Traffic is normal. One day, the underlying physical host experiences a hardware failure. The instance terminates.
- _The Impact:_ Total downtime until an engineer manually launches a new instance, configures it, and points DNS to it. If this happens at 3 AM on a Saturday, the outage could last hours.
- _The Mandate:_ Even a single instance should be placed inside an Auto Scaling Group (ASG) with `min=1, max=1, desired=1`. If the instance fails a health check or the underlying host degrades, the ASG will automatically terminate the unhealthy instance and launch a fresh replacement from the Launch Template. This is called "self-healing infrastructure."

## 6. S3 Storage & Bucket Management

### Purpose

Provision secure, scalable object storage for assets, logs, and static sites.

### What You Will Achieve

- Create and manage S3 buckets safely
- Apply correct access controls and encryption
- Host static sites and generate pre-signed URLs

### Step-by-Step Implementation

1. Create buckets and upload objects (Section 6.2).
2. Configure permissions and public access blocks (Section 6.3).
3. Decide public vs. private access patterns (Section 6.4).
4. Enable encryption and lifecycle policies (Sections 6.7 and 6.8).

### Commands

All required AWS CLI commands are included under each step.

### Validation

Run the `aws s3 ls` and policy validation steps after each change.

### Common Errors

- Bucket name already exists (global namespace).
- AccessDenied due to missing KMS permissions.

### Pro Tips

- Default to private buckets with CloudFront OAC.
- Enable versioning for critical data.

### 6.1 What is Amazon S3?

Amazon Simple Storage Service (S3) is an infinitely scalable, highly durable object storage service. It is not a file system. There are no directories, no inodes, no mount points. S3 stores data as flat key-value pairs inside logical containers called "buckets." The "key" is the full object path (e.g., `uploads/2026/05/invoice.pdf`), and the "value" is the binary blob of the file itself, plus metadata.

#### Why is S3 the default storage primitive?

S3 provides 99.999999999% (11 nines) of data durability. This means if you store 10 million objects, you can statistically expect to lose a single object once every 10,000 years. AWS achieves this by automatically replicating every object across a minimum of three physically separated Availability Zones within a region. No other storage medium on the planet offers this level of durability at this cost.

#### When to use S3:

- Storing application assets: user-uploaded images, documents, videos, and PDFs.
- Hosting static websites and Single Page Applications (SPAs) like React or Angular builds.
- Centralized logging sink for CloudTrail, ALB access logs, VPC Flow Logs, and application logs.
- Data lake foundation for analytics pipelines (Athena, Glue, EMR).
- Terraform state backend (with DynamoDB locking).
- Backup and disaster recovery archives.

#### When NOT to use S3:

- **As a database.** S3 is not designed for transactional read/write workloads with sub-millisecond latency requirements. Use DynamoDB or RDS for that.
- **As a shared file system for concurrent writes.** If multiple EC2 instances need to read and write to the same file simultaneously (like a shared NFS mount), use Amazon EFS (Elastic File System). S3 does not support file locking.
- **For frequently mutated small objects.** S3 has per-request pricing. If your application writes a 1KB config file 10,000 times per second, the PUT request costs will far exceed the value of the storage. Use DynamoDB or ElastiCache for high-frequency key-value updates.

---

### 6.2 Bucket Creation & Core Operations (CLI)

#### Step 1: Create the Bucket

S3 bucket names are globally unique across all AWS accounts worldwide. Choose a name that follows the enterprise naming convention.

```bash
# Create a bucket in the us-east-1 region
aws s3 mb s3://acme-corp-prod-assets --region us-east-1
```

_Validation:_

```bash
# List all buckets to confirm creation
aws s3 ls

# Expected output includes:
# 2026-05-01 10:00:00 acme-corp-prod-assets
```

#### Step 2: Upload Objects

```bash
# Upload a single file
aws s3 cp ./build/index.html s3://acme-corp-prod-assets/frontend/index.html

# Upload an entire directory recursively
aws s3 cp ./build/ s3://acme-corp-prod-assets/frontend/ --recursive

# Sync a local directory (only uploads changed files — idempotent)
aws s3 sync ./build/ s3://acme-corp-prod-assets/frontend/ --delete
```

_Validation:_

```bash
# List the contents of the bucket to confirm the upload
aws s3 ls s3://acme-corp-prod-assets/frontend/ --recursive --human-readable

# Download a file to verify integrity
aws s3 cp s3://acme-corp-prod-assets/frontend/index.html ./downloaded-index.html
diff ./build/index.html ./downloaded-index.html
# If 'diff' produces no output, the files are identical.
```

#### Step 3: Delete Objects and Buckets

```bash
# Delete a single object
aws s3 rm s3://acme-corp-prod-assets/frontend/old-file.js

# Delete all objects in a bucket (DANGER — irreversible without versioning)
aws s3 rm s3://acme-corp-prod-assets --recursive

# Delete the bucket itself (must be empty first)
aws s3 rb s3://acme-corp-prod-assets
```

---

### 6.3 Permissions: The Multi-Layer Access Control Model

S3 access control is the single most breached component in the entire AWS ecosystem. Understanding the layered permission model is not optional—it is survival.

#### Layer 1: Block Public Access (Account-Level Kill Switch)

AWS provides an account-level setting called "S3 Block Public Access." When enabled, it overrides all bucket policies and ACLs to prevent any bucket in the account from being made public, regardless of individual bucket configuration.

```bash
# Enable Block Public Access at the ACCOUNT level (mandatory for all accounts)
aws s3control put-public-access-block \
    --account-id 123456789012 \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

_Validation:_

```bash
aws s3control get-public-access-block --account-id 123456789012
# All four settings must be 'true'.
```

#### Layer 2: Bucket Policy (Resource-Based Policy)

A JSON policy attached directly to the bucket that defines who (which AWS principals) can perform which actions on the bucket and its objects.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAppServerReadOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/prod-app-server-role"
      },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::acme-corp-prod-assets",
        "arn:aws:s3:::acme-corp-prod-assets/*"
      ]
    }
  ]
}
```

```bash
# Apply the bucket policy
aws s3api put-bucket-policy \
    --bucket acme-corp-prod-assets \
    --policy file://bucket-policy.json
```

#### Layer 3: IAM Policies (Identity-Based)

The IAM role or user attempting to access the bucket must also have an IAM policy granting the corresponding S3 actions. Both the bucket policy AND the IAM policy must allow the action for it to succeed (unless the bucket policy uses a wildcard principal `"*"`).

#### Layer 4: Access Control Lists (ACLs) — DEPRECATED

ACLs are the legacy, pre-2012 permission model. They are coarse-grained, confusing, and the primary cause of accidental public exposure. **ACLs must be disabled on all new buckets.**

```bash
# Enforce Bucket Owner Enforced (disables ACLs entirely)
aws s3api put-bucket-ownership-controls \
    --bucket acme-corp-prod-assets \
    --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerEnforced"}]}'
```

---

### 6.4 Public vs. Private Buckets

#### Private Buckets (The Default and the Enterprise Standard)

By default, every S3 bucket is private. Only the AWS account owner (and principals granted explicit permission) can access the objects. This is the correct posture for 99% of enterprise use cases.

- _Use cases:_ Application data, user uploads, database backups, Terraform state files, log archives.

#### Public Buckets (Extremely Rare and Heavily Regulated)

A public bucket allows anyone on the internet to read (and potentially write to) objects without authentication. The only legitimate enterprise use case for a truly public bucket is hosting a static website directly from S3 without CloudFront—and even that is increasingly replaced by CloudFront + Origin Access Control (OAC).

- **When to use:** Almost never. If you must serve public assets, use a private S3 bucket as a CloudFront origin with OAC. This keeps the bucket locked down while CloudFront serves the content globally.
- **When NOT to use:** Storing any sensitive data, PII, credentials, API keys, database dumps, or internal documentation.

---

### 6.5 Pre-Signed URLs: Temporary, Scoped Access

#### What is it?

A Pre-Signed URL is a time-limited, cryptographically signed URL that grants temporary access to a specific object in a private S3 bucket. The URL encodes the AWS credentials, the allowed action (GET or PUT), the expiration timestamp, and a signature. Anyone with the URL can perform the action until the URL expires—no AWS credentials required on the client side.

#### Why is this critical?

It solves the fundamental problem of allowing untrusted clients (mobile apps, browser-based JavaScript, third-party vendors) to upload or download files from a private bucket without ever exposing AWS credentials to the client.

#### When to use:

- Allowing a user's browser to directly upload a profile picture to S3 (bypassing your backend server, which avoids bandwidth bottlenecks).
- Generating temporary download links for invoices, reports, or exported CSVs that expire after 15 minutes.
- Sharing a specific log file with a vendor for debugging without giving them access to the entire bucket.

#### When NOT to use:

- For objects that must be permanently and publicly accessible (use CloudFront + OAC).
- If the URL will be cached or indexed by search engines (it will expire, leading to broken links).

#### Generating a Pre-Signed URL:

```bash
# Generate a download URL that expires in 3600 seconds (1 hour)
aws s3 presign s3://acme-corp-prod-assets/reports/monthly-revenue-2026-04.pdf \
    --expires-in 3600

# Output: https://acme-corp-prod-assets.s3.amazonaws.com/reports/monthly-revenue-2026-04.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Expires=3600&X-Amz-Signature=...
```

_Validation:_

```bash
# Test the generated URL from any machine (even without AWS CLI)
curl -I "https://acme-corp-prod-assets.s3.amazonaws.com/reports/monthly-revenue-2026-04.pdf?X-Amz-Algorithm=..."
# Expected: HTTP/1.1 200 OK

# Wait for the URL to expire, then test again
# Expected: HTTP/1.1 403 Forbidden (Access Denied)
```

---

### 6.6 Static Website Hosting on S3

#### What is it?

S3 can serve static web content (HTML, CSS, JavaScript, images) directly via an HTTP endpoint, acting as a simple web server without any EC2 instances, Nginx, or containers.

#### When to use:

- Hosting React, Angular, or Vue.js Single Page Applications (SPAs) for internal tools.
- Landing pages, documentation sites (Docusaurus, MkDocs), or marketing microsites.

#### When NOT to use:

- Applications requiring server-side rendering (SSR), dynamic API routes, or database connections. Use ECS/EKS/Lambda for those.
- If you need HTTPS with a custom domain. S3 website endpoints only support HTTP natively. You must place CloudFront in front of S3 for HTTPS with ACM certificates.

#### Step-by-Step Setup:

```bash
# 1. Enable static website hosting
aws s3 website s3://acme-corp-marketing-site \
    --index-document index.html \
    --error-document error.html

# 2. Upload the built SPA
aws s3 sync ./build/ s3://acme-corp-marketing-site/ --delete

# 3. For a PRIVATE bucket behind CloudFront (RECOMMENDED):
#    No public access needed. CloudFront OAC handles auth.
#
# For a TRULY PUBLIC static site (use with extreme caution):
# You must disable Block Public Access on THIS bucket and add a public read policy.
```

_Validation:_

```bash
# The S3 website endpoint format is:
# http://<bucket-name>.s3-website-<region>.amazonaws.com
curl -I http://acme-corp-marketing-site.s3-website-us-east-1.amazonaws.com
# Expected: HTTP/1.1 200 OK
```

---

### 6.7 Encryption: Data Protection at Rest

Every object stored in S3 must be encrypted at rest. This is a non-negotiable enterprise mandate.

#### Server-Side Encryption Options:

1. **SSE-S3 (S3-Managed Keys):** AWS manages the keys entirely. Zero configuration. This is the default since January 2023—all new objects are automatically encrypted with SSE-S3.
   - _When to use:_ When compliance does not require you to control or audit the encryption key lifecycle.
2. **SSE-KMS (KMS-Managed Keys):** You create a Customer Managed Key (CMK) in AWS KMS. S3 uses this key to encrypt objects. Every encryption/decryption call is logged in CloudTrail, providing a full audit trail of who accessed what data and when.
   - _When to use:_ When compliance (SOC2, HIPAA, PCI-DSS) mandates key rotation auditing, cross-account key sharing control, or the ability to immediately revoke access to all encrypted data by disabling the key.
3. **SSE-C (Customer-Provided Keys):** You provide your own encryption key with every PUT and GET request. AWS never stores the key.
   - _When to use:_ Extremely rare. Only when regulatory requirements mandate that AWS must never possess the encryption key at rest (even ephemerally during processing).

```bash
# Enable default SSE-KMS encryption on a bucket
aws s3api put-bucket-encryption \
    --bucket acme-corp-prod-assets \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678"
        },
        "BucketKeyEnabled": true
      }]
    }'
```

_Validation:_

```bash
aws s3api get-bucket-encryption --bucket acme-corp-prod-assets
# Confirm SSEAlgorithm is "aws:kms" and the correct KMS key ARN is listed.
```

---

### 6.8 Lifecycle Policies: Automated Cost Optimization

S3 offers multiple storage classes with decreasing cost (and increasing retrieval latency). Lifecycle policies automatically transition objects between classes based on age.

| Storage Class                | Cost (per GB/mo) | Retrieval    | Use Case                                |
| ---------------------------- | ---------------- | ------------ | --------------------------------------- |
| S3 Standard                  | ~$0.023          | Instant      | Active application data                 |
| S3 Infrequent Access (IA)    | ~$0.0125         | Instant      | Backups accessed monthly                |
| S3 Glacier Instant Retrieval | ~$0.004          | Milliseconds | Compliance archives needing rare access |
| S3 Glacier Deep Archive      | ~$0.00099        | 12-48 hours  | Regulatory archives held for 7+ years   |

```bash
# Apply a lifecycle policy: move to IA after 30 days, Glacier after 90, delete after 365
aws s3api put-bucket-lifecycle-configuration \
    --bucket acme-corp-prod-logs \
    --lifecycle-configuration '{
      "Rules": [{
        "ID": "LogRetentionPolicy",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Transitions": [
          {"Days": 30, "StorageClass": "STANDARD_IA"},
          {"Days": 90, "StorageClass": "GLACIER"}
        ],
        "Expiration": {"Days": 365}
      }]
    }'
```

_Validation:_

```bash
aws s3api get-bucket-lifecycle-configuration --bucket acme-corp-prod-logs
```

---

### 6.9 Data Leak Scenarios: Real-World S3 Breaches

S3 data leaks have caused more public embarrassment and regulatory fines than almost any other AWS misconfiguration. These are not hypothetical scenarios—they are documented incidents.

#### Breach 1: The Misconfigured Bucket Policy with Principal "\*"

- _The Scenario:_ A developer writes a bucket policy to allow their application to read objects. Instead of specifying the app's IAM role ARN as the `Principal`, they use `"Principal": "*"` (meaning "anyone in the world"). They intended to restrict by `Condition` but forgot to add the condition block.
- _The Impact:_ Every object in the bucket—including customer PII, internal financial documents, and API keys—becomes downloadable by anyone with the bucket URL. Automated scanners (like GrayhatWarfare) index the bucket within hours.
- _The Mandate:_ Never use `"Principal": "*"` in a bucket policy unless it is a truly public static website bucket. All bucket policies must be peer-reviewed in Pull Requests with mandatory S3 policy linting (via `aws-s3-policy-checker` or Checkov).

#### Breach 2: The Forgotten Log Bucket

- _The Scenario:_ An ALB is configured to dump access logs into an S3 bucket. The bucket was created years ago with legacy ACLs that granted `AuthenticatedUsers` read access (which means any person with any AWS account, not just your organization).
- _The Impact:_ Access logs contain source IP addresses, user agents, request paths (which often include session tokens in query strings), and response codes. An attacker mines these logs to replay session tokens and hijack active user accounts.
- _The Mandate:_ All log buckets must have ACLs disabled (`BucketOwnerEnforced`), Block Public Access enabled, SSE-KMS encryption, and access restricted exclusively to the logging service's IAM role.

#### Breach 3: The Versioning Oversight

- _The Scenario:_ A security team discovers a publicly exposed bucket and immediately removes the public bucket policy. They believe the incident is resolved.
- _The Impact:_ S3 Versioning was enabled. While the current version of each object is now private, all previous versions of those objects still have the old, public ACL permissions. The attacker had already bookmarked version-specific URLs and continues to download sensitive data for weeks.
- _The Mandate:_ When remediating a public bucket, you must also audit and remediate all object versions. Use `aws s3api list-object-versions` to enumerate every version and `aws s3api put-object-acl` to reset permissions.

#### Breach 4: Cross-Account Access via Overly Broad STS AssumeRole

- _The Scenario:_ A vendor is granted cross-account access to a specific S3 prefix (`vendor-data/*`). The IAM trust policy, however, allows the vendor's entire AWS account to assume the role, rather than a specific IAM role within their account.
- _The Impact:_ Any IAM user or role within the vendor's account—including a compromised CI/CD runner—can assume the role and access all data in the allowed prefix. The blast radius is the vendor's entire organization, not just the authorized individual.
- _The Mandate:_ Cross-account trust policies must always specify the exact IAM role ARN (`arn:aws:iam::VENDOR_ACCT:role/specific-role-name`), never the account root (`arn:aws:iam::VENDOR_ACCT:root`).

---

### 6.10 Troubleshooting S3 Access Issues

- _Issue:_ `AccessDenied` when trying to `GetObject` despite having an IAM policy allowing `s3:GetObject`.
- _Resolution:_ Check all four layers: (1) Is Block Public Access overriding? (2) Does the Bucket Policy explicitly `Deny` your principal? An explicit `Deny` always overrides an `Allow`. (3) Is SSE-KMS enabled, and does your IAM role have `kms:Decrypt` permission on the CMK? (4) Is there a VPC Endpoint policy restricting access to specific buckets?

- _Issue:_ `SlowDown` errors (HTTP 503) during high-throughput uploads.
- _Resolution:_ S3 automatically partitions keys by prefix. If all objects share the same prefix (e.g., `logs/2026-05-01/...`), S3 may throttle. Distribute objects across randomized prefixes or use hex hash prefixes.

## 7. Database Strategy (RDS & Alternatives)

### Purpose

Choose and provision a production-grade database with strong durability and recovery controls.

### What You Will Achieve

- Select the right database engine for your workload
- Launch an RDS instance with secure defaults
- Implement backups, scaling, and safe access patterns

### Step-by-Step Implementation

1. Choose the engine (Section 7.2).
2. Create the subnet group and RDS instance (Section 7.3).
3. Store credentials in Secrets Manager (Section 7.3 Step 3).
4. Validate connectivity and set up backups (Sections 7.4 and 7.5).

### Commands

CLI commands are provided for subnet group creation, RDS provisioning, and backups.

### Validation

Confirm the DB is `available`, endpoints resolve, and you can connect via SSM or SSH tunnel.

### Common Errors

- RDS in a public subnet (security risk).
- Security group does not allow app-to-db traffic.

### Pro Tips

- Use Multi-AZ for production.
- Store credentials only in Secrets Manager.

### 7.1 What is Amazon RDS?

Amazon Relational Database Service (RDS) is a fully managed service that provisions, patches, backs up, and maintains relational database engines on your behalf. When you create an RDS instance, AWS deploys a dedicated EC2 instance behind the scenes, installs the database engine, configures storage on EBS volumes, sets up automated backups to S3, and provides a DNS endpoint for your application to connect to. You never SSH into the underlying host. You never run `apt upgrade` to patch the engine. AWS handles the undifferentiated heavy lifting of database administration.

#### Why is RDS the enterprise default?

Operating a production-grade relational database is arguably the single most complex and high-risk task in all of infrastructure engineering. A single misconfigured `pg_hba.conf`, a missed vacuum operation, or a corrupted WAL segment can result in catastrophic, irrecoverable data loss. RDS eliminates entire categories of human error by automating the operational mechanics that require deep DBA expertise.

> ⚠️ Cost Warning: RDS bills per hour, storage, and backup retention. Multi-AZ roughly doubles compute cost. Keep dev/staging instances small and delete unused databases to prevent silent monthly charges.

#### When to use RDS:

- Any application requiring ACID-compliant relational storage (user accounts, financial transactions, inventory management, order processing).
- When your team does not have a dedicated, senior Database Administrator (DBA) on staff. RDS compensates for the absence of deep engine-level expertise.
- When automated backups, point-in-time recovery, and Multi-AZ failover are hard requirements (which they should be for all production data).
- When compliance mandates encryption at rest (RDS integrates natively with KMS) and audit logging (RDS integrates with CloudWatch and CloudTrail).

#### When NOT to use RDS:

- **Custom database engines or exotic extensions:** RDS supports MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and Amazon Aurora. If your application requires CockroachDB, TimescaleDB with specific C extensions, or a custom-compiled PostgreSQL fork, RDS cannot host it. You must self-manage on EC2.
- **Extreme performance tuning requirements:** RDS abstracts the operating system. You cannot tune kernel parameters (`vm.swappiness`, `vm.dirty_ratio`), configure custom filesystem mount options (`noatime`, `data=writeback`), or pin NUMA nodes. If your workload requires this level of bare-metal optimization, you need EC2.
- **Multi-master write scaling:** Standard RDS supports a single primary writer. If your application requires geographically distributed multi-master writes, consider Amazon Aurora Global Database or DynamoDB Global Tables.

---

### 7.2 Engine Selection: MySQL vs. PostgreSQL vs. Aurora

Choosing the wrong database engine is a decision that haunts an organization for years. Migrations between engines are extraordinarily expensive and risky. This decision must be made deliberately, not by accident or familiarity.

#### 7.2.1 MySQL (RDS for MySQL)

- **What:** The world's most widely deployed open-source relational database. Known for speed, simplicity, and massive community support.
- **When to use:**
  - Simple CRUD-heavy web applications (WordPress, Laravel, Express.js apps).
  - When the team has deep MySQL operational experience.
  - Read-heavy workloads leveraging MySQL's efficient query cache and replication model.
- **When NOT to use:**
  - Complex analytical queries requiring advanced window functions, CTEs, or JSON path operations. MySQL historically lags behind PostgreSQL in SQL standards compliance.
  - Applications requiring robust JSONB indexing, full-text search with ranking, or GIS/PostGIS spatial queries.
- **Real-World Mistake:** A team picks MySQL because "it's what we've always used." Two years later, a new feature requires recursive CTEs and advanced JSON aggregation. The team discovers MySQL's support is limited, and they face a 6-month migration to PostgreSQL under immense business pressure.

#### 7.2.2 PostgreSQL (RDS for PostgreSQL)

- **What:** The most advanced open-source relational database. PostgreSQL prioritizes standards compliance, extensibility, and data integrity above raw speed.
- **When to use:**
  - Applications requiring complex relational modeling (foreign keys, constraints, triggers, stored procedures).
  - JSONB-heavy workloads where you need to index and query semi-structured data alongside relational columns.
  - GIS and geospatial applications using the PostGIS extension.
  - Compliance-heavy environments requiring row-level security policies.
- **When NOT to use:**
  - Extremely simple key-value or document-only workloads where DynamoDB or MongoDB would be architecturally simpler and cheaper.
- **Enterprise Recommendation:** PostgreSQL is our default engine for all new projects unless a specific, documented technical requirement mandates otherwise.

#### 7.2.3 Amazon Aurora

- **What:** A proprietary AWS database engine that is wire-compatible with MySQL and PostgreSQL but uses a completely re-engineered, cloud-native storage layer. Aurora decouples compute from storage: the storage layer automatically replicates data six ways across three AZs and can grow up to 128 TiB without manual provisioning.
- **When to use:**
  - High-throughput production workloads requiring up to 5x the throughput of standard MySQL and 3x the throughput of standard PostgreSQL.
  - When you need up to 15 read replicas (standard RDS supports 5).
  - When Aurora Serverless v2 fits your traffic pattern: the database scales compute capacity up and down automatically based on load, and you pay per ACU (Aurora Capacity Unit) consumed.
- **When NOT to use:**
  - Budget-constrained development or staging environments. Aurora's minimum cost is significantly higher than a `db.t3.micro` RDS instance.
  - When vendor lock-in is a critical concern. Aurora's storage engine is proprietary and cannot be replicated outside of AWS.

---

### 7.3 Provisioning RDS: Step-by-Step (CLI)

#### Step 1: Create a DB Subnet Group

RDS instances must be launched into a DB Subnet Group—a collection of at least two subnets in different Availability Zones within your VPC. This is required for Multi-AZ deployments.

```bash
aws rds create-db-subnet-group \
    --db-subnet-group-name prod-db-subnet-group \
    --db-subnet-group-description "Production database subnets across us-east-1a and us-east-1b" \
    --subnet-ids '["subnet-isolated-1a","subnet-isolated-1b"]'
```

#### Step 2: Create the RDS Instance

```bash
aws rds create-db-instance \
    --db-instance-identifier prod-auth-db \
    --db-instance-class db.m5.large \
    --engine postgres \
    --engine-version 16.4 \
    --master-username dbadmin \
    --master-user-password "$(aws secretsmanager get-random-password --password-length 32 --query 'RandomPassword' --output text)" \
    --allocated-storage 100 \
    --storage-type gp3 \
    --storage-encrypted \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678 \
    --vpc-security-group-ids sg-db-prod \
    --db-subnet-group-name prod-db-subnet-group \
    --multi-az \
    --backup-retention-period 35 \
    --preferred-backup-window "03:00-04:00" \
    --preferred-maintenance-window "Sun:05:00-Sun:06:00" \
    --deletion-protection \
    --copy-tags-to-snapshot \
    --tags Key=Environment,Value=Production Key=Service,Value=AuthAPI
```

_Critical Flags Explained:_

- `--multi-az`: Deploys a synchronous standby replica in a second AZ. If the primary AZ fails, RDS automatically fails over to the standby in approximately 60-120 seconds.
- `--storage-encrypted`: Encrypts all data, logs, snapshots, and replicas at rest using the specified KMS key.
- `--deletion-protection`: Prevents accidental database deletion via CLI or console. Must be explicitly disabled before deletion.
- `--backup-retention-period 35`: Retains automated daily snapshots for 35 days (the maximum). This directly defines your Recovery Point Objective (RPO).

_Validation:_

```bash
# Wait for the instance to become available (can take 10-15 minutes)
aws rds wait db-instance-available --db-instance-identifier prod-auth-db

# Describe the instance to get the endpoint
aws rds describe-db-instances \
    --db-instance-identifier prod-auth-db \
    --query 'DBInstances[0].[Endpoint.Address, Endpoint.Port, MultiAZ, StorageEncrypted]' \
    --output table
```

#### Step 3: Store the Password in Secrets Manager

The master password was generated randomly in the provisioning command. It must be immediately stored in AWS Secrets Manager—never in a `.env` file, a Slack message, or a Confluence page.

```bash
aws secretsmanager create-secret \
    --name prod/auth-db/master-credentials \
    --description "Master credentials for the production Auth API PostgreSQL database" \
    --secret-string '{"username":"dbadmin","password":"THE_GENERATED_PASSWORD","host":"prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com","port":"5432","dbname":"authdb"}'
```

---

### 7.4 Connecting to the Database

RDS instances in the Isolated Database Tier have no public IP address and no internet route. You must tunnel through a bastion host or use SSM Session Manager port forwarding.

#### Method 1: SSM Port Forwarding (Preferred — No Bastion Required)

```bash
# Start a port forwarding session through SSM to the RDS endpoint
aws ssm start-session \
    --target i-bastion-instance-id \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{"host":["prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

Now, in a separate terminal on your local machine:

```bash
# Connect to PostgreSQL via the local forwarded port
psql -h 127.0.0.1 -p 5432 -U dbadmin -d authdb

# For MySQL:
mysql -h 127.0.0.1 -P 3306 -u dbadmin -p authdb
```

_Validation:_

```bash
# Inside the psql shell, verify connectivity and version
SELECT version();
# Expected: PostgreSQL 16.4 on x86_64-pc-linux-gnu, compiled by gcc...

# Check current database size
SELECT pg_size_pretty(pg_database_size('authdb'));

# List all tables
\dt
```

#### Method 2: SSH Tunnel via Bastion Host

```bash
# From your local machine, create an SSH tunnel
ssh -i bastion-key.pem -L 5432:prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com:5432 ec2-user@bastion-public-ip -N

# In another terminal, connect via the tunnel
psql -h 127.0.0.1 -p 5432 -U dbadmin -d authdb
```

---

### 7.5 Backup Strategy: RPO and RTO

Data loss is the one failure mode from which a company may never recover. The backup strategy is the last line of defense.

#### What are RPO and RTO?

- **Recovery Point Objective (RPO):** The maximum acceptable amount of data loss measured in time. An RPO of 1 hour means you accept losing up to 1 hour of database transactions in the worst-case scenario.
- **Recovery Time Objective (RTO):** The maximum acceptable amount of downtime. An RTO of 30 minutes means the database must be fully operational within 30 minutes of a failure declaration.

#### RDS Automated Backup Mechanisms

1. **Automated Daily Snapshots:**
   - _What:_ RDS takes a full volume-level snapshot of the database instance once per day during the defined backup window.
   - _Retention:_ Configurable from 1 to 35 days. We mandate 35 days for production.
   - _RPO Impact:_ Without transaction logs, your RPO would be up to 24 hours (the time since the last snapshot).

2. **Continuous Transaction Log Archival:**
   - _What:_ RDS continuously archives database transaction logs (WAL for PostgreSQL, binlogs for MySQL) to S3 every 5 minutes.
   - _RPO Impact:_ This gives you an effective RPO of approximately 5 minutes for point-in-time recovery. You can restore the database to any second within the backup retention window.

3. **Manual Snapshots:**
   - _What:_ On-demand snapshots triggered before risky operations (major schema migrations, engine upgrades).
   - _Retention:_ Manual snapshots are retained indefinitely until you explicitly delete them. They are NOT subject to the backup retention period.

```bash
# Create a manual snapshot before a risky migration
aws rds create-db-snapshot \
    --db-instance-identifier prod-auth-db \
    --db-snapshot-identifier pre-migration-snapshot-2026-05-01

# Verify the snapshot
aws rds describe-db-snapshots \
    --db-snapshot-identifier pre-migration-snapshot-2026-05-01 \
    --query 'DBSnapshots[0].[Status, SnapshotCreateTime, AllocatedStorage]' \
    --output table
```

#### Point-in-Time Recovery (PITR)

```bash
# Restore the database to a specific timestamp (creates a NEW instance)
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier prod-auth-db \
    --target-db-instance-identifier prod-auth-db-restored \
    --restore-time "2026-05-01T10:30:00Z" \
    --db-instance-class db.m5.large \
    --db-subnet-group-name prod-db-subnet-group \
    --vpc-security-group-ids sg-db-prod
```

_Critical Note:_ PITR creates a **new** RDS instance. It does not overwrite the existing one. After validation, you must update your application's connection string or DNS CNAME to point to the restored instance.

---

### 7.6 Scaling the Database

#### Vertical Scaling (Scale Up)

Increasing the instance class (e.g., `db.m5.large` → `db.m5.2xlarge`) to get more CPU, RAM, and I/O.

```bash
aws rds modify-db-instance \
    --db-instance-identifier prod-auth-db \
    --db-instance-class db.m5.2xlarge \
    --apply-immediately
```

- _Impact:_ This triggers a brief downtime (typically 5-15 minutes) while RDS stops the old instance and boots a new one with the larger class. In Multi-AZ deployments, RDS performs the upgrade on the standby first, then fails over, minimizing downtime to approximately 60 seconds.
- **When to use:** When your CloudWatch metrics consistently show CPU utilization above 70%, `FreeableMemory` dropping below 1 GiB, or `ReadIOPS`/`WriteIOPS` hitting the volume's baseline.
- **When NOT to use:** As the first response to slow queries. Before scaling up, analyze and optimize your SQL queries, add missing indexes, and review connection pooling. Throwing hardware at a bad query is an expensive band-aid.

#### Horizontal Scaling: Read Replicas

Read Replicas are asynchronous copies of the primary database. They receive a continuous stream of replicated changes from the primary but serve read traffic independently.

```bash
# Create a Read Replica
aws rds create-db-instance-read-replica \
    --db-instance-identifier prod-auth-db-read1 \
    --source-db-instance-identifier prod-auth-db \
    --db-instance-class db.m5.large \
    --availability-zone us-east-1b
```

#### When to use Read Replicas:

- Your application is read-heavy (dashboards, reporting, search, product catalogs). Route all `SELECT` queries to the read replica and all `INSERT`/`UPDATE`/`DELETE` queries to the primary.
- Offloading expensive analytical queries from the primary to prevent them from impacting transactional performance.
- Cross-region read replicas for reducing read latency for globally distributed users.

#### When NOT to use Read Replicas:

- As a substitute for Multi-AZ HA. Read replicas use **asynchronous** replication. If the primary fails and you promote a read replica, you will lose the transactions that had not yet been replicated (the "replication lag"). Multi-AZ uses **synchronous** replication with zero data loss.
- If your application is write-heavy. Read replicas do not help with write scaling. Every write still goes to the single primary.

_Validation:_

```bash
# Check the replication lag of the read replica
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name ReplicaLag \
    --dimensions Name=DBInstanceIdentifier,Value=prod-auth-db-read1 \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --output table
```

_Troubleshooting:_ If `ReplicaLag` consistently exceeds 30 seconds, the read replica's instance class may be too small to keep up with the write volume on the primary. Scale up the replica, or investigate long-running queries on the replica that are blocking the replication thread.

---

### 7.7 RDS vs. Self-Managed Database on EC2

This is one of the most critical architectural decisions. The table below provides a definitive comparison.

| Dimension              | Amazon RDS (Managed)                                    | Self-Managed on EC2                                                                     |
| ---------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **OS Access**          | None. No SSH.                                           | Full root access. Kernel tuning, custom filesystems.                                    |
| **Patching**           | Automated during maintenance window.                    | Manual. You must track CVEs and apply patches yourself.                                 |
| **Backups**            | Automated daily snapshots + continuous WAL archival.    | Manual. You must configure `pg_dump`, `xtrabackup`, or WAL-G.                           |
| **Multi-AZ HA**        | One checkbox. Fully automated failover.                 | Must manually configure streaming replication, `repmgr`, and Keepalived/VIP management. |
| **Scaling**            | One CLI command to resize or add read replicas.         | Manual instance launch, data sync, replication setup.                                   |
| **Engine Flexibility** | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora. | Any engine: CockroachDB, MongoDB, ClickHouse, custom forks.                             |
| **Cost**               | Higher per-instance (managed service premium).          | Lower per-instance, but exponentially higher labor cost.                                |
| **Operational Risk**   | Low. AWS SLA guarantees 99.95% uptime.                  | High. A single DBA mistake can cause irrecoverable data loss.                           |

#### The Enterprise Verdict:

- **Default to RDS.** Unless there is a documented, technically justified reason to self-manage (exotic engine, bare-metal tuning, regulatory restriction), RDS is the mandatory choice. The labor cost of a single production database outage—engineer salaries during incident response, lost revenue, customer churn, SLA penalty payouts—almost always exceeds the managed service premium.
- **If you self-manage, you own everything:** Patching, backup verification (do you actually test restoring from your backups?), replication monitoring, failover automation, connection pooling, certificate rotation, and storage capacity planning. Most teams underestimate this burden by an order of magnitude.

---

### 7.8 Real-World Database Failures

#### Failure 1: The Untested Backup

- _The Scenario:_ A company runs nightly `pg_dump` backups to S3 for their self-managed PostgreSQL on EC2. The cron job runs for 2 years without issues. One day, the primary database suffers unrecoverable disk corruption.
- _The Mistake:_ The team attempts to restore from the latest backup and discovers that `pg_dump` had been silently failing for the last 6 months due to a disk space issue on the instance. The dump files on S3 are 0 bytes. Nobody ever checked.
- _The Impact:_ 6 months of customer data permanently lost. The company faces class-action lawsuits and regulatory fines.
- _The Mandate:_ Backups that are not regularly tested by performing a full restoration to a staging environment are not backups. They are hopes. Automated backup verification must be a CI/CD pipeline stage.

#### Failure 2: The Connection Pool Starvation

- _The Scenario:_ A Node.js API connects to RDS PostgreSQL. Each HTTP request opens a new database connection and closes it when the response is sent. Under load, the connection count rises from 50 to 500 to the RDS maximum of `max_connections` (determined by instance RAM).
- _The Mistake:_ Once `max_connections` is exhausted, every new API request fails with `FATAL: too many connections for role "appuser"`. The entire API crashes under moderate load.
- _The Impact:_ Complete production outage during peak traffic.
- _The Mandate:_ Applications must never connect directly to RDS. Always use a connection pooler: `PgBouncer` (sidecar or standalone) for PostgreSQL, or the built-in connection pool in your application's ORM (e.g., Sequelize `pool: { max: 20, min: 5 }`). For serverless architectures (Lambda), use RDS Proxy, which is a managed connection pooler designed specifically for high-concurrency, short-lived function connections.

#### Failure 3: The Accidental Production Drop

- _The Scenario:_ A developer connects to the production database thinking it is the staging database (because both endpoints look identical: `prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com` vs. `staging-auth-db.xyzxyz.us-east-1.rds.amazonaws.com`). They run `DROP TABLE users;`.
- _The Impact:_ All user data is immediately deleted. The team must perform a PITR restoration, which takes 45 minutes and results in data loss for transactions that occurred between the last WAL checkpoint and the drop.
- _The Mandate:_ (1) Set `deletion-protection` on all production instances. (2) Use distinct, highly visible database naming conventions (e.g., set the PostgreSQL `application_name` connection parameter to include the environment). (3) Restrict production database credentials to a tightly scoped IAM role that only the production application's service account can assume—developers should connect to production read replicas only, with read-only credentials.

## 8. Project Deployment (Backend + Frontend)

### Purpose

Deploy backend and frontend services safely with repeatable, validated steps.

### What You Will Achieve

- Deploy a Node.js backend with PM2 and Nginx
- Build and ship a React frontend to Nginx or S3/CloudFront
- Validate deployments and understand rollback posture

### Step-by-Step Implementation

1. Perform backend deployment steps (Section 8.2).
2. Configure PM2 and make it persistent (Section 8.3).
3. Build and deploy the frontend (Section 8.4).
4. Use the runbook checklist (Section 8.5).

### Commands

All deployment commands are provided inline under each step.

### Validation

Run local and external health checks after each deployment step.

### Common Errors

- Skipping `npm run build` before PM2 reload.
- Missing secrets or incorrect environment variables.

### Pro Tips

- Automate these steps in CI/CD as soon as possible.
- Keep deployments atomic and documented.

### 8.1 The Deployment Philosophy

Deployment is the most dangerous moment in the lifecycle of any software system. It is the precise instant where untested assumptions collide with production reality. Every outage post-mortem in our history can be traced back to one of three deployment sins: deploying un-validated artifacts, deploying with incorrect environment configuration, or deploying without a rollback plan.

**Simple Architecture Flow (Baseline):** User → Domain → Nginx → Node App → Database

This is the minimum production path used throughout this playbook. Every step you configure should reinforce this flow: DNS points to the server, Nginx terminates HTTP(S) and proxies to Node.js, and Node.js persists state in a database.

This section codifies the mechanical, step-by-step process for deploying a Node.js backend API and a React frontend application to our AWS infrastructure. While mature organizations should automate these flows via CI/CD pipelines (covered in a later section), every engineer must understand the manual execution path. Automation is not magic—it is scripted manual steps. If you cannot deploy manually, you cannot debug a broken pipeline.

---

### 8.2 Backend Deployment: Node.js Application

#### 8.2.0 Fast Path (Minimum Required Commands)

If you just need the bare-minimum sequence (and already have server access), use this:

```bash
# Install Node.js with version control (NVM)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts --latest-npm
node -v
npm -v

# Optional: update npm to the latest stable version
npm install -g npm@latest

# Clone the repo
cd /opt/apps
git clone https://github.com/your-org/your-app.git
cd your-app

# Install dependencies and build
npm ci --production
npm run build
```

_Note:_ `npm` ships with Node.js. The `npm -v` check confirms it is installed.

#### 8.2.1 Pre-Deployment Checklist

Before touching the server, verify these conditions locally:

1. The application builds successfully: `npm run build` exits with code 0.
2. All tests pass: `npm test` exits with code 0.
3. Environment variables are documented and the production values are staged in AWS Secrets Manager or SSM Parameter Store.
4. The database schema migration is compatible with the currently running application version (backward-compatible migrations only for zero-downtime deploys).
5. A rollback plan exists: the previous working Git commit SHA is documented.

#### 8.2.2 Step 1: Access the Server

Connect to the EC2 instance in the Private Application Subnet via SSM Session Manager (preferred) or the Bastion SSH tunnel (as documented in Section 5.4).

```bash
# Preferred: SSM Session Manager
aws ssm start-session --target i-0abc1234def56789

# Alternative: SSH via Bastion
ssh -i bastion-key.pem ec2-user@bastion-public-ip
ssh -i app-key.pem ec2-user@10.0.10.45
```

_Validation:_

```bash
whoami
hostname
pwd
```

_Common Errors:_

- **SSM session fails:** Instance profile missing `AmazonSSMManagedInstanceCore`.
- **SSH timeout:** Private instance has no public IP; use SSM or a bastion.

#### 8.2.3 Step 2: Clone the Repository

On the first deployment only, clone the repository. For subsequent deployments, use `git pull`.

##### If You Use SSH (Recommended)

1. Generate a deploy key on the server:

```bash
ssh-keygen -t ed25519 -C "deploy@auth-api" -f ~/.ssh/id_ed25519 -N ""
```

2. Add GitHub to known hosts (prevents the interactive prompt):

```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

3. Copy the public key and add it to your GitHub repo as a **Deploy Key**:

```bash
cat ~/.ssh/id_ed25519.pub
```

**GitHub Steps:** Repo → **Settings** → **Deploy keys** → **Add deploy key** → paste the key → enable **Read-only** (or **Write** only if required for CI).

4. Test SSH access:

```bash
ssh -T git@github.com
```

_Expected:_ A success message like "You've successfully authenticated".

##### If You Use HTTPS (Alternative)

Use a Personal Access Token (PAT) instead of a password.

1. Create a PAT in GitHub: **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
2. Enable scopes: `repo` (private repositories), `read:packages` if needed.
3. Clone using HTTPS:

```bash
git clone https://github.com/acme-corp/auth-api.git
```

4. When prompted for a password, paste the PAT.

_Common Errors:_

- **Permission denied (publickey):** Deploy key not added to GitHub or wrong key file.
- **Host key verification failed:** Run `ssh-keyscan github.com >> ~/.ssh/known_hosts`.
- **Invalid username/password (HTTPS):** Use a PAT, not your GitHub password.

```bash
# First-time deployment: Clone the repository
cd /opt
sudo mkdir -p apps
sudo chown ec2-user:ec2-user apps
cd apps
git clone git@github.com:acme-corp/auth-api.git
cd auth-api

# Subsequent deployments: Pull the latest code
cd /opt/apps/auth-api
git fetch --all
git checkout main
git pull origin main
```

_Validation:_

```bash
# Verify you are on the correct branch and commit
git log -1 --oneline
# Expected: abc1234 feat: implement password reset flow

# Verify the commit matches the release tag
git describe --tags --always
```

#### 8.2.4 Step 3: Install Dependencies

```bash
# Install production dependencies only (excludes devDependencies)
npm ci --production

# WHY 'npm ci' and NOT 'npm install'?
# 'npm ci' performs a clean install from package-lock.json.
# It deletes node_modules first, ensuring deterministic builds.
# 'npm install' may resolve different versions if package-lock.json
# is out of sync, causing "works on my machine" failures.
```

_Validation:_

```bash
# Verify node_modules was created and is not empty
ls -la node_modules/ | head -5

# Check for any npm audit vulnerabilities
npm audit --production
# Address CRITICAL and HIGH severity issues before deploying.
```

#### 8.2.5 Step 4: Build the Application

If your Node.js application uses TypeScript, Babel, or any build step:

```bash
# Compile TypeScript to JavaScript (or run your build script)
npm run build

# Verify the build output exists
ls -la dist/
# Expected: compiled JavaScript files in the dist/ directory
```

_Troubleshooting:_

- _Issue:_ `npm run build` fails with `JavaScript heap out of memory`.
- _Resolution:_ The default Node.js heap size is ~1.7 GiB. For large builds, increase it: `NODE_OPTIONS="--max-old-space-size=4096" npm run build`. If this is a recurring problem, your build server instance class is too small.

#### 8.2.6 Step 5: Configure Environment Variables

**This is the single most critical step.** An application deployed with the wrong database URL, API key, or JWT secret will either crash immediately or—far worse—silently connect to the wrong database and corrupt staging/production data.

##### The Enterprise Mandate: Never Hardcode Secrets

Environment variables must be injected at runtime from a secure, centralized source. There are three acceptable strategies, ranked by maturity:

**Strategy 1: SSM Parameter Store (Minimum Viable)**
Store each variable as a `SecureString` parameter in AWS Systems Manager.

```bash
# Store a secret in SSM
aws ssm put-parameter \
    --name "/prod/auth-api/DATABASE_URL" \
    --type "SecureString" \
    --value "postgresql://dbadmin:PASSWORD@prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com:5432/authdb"

# Retrieve it on the server (via the instance's IAM role)
export DATABASE_URL=$(aws ssm get-parameter --name "/prod/auth-api/DATABASE_URL" --with-decryption --query 'Parameter.Value' --output text)
```

**Strategy 2: AWS Secrets Manager (Recommended for DB Credentials)**
Secrets Manager provides automatic rotation and structured JSON secrets.

```bash
# Retrieve the full secret JSON
SECRET_JSON=$(aws secretsmanager get-secret-value --secret-id prod/auth-db/master-credentials --query 'SecretString' --output text)

# Parse individual values using jq
export DB_HOST=$(echo $SECRET_JSON | jq -r '.host')
export DB_USER=$(echo $SECRET_JSON | jq -r '.username')
export DB_PASS=$(echo $SECRET_JSON | jq -r '.password')
export DB_PORT=$(echo $SECRET_JSON | jq -r '.port')
export DB_NAME=$(echo $SECRET_JSON | jq -r '.dbname')
```

**Strategy 3: .env File from Secrets Manager (Pragmatic for PM2)**
For PM2-managed applications, create a transient `.env` file at deploy time, loaded by the application, and never committed to Git.

```bash
# Generate the .env file from SSM parameters
cat > /opt/apps/auth-api/.env <<EOF
NODE_ENV=production
PORT=3000
DATABASE_URL=$(aws ssm get-parameter --name "/prod/auth-api/DATABASE_URL" --with-decryption --query 'Parameter.Value' --output text)
JWT_SECRET=$(aws ssm get-parameter --name "/prod/auth-api/JWT_SECRET" --with-decryption --query 'Parameter.Value' --output text)
REDIS_URL=$(aws ssm get-parameter --name "/prod/auth-api/REDIS_URL" --with-decryption --query 'Parameter.Value' --output text)
S3_BUCKET=$(aws ssm get-parameter --name "/prod/auth-api/S3_BUCKET" --query 'Parameter.Value' --output text)
EOF

# Restrict file permissions (CRITICAL — only the app user should read this)
chmod 600 /opt/apps/auth-api/.env
```

_Validation:_

```bash
# Verify the .env file exists and is not world-readable
ls -la /opt/apps/auth-api/.env
# Expected: -rw------- 1 ec2-user ec2-user ... .env

# Verify .env is in .gitignore
grep ".env" /opt/apps/auth-api/.gitignore
# Expected: .env
```

##### Quick Environment Variable Strategy (Beginner-Friendly)

Use `.env` for runtime configuration on the server and `export` for immediate shell testing.

```bash
# Temporary export in the current shell session
export PORT=3000
export DB_HOST=localhost
export DB_USER=appuser
export DB_PASSWORD='replace-me'
```

**Why not hardcode secrets?** Hardcoded credentials leak into Git history, logs, and build artifacts. Always inject secrets at runtime via SSM/Secrets Manager and keep `.env` out of Git.

---

### 8.2.7 Domain + DNS (Route 53 or External)

#### What is DNS?

DNS is the internet's phonebook. Your domain (e.g., `api.yourcompany.com`) must resolve to your server's public IP (Elastic IP) or your load balancer's DNS name so users can reach your application.

#### Option A: Route 53 (Recommended)

1. Route 53 → **Hosted zones** → **Create hosted zone** for `yourdomain.com`.
2. Update your domain registrar to use the Route 53 name servers listed in the hosted zone.
3. Create an **A record** pointing to your EC2 **Elastic IP** (or ALB DNS name via Alias).

```bash
# Create the hosted zone
aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference "prod-$(date +%Y%m%d%H%M%S)"

# Create an A record to the Elastic IP
cat > /tmp/route53-record.json <<'EOF'
{
  "Comment": "A record for EC2",
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "yourdomain.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [{"Value": "YOUR_ELASTIC_IP"}]
    }
  }]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID_HERE \
  --change-batch file:///tmp/route53-record.json
```

#### Option B: External DNS Provider

If you are not using Route 53, create the same **A record** in your DNS provider (GoDaddy, Namecheap, Cloudflare) pointing to the Elastic IP or ALB DNS name.

_Validation:_

```bash
dig +short yourdomain.com
# Expected: your Elastic IP (or the ALB DNS resolution)
```

_Note:_ DNS propagation can take 5-30 minutes (sometimes longer). Do not proceed to HTTPS until the domain resolves correctly.

---

### 8.2.8 Enable HTTPS (Certbot + Nginx)

TLS/HTTPS encrypts traffic between users and your server. It is mandatory for production.

**Ubuntu/Debian:**

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com
```

**Amazon Linux 2023:**

```bash
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com
```

_Validation:_

```bash
sudo certbot renew --dry-run
```

_Auto-renewal check:_

```bash
sudo systemctl status certbot.timer
```

---

### 8.3 PM2: Production Process Management

#### What is PM2?

PM2 is a production-grade process manager for Node.js. It provides automatic restarts on crash, cluster mode for multi-core utilization, log management, graceful shutdown handling, and system startup integration.

#### Why NOT just `node server.js`?

Running `node server.js` directly in production is catastrophically fragile:

- If the process crashes (uncaught exception, OOM kill), it stays dead. No automatic restart.
- If the SSH session disconnects, the process dies (unless you used `nohup` or `screen`, which are crude hacks).
- You get no log rotation, no cluster mode, no monitoring.

#### When to use PM2:

- Any Node.js application deployed directly on EC2 instances.
- When the team is not yet using containerization (Docker/ECS/EKS).

#### When NOT to use PM2:

- If the application is containerized and deployed on ECS Fargate or EKS. The container orchestrator handles restarts, scaling, and log collection. PM2 inside a Docker container is redundant and actively harmful (it interferes with the container's signal handling and defeats the purpose of `docker restart` policies).

#### Step 1: Install PM2 Globally

```bash
npm install pm2 -g

# Verify installation
pm2 --version
# Expected: 5.x.x
```

#### Minimal PM2 Lifecycle Commands (Required)

```bash
pm2 start app.js --name my-app
pm2 save
pm2 startup
pm2 logs
```

Restart example:

```bash
pm2 restart my-app
```

#### Step 2: Create the PM2 Ecosystem Configuration File

Instead of passing flags via the command line, use a declarative `ecosystem.config.js` file. This is version-controlled alongside the application code.

```bash
cat > /opt/apps/auth-api/ecosystem.config.js <<'EOF'
module.exports = {
  apps: [{
    name: "auth-api",
    script: "./dist/server.js",        // Entry point (compiled JS)
    instances: "max",                   // Cluster mode: spawn one worker per CPU core
    exec_mode: "cluster",              // Enable cluster mode for multi-core utilization
    max_memory_restart: "512M",        // Auto-restart if a worker exceeds 512MB RAM
    env: {
      NODE_ENV: "production",
      PORT: 3000
    },
    // Graceful shutdown: PM2 sends SIGINT, waits 5 seconds for connections to drain
    kill_timeout: 5000,
    // Wait 3 seconds after boot before considering the process "online"
    wait_ready: true,
    listen_timeout: 10000,
    // Log configuration
    error_file: "/var/log/pm2/auth-api-error.log",
    out_file: "/var/log/pm2/auth-api-out.log",
    merge_logs: true,
    log_date_format: "YYYY-MM-DD HH:mm:ss Z"
  }]
};
EOF
```

#### Step 3: Start the Application with PM2

```bash
# Create the log directory
sudo mkdir -p /var/log/pm2
sudo chown ec2-user:ec2-user /var/log/pm2

# Start the application using the ecosystem file
cd /opt/apps/auth-api
pm2 start ecosystem.config.js

# Verify the application is running
pm2 status
```

_Validation:_

```bash
# Expected output of 'pm2 status':
# ┌────┬──────────┬─────────┬─────────┬──────────┬────────┬──────┐
# │ id │ name     │ mode    │ status  │ restarts │ uptime │ cpu  │
# ├────┼──────────┼─────────┼─────────┼──────────┼────────┼──────┤
# │ 0  │ auth-api │ cluster │ online  │ 0        │ 5s     │ 2%   │
# │ 1  │ auth-api │ cluster │ online  │ 0        │ 5s     │ 1%   │
# └────┴──────────┴─────────┴─────────┴──────────┴────────┴──────┘

# Test the API endpoint locally
curl -s http://localhost:3000/health
# Expected: {"status":"ok","version":"1.2.3","uptime":5}
```

#### Step 4: Essential PM2 Operations

```bash
# Restart all instances (graceful — zero downtime in cluster mode)
pm2 reload auth-api

# Hard restart (kills and restarts — brief downtime)
pm2 restart auth-api

# Stop the application
pm2 stop auth-api

# View real-time logs (stdout + stderr combined)
pm2 logs auth-api

# View only error logs
pm2 logs auth-api --err

# View the last 200 lines of logs
pm2 logs auth-api --lines 200

# Monitor CPU, Memory, and loop delay in real-time
pm2 monit

# Delete the process from PM2's process list entirely
pm2 delete auth-api
```

#### Step 5: Configure PM2 to Survive Reboots

By default, PM2 processes do not survive an instance reboot. You must generate a startup script.

```bash
# Generate the startup script for systemd
pm2 startup systemd

# PM2 will output a command like:
# sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v22.x.x/bin pm2 startup systemd -u ec2-user --hp /home/ec2-user
# EXECUTE THAT EXACT COMMAND.

# Save the current process list so PM2 knows what to restore on boot
pm2 save
```

_Validation:_

```bash
# Reboot the instance and verify PM2 auto-started the application
sudo reboot
# Wait 60 seconds, then reconnect
aws ssm start-session --target i-0abc1234def56789
pm2 status
# The auth-api process should be 'online' with a fresh uptime.
```

#### Step 6: Configure PM2 Log Rotation (Required)

PM2 does not rotate logs by default. Unbounded logs can fill disk and crash the server.

```bash
# Install the log rotation module
pm2 install pm2-logrotate

# Configure rotation limits
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
pm2 set pm2-logrotate:rotateInterval '0 0 * * *'
```

_Validation:_

```bash
pm2 conf pm2-logrotate
# Confirm max_size and retain are set as expected
```

_Common Errors:_

- **Module not found:** Ensure PM2 is installed globally for the current user.
- **Logs still growing:** Confirm you are writing logs to files (see ecosystem config).

---

### 8.4 Frontend Deployment: React Application

#### 8.4.1 Build the React Application

The React application is built locally or in a CI/CD pipeline, producing a static `build/` directory containing HTML, CSS, and JavaScript bundles. There is no server-side runtime.

```bash
# Clone the frontend repository (if not already cloned)
git clone git@github.com:acme-corp/dashboard-frontend.git
cd dashboard-frontend

# Install dependencies
npm ci

# Set the production API URL via environment variable
# React (Create React App) uses REACT_APP_ prefix
# Vite uses VITE_ prefix
export REACT_APP_API_URL=https://api.acme-corp.com
export REACT_APP_ENV=production

# Build the production bundle
npm run build

# Verify the build output
ls -la build/
# Expected: index.html, static/css/, static/js/, static/media/
du -sh build/
# Expected: ~2-15 MB depending on application complexity
```

_Critical Note on Frontend Environment Variables:_
Frontend environment variables are **baked into the JavaScript bundle at build time**. They are not runtime secrets—they are embedded in the static files and visible to anyone who opens the browser's DevTools. **Never put secret API keys, database passwords, or internal tokens in frontend environment variables.** Only public-facing configuration (API base URL, feature flags, analytics IDs) belongs here.

#### 8.4.2 Deployment Option A: Nginx on EC2 (Traditional)

If the frontend is served from the same EC2 instance as the backend (common for small projects or internal tools):

```bash
# Copy the build artifacts to Nginx's serving directory
sudo rm -rf /var/www/dashboard/*
sudo cp -r build/* /var/www/dashboard/

# Set correct ownership
sudo chown -R nginx:nginx /var/www/dashboard/
```

Configure Nginx to serve the SPA with proper client-side routing support:

```bash
sudo tee /etc/nginx/conf.d/dashboard.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name dashboard.acme-corp.com;

    root /var/www/dashboard;
    index index.html;

    # SPA Routing: All unknown paths should serve index.html
    # This allows React Router / Vue Router to handle client-side routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets aggressively (they have content hashes in filenames)
    location /static/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Proxy API requests to the backend (avoids CORS issues)
    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

_Validation:_

```bash
# Test from the server
curl -I http://localhost
# Expected: HTTP/1.1 200 OK, Content-Type: text/html

# Test a deep React route (should still return index.html, not 404)
curl -I http://localhost/users/profile/settings
# Expected: HTTP/1.1 200 OK (served by try_files -> /index.html)
```

#### 8.4.3 Deployment Option B: S3 + CloudFront (Enterprise Standard)

For production frontends, the enterprise standard is deploying static assets to a private S3 bucket and serving them globally via CloudFront CDN. This eliminates the need for EC2 instances to serve static files, provides global edge caching, automatic HTTPS via ACM, and costs pennies per month.

```bash
# Sync the build directory to the S3 bucket
aws s3 sync build/ s3://acme-corp-dashboard-frontend/ \
    --delete \
    --cache-control "public, max-age=31536000, immutable" \
    --exclude "index.html" \
    --exclude "service-worker.js"

# Upload index.html with NO CACHE (must always be fresh)
aws s3 cp build/index.html s3://acme-corp-dashboard-frontend/index.html \
    --cache-control "no-cache, no-store, must-revalidate"

# Upload service-worker.js with NO CACHE
aws s3 cp build/service-worker.js s3://acme-corp-dashboard-frontend/service-worker.js \
    --cache-control "no-cache, no-store, must-revalidate" 2>/dev/null || true
```

_Why the split caching strategy?_

- **Static assets** (`/static/js/main.abc123.js`): These filenames contain content hashes. If the code changes, the hash changes, generating a new filename. It is safe to cache these for 1 year (`max-age=31536000`) because the filename itself guarantees freshness.
- **`index.html`**: This file references the hashed asset filenames. If a user's browser caches `index.html`, they will continue loading the old JavaScript bundles even after a new deployment. Setting `no-cache` on `index.html` forces the browser to always fetch the latest version, which then points to the new hashed assets.

```bash
# Invalidate the CloudFront cache to force edge nodes to fetch the new index.html
aws cloudfront create-invalidation \
    --distribution-id E1A2B3C4D5E6F7 \
    --paths "/index.html" "/service-worker.js"
```

_Validation:_

```bash
# Check the CloudFront distribution URL
curl -I https://d1234567890.cloudfront.net/
# Expected: HTTP/2 200, x-cache: Hit from cloudfront (after first request)

# Verify the cache headers on a static asset
curl -I https://d1234567890.cloudfront.net/static/js/main.abc123.js
# Expected: Cache-Control: public, max-age=31536000, immutable

# Verify index.html is NOT cached
curl -I https://d1234567890.cloudfront.net/index.html
# Expected: Cache-Control: no-cache, no-store, must-revalidate
```

---

### 8.5 The Complete Deployment Runbook (Backend)

For reference, here is the condensed, sequential deployment checklist an engineer executes:

```bash
# 1. Connect to the server
aws ssm start-session --target i-0abc1234def56789

# 2. Navigate to the application directory
cd /opt/apps/auth-api

# 3. Pull latest code
git pull origin main

# 4. Install dependencies
npm ci --production

# 5. Build
npm run build

# 6. Regenerate .env from Secrets Manager / SSM
# (run the .env generation script from Section 8.2.6 Strategy 3)

# 7. Graceful reload (zero-downtime in cluster mode)
pm2 reload auth-api

# 8. Verify health
curl -s http://localhost:3000/health

# 9. Check logs for errors
pm2 logs auth-api --lines 50 --err

# 10. Save PM2 process list
pm2 save
```

---

### 8.6 Failure + Rollback Strategy (Mandatory)

If a deployment fails, you must be able to return to the last known-good version within minutes. This is non-negotiable for production systems.

#### Common Failure Signals

- Health check returns non-200 or times out after deploy.
- PM2 shows `errored` or repeated restarts.
- Logs show missing env vars, failed DB migrations, or runtime exceptions.

#### Fast Rollback (Single Server)

```bash
# Move back to the last known-good commit or tag
git fetch --all --tags
git checkout previous-version

# Reinstall dependencies and rebuild if required
npm ci --production
npm run build

# Restart the app
pm2 restart app
```

#### Real Scenarios and What to Do

1. **Bad environment variable:** Revert the `.env` file to the previous version and restart PM2.
2. **Broken build artifact:** `git checkout` the last known-good commit, rebuild, then restart PM2.
3. **Database migration broke production:** Roll forward with a fix or restore from backup if data was corrupted. Never `DROP` in a panic.

_Validation:_

```bash
curl -sf http://localhost:3000/health
pm2 status
```

---

### 8.7 Real-World Deployment Mistakes

#### Mistake 1: The Forgotten `npm run build`

- _The Scenario:_ A developer pulls the latest code, runs `pm2 reload`, and walks away. They forgot to run `npm run build` after pulling the new TypeScript source code.
- _The Impact:_ PM2 reloads the old compiled JavaScript from the `dist/` directory. The "new deployment" is actually the previous version running. The developer marks the deployment as complete in Jira. Two days later, QA reports that the new feature is missing. The team spends hours debugging before realizing the build step was skipped entirely.
- _The Mandate:_ Deployment scripts must be atomic and sequential. The `build` step must complete successfully (exit code 0) before the `reload` step is triggered. This is trivially enforced in a CI/CD pipeline but easily forgotten in manual deployments.

#### Mistake 2: The `.env` File Committed to Git

- _The Scenario:_ A developer creates a `.env` file on their laptop with production database credentials for local testing. They forget to add `.env` to `.gitignore`. They commit and push. The credentials are now in the Git history forever—even if they immediately delete the file and push again.
- _The Impact:_ Anyone with repository access (including former employees, open-source contributors if the repo is public, or compromised CI/CD runners) can extract production database credentials from the Git history. The database is compromised.
- _The Mandate:_ (1) `.env` must be in `.gitignore` from the first commit of any repository. (2) Use `git-secrets` or `trufflehog` as a pre-commit hook to scan for credential patterns. (3) If a secret is ever committed, immediately rotate the credential in AWS Secrets Manager—do not just delete the commit.

#### Mistake 3: The `npm install` vs. `npm ci` Disaster

- _The Scenario:_ The CI pipeline runs `npm install` instead of `npm ci`. A transitive dependency releases a buggy patch version between the time the developer tested locally and the time the CI pipeline runs. The `package-lock.json` is ignored, and a different version of a critical library is installed on the production server.
- _The Impact:_ The application crashes in production with a cryptic error from a dependency the developer never directly imported. Debugging takes 6 hours because the error cannot be reproduced locally (where the original, working version is cached in `node_modules`).
- _The Mandate:_ `npm ci` is the only acceptable command for CI/CD and production deployments. It guarantees a deterministic, reproducible install from `package-lock.json`.

#### Mistake 4: Frontend Deployment Without CloudFront Invalidation

- _The Scenario:_ The team deploys a critical bug fix to the React frontend. They run `aws s3 sync` to upload the new build. They verify the fix by loading the site in an incognito window—it works. They announce the fix is live.
- _The Impact:_ CloudFront edge nodes are still serving the old, cached `index.html` from 20+ global points of presence. 80% of users in other geographic regions continue to experience the bug for up to 24 hours (CloudFront's default TTL). The support team is flooded with tickets.
- _The Mandate:_ Every S3 frontend deployment must be immediately followed by a CloudFront cache invalidation for `/index.html`. The deployment is not complete until the invalidation status transitions to `Completed`.

#### Mistake 5: Running PM2 as Root

- _The Scenario:_ An engineer runs `sudo pm2 start ecosystem.config.js` to avoid permission issues when binding to port 80.
- _The Impact:_ The Node.js application now runs as root. If an attacker exploits an application vulnerability (SSRF, prototype pollution, RCE), they gain root-level access to the entire operating system. They can read `/etc/shadow`, install rootkits, pivot to other instances via the instance's IAM role, or wipe the filesystem.
- _The Mandate:_ Node.js applications must never run as root. Use Nginx as a reverse proxy to listen on port 80/443 and forward traffic to the Node.js app on a high port (3000, 8080). If you absolutely must bind to a privileged port without Nginx, use `setcap` to grant the Node.js binary the `CAP_NET_BIND_SERVICE` capability: `sudo setcap 'cap_net_bind_service=+ep' $(which node)`.

## 9. Scaling Strategies (Vertical & Horizontal)

### Purpose

Ensure the system can scale safely under load without sacrificing availability or cost control.

### What You Will Achieve

- Understand vertical vs. horizontal scaling trade-offs
- Configure Auto Scaling Groups and ALBs correctly
- Apply the right scaling strategy for real traffic patterns

### Step-by-Step Implementation

1. Evaluate vertical scaling limits (Section 9.2).
2. Ensure app readiness for horizontal scaling (Section 9.3).
3. Create ASGs and scaling policies (Section 9.4).
4. Configure ALB and target groups (Section 9.5).

### Commands

All ASG, ALB, and scaling commands are provided inline.

### Validation

Verify healthy targets, scaling events, and alarm-driven policies are working.

### Common Errors

- Scaling a stateful app without externalizing sessions.
- Setting max size too high and incurring runaway costs.

### Pro Tips

- Pre-scale for predictable events.
- Use WAF + rate limits to avoid scaling during attacks.

### 9.1 The Scaling Imperative: Why This Section Can Save the Company

Scaling is the engineering discipline of ensuring that an application continues to serve users reliably and performantly as demand increases—whether that increase is gradual organic growth or an instantaneous 100x traffic spike from a viral social media post, a flash sale, or a DDoS attack. A system that cannot scale is a system with a countdown timer. The only question is when it breaks, not if.

In the cloud, scaling is not just a technical concern—it is a financial one. Over-provisioning (running 20 servers when you need 3) bleeds money. Under-provisioning (running 3 servers when you need 20) bleeds customers. The objective is **elastic scaling**: the infrastructure automatically expands under load and contracts when load recedes, aligning cloud spend precisely with business demand.

---

### 9.2 Vertical Scaling (Scale Up / Scale Down)

#### What is it?

Vertical scaling means increasing the capacity of a single machine. You replace a small instance with a larger one: more CPU cores, more RAM, more network bandwidth, more I/O throughput. In AWS, this means changing the EC2 instance type (e.g., `t3.medium` → `m5.2xlarge`) or the RDS instance class (e.g., `db.m5.large` → `db.m5.4xlarge`).

#### Why is it the simplest form of scaling?

It requires zero architectural changes to the application. A single-threaded, stateful monolithic application that runs on one server will run identically on a bigger server—just faster. There is no need to implement load balancing, session management, or distributed caching. The application code remains untouched.

#### When to use Vertical Scaling:

- **Databases (RDS/Aurora):** Relational databases are inherently difficult to horizontally scale for writes. Vertical scaling is often the first and most effective response to database performance degradation. Doubling the RAM on an RDS instance can dramatically improve query performance because PostgreSQL/MySQL use free memory for buffer pool caching—more RAM means more data cached in memory, fewer disk reads.
- **Legacy monoliths:** Applications that maintain in-memory state (e.g., in-process session stores, local caches) and cannot be trivially distributed across multiple nodes.
- **Quick emergency response:** When production is on fire and you need immediate relief, scaling up a single instance is faster than re-architecting the application for horizontal scaling.

#### When NOT to use Vertical Scaling:

- **As a long-term strategy.** Every instance family has a maximum size. The largest general-purpose instance (`m7i.48xlarge`) has 192 vCPUs and 768 GiB RAM. If your workload outgrows this, vertical scaling hits a hard ceiling.
- **When high availability is critical.** Vertical scaling creates a Single Point of Failure (SPOF). If that one massive instance crashes, there is no failover. The application is completely offline until a new instance is provisioned.
- **For stateless, horizontally scalable applications.** If your Node.js API is stateless (no in-memory sessions, no local file storage), horizontal scaling with an Auto Scaling Group is architecturally superior, cheaper, and more resilient.

#### The Downtime Problem

Vertical scaling on EC2 requires **stopping** the instance, changing the instance type, and **starting** it again. This causes downtime—typically 2-5 minutes for EC2, and 5-15 minutes for RDS (unless Multi-AZ is enabled, which reduces it to approximately 60 seconds via automated failover).

```bash
# EC2: Stop, resize, start
aws ec2 stop-instances --instance-ids i-0abc1234def56789
aws ec2 wait instance-stopped --instance-ids i-0abc1234def56789

aws ec2 modify-instance-attribute \
    --instance-id i-0abc1234def56789 \
    --instance-type '{"Value": "m5.2xlarge"}'

aws ec2 start-instances --instance-ids i-0abc1234def56789
aws ec2 wait instance-running --instance-ids i-0abc1234def56789
```

_Validation:_

```bash
aws ec2 describe-instances \
    --instance-ids i-0abc1234def56789 \
    --query 'Reservations[0].Instances[0].[InstanceType, State.Name]' \
    --output text
# Expected: m5.2xlarge  running
```

#### The Vertical Scaling Limits Table

| Dimension | Practical Ceiling                        | What Happens at the Ceiling                             |
| --------- | ---------------------------------------- | ------------------------------------------------------- |
| CPU       | 192 vCPUs (`m7i.48xlarge`)               | Cannot scale further. Must distribute workload.         |
| Memory    | 24 TiB (`u-24tb1.112xlarge` High Memory) | Extremely expensive. Reserved for SAP HANA workloads.   |
| Network   | 200 Gbps (`p5.48xlarge`)                 | Diminishing returns. Application rarely saturates this. |
| EBS I/O   | 260,000 IOPS (`io2 Block Express`)       | Must move to instance store NVMe for higher I/O.        |

---

### 9.3 Horizontal Scaling (Scale Out / Scale In)

#### What is it?

Horizontal scaling means adding more instances of the same size behind a load balancer. Instead of one big server, you run ten identical small servers. Traffic is distributed across all of them. If one crashes, the remaining nine continue serving users.

#### Why is this the enterprise standard?

- **Eliminates SPOF:** No single instance failure can take down the application.
- **Infinitely extensible:** There is no theoretical ceiling. You can add hundreds of instances.
- **Cost-efficient:** You scale in (remove instances) during off-peak hours, paying only for what you actively use.
- **Enables zero-downtime deployments:** Rolling updates replace one instance at a time while the others continue serving traffic.

#### When to use Horizontal Scaling:

- Stateless HTTP APIs (the vast majority of microservices).
- Web servers, API gateways, and backend-for-frontend (BFF) layers.
- Worker processes consuming from SQS queues (each worker instance processes messages independently).

#### When NOT to use Horizontal Scaling:

- **Stateful applications with local storage.** If your application writes user session data to local disk or maintains an in-memory cache that is not synchronized across instances, each instance will have a different view of the state. Users will get inconsistent responses depending on which instance handles their request. You must externalize state to ElastiCache (Redis) or DynamoDB before horizontal scaling is viable.
- **Single-writer databases.** You cannot horizontally scale RDS write traffic by adding more RDS instances. All writes must go to a single primary. (Read traffic can be horizontally scaled via Read Replicas, as covered in Section 7.6.)

#### The Prerequisite: Application Readiness Checklist

Before enabling horizontal scaling, your application must satisfy all of the following:

1. **Stateless:** No in-memory sessions. Sessions are stored in Redis or a database.
2. **No local file dependencies:** Uploaded files go directly to S3, not `/tmp` on the instance.
3. **Health check endpoint:** A `/health` route that returns HTTP 200 when the app is ready to serve traffic.
4. **Graceful shutdown:** The application handles `SIGTERM` by stopping new request acceptance and draining in-flight connections before exiting.
5. **Externalized configuration:** Environment variables from SSM/Secrets Manager, not hardcoded in local files that differ per instance.

---

### 9.4 Auto Scaling Groups (ASG): The Engine of Elasticity

#### What is it?

An Auto Scaling Group (ASG) is an AWS service that automatically manages a fleet of EC2 instances. You define a **Launch Template** (the blueprint for each instance: AMI, instance type, Security Groups, user data script) and a **Scaling Policy** (the rules that determine when to add or remove instances). The ASG continuously monitors CloudWatch metrics and adjusts the fleet size to match demand.

#### Why is this non-negotiable for production?

Without an ASG, a single EC2 instance is a fragile, mortal pet. With an ASG, your infrastructure becomes a self-healing, auto-scaling organism. If an instance fails a health check, the ASG automatically terminates it and launches a replacement. If CPU usage spikes, the ASG adds more instances. If traffic drops at 3 AM, the ASG removes excess instances to save money.

#### Step 1: Create a Launch Template

```bash
aws ec2 create-launch-template \
    --launch-template-name prod-api-launch-template \
    --version-description "v1 - Golden AMI with Node.js 22 and Nginx" \
    --launch-template-data '{
      "ImageId": "ami-golden-v1-2026-05-01",
      "InstanceType": "m5.large",
      "KeyName": "prod-ssh-keypair",
      "SecurityGroupIds": ["sg-app-prod"],
      "IamInstanceProfile": {"Name": "EC2-SSM-Role"},
      "UserData": "'$(base64 -w 0 <<'USERDATA'
#!/bin/bash
cd /opt/apps/auth-api
git pull origin main
npm ci --production
npm run build
# Generate .env from SSM
cat > .env <<EOF
NODE_ENV=production
PORT=3000
DATABASE_URL=$(aws ssm get-parameter --name "/prod/auth-api/DATABASE_URL" --with-decryption --query 'Parameter.Value' --output text)
JWT_SECRET=$(aws ssm get-parameter --name "/prod/auth-api/JWT_SECRET" --with-decryption --query 'Parameter.Value' --output text)
EOF
pm2 start ecosystem.config.js
pm2 save
USERDATA
)'",
      "TagSpecifications": [{
        "ResourceType": "instance",
        "Tags": [
          {"Key": "Name", "Value": "prod-api-asg-instance"},
          {"Key": "Environment", "Value": "Production"},
          {"Key": "Service", "Value": "AuthAPI"}
        ]
      }]
    }'
```

#### Step 2: Create the Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name prod-api-asg \
    --launch-template LaunchTemplateName=prod-api-launch-template,Version='$Latest' \
    --min-size 2 \
    --max-size 10 \
    --desired-capacity 2 \
    --vpc-zone-identifier "subnet-private-1a,subnet-private-1b" \
    --health-check-type ELB \
    --health-check-grace-period 300 \
    --target-group-arns "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-api-tg/abc123" \
    --tags Key=Environment,Value=Production,PropagateAtLaunch=true
```

_Critical Flags Explained:_

- `--min-size 2`: The ASG will never go below 2 instances, even during absolute minimum traffic. This guarantees High Availability across two AZs.
- `--max-size 10`: The absolute ceiling. Even if traffic spikes infinitely, the ASG will not exceed 10 instances (to prevent runaway costs from a misconfigured scaling policy or a DDoS attack).
- `--health-check-type ELB`: Instead of relying on basic EC2 status checks (which only verify the hypervisor, not the application), the ASG uses the Load Balancer's health checks. If the ALB's `/health` endpoint returns 5xx for 3 consecutive checks, the ASG considers the instance unhealthy and replaces it.
- `--health-check-grace-period 300`: After launching a new instance, the ASG waits 300 seconds (5 minutes) before starting health checks. This gives the application time to boot, install dependencies, and start PM2.

_Validation:_

```bash
aws autoscaling describe-auto-scaling-groups \
    --auto-scaling-group-names prod-api-asg \
    --query 'AutoScalingGroups[0].[MinSize, MaxSize, DesiredCapacity, Instances[*].[InstanceId, HealthStatus, LifecycleState]]' \
    --output table
```

#### Step 3: Configure Scaling Policies

**Target Tracking Scaling (Recommended):** You define a target metric value, and AWS automatically adjusts the fleet size to maintain it.

```bash
# Scale to maintain average CPU utilization at 60%
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name prod-api-asg \
    --policy-name cpu-target-tracking \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration '{
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ASGAverageCPUUtilization"
      },
      "TargetValue": 60.0,
      "ScaleInCooldown": 300,
      "ScaleOutCooldown": 60
    }'
```

_Why 60% target and not 80%?_ Scaling out is not instant. It takes 3-5 minutes to launch a new instance, boot the OS, pull the code, build, and pass health checks. If you set the target to 80%, a sudden spike will push the existing instances to 100% before the new instances are ready, causing latency spikes or timeouts. A 60% target provides headroom for the scaling lag.

_Why asymmetric cooldowns?_ `ScaleOutCooldown: 60` means the ASG can add instances rapidly (check every 60 seconds during a spike). `ScaleInCooldown: 300` means the ASG waits 5 minutes before removing instances, preventing "flapping" where instances are repeatedly added and removed during fluctuating traffic.

---

### 9.5 Application Load Balancer (ALB): The Traffic Director

#### What is it?

The Application Load Balancer (ALB) operates at Layer 7 (HTTP/HTTPS). It receives all incoming client requests on a single DNS endpoint and distributes them across the healthy instances registered in a Target Group. It inspects HTTP headers, paths, and host headers to make intelligent routing decisions.

> ⚠️ Cost Warning: ALBs are billed per hour and per LCU (Load Balancer Capacity Unit). Even idle ALBs in dev/staging incur charges. Delete unused ALBs and consolidate where possible.

#### Why is it mandatory for horizontal scaling?

Without a load balancer, there is no way to distribute traffic across multiple instances. Each instance has its own IP address. You would need to expose all 10 IPs to clients and rely on client-side DNS round-robin—which is unreliable, does not perform health checks, and exposes your internal infrastructure topology.

#### When to use ALB:

- Any HTTP/HTTPS application with multiple backend instances.
- When you need path-based routing (e.g., `/api/*` → API Target Group, `/admin/*` → Admin Target Group).
- When you need host-based routing (e.g., `api.acme.com` → API instances, `dashboard.acme.com` → Dashboard instances).
- When you need WebSocket support, HTTP/2, gRPC, or sticky sessions.

#### When NOT to use ALB:

- **Raw TCP/UDP traffic** (non-HTTP protocols like MQTT, custom binary protocols, gaming servers). Use a Network Load Balancer (NLB) instead, which operates at Layer 4 and passes raw TCP/UDP packets without HTTP inspection.
- **Extreme low-latency requirements** (<1ms added latency). NLB has lower latency because it does not inspect HTTP headers.

#### Step 1: Create the ALB

```bash
# Create the ALB in public subnets
aws elbv2 create-load-balancer \
    --name prod-api-alb \
    --subnets subnet-public-1a subnet-public-1b \
    --security-groups sg-alb-prod \
    --scheme internet-facing \
    --type application \
    --tags Key=Environment,Value=Production
```

#### Step 2: Create a Target Group

```bash
aws elbv2 create-target-group \
    --name prod-api-tg \
    --protocol HTTP \
    --port 80 \
    --vpc-id vpc-0abcd1234 \
    --target-type instance \
    --health-check-protocol HTTP \
    --health-check-path /health \
    --health-check-interval-seconds 15 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3
```

_Health Check Configuration Explained:_

- **Path `/health`:** The ALB sends an HTTP GET to `/health` on each registered instance.
- **Interval 15s:** The ALB checks every 15 seconds.
- **Unhealthy Threshold 3:** If an instance fails 3 consecutive checks (45 seconds total), the ALB stops sending traffic to it and the ASG is notified to replace it.
- **Healthy Threshold 2:** A newly launched instance must pass 2 consecutive checks (30 seconds) before receiving traffic.

#### Step 3: Create an HTTPS Listener

```bash
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/prod-api-alb/abc123 \
    --protocol HTTPS \
    --port 443 \
    --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
    --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/xyz-cert \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-api-tg/abc123

# Redirect HTTP (port 80) to HTTPS (port 443)
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/prod-api-alb/abc123 \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

_Validation:_

```bash
# Check ALB DNS and test connectivity
ALB_DNS=$(aws elbv2 describe-load-balancers --names prod-api-alb --query 'LoadBalancers[0].DNSName' --output text)
curl -I https://$ALB_DNS/health
# Expected: HTTP/2 200

# Check registered targets and their health
aws elbv2 describe-target-health \
    --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-api-tg/abc123 \
    --output table
# Expected: All targets show "healthy"
```

---

### 9.6 Real-World Traffic Spike Scenarios

#### Scenario 1: The Viral Product Launch

- _The Situation:_ An e-commerce company launches a limited-edition product at 12:00 PM. Marketing sends push notifications to 2 million users simultaneously. Traffic jumps from 500 RPS to 50,000 RPS in 30 seconds.
- _What happens WITHOUT scaling:_ The 2 EC2 instances immediately saturate at 100% CPU. Requests queue up. Latency jumps from 100ms to 30 seconds. Users see timeout errors. 90% of purchase attempts fail. The company loses hundreds of thousands of dollars in revenue.
- _What happens WITH proper scaling:_ The ASG's Target Tracking policy detects CPU averaging 60% within 60 seconds. It begins launching new instances. Within 5 minutes, the fleet grows from 2 to 8 instances. However, 5 minutes is too slow for a 30-second spike.
- _The Lesson:_ For predictable spikes (scheduled flash sales, marketing campaigns, live events), use **Scheduled Scaling** to pre-warm the fleet before the event.

```bash
# Pre-scale the ASG 30 minutes before the launch
aws autoscaling put-scheduled-update-group-action \
    --auto-scaling-group-name prod-api-asg \
    --scheduled-action-name pre-launch-scale-out \
    --start-time "2026-05-01T11:30:00Z" \
    --desired-capacity 10 \
    --min-size 10

# Scale back down 2 hours after the launch
aws autoscaling put-scheduled-update-group-action \
    --auto-scaling-group-name prod-api-asg \
    --scheduled-action-name post-launch-scale-in \
    --start-time "2026-05-01T14:00:00Z" \
    --desired-capacity 2 \
    --min-size 2
```

#### Scenario 2: The Unexpected Organic Spike

- _The Situation:_ A SaaS company's product is featured in a major tech publication at 9 AM on a Tuesday. Nobody was warned. Traffic increases 20x over 2 hours.
- _What happens WITH Target Tracking:_ The ASG gradually adds instances over the 2-hour ramp. Because the increase is gradual (not instantaneous), the scaling policy has time to react. The fleet grows from 2 → 4 → 6 → 8 instances smoothly. Users experience slightly elevated latency (200ms vs. 100ms) during the first 10 minutes but no errors.
- _The Lesson:_ Target Tracking handles gradual ramps well. It struggles with instantaneous spikes. For truly unpredictable, instantaneous spikes, consider **Predictive Scaling** (machine learning-based) or over-provisioning with a higher `min-size`.

#### Scenario 3: The DDoS Attack Masquerading as Legitimate Traffic

- _The Situation:_ An attacker sends 100,000 RPS of seemingly legitimate HTTP GET requests to the API. The ASG faithfully scales from 2 to 10 instances (the `max-size`). The instances are overwhelmed. The attacker continues.
- _What happens WITHOUT limits:_ If `max-size` is set too high (e.g., 100), the ASG will scale to 100 instances, costing $10,000+/day, while the attacker sustains the attack.
- _The Lesson:_ (1) Always set a hard `max-size` ceiling that aligns with your budget, not your optimism. (2) Deploy AWS WAF (Web Application Firewall) in front of the ALB to rate-limit by IP, block known bot signatures, and geo-restrict traffic. (3) Enable AWS Shield Advanced for automatic DDoS mitigation on the ALB.

---

### 9.7 Failure Handling: Self-Healing Infrastructure

#### Instance Health Check Failure

- _Trigger:_ The ALB's `/health` endpoint returns HTTP 500 three consecutive times.
- _ASG Response:_ The ASG marks the instance as `Unhealthy`, terminates it, and launches a fresh replacement from the Launch Template. The replacement instance boots, pulls the latest code via `UserData`, starts PM2, and is registered with the Target Group after passing health checks.
- _Result:_ The entire recovery happens automatically within 5-7 minutes. No human intervention required. No pager notification for a single instance failure.

#### Availability Zone Failure

- _Trigger:_ An entire AZ (`us-east-1a`) experiences a power outage.
- _ASG Response:_ The ASG detects that all instances in `us-east-1a` are unhealthy. It immediately launches replacement instances in `us-east-1b` (the other AZ in the `vpc-zone-identifier`). The ALB automatically stops routing traffic to the dead AZ and directs 100% of traffic to the healthy AZ.
- _Result:_ Users experience a brief increase in latency (due to reduced capacity while new instances boot) but no hard downtime.

#### Cascading Failure (Database Overload)

- _Trigger:_ The database becomes overloaded (connection pool exhaustion, slow query lock contention). The application's `/health` endpoint starts timing out because it includes a database connectivity check.
- _ASG Response:_ The ASG sees all instances as `Unhealthy`. It begins terminating and replacing them. The new instances also fail the health check because the database is still overloaded. The ASG enters a death spiral—continuously terminating and launching instances.
- _The Mitigation:_ (1) The `/health` endpoint should have a **shallow** mode (checks only if the Node.js process is alive) and a **deep** mode (checks database connectivity). The ALB should use the shallow health check. (2) Set `health-check-grace-period` to at least 300 seconds to prevent premature termination. (3) Implement circuit breakers in the application code (e.g., using `opossum` for Node.js) that fail fast instead of hanging on database connections.

---

### 9.8 Scaling Decision Matrix: Quick Reference

| Signal                                      | Action                                               |
| ------------------------------------------- | ---------------------------------------------------- |
| CPU > 60% sustained for 5 min               | Scale out (add instances via ASG)                    |
| CPU < 20% sustained for 15 min              | Scale in (remove instances via ASG)                  |
| Memory > 80%                                | Scale up (increase instance type) OR fix memory leak |
| Request latency p99 > 2s                    | Scale out AND investigate application bottleneck     |
| Database connections near `max_connections` | Add read replicas OR scale up RDS instance class     |
| Predictable spike (flash sale, launch)      | Scheduled scaling 30 min before the event            |
| Sudden unpredictable spike                  | Target Tracking + higher `min-size` baseline         |
| DDoS-pattern traffic                        | WAF rate limiting + Shield Advanced, NOT scaling out |

## 10. CI/CD Pipeline

### Purpose

Automate build, test, and deployment to reduce human error and improve release speed.

### What You Will Achieve

- Understand CI/CD stages and failure modes
- Implement production-grade GitHub Actions pipelines
- Choose the correct deployment strategy (rolling vs. blue-green)

### Step-by-Step Implementation

1. Review pipeline architecture (Section 10.2).
2. Implement backend and frontend workflows (Sections 10.3 and 10.4).
3. Apply deployment strategies and safeguards (Sections 10.6 and 10.7).

### Commands

Pipeline commands are embedded in the workflow YAML examples.

### Validation

Confirm each pipeline stage completes and health checks pass post-deploy.

### Common Errors

- Using long-lived AWS keys instead of OIDC.
- Deploying from `main` without a pinned commit.

### Pro Tips

- Add manual approvals for production.
- Keep build artifacts immutable and versioned.

### 10.1 Why CI/CD Matters: The End of Manual Deployment

#### What is CI/CD?

**Continuous Integration (CI)** is the practice of automatically building and testing every code change the moment it is pushed to a version control repository. **Continuous Deployment (CD)** extends this by automatically deploying the validated artifact to production (or staging) environments without human intervention beyond the initial code review and merge approval.

#### Why is this non-negotiable in an enterprise?

Manual deployment, as documented in Section 8, is the training-wheels phase. It is acceptable for initial bootstrapping and emergency break-glass scenarios, but it is fundamentally incompatible with a mature engineering organization. Manual deployments suffer from three fatal pathologies:

1. **Human Error at Scale:** When a single engineer deploys once a week, the risk of forgetting `npm run build` or uploading the wrong `.env` file is manageable. When 15 engineers deploy 30 microservices across 3 environments daily, human error becomes a statistical certainty. CI/CD eliminates the human from the mechanical execution entirely.
2. **Audit Vacuum:** When an engineer SSHs into a production server and runs commands, there is no centralized, immutable record of exactly what was deployed, by whom, at what time, and from which Git commit. CI/CD pipelines produce a complete, tamper-proof audit trail for every deployment—satisfying SOC2, HIPAA, and PCI-DSS compliance requirements.
3. **Velocity Strangulation:** If deployment requires a senior engineer to spend 45 minutes executing a manual runbook, the organization can only deploy a few times per day at best. High-performing teams deploy dozens to hundreds of times per day. CI/CD reduces deployment time from 45 minutes to 3 minutes—fully automated.

#### When to implement CI/CD:

- Immediately. From the very first commit to the repository. Even a simple pipeline that just runs `npm test` on every push provides immense value by catching broken code before it reaches `main`.

#### When NOT to implement CI/CD:

- There is no legitimate scenario where CI/CD should be avoided. The only variable is the _depth_ of automation. A one-person prototype might have CI only (build and test). A production system must have full CI/CD with automated deployment and rollback.

---

### 10.2 The Build → Test → Deploy Pipeline Architecture

Every CI/CD pipeline, regardless of tooling (GitHub Actions, GitLab CI, Jenkins, AWS CodePipeline), follows the same fundamental three-stage architecture:

#### Stage 1: Build (Compile and Package)

- **What happens:** The source code is checked out from Git. Dependencies are installed (`npm ci`). The application is compiled (`npm run build` for TypeScript/React). A deployable artifact is produced (a `dist/` directory, a Docker image, or a zip file).
- **Failure mode:** If the build fails (syntax error, missing dependency, type error), the pipeline stops immediately. No deployment occurs.
- **Artifact output:** The build artifact must be versioned and stored in an artifact registry (S3, ECR, GitHub Packages) so that it can be traced back to the exact Git commit.

#### Stage 2: Test (Validate Correctness)

- **What happens:** Automated tests are executed against the build artifact. This includes unit tests, integration tests, linting (ESLint, Prettier), security scanning (npm audit, Snyk), and optionally end-to-end tests (Playwright, Cypress).
- **Failure mode:** If any test fails, the pipeline stops. The artifact is not deployed. The developer is notified via Slack/email with the specific test failure output.
- **The Golden Rule:** Tests must be fast and deterministic. A test suite that takes 30 minutes or produces flaky, non-deterministic results will be ignored by developers and defeated the purpose entirely.

#### Stage 3: Deploy (Ship to Environment)

- **What happens:** The validated artifact is deployed to the target environment (Staging or Production). This can be SSH + PM2 reload, `aws s3 sync` for frontends, `aws ecs update-service` for containers, or an ASG rolling update.
- **Failure mode:** If the deployment fails (health check failure, connection timeout), the pipeline triggers an automatic rollback to the previous known-good version.
- **The Gate:** Production deployments should require an explicit approval step (a manual button click in the pipeline UI or a Slack approval bot) to prevent unintentional pushes to production from a rogue merge.

---

### 10.2.1 Minimal GitHub Actions Deployment (SSH + PM2)

This is the smallest usable GitHub Actions workflow that deploys to a single EC2 server over SSH and restarts PM2.

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: ssh user@server "cd app && git pull && pm2 restart app"
```

**Step-by-step explanation:**

- `on.push.branches: [main]` triggers the workflow when code is pushed to `main`.
- `actions/checkout@v3` pulls your repository into the runner so it can access the code.
- `ssh user@server "cd app && git pull && pm2 restart app"` logs into the server, pulls the latest code, and restarts the app.

_Required setup:_ store your SSH private key in GitHub Secrets (e.g., `DEPLOY_KEY`) and use `ssh-agent` in the workflow, or configure GitHub Actions OIDC + SSM for passwordless, keyless deployments.

---

### 10.3 GitHub Actions: Production-Grade Pipeline (Backend)

Below is a complete, production-ready GitHub Actions workflow for our Node.js backend API. This is not a tutorial example—it is the actual template used in our enterprise repositories.

```yaml
# .github/workflows/deploy-backend.yml
name: Backend CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Prevent concurrent deployments to the same environment
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

env:
  NODE_VERSION: "22"
  AWS_REGION: "us-east-1"

# Required for OIDC authentication with AWS (no long-lived keys)
permissions:
  id-token: write
  contents: read

jobs:
  # ============================================================
  # STAGE 1: BUILD & LINT
  # ============================================================
  build:
    name: Build & Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install Dependencies (Deterministic)
        run: npm ci

      - name: Lint Code (ESLint + Prettier)
        run: npm run lint

      - name: Build Application
        run: npm run build

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact-${{ github.sha }}
          path: dist/
          retention-days: 7

  # ============================================================
  # STAGE 2: TEST
  # ============================================================
  test:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest
    needs: build
    services:
      # Spin up a PostgreSQL container for integration tests
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U testuser"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      NODE_ENV: test
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install Dependencies
        run: npm ci

      - name: Run Database Migrations
        run: npm run db:migrate

      - name: Run Unit Tests
        run: npm test -- --coverage --ci

      - name: Run Security Audit
        run: npm audit --production --audit-level=high

  # ============================================================
  # STAGE 3: DEPLOY TO STAGING
  # ============================================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: staging
      url: https://staging-api.acme-corp.com
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC - No Long-Lived Keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Download Build Artifact
        uses: actions/download-artifact@v4
        with:
          name: build-artifact-${{ github.sha }}
          path: dist/

      - name: Deploy to Staging via SSM Run Command
        run: |
          aws ssm send-command \
            --instance-ids "i-staging-instance-id" \
            --document-name "AWS-RunShellScript" \
            --parameters 'commands=[
              "cd /opt/apps/auth-api",
              "git pull origin main",
              "npm ci --production",
              "npm run build",
              "pm2 reload auth-api",
              "sleep 5",
              "curl -sf http://localhost:3000/health || exit 1"
            ]' \
            --output text

      - name: Validate Staging Health
        run: |
          sleep 30
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://staging-api.acme-corp.com/health)
          if [ "$STATUS" != "200" ]; then
            echo "Staging health check FAILED with status $STATUS"
            exit 1
          fi
          echo "Staging deployment validated successfully."

  # ============================================================
  # STAGE 4: DEPLOY TO PRODUCTION (Requires Manual Approval)
  # ============================================================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://api.acme-corp.com
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to Production via ASG Rolling Update
        run: |
          # Update the Launch Template with the new Git commit SHA as a tag
          aws ec2 create-launch-template-version \
            --launch-template-name prod-api-launch-template \
            --source-version '$Latest' \
            --version-description "deploy-${{ github.sha }}"

          # Trigger a rolling refresh of the ASG
          aws autoscaling start-instance-refresh \
            --auto-scaling-group-name prod-api-asg \
            --preferences '{
              "MinHealthyPercentage": 90,
              "InstanceWarmup": 300
            }'

      - name: Wait for Instance Refresh to Complete
        run: |
          echo "Waiting for ASG instance refresh to complete..."
          while true; do
            STATUS=$(aws autoscaling describe-instance-refreshes \
              --auto-scaling-group-name prod-api-asg \
              --query 'InstanceRefreshes[0].Status' \
              --output text)
            echo "Current status: $STATUS"
            if [ "$STATUS" == "Successful" ]; then
              echo "Production deployment completed successfully."
              break
            elif [ "$STATUS" == "Failed" ] || [ "$STATUS" == "Cancelled" ]; then
              echo "Production deployment FAILED. Initiating rollback."
              exit 1
            fi
            sleep 30
          done

      - name: Validate Production Health
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.acme-corp.com/health)
          if [ "$STATUS" != "200" ]; then
            echo "Production health check FAILED with status $STATUS"
            exit 1
          fi
          echo "Production is healthy. Deployment complete."
```

---

### 10.4 GitHub Actions: Production-Grade Pipeline (Frontend)

```yaml
# .github/workflows/deploy-frontend.yml
name: Frontend CI/CD Pipeline

on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "public/**"
      - "package.json"
      - "package-lock.json"

permissions:
  id-token: write
  contents: read

env:
  NODE_VERSION: "22"
  AWS_REGION: "us-east-1"
  S3_BUCKET: "acme-corp-dashboard-frontend"
  CLOUDFRONT_DISTRIBUTION_ID: "E1A2B3C4D5E6F7"

jobs:
  build-and-deploy:
    name: Build, Test & Deploy Frontend
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://dashboard.acme-corp.com
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install Dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Run Tests
        run: npm test -- --ci --watchAll=false

      - name: Build Production Bundle
        run: npm run build
        env:
          REACT_APP_API_URL: https://api.acme-corp.com
          REACT_APP_ENV: production

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-frontend-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy Static Assets to S3
        run: |
          # Upload hashed assets with aggressive caching
          aws s3 sync build/ s3://${{ env.S3_BUCKET }}/ \
            --delete \
            --cache-control "public, max-age=31536000, immutable" \
            --exclude "index.html" \
            --exclude "service-worker.js" \
            --exclude "asset-manifest.json"

          # Upload index.html with NO CACHE
          aws s3 cp build/index.html s3://${{ env.S3_BUCKET }}/index.html \
            --cache-control "no-cache, no-store, must-revalidate"

      - name: Invalidate CloudFront Cache
        run: |
          INVALIDATION_ID=$(aws cloudfront create-invalidation \
            --distribution-id ${{ env.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/index.html" "/service-worker.js" "/asset-manifest.json" \
            --query 'Invalidation.Id' \
            --output text)
          echo "Invalidation ID: $INVALIDATION_ID"

          # Wait for invalidation to complete
          aws cloudfront wait invalidation-completed \
            --distribution-id ${{ env.CLOUDFRONT_DISTRIBUTION_ID }} \
            --id $INVALIDATION_ID
          echo "CloudFront invalidation completed."

      - name: Validate Deployment
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://dashboard.acme-corp.com)
          if [ "$STATUS" != "200" ]; then
            echo "Frontend deployment validation FAILED with status $STATUS"
            exit 1
          fi
          echo "Frontend deployment validated successfully."
```

---

### 10.5 Critical Pipeline Design Decisions Explained

#### OIDC Authentication (No Long-Lived AWS Keys)

The pipeline uses `aws-actions/configure-aws-credentials@v4` with `role-to-assume`. This leverages OpenID Connect (OIDC) federation between GitHub and AWS IAM. GitHub generates a short-lived JWT token for each workflow run, and AWS STS exchanges it for temporary credentials. **No IAM Access Keys are stored as GitHub Secrets.** This eliminates the risk of key leakage entirely.

#### Concurrency Control

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

This prevents two pipeline runs from deploying simultaneously to the same environment. If a developer pushes commit A, and then immediately pushes commit B, the second run waits for the first to complete rather than canceling it (which could leave the environment in a half-deployed state).

#### Environment Protection Rules

The `environment: production` declaration in the deploy job triggers GitHub's Environment Protection Rules. This can be configured in the repository settings to require:

- Manual approval from designated reviewers before the deploy job starts.
- A minimum wait timer (e.g., 10 minutes) to give the team time to cancel if a bad merge was detected.
- Branch restrictions (only the `main` branch can deploy to production).

---

### 10.6 Deployment Strategies: Rolling vs. Blue-Green

#### 10.6.1 Rolling Deployment

##### What is it?

A rolling deployment replaces instances in the fleet incrementally—one at a time (or a small batch at a time). While some instances run the new version, others still run the old version. Traffic is gradually shifted as new instances pass health checks and old instances are terminated.

##### How it works with ASG:

The `start-instance-refresh` API (used in the pipeline above) orchestrates this automatically:

1. ASG launches a new instance from the updated Launch Template.
2. The new instance boots, pulls code, starts the application, and passes the ALB health check.
3. ASG terminates one old instance.
4. Repeat until all instances are running the new version.

##### When to use:

- Standard, routine deployments where brief version inconsistency is acceptable.
- Applications where the API contract between versions is backward-compatible (e.g., a new endpoint was added, but no existing endpoints changed their response schema).

##### When NOT to use:

- When the deployment includes a **breaking database migration** that is incompatible with the old code. During the rolling period, old-version instances will crash when they encounter the new schema.
- When you need the ability to **instantly roll back** the entire fleet. Rolling back a rolling deployment requires another full rolling cycle (10-15 minutes), not an instant switch.

##### Advantages:

- Zero downtime (at least `MinHealthyPercentage` of the fleet is always serving traffic).
- Low resource cost (you never run two full fleets simultaneously).

##### Disadvantages:

- During the rollout, both old and new versions are serving traffic simultaneously. If the API response format changed, clients may receive inconsistent responses.
- Rollback is slow (another full rolling cycle).

```bash
# Trigger a rolling instance refresh
aws autoscaling start-instance-refresh \
    --auto-scaling-group-name prod-api-asg \
    --preferences '{
      "MinHealthyPercentage": 90,
      "InstanceWarmup": 300,
      "MaxHealthyPercentage": 110
    }'

# Monitor the progress
aws autoscaling describe-instance-refreshes \
    --auto-scaling-group-name prod-api-asg \
    --query 'InstanceRefreshes[0].[Status, PercentageComplete, StatusReason]' \
    --output table
```

---

#### 10.6.2 Blue-Green Deployment

##### What is it?

Blue-Green deployment maintains two identical, independent environments: **Blue** (the current production fleet) and **Green** (the new version fleet). Both fleets are fully provisioned and running simultaneously. Traffic is routed entirely to Blue. The Green fleet is deployed, validated, and health-checked in complete isolation. Once Green is confirmed healthy, traffic is switched from Blue to Green in a single, atomic operation (typically a DNS swap or ALB Target Group switch). If Green has a problem, traffic is instantly switched back to Blue.

##### How it works with ALB Target Groups:

1. **Blue Target Group** (`prod-api-tg-blue`): Currently receiving 100% of production traffic via the ALB listener.
2. **Green Target Group** (`prod-api-tg-green`): A new set of instances running the new version, registered here. They receive zero production traffic but are accessible via an internal test URL.
3. **Validation:** QA or automated smoke tests hit the Green Target Group directly to verify the new version.
4. **Switch:** The ALB listener's default action is updated to forward traffic to the Green Target Group. Blue becomes the standby.

```bash
# Step 1: Deploy the new version to the Green Target Group
# (Launch new instances, register them with prod-api-tg-green)

# Step 2: Validate Green environment
curl -s https://green-internal.acme-corp.com/health
# Expected: {"status":"ok","version":"2.0.0"}

# Step 3: Switch traffic from Blue to Green (atomic)
aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/prod-api-alb/abc123/def456 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-api-tg-green/xyz789

# Step 4: Validate production
curl -s https://api.acme-corp.com/health
# Expected: {"status":"ok","version":"2.0.0"}

# Step 5: If something goes wrong, INSTANT rollback to Blue
aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/prod-api-alb/abc123/def456 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-api-tg-blue/abc123
```

##### When to use:

- High-risk deployments: major version releases, breaking API changes, database schema migrations.
- When instant rollback (measured in seconds, not minutes) is a hard requirement.
- Compliance-heavy environments requiring pre-production validation of the exact production artifact.

##### When NOT to use:

- Routine, low-risk deployments. Running two full production fleets simultaneously doubles your compute cost during the deployment window.
- When budget is extremely tight and the application is simple enough that rolling deployments suffice.

##### Advantages:

- **Instant rollback:** Switch the ALB listener back to Blue in under 5 seconds. No waiting for new instances to boot.
- **Zero-risk validation:** The Green environment is tested with real production-like traffic patterns before it receives a single real user request.
- **No version mixing:** At any given moment, 100% of traffic goes to either Blue or Green. Never both.

##### Disadvantages:

- **Double the cost** during the deployment window (both Blue and Green fleets are running).
- **Database schema complexity:** If the new version requires a schema migration, both Blue and Green must be compatible with the same database schema (backward-compatible migrations are mandatory).

---

### 10.7 Deployment Strategy Decision Matrix

| Factor                     | Rolling Deployment                                     | Blue-Green Deployment                     |
| -------------------------- | ------------------------------------------------------ | ----------------------------------------- |
| **Rollback Speed**         | Slow (5-15 min rolling cycle)                          | Instant (ALB listener switch, <5 sec)     |
| **Cost During Deploy**     | Low (same fleet, replaced incrementally)               | High (2x fleet running simultaneously)    |
| **Version Mixing**         | Yes (old + new during rollout)                         | No (100% Blue OR 100% Green)              |
| **Validation Before Live** | Limited (new instances serve real traffic immediately) | Full (Green tested in isolation)          |
| **Complexity**             | Low (native ASG feature)                               | Medium (requires dual Target Groups, DNS) |
| **Best For**               | Routine feature releases                               | Major releases, breaking changes          |

---

### 10.8 Real-World CI/CD Failures

#### Failure 1: The Self-Referencing Rollback Loop

- _The Scenario:_ A pipeline deploys a buggy version that crashes on startup. The ASG health check detects the failure and terminates the instance. The ASG launches a replacement—which pulls the same buggy code from `main` and also crashes. This loop continues indefinitely.
- _The Impact:_ The ASG continuously terminates and launches instances, consuming EC2 capacity and generating noise in CloudWatch alarms, while the application remains completely offline.
- _The Mandate:_ The Launch Template's `UserData` should pull a specific, pinned Git tag or commit SHA—not `main`. The pipeline should update the Launch Template with the exact deployed commit before triggering the instance refresh. If an instance fails, the ASG re-launches with the same known-good (or known-bad) pinned version, not whatever happens to be on `main` at that moment.

#### Failure 2: The Database Migration Race Condition

- _The Scenario:_ A developer adds a new column to the `users` table (`ALTER TABLE users ADD COLUMN avatar_url TEXT`). The pipeline runs the migration and then deploys the new code that reads `avatar_url`. During a rolling deployment, old-version instances (which do not know about `avatar_url`) crash because the ORM schema validation fails against the updated database.
- _The Impact:_ 50% of production traffic hits crashing old instances during the rolling period.
- _The Mandate:_ Database migrations must be **backward-compatible**. The migration should be deployed in a separate release _before_ the code that depends on it. Phase 1: Add the column (old code ignores it). Phase 2 (next release): Deploy the code that reads the column. Phase 3 (future release): Remove the old code path. This is called the "expand-contract" migration pattern.

#### Failure 3: The Forgotten `paths` Filter

- _The Scenario:_ A frontend pipeline is triggered on every push to `main`, regardless of what changed. A developer updates the `README.md` file. The pipeline runs a full build, test, S3 sync, and CloudFront invalidation—deploying the exact same frontend code with zero changes.
- _The Impact:_ Unnecessary compute costs, wasted CI/CD minutes, and CloudFront invalidation costs ($0.005 per path per invalidation—which adds up at 50 deployments per day).
- _The Mandate:_ Use `paths` filters in the workflow trigger to only run the pipeline when relevant source files change.

```yaml
on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "public/**"
      - "package.json"
      - "package-lock.json"
```

## 11. Monitoring & Logging

### Purpose

Ensure you can detect, diagnose, and respond to issues quickly using metrics and logs.

### What You Will Achieve

- Establish required monitoring metrics across layers
- Ship logs centrally to CloudWatch Logs
- Configure alarms and dashboards for production

### Step-by-Step Implementation

1. Define critical metrics and install the CloudWatch Agent (Sections 11.3 and 11.4).
2. Centralize logs and query them (Section 11.5).
3. Create alarms and dashboards (Sections 11.6 and 11.7).

### Commands

All CloudWatch CLI commands are included inline.

### Validation

Confirm metrics and log streams appear in CloudWatch within 2-3 minutes.

### Common Errors

- Missing memory/disk metrics because the agent is not installed.
- Alert fatigue from poorly tuned thresholds.

### Pro Tips

- Keep alarms minimal and actionable.
- Use structured JSON logs for faster incident response.

### 11.1 The Monitoring Imperative: You Cannot Fix What You Cannot See

#### What is monitoring?

Monitoring is the continuous, automated observation of infrastructure health, application performance, and business metrics. It transforms an opaque, invisible system into a transparent, diagnosable one. Without monitoring, every incident begins with the same catastrophic question: "Is something wrong? We don't know. A customer just called."

#### Why does monitoring fail in most organizations?

Because most teams treat monitoring as an afterthought—something bolted on after the application is already in production. They set up a single CPU alarm on an EC2 instance, declare monitoring "done," and move on. Then, at 2 AM, the application crashes because the disk filled up with unrotated PM2 logs, and nobody receives an alert because nobody was monitoring disk usage.

Monitoring is not a task. It is a system. And like any system, it must be designed, tested, and maintained with the same rigor as the application code itself.

#### The Three Pillars of Observability:

1. **Metrics:** Numerical time-series data points (CPU %, memory %, request count, error rate, latency percentiles). Metrics tell you _what_ is happening.
2. **Logs:** Detailed, timestamped text records of events (application errors, HTTP access logs, database query logs). Logs tell you _why_ something is happening.
3. **Traces:** End-to-end request flow tracking across distributed microservices (AWS X-Ray, Jaeger, OpenTelemetry). Traces tell you _where_ the bottleneck is in a multi-service call chain.

This section focuses on Metrics and Logs via Amazon CloudWatch—the native AWS observability platform.

---

### 11.2 CloudWatch: The Unified Observability Platform

#### What is CloudWatch?

Amazon CloudWatch is a centralized monitoring and observability service that collects metrics, logs, and events from nearly every AWS resource. When you launch an EC2 instance, RDS database, or ALB, AWS automatically pushes basic metrics (CPU, network, disk) to CloudWatch at no additional cost. You can then create dashboards to visualize them, alarms to alert on thresholds, and log queries to diagnose incidents.

#### CloudWatch Components:

| Component                   | What It Does                                                                    |
| --------------------------- | ------------------------------------------------------------------------------- |
| **CloudWatch Metrics**      | Time-series data points (e.g., EC2 CPUUtilization, ALB RequestCount).           |
| **CloudWatch Logs**         | Centralized log ingestion and storage (application logs, VPC Flow Logs).        |
| **CloudWatch Alarms**       | Threshold-based alerts that trigger SNS notifications, Auto Scaling, or Lambda. |
| **CloudWatch Dashboards**   | Visual dashboards with graphs, numbers, and maps for real-time status.          |
| **CloudWatch Log Insights** | SQL-like query language for searching and analyzing log data at scale.          |

---

### 11.3 What to Monitor: The Enterprise Metric Taxonomy

Most teams monitor too little, too late. Below is the definitive list of metrics that must be monitored for every production system, organized by infrastructure layer.

#### Layer 1: Compute (EC2 / ECS / Lambda)

| Metric                        | CloudWatch Name           | Alert Threshold | Why It Matters                                                     |
| ----------------------------- | ------------------------- | --------------- | ------------------------------------------------------------------ |
| CPU Utilization               | `CPUUtilization`          | > 70% for 5 min | Sustained high CPU indicates under-provisioning or a runaway loop. |
| Memory Utilization            | Custom (CloudWatch Agent) | > 80%           | Not a default metric. Must install the CloudWatch Agent.           |
| Disk Usage                    | Custom (CloudWatch Agent) | > 85%           | Disk full = application crash, log write failure, DB corruption.   |
| CPU Credit Balance (T-family) | `CPUCreditBalance`        | < 20 credits    | Approaching zero means imminent CPU throttling.                    |
| Instance Status Check         | `StatusCheckFailed`       | > 0             | Hypervisor-level failure. Instance is compromised.                 |

#### Layer 2: Load Balancer (ALB)

| Metric               | CloudWatch Name             | Alert Threshold | Why It Matters                                                           |
| -------------------- | --------------------------- | --------------- | ------------------------------------------------------------------------ |
| HTTP 5xx Errors      | `HTTPCode_Target_5XX_Count` | > 10 in 1 min   | Backend application is crashing or returning errors.                     |
| HTTP 4xx Errors      | `HTTPCode_Target_4XX_Count` | > 100 in 5 min  | Potential bot attack, API abuse, or broken client integration.           |
| Target Response Time | `TargetResponseTime`        | p99 > 2 seconds | Application is slow. Investigate queries, connection pool, or CPU.       |
| Healthy Host Count   | `HealthyHostCount`          | < 2             | Fewer than 2 healthy instances = no redundancy, single point of failure. |
| Unhealthy Host Count | `UnHealthyHostCount`        | > 0             | An instance is failing health checks. ASG should replace it.             |

#### Layer 3: Database (RDS)

| Metric               | CloudWatch Name          | Alert Threshold      | Why It Matters                                                    |
| -------------------- | ------------------------ | -------------------- | ----------------------------------------------------------------- |
| CPU Utilization      | `CPUUtilization`         | > 70% for 10 min     | Query optimization needed before scaling up.                      |
| Freeable Memory      | `FreeableMemory`         | < 500 MiB            | Buffer pool is starved. Scale up instance class.                  |
| Database Connections | `DatabaseConnections`    | > 80% of max         | Connection pool starvation imminent. Deploy PgBouncer.            |
| Read/Write IOPS      | `ReadIOPS` / `WriteIOPS` | Near volume baseline | Storage bottleneck. Upgrade to gp3 with provisioned IOPS.         |
| Replica Lag          | `ReplicaLag`             | > 30 seconds         | Read replicas falling behind. Scale replica or reduce write load. |
| Free Storage Space   | `FreeStorageSpace`       | < 10 GiB             | Database will crash if disk fills. Enable storage autoscaling.    |

#### Layer 4: Application (Custom Metrics)

These are the most important metrics and the ones most teams neglect entirely. AWS cannot monitor your application logic—you must instrument it yourself.

| Metric                       | Source                               | Alert Threshold        | Why It Matters                                        |
| ---------------------------- | ------------------------------------ | ---------------------- | ----------------------------------------------------- |
| Request Rate (RPS)           | Application / ALB `RequestCount`     | Sudden 10x spike       | Traffic anomaly—viral event or DDoS.                  |
| Error Rate (% of 5xx)        | Application logs                     | > 1% of total requests | Business-impacting failures. Investigate immediately. |
| Latency p50 / p95 / p99      | Application middleware               | p99 > 2 seconds        | Tail latency affects the worst user experiences.      |
| Queue Depth (SQS)            | `ApproximateNumberOfMessagesVisible` | Growing continuously   | Workers cannot keep up. Scale out consumers.          |
| Active WebSocket Connections | Application                          | Near instance limit    | Connection saturation.                                |

---

### 11.4 Installing the CloudWatch Agent for Memory & Disk Metrics

AWS does not collect memory or disk usage by default because these are OS-level metrics that the hypervisor cannot see. You must install the CloudWatch Unified Agent on every EC2 instance.

```bash
# Download and install the CloudWatch Agent (Amazon Linux 2023)
sudo dnf install amazon-cloudwatch-agent -y

# Create the agent configuration
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<'EOF'
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "metrics": {
    "namespace": "CustomEC2Metrics",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/"],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF

# Start the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
    -s
```

_Validation:_

```bash
# Check the agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
# Expected: "status": "running"

# Verify metrics are appearing in CloudWatch (wait 2-3 minutes)
aws cloudwatch list-metrics --namespace CustomEC2Metrics --output table
# Expected: mem_used_percent and disk_used_percent metrics are listed
```

---

### 11.5 CloudWatch Logs: Centralized Log Management

#### Why centralize logs?

When your application runs on a single EC2 instance, you can SSH in and `tail -f` the log file. When your application runs on 10 auto-scaled instances (which are continuously terminated and replaced), the logs die with the instance. CloudWatch Logs provides a durable, centralized, searchable log store that survives instance termination.

#### Sending Application Logs to CloudWatch

**Method 1: CloudWatch Agent (Recommended for PM2 / File-Based Logs)**

Add a `logs` section to the CloudWatch Agent configuration:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/pm2/auth-api-out.log",
            "log_group_name": "/prod/auth-api/application",
            "log_stream_name": "{instance_id}/stdout",
            "timezone": "UTC",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/pm2/auth-api-error.log",
            "log_group_name": "/prod/auth-api/errors",
            "log_stream_name": "{instance_id}/stderr",
            "timezone": "UTC",
            "retention_in_days": 90
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/prod/nginx/access",
            "log_stream_name": "{instance_id}/access",
            "timezone": "UTC",
            "retention_in_days": 30
          }
        ]
      }
    }
  }
}
```

#### Viewing Logs via CLI

```bash
# List all log groups
aws logs describe-log-groups --query 'logGroups[*].logGroupName' --output table

# View the most recent log events from a specific log group
aws logs tail /prod/auth-api/errors --follow

# View logs from the last 1 hour
aws logs tail /prod/auth-api/application --since 1h

# Filter logs for a specific error pattern
aws logs filter-log-events \
    --log-group-name /prod/auth-api/errors \
    --filter-pattern "ERROR" \
    --start-time $(date -u -d '1 hour ago' +%s000) \
    --query 'events[*].[timestamp, message]' \
    --output text
```

#### CloudWatch Logs Insights: SQL-Like Log Analysis

Logs Insights allows you to write structured queries against your log data. This is indispensable during incident response.

```bash
# Find the top 10 most frequent error messages in the last 24 hours
aws logs start-query \
    --log-group-name /prod/auth-api/errors \
    --start-time $(date -u -d '24 hours ago' +%s) \
    --end-time $(date -u +%s) \
    --query-string '
      fields @timestamp, @message
      | filter @message like /ERROR/
      | stats count(*) as errorCount by @message
      | sort errorCount desc
      | limit 10
    '
# This returns a queryId. Use it to fetch results:
aws logs get-query-results --query-id "QUERY_ID_FROM_ABOVE"
```

```bash
# Find the slowest API requests in the last hour (if your app logs response times)
aws logs start-query \
    --log-group-name /prod/auth-api/application \
    --start-time $(date -u -d '1 hour ago' +%s) \
    --end-time $(date -u +%s) \
    --query-string '
      fields @timestamp, @message
      | parse @message "* * * *ms" as method, path, statusCode, responseTime
      | filter responseTime > 1000
      | sort responseTime desc
      | limit 20
    '
```

---

### 11.6 CloudWatch Alarms: Automated Incident Detection

An alarm without an action is just a decoration. Every alarm must trigger a notification (SNS → Slack/PagerDuty/Email) or an automated remediation (Lambda function, ASG scaling action).

#### Step 1: Create an SNS Topic for Alert Routing

```bash
# Create the SNS topic
aws sns create-topic --name prod-critical-alerts
# Output: TopicArn

# Subscribe the DevOps team's email
aws sns subscribe \
    --topic-arn arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --protocol email \
    --notification-endpoint devops-alerts@acme-corp.com

# Subscribe a Slack webhook (via AWS Chatbot or Lambda)
aws sns subscribe \
    --topic-arn arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --protocol https \
    --notification-endpoint https://hooks.slack.com/services/T00000/B00000/XXXXXXXX
```

#### Step 2: Create Critical Alarms

```bash
# ALARM 1: High CPU on EC2 instances
aws cloudwatch put-metric-alarm \
    --alarm-name "prod-api-high-cpu" \
    --alarm-description "API server CPU above 70% for 5 minutes" \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --dimensions Name=AutoScalingGroupName,Value=prod-api-asg \
    --statistic Average \
    --period 300 \
    --threshold 70 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --ok-actions arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --treat-missing-data notBreaching

# ALARM 2: ALB 5xx errors exceeding threshold
aws cloudwatch put-metric-alarm \
    --alarm-name "prod-alb-5xx-spike" \
    --alarm-description "ALB returning more than 10 5xx errors in 1 minute" \
    --namespace AWS/ApplicationELB \
    --metric-name HTTPCode_Target_5XX_Count \
    --dimensions Name=LoadBalancer,Value=app/prod-api-alb/abc123 \
    --statistic Sum \
    --period 60 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --treat-missing-data notBreaching

# ALARM 3: RDS free storage space critically low
aws cloudwatch put-metric-alarm \
    --alarm-name "prod-db-low-storage" \
    --alarm-description "RDS free storage below 5 GiB" \
    --namespace AWS/RDS \
    --metric-name FreeStorageSpace \
    --dimensions Name=DBInstanceIdentifier,Value=prod-auth-db \
    --statistic Average \
    --period 300 \
    --threshold 5368709120 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --treat-missing-data breaching

# ALARM 4: Zero healthy targets behind the ALB
aws cloudwatch put-metric-alarm \
    --alarm-name "prod-alb-no-healthy-targets" \
    --alarm-description "CRITICAL: No healthy targets behind the ALB" \
    --namespace AWS/ApplicationELB \
    --metric-name HealthyHostCount \
    --dimensions Name=TargetGroup,Value=targetgroup/prod-api-tg/abc123 Name=LoadBalancer,Value=app/prod-api-alb/abc123 \
    --statistic Minimum \
    --period 60 \
    --threshold 1 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:prod-critical-alerts \
    --treat-missing-data breaching
```

_Validation:_

```bash
# List all alarms and their current states
aws cloudwatch describe-alarms \
    --alarm-name-prefix "prod-" \
    --query 'MetricAlarms[*].[AlarmName, StateValue, StateReason]' \
    --output table
# Expected: All alarms should be in 'OK' state during normal operations
```

---

### 11.7 CloudWatch Dashboards: The War Room Display

A dashboard is the single pane of glass that the on-call engineer opens when an alarm fires. It must answer four questions in under 10 seconds:

1. **Is the system up?** (Healthy host count, error rate)
2. **Is the system fast?** (p99 latency, response time)
3. **Is the system under pressure?** (CPU, memory, connection count)
4. **Is the system growing?** (Request rate, queue depth)

```bash
# Create a CloudWatch Dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "Production-Overview" \
    --dashboard-body '{
      "widgets": [
        {
          "type": "metric",
          "x": 0, "y": 0, "width": 12, "height": 6,
          "properties": {
            "title": "ALB Request Count & 5xx Errors",
            "metrics": [
              ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/prod-api-alb/abc123", {"stat": "Sum", "period": 60}],
              ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", "app/prod-api-alb/abc123", {"stat": "Sum", "period": 60, "color": "#d62728"}]
            ],
            "view": "timeSeries",
            "region": "us-east-1",
            "period": 60
          }
        },
        {
          "type": "metric",
          "x": 12, "y": 0, "width": 12, "height": 6,
          "properties": {
            "title": "API Latency (p50, p95, p99)",
            "metrics": [
              ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/prod-api-alb/abc123", {"stat": "p50", "period": 60}],
              ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/prod-api-alb/abc123", {"stat": "p95", "period": 60}],
              ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/prod-api-alb/abc123", {"stat": "p99", "period": 60, "color": "#d62728"}]
            ],
            "view": "timeSeries",
            "region": "us-east-1"
          }
        },
        {
          "type": "metric",
          "x": 0, "y": 6, "width": 8, "height": 6,
          "properties": {
            "title": "EC2 ASG CPU Utilization",
            "metrics": [
              ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "prod-api-asg", {"stat": "Average", "period": 60}]
            ],
            "view": "timeSeries",
            "region": "us-east-1",
            "annotations": {
              "horizontal": [{"value": 70, "label": "Alert Threshold", "color": "#d62728"}]
            }
          }
        },
        {
          "type": "metric",
          "x": 8, "y": 6, "width": 8, "height": 6,
          "properties": {
            "title": "RDS Database Connections",
            "metrics": [
              ["AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", "prod-auth-db", {"stat": "Average", "period": 60}]
            ],
            "view": "timeSeries",
            "region": "us-east-1"
          }
        },
        {
          "type": "metric",
          "x": 16, "y": 6, "width": 8, "height": 6,
          "properties": {
            "title": "ALB Healthy Host Count",
            "metrics": [
              ["AWS/ApplicationELB", "HealthyHostCount", "TargetGroup", "targetgroup/prod-api-tg/abc123", "LoadBalancer", "app/prod-api-alb/abc123", {"stat": "Minimum", "period": 60}]
            ],
            "view": "singleValue",
            "region": "us-east-1"
          }
        }
      ]
    }'
```

---

### 11.8 Why Monitoring Fails in Real Systems

#### Failure 1: Alert Fatigue (The Boy Who Cried Wolf)

- _The Scenario:_ A team configures 150 CloudWatch alarms. 40 of them are poorly calibrated—they fire every day for non-critical issues like a brief CPU spike during a cron job or a transient network blip. Engineers receive 40+ Slack notifications daily.
- _The Impact:_ Within two weeks, every engineer mutes the `#prod-alerts` Slack channel. When a genuinely critical alarm fires (database disk space at 0%), nobody notices for 45 minutes because the channel is a wall of noise.
- _The Mandate:_ Alarms must be tiered:
  - **P1 (Critical / PagerDuty):** Pages the on-call engineer immediately. Reserved for: total outage, zero healthy hosts, data loss risk. Maximum 2-3 P1 alarms per service.
  - **P2 (Warning / Slack #prod-alerts):** Requires investigation within 1 hour. High error rate, elevated latency, disk > 80%.
  - **P3 (Informational / Slack #prod-metrics):** No action required. Awareness only. Daily cost anomaly reports, deployment notifications.

#### Failure 2: Missing the Memory Metric

- _The Scenario:_ A Node.js application has a slow memory leak. Over 72 hours, heap usage grows from 200 MiB to the instance's 8 GiB limit. The Linux OOM Killer terminates the Node.js process. PM2 restarts it. The cycle repeats every 3 days.
- _The Impact:_ The team never notices because they only monitor CPU. Memory is not a default CloudWatch metric. The intermittent PM2 restarts cause brief API outages every 3 days that are attributed to "network issues."
- _The Mandate:_ Install the CloudWatch Agent on every instance to collect `mem_used_percent` and `disk_used_percent`. These are not optional—they are as critical as CPU.

#### Failure 3: Logging Without Structure

- _The Scenario:_ A developer uses `console.log("Error: something went wrong with user " + userId)` throughout the application. When an incident occurs, the engineer searches CloudWatch Logs for "Error" and gets 50,000 unstructured results with no severity, no request ID, and no stack trace.
- _The Impact:_ Mean Time to Resolution (MTTR) is 2+ hours because the engineer cannot isolate the specific error from the noise.
- _The Mandate:_ Application logs must be structured JSON. Every log entry must include: `timestamp`, `level` (INFO, WARN, ERROR), `requestId` (correlation ID from the ALB or API Gateway), `message`, and `metadata` (user ID, endpoint, duration).

```json
{
  "timestamp": "2026-05-01T10:30:00.000Z",
  "level": "ERROR",
  "requestId": "abc-123-def-456",
  "message": "Failed to create user",
  "error": "duplicate key value violates unique constraint \"users_email_key\"",
  "metadata": {
    "endpoint": "POST /api/users",
    "userId": null,
    "email": "john@example.com",
    "durationMs": 45
  }
}
```

#### Failure 4: Monitoring the Infrastructure But Not the Business

- _The Scenario:_ All infrastructure metrics are green: CPU at 30%, memory at 50%, zero 5xx errors, latency at 80ms. But revenue dropped 40% overnight.
- _The Impact:_ The application had a logic bug that caused the checkout flow to silently fail—returning a 200 OK with an empty cart instead of processing the payment. No errors, no crashes, no alarms.
- _The Mandate:_ Monitor business-level metrics alongside infrastructure metrics:
  - Orders per minute (compared to historical baseline).
  - Signup conversion rate.
  - Payment success/failure ratio.
  - Active user count (if it drops 50% on a weekday, something is broken regardless of what the CPU says).

## 12. Advanced Security Practices

### Purpose

Harden the environment against real-world attacks with layered controls.

### What You Will Achieve

- Apply least-privilege access patterns at scale
- Manage secrets lifecycle with rotation and auditability
- Implement network isolation and threat detection

### Step-by-Step Implementation

1. Apply PoLP and permission boundaries (Section 12.2).
2. Configure Secrets Manager lifecycle controls (Section 12.3).
3. Add VPC endpoints, WAF, GuardDuty, and CloudTrail (Sections 12.4-12.6).

### Commands

All IAM, Secrets Manager, and security service commands are included inline.

### Validation

Verify GuardDuty is enabled, CloudTrail is logging, and WAF rules are active.

### Common Errors

- Overly broad IAM policies (`*` actions/resources).
- Leaving IMDSv1 enabled.

### Pro Tips

- Automate policy checks with IaC and security linters.
- Treat secrets as code: rotate, audit, and revoke quickly.

### 12.1 Security Is Not a Feature—It Is the Foundation

Security in a cloud environment is not a checklist item that gets addressed in "Phase 2" after the MVP ships. It is the invisible scaffolding upon which every other section of this playbook rests. A misconfigured IAM policy, a leaked secret, or an unpatched AMI can unravel months of infrastructure engineering in a matter of minutes.

This section builds upon the foundational IAM coverage in Section 3 and elevates it to advanced, production-hardened security practices. We cover three pillars in depth: access control (Least Privilege), secrets lifecycle management, and network-level isolation. Each subsection includes the mechanism, the mandate, and the real-world breach that validates why it exists.

---

### 12.2 The Principle of Least Privilege (PoLP): The Deep Dive

#### What is it?

The Principle of Least Privilege states that every identity (user, role, service, application) should be granted the absolute minimum permissions required to perform its specific function—and nothing more. Not "probably safe." Not "we'll lock it down later." The minimum, from day one.

#### Why does it matter at this depth?

Because privilege escalation is the most common attack path in cloud breaches. An attacker who compromises a low-privilege Lambda function cannot do much damage. But if that Lambda's execution role has `"Action": "*", "Resource": "*"`, the attacker now has full administrative control over the entire AWS account—databases, S3 buckets, IAM itself, billing.

#### The Mechanics of Implementing PoLP in AWS

##### Step 1: Start with Zero Permissions

Every new IAM Role or User starts with zero permissions. You add policies incrementally as the application's actual needs are discovered.

##### Step 2: Use AWS IAM Access Analyzer

IAM Access Analyzer monitors CloudTrail logs to identify which API calls a role actually makes over a period of time. It then generates a scoped-down policy based on actual usage—not guesswork.

```bash
# Create an Access Analyzer
aws accessanalyzer create-analyzer \
    --analyzer-name production-analyzer \
    --type ACCOUNT

# After 30 days, generate a policy based on actual usage
aws accessanalyzer start-policy-generation \
    --policy-generation-details '{
      "principalArn": "arn:aws:iam::123456789012:role/prod-app-server-role"
    }'
```

##### Step 3: Implement Permission Boundaries

A Permission Boundary is an advanced IAM feature that defines the _maximum_ permissions a role can ever have, regardless of what identity-based policies are attached to it. Even if an administrator accidentally attaches `AdministratorAccess` to the role, the boundary prevents the role from exceeding its ceiling.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlySpecificServices",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "sqs:*",
        "secretsmanager:GetSecretValue",
        "logs:*",
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAMEscalation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateRole",
        "iam:CreateUser",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy",
        "iam:CreateAccessKey",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyInfrastructureMutation",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket",
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# Attach the permission boundary to a role
aws iam put-role-permissions-boundary \
    --role-name prod-app-server-role \
    --permissions-boundary arn:aws:iam::123456789012:policy/AppServerPermissionBoundary
```

##### Step 4: Enforce Conditions and Resource-Level Restrictions

Never write `"Resource": "*"` when you know the specific ARN. Never omit `Condition` blocks when context is available.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnlyOnSpecificBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::acme-corp-prod-assets",
        "arn:aws:s3:::acme-corp-prod-assets/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

This policy allows S3 read access ONLY to a specific bucket, ONLY from the `us-east-1` region, ONLY from the VPC's private IP range. If an attacker exfiltrates the credentials and attempts to use them from their own machine, the `aws:SourceIp` condition blocks the request.

#### The PoLP Anti-Patterns (What NOT to Do)

| Anti-Pattern                                      | Why It Is Dangerous                                                                   |
| ------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `"Action": "*", "Resource": "*"`                  | Grants god-mode. If compromised, the attacker owns the entire account.                |
| Attaching `AdministratorAccess` to an app role    | The app can create IAM users, modify billing, and delete CloudTrail logs.             |
| Using `PowerUserAccess` for all developers        | Developers can create IAM roles with escalated privileges (privilege escalation).     |
| Sharing one IAM role across all microservices     | A breach in one service gives the attacker access to every other service's resources. |
| Using `iam:PassRole` without resource restriction | Allows passing any role to any service, enabling privilege escalation.                |

---

### 12.3 AWS Secrets Manager: Full Lifecycle Management

Section 3.5 introduced Secrets Manager at a conceptual level. This section covers the full operational lifecycle—creation, retrieval, rotation, auditing, and emergency revocation.

#### Why is Secrets Manager critical at the advanced level?

Because secrets are the skeleton keys of your infrastructure. A leaked database password, an exposed API key, or a hardcoded JWT signing secret is not a "security finding"—it is an active breach vector. Secrets Manager ensures that secrets are centrally stored, encrypted at rest with KMS, automatically rotated, and audited via CloudTrail.

#### The Complete Secret Lifecycle

##### Phase 1: Create the Secret

```bash
aws secretsmanager create-secret \
    --name prod/payments/stripe-api-key \
    --description "Stripe live API key for the payments service" \
    --secret-string '{"api_key":"sk_live_abc123xyz789","webhook_secret":"whsec_def456"}' \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678 \
    --tags Key=Environment,Value=Production Key=Service,Value=PaymentsAPI Key=Owner,Value=platform-team
```

##### Phase 2: Retrieve the Secret (Application Code)

The application retrieves secrets at startup or on-demand. Never cache secrets indefinitely—always respect the rotation interval.

```bash
# CLI retrieval (for scripts/debugging only)
aws secretsmanager get-secret-value \
    --secret-id prod/payments/stripe-api-key \
    --query 'SecretString' \
    --output text | jq .
```

Node.js application retrieval pattern:

```javascript
const {
  SecretsManagerClient,
  GetSecretValueCommand,
} = require("@aws-sdk/client-secrets-manager");

const client = new SecretsManagerClient({ region: "us-east-1" });

async function getSecret(secretName) {
  const response = await client.send(
    new GetSecretValueCommand({ SecretId: secretName }),
  );
  return JSON.parse(response.SecretString);
}

// Usage at application startup
const dbCredentials = await getSecret("prod/auth-db/master-credentials");
const pool = new Pool({
  host: dbCredentials.host,
  user: dbCredentials.username,
  password: dbCredentials.password,
  database: dbCredentials.dbname,
  port: dbCredentials.port,
});
```

##### Phase 3: Automatic Rotation

Secrets Manager can automatically rotate database credentials by invoking a Lambda function on a configurable schedule.

```bash
# Enable automatic rotation every 30 days
aws secretsmanager rotate-secret \
    --secret-id prod/auth-db/master-credentials \
    --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function/SecretsManagerRDSRotation \
    --rotation-rules '{"AutomaticallyAfterDays": 30}'
```

_How the rotation Lambda works internally:_

1. **createSecret:** Generates a new random password and stores it as the `AWSPENDING` version.
2. **setSecret:** Connects to the RDS instance using the current (active) password and changes the database user's password to the new one.
3. **testSecret:** Connects to the database using the new password to verify it works.
4. **finishSecret:** Promotes the `AWSPENDING` version to `AWSCURRENT`. The old password is moved to `AWSPREVIOUS` (kept for rollback).

_Validation:_

```bash
# Check the rotation status
aws secretsmanager describe-secret \
    --secret-id prod/auth-db/master-credentials \
    --query '{RotationEnabled: RotationEnabled, NextRotationDate: NextRotationDate, LastRotatedDate: LastRotatedDate}' \
    --output table
```

##### Phase 4: Emergency Secret Revocation

If a secret is leaked (found in a public GitHub repo, exposed in application logs, stolen by an attacker):

```bash
# Step 1: IMMEDIATELY rotate the secret (forces a new password NOW)
aws secretsmanager rotate-secret \
    --secret-id prod/auth-db/master-credentials \
    --rotate-immediately

# Step 2: Invalidate all active sessions that may be using the old credential
# (For database: force-disconnect all active connections)
# (For API keys: revoke the key at the provider—Stripe, Twilio, etc.)

# Step 3: Audit CloudTrail for unauthorized usage of the old secret
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=ResourceName,AttributeValue=prod/auth-db/master-credentials \
    --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
    --query 'Events[*].[EventTime, Username, EventName, SourceIPAddress]' \
    --output table
```

---

### 12.4 Network Isolation: Defense in Depth

Network isolation ensures that even if an application-level vulnerability is exploited, the attacker's lateral movement is contained within the smallest possible blast radius.

#### 12.4.1 VPC Endpoints: Eliminating Internet Exposure for AWS API Calls

##### What is the problem?

When an EC2 instance in a private subnet calls `aws s3 get-object`, the request leaves the VPC, traverses the NAT Gateway, exits to the public internet, reaches the S3 public API endpoint, and returns via the same path. This creates two risks:

1. The traffic traverses the public internet (even though it is TLS-encrypted, metadata leakage and man-in-the-middle downgrade attacks are possible).
2. The NAT Gateway becomes a bottleneck and a cost center for high-volume S3 operations.

##### What is the solution?

A VPC Endpoint creates a private, direct connection between your VPC and the AWS service. Traffic never leaves the AWS network backbone.

```bash
# Create a Gateway Endpoint for S3 (free, no per-hour charge)
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0abcd1234 \
    --service-name com.amazonaws.us-east-1.s3 \
    --route-table-ids rtb-private-rt \
    --policy-document '{
      "Statement": [{
        "Effect": "Allow",
        "Principal": "*",
        "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
        "Resource": [
          "arn:aws:s3:::acme-corp-prod-assets",
          "arn:aws:s3:::acme-corp-prod-assets/*",
          "arn:aws:s3:::acme-corp-prod-logs",
          "arn:aws:s3:::acme-corp-prod-logs/*"
        ]
      }]
    }'

# Create an Interface Endpoint for Secrets Manager
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0abcd1234 \
    --service-name com.amazonaws.us-east-1.secretsmanager \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-private-1a subnet-private-1b \
    --security-group-ids sg-vpce-secretsmanager \
    --private-dns-enabled
```

_Validation:_

```bash
# From a private instance, verify S3 traffic uses the VPC Endpoint
aws s3 ls s3://acme-corp-prod-assets/ --debug 2>&1 | grep "endpoint"
# The debug output should show the endpoint URL as the VPC endpoint, not the public S3 URL.
```

#### 12.4.2 Security Group Chaining: Micro-Segmentation

The enterprise mandate from Section 5.5 stated "reference other Security Groups, not IPs." Here is the complete chain for a 3-tier architecture:

```
Internet → ALB (sg-alb)
              ↓ Port 80/443 Inbound from 0.0.0.0/0

ALB → App Servers (sg-app)
              ↓ Port 3000 Inbound ONLY from sg-alb

App Servers → Database (sg-db)
              ↓ Port 5432 Inbound ONLY from sg-app

App Servers → Redis (sg-redis)
              ↓ Port 6379 Inbound ONLY from sg-app
```

```bash
# sg-app: Only allow traffic from the ALB Security Group
aws ec2 authorize-security-group-ingress \
    --group-id sg-app-prod \
    --protocol tcp \
    --port 3000 \
    --source-group sg-alb-prod

# sg-db: Only allow traffic from the App Server Security Group
aws ec2 authorize-security-group-ingress \
    --group-id sg-db-prod \
    --protocol tcp \
    --port 5432 \
    --source-group sg-app-prod

# sg-redis: Only allow traffic from the App Server Security Group
aws ec2 authorize-security-group-ingress \
    --group-id sg-redis-prod \
    --protocol tcp \
    --port 6379 \
    --source-group sg-app-prod
```

With this chain, even if an attacker compromises a publicly accessible ALB rule, they cannot directly reach the database because `sg-db` only accepts connections from `sg-app`, not from `sg-alb`.

#### 12.4.3 AWS WAF: Application-Layer Firewall

AWS WAF (Web Application Firewall) sits in front of the ALB and inspects every HTTP request before it reaches your application. It can block requests based on IP reputation, geographic origin, SQL injection patterns, cross-site scripting (XSS) payloads, and rate limiting.

```bash
# Create a WAF Web ACL with AWS Managed Rule Groups
aws wafv2 create-web-acl \
    --name prod-api-waf \
    --scope REGIONAL \
    --default-action '{"Allow": {}}' \
    --rules '[
      {
        "Name": "AWSManagedRulesCommonRuleSet",
        "Priority": 1,
        "Statement": {
          "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesCommonRuleSet"
          }
        },
        "OverrideAction": {"None": {}},
        "VisibilityConfig": {
          "SampledRequestsEnabled": true,
          "CloudWatchMetricsEnabled": true,
          "MetricName": "CommonRuleSet"
        }
      },
      {
        "Name": "AWSManagedRulesSQLiRuleSet",
        "Priority": 2,
        "Statement": {
          "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesSQLiRuleSet"
          }
        },
        "OverrideAction": {"None": {}},
        "VisibilityConfig": {
          "SampledRequestsEnabled": true,
          "CloudWatchMetricsEnabled": true,
          "MetricName": "SQLiRuleSet"
        }
      },
      {
        "Name": "RateLimitRule",
        "Priority": 3,
        "Statement": {
          "RateBasedStatement": {
            "Limit": 2000,
            "AggregateKeyType": "IP"
          }
        },
        "Action": {"Block": {}},
        "VisibilityConfig": {
          "SampledRequestsEnabled": true,
          "CloudWatchMetricsEnabled": true,
          "MetricName": "RateLimit"
        }
      }
    ]' \
    --visibility-config '{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"ProdAPIWAF"}'
```

_This WAF configuration:_

- Blocks common attack vectors (path traversal, host header injection, oversized requests) via the Common Rule Set.
- Blocks SQL injection attempts via the SQLi Rule Set.
- Rate-limits any single IP to 2,000 requests per 5 minutes, blocking beyond that threshold.

---

### 12.5 AWS GuardDuty: Automated Threat Detection

#### What is it?

Amazon GuardDuty is a managed threat detection service that continuously monitors CloudTrail management events, VPC Flow Logs, and DNS query logs for malicious activity. It uses machine learning anomaly detection to identify threats that static rules would miss.

#### What does it detect?

- **Compromised EC2 Instances:** Instances communicating with known command-and-control (C2) servers, bitcoin mining pools, or Tor exit nodes.
- **Compromised IAM Credentials:** API calls from unusual geographic locations, at unusual times, or using unusual API patterns (e.g., an application role suddenly calling `iam:CreateUser`).
- **Cryptocurrency Mining:** GPU or CPU usage patterns consistent with mining activity.
- **S3 Bucket Enumeration:** An external entity attempting to list or access S3 buckets they do not own.

```bash
# Enable GuardDuty (one-time setup per account per region)
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# List active findings
aws guardduty list-findings \
    --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
    --finding-criteria '{
      "Criterion": {
        "severity": {"Gte": 7}
      }
    }'
```

_The Enterprise Mandate:_ GuardDuty must be enabled in every AWS account and every region. High-severity findings (severity ≥ 7) must trigger a P1 PagerDuty alert immediately.

---

### 12.6 CloudTrail: The Immutable Audit Log

#### What is it?

AWS CloudTrail records every API call made in your AWS account—who called what API, from which IP address, at what time, and what the result was. It is the immutable forensic record that investigators and auditors rely on during and after a security incident.

#### Why is it critical?

Without CloudTrail, you have no way to answer: "Who deleted the production database?" "When was this IAM role's policy modified?" "Has anyone been accessing this S3 bucket from outside our VPC?"

```bash
# Create an organization-wide trail that logs to a centralized S3 bucket
aws cloudtrail create-trail \
    --name org-security-trail \
    --s3-bucket-name acme-corp-cloudtrail-logs \
    --is-multi-region-trail \
    --enable-log-file-validation \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/trail-encryption-key \
    --include-global-service-events

# Start logging
aws cloudtrail start-logging --name org-security-trail
```

_Critical Flags:_

- `--is-multi-region-trail`: Captures API calls in ALL regions, not just the trail's home region. Attackers frequently operate in regions you are not monitoring.
- `--enable-log-file-validation`: CloudTrail generates a digest file with a cryptographic hash of each log file. If an attacker attempts to modify or delete log entries, the hash validation will fail, proving tampering occurred.

_Validation:_

```bash
# Verify the trail is active and logging
aws cloudtrail get-trail-status --name org-security-trail \
    --query '{IsLogging: IsLogging, LatestDeliveryTime: LatestDeliveryTime}'
# Expected: IsLogging: true, LatestDeliveryTime: recent timestamp

# Search for a specific event (e.g., who deleted an S3 bucket)
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket \
    --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
    --query 'Events[*].[EventTime, Username, SourceIPAddress, Resources[0].ResourceName]' \
    --output table
```

---

### 12.7 Real-World Breach Scenarios

#### Breach 1: The SSRF to IMDS Credential Theft

- _The Scenario:_ A Node.js application has an endpoint that fetches a URL provided by the user (e.g., a "preview link" feature). An attacker crafts a request pointing to `http://169.254.169.254/latest/meta-data/iam/security-credentials/prod-app-server-role`. The application dutifully fetches this internal URL and returns the EC2 instance's IAM role credentials—temporary AccessKeyId, SecretAccessKey, and SessionToken—to the attacker.
- _The Impact:_ The attacker now possesses valid AWS credentials with whatever permissions the EC2 instance role has. If the role has broad S3 access, the attacker exfiltrates every object in every allowed bucket.
- _The Mandate:_
  1. Enable **IMDSv2 (Instance Metadata Service v2)** on all EC2 instances. IMDSv2 requires a session token obtained via a PUT request, which cannot be obtained via an SSRF attack because the attacker cannot inject custom HTTP methods through the application's URL fetcher.
  2. Validate and sanitize all user-provided URLs. Block requests to RFC 1918 private IP ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) and the link-local IMDS address (`169.254.169.254`).

```bash
# Enforce IMDSv2 on all existing instances
aws ec2 modify-instance-metadata-options \
    --instance-id i-0abc1234def56789 \
    --http-tokens required \
    --http-endpoint enabled
```

#### Breach 2: The Overly Permissive Lambda Execution Role

- _The Scenario:_ A developer creates a Lambda function for image resizing. They attach the `AmazonS3FullAccess` managed policy to the Lambda's execution role because "it needs to read and write to S3." The Lambda processes user-uploaded images.
- _The Attack:_ An attacker uploads a specially crafted image that exploits a vulnerability in the image processing library (e.g., ImageMagick's `CVE-2016-3714`). The exploit achieves Remote Code Execution (RCE) inside the Lambda's runtime environment. Because the Lambda's role has `s3:*` on `*`, the attacker's code enumerates and downloads every object from every S3 bucket in the account—including database backups, Terraform state files (which contain plaintext secrets), and customer PII.
- _The Impact:_ Complete data breach. Every S3 bucket in the account is compromised.
- _The Mandate:_ The Lambda execution role should have been scoped to `s3:GetObject` and `s3:PutObject` on a single, specific bucket (`arn:aws:s3:::acme-corp-image-uploads/*`). The blast radius of the same RCE exploit would have been limited to one bucket of user-uploaded images—sensitive, but not catastrophic.

#### Breach 3: The Forgotten Development Environment

- _The Scenario:_ A startup has a `dev` AWS account that was set up 3 years ago by a former employee. The account has no CloudTrail logging, no GuardDuty, and no billing alerts. The root account password is `admin123` with no MFA. The `dev` account is peered with the `prod` account via VPC Peering for "convenience."
- _The Attack:_ An attacker brute-forces the root account password (no MFA, no lockout). They discover the VPC Peering connection. They launch an EC2 instance in the `dev` VPC and, because the peering route tables were misconfigured to allow all traffic, they directly access the production RDS database on port 5432 through the peering connection.
- _The Impact:_ Production database fully compromised via a forgotten dev account.
- _The Mandate:_
  1. Every AWS account—including dev, staging, and sandbox—must have CloudTrail, GuardDuty, and root MFA enabled from day one.
  2. VPC Peering route tables must use the narrowest possible CIDR blocks. Never route `0.0.0.0/0` through a peering connection.
  3. Security Groups in the production VPC must explicitly allow only specific, expected source IPs from the peered VPC—not the entire peered CIDR block.

#### Breach 4: The Exposed Terraform State File

- _The Scenario:_ A DevOps engineer stores Terraform state in an S3 bucket. The Terraform state file contains the plaintext values of every resource attribute—including RDS master passwords, IAM access keys generated via `aws_iam_access_key`, and Secrets Manager secret values passed as variables. The S3 bucket has versioning enabled but no encryption and a bucket policy with `"Principal": "*"` that was added during debugging and never removed.
- _The Impact:_ An attacker discovers the bucket (via S3 bucket enumeration or accidental Google indexing). They download the `terraform.tfstate` file and extract every credential in the infrastructure. Every database, every API key, every secret is compromised simultaneously.
- _The Mandate:_
  1. Terraform state buckets must have SSE-KMS encryption, Block Public Access, versioning, and `BucketOwnerEnforced` ACL ownership.
  2. State files must never be stored locally or committed to Git.
  3. Use `sensitive = true` on all secret-containing Terraform outputs to prevent them from appearing in CLI output and plan files.
  4. Implement DynamoDB state locking to prevent concurrent state corruption.

#### Breach 5: The Supply Chain Attack via Compromised npm Package

- _The Scenario:_ A popular npm package (`event-stream`) is acquired by a new maintainer who injects malicious code into a minor version release. The malicious code targets a specific cryptocurrency wallet application. A company's CI/CD pipeline runs `npm install` (not `npm ci`), pulling the latest minor version of the compromised dependency. The malicious code executes during the build step on the CI server, which has an IAM role with broad AWS permissions for deployment.
- _The Impact:_ The attacker's code, running inside the CI/CD environment, uses the CI server's IAM role to access Secrets Manager, exfiltrate production secrets, and establish a persistent backdoor IAM user.
- _The Mandate:_
  1. Always use `npm ci` for deterministic builds from `package-lock.json`.
  2. Pin dependency versions exactly (`"express": "4.18.2"`, not `"express": "^4.18.0"`).
  3. Run `npm audit` and integrate Snyk or Socket.dev into the CI pipeline to detect known vulnerabilities and anomalous package behavior.
  4. The CI/CD runner's IAM role must follow PoLP—it should only have permissions to deploy, not to read all secrets or modify IAM. Use separate roles for build vs. deploy stages.

## 13. Troubleshooting Guide

### Purpose

Provide a repeatable, layered approach to debugging production issues.

### What You Will Achieve

- Quickly isolate issues by network, host, service, app, and dependency layers
- Resolve the most common SSH, PM2, and Nginx failures
- Execute a 60-second triage when production is down

### Step-by-Step Implementation

1. Follow the 5-layer diagnostic framework (Section 13.1).
2. Use the SSH and application issue flowcharts (Sections 13.2 and 13.3).
3. Apply system-level diagnostics and the 60-second runbook (Sections 13.7 and 13.8).

### Commands

All diagnostic commands are provided inline for each scenario.

### Validation

Confirm the failing layer, apply the fix, and re-run the exact failing check.

### Common Errors

- Guessing the root cause without checking logs.
- Restarting services without understanding the failure mode.

### Pro Tips

- Save command outputs in incident notes for faster handoffs.
- Practice the 60-second runbook before an incident occurs.

### 13.1 The Debugging Mindset: Think in Layers, Not in Guesses

The single most destructive behavior in incident response is guessing. An engineer sees "the site is down," assumes "the server must have crashed," SSHs in (if they even can), restarts PM2, and walks away. The site comes back up. Twenty minutes later it goes down again. They restart PM2 again. This cycle repeats five times before someone finally reads a log file and discovers that the RDS instance ran out of storage 6 hours ago and every database write has been silently failing.

Effective troubleshooting is not a talent—it is a discipline. It follows a systematic, layered approach that eliminates possibilities methodically. You start at the outermost layer (can I reach the server at all?) and work inward toward the application logic.

#### The 5-Layer Diagnostic Framework

Every production issue falls into one of these five layers. Diagnose them in order—never skip a layer.

```
Layer 1: NETWORK     → Can I reach the machine?        (ping, telnet, traceroute, Security Groups)
Layer 2: OS / HOST   → Is the machine healthy?          (CPU, memory, disk, processes)
Layer 3: SERVICE     → Is the application process running? (pm2 status, systemctl, lsof)
Layer 4: APPLICATION → Is the application behaving correctly? (logs, health endpoint, error rates)
Layer 5: DEPENDENCY  → Are external dependencies healthy?  (database, Redis, S3, third-party APIs)
```

Never attempt Layer 4 debugging (reading application logs) until you have confirmed Layers 1-3 are healthy. If the machine is out of disk space (Layer 2), reading application logs is meaningless because the application cannot even write logs.

---

### 13.2 SSH Issues: "I Can't Even Get In"

SSH failures are the most common and the most frustrating class of infrastructure issues because they prevent you from diagnosing everything else. There are exactly four reasons an SSH connection can fail, and you must systematically eliminate each one.

#### Problem 1: "Permission denied (publickey)"

This error means the SSH handshake reached the server, but authentication failed. The server is alive and reachable—the issue is with your credentials.

##### Diagnostic Flowchart:

```
"Permission denied (publickey)" received
│
├─ Are you using the correct key file?
│   $ ssh -i /path/to/key.pem ec2-user@IP -v
│   Look for: "Offering public key: /path/to/key.pem"
│   If the path is wrong or the file doesn't exist → fix the path.
│
├─ Does the key file have correct permissions?
│   $ ls -la /path/to/key.pem
│   Must be: -rw------- (600) or -r-------- (400)
│   If it's 644 or 755 → SSH refuses to use it.
│   $ chmod 400 /path/to/key.pem
│
├─ Are you using the correct username?
│   Amazon Linux / Amazon Linux 2023: ec2-user
│   Ubuntu: ubuntu
│   CentOS / RHEL: centos or ec2-user
│   Debian: admin
│   SUSE: ec2-user
│   If wrong username → "Permission denied" even with the correct key.
│
├─ Was the instance launched with this key pair?
│   $ aws ec2 describe-instances \
│       --instance-ids i-0abc1234def56789 \
│       --query 'Reservations[0].Instances[0].KeyName' \
│       --output text
│   If the key pair name doesn't match your .pem file → wrong key.
│
└─ Was the authorized_keys file on the instance modified or corrupted?
    This happens when someone manually edits ~/.ssh/authorized_keys
    and introduces a typo or removes the key entry.
    Recovery: Use SSM Session Manager (no SSH key required) to fix it:
    $ aws ssm start-session --target i-0abc1234def56789
    Then inside: cat ~/.ssh/authorized_keys
```

##### The Full Debug Command:

```bash
# SSH with maximum verbosity to see exactly where authentication fails
ssh -i key.pem ec2-user@54.123.45.67 -vvv 2>&1 | tee ssh-debug.log

# Key lines to look for in the output:
# "Offering public key: key.pem RSA SHA256:abc123..."  → Key is being offered
# "Server accepts key: ..."                             → Key was accepted (auth succeeds)
# "No more authentication methods to try"               → All keys were rejected
```

#### Problem 2: "Connection timed out" or "Connection refused"

This error means the SSH handshake never even started. The packets are not reaching the server or the server is not listening on port 22.

##### Diagnostic Flowchart:

```
"Connection timed out" received
│
├─ LAYER 1: Is the instance running?
│   $ aws ec2 describe-instances \
│       --instance-ids i-0abc1234def56789 \
│       --query 'Reservations[0].Instances[0].State.Name' \
│       --output text
│   Expected: "running"
│   If "stopped" or "terminated" → start the instance.
│
├─ LAYER 1: Does the instance have a public IP? (if connecting directly)
│   $ aws ec2 describe-instances \
│       --instance-ids i-0abc1234def56789 \
│       --query 'Reservations[0].Instances[0].PublicIpAddress' \
│       --output text
│   If "None" → the instance is in a private subnet (no public IP).
│   Use SSM Session Manager or a bastion host instead.
│
├─ LAYER 1: Is port 22 open in the Security Group?
│   $ aws ec2 describe-security-groups \
│       --group-ids sg-app-prod \
│       --query 'SecurityGroups[0].IpPermissions[?FromPort==`22`]' \
│       --output table
│   If empty → port 22 is not allowed. Add an ingress rule.
│   If source is "10.0.0.0/8" → you can only SSH from within the VPC.
│
├─ LAYER 1: Is there an Internet Gateway attached and a route to 0.0.0.0/0?
│   $ aws ec2 describe-route-tables \
│       --filters Name=association.subnet-id,Values=subnet-public-1a \
│       --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]' \
│       --output table
│   If no IGW route → the subnet is private, not public.
│
├─ LAYER 1: Is the NACL (Network ACL) blocking traffic?
│   NACLs are stateless and have explicit deny rules.
│   $ aws ec2 describe-network-acls \
│       --filters Name=association.subnet-id,Values=subnet-public-1a \
│       --query 'NetworkAcls[0].Entries' \
│       --output table
│   Check for DENY rules on port 22 with a lower rule number than the ALLOW rule.
│
└─ LAYER 2: Is the SSH daemon running on the instance?
    If you can connect via SSM Session Manager:
    $ sudo systemctl status sshd
    If inactive → $ sudo systemctl start sshd
    If the SSH port was changed from 22 to another port:
    $ sudo grep "^Port" /etc/ssh/sshd_config
```

##### Quick Connectivity Test Commands:

```bash
# Test if port 22 is reachable (from your local machine)
nc -zv 54.123.45.67 22 -w 5
# Expected: Connection to 54.123.45.67 22 port [tcp/ssh] succeeded!
# If "timed out" → network/firewall issue (Security Group, NACL, route table)
# If "Connection refused" → port is reachable but nothing is listening (sshd is down)

# Trace the network path to identify where packets are being dropped
traceroute -T -p 22 54.123.45.67

# Test from within the VPC (if you have SSM access to another instance)
aws ssm start-session --target i-another-instance
nc -zv 10.0.10.45 22 -w 5
```

---

### 13.3 Application Issues: "The App Is Down"

Once you have confirmed SSH/SSM access is working (Layer 1 resolved), you move to diagnosing why the application is not serving traffic.

#### Problem 3: "Port not open" / "Connection refused on port 3000"

The ALB health check returns unhealthy, or `curl http://localhost:3000/health` returns "Connection refused."

##### Diagnostic Flowchart:

```
"Connection refused" on application port
│
├─ LAYER 3: Is the process running at all?
│   $ pm2 status
│   If "errored" or "stopped" → the process crashed. Check logs.
│   If no pm2 processes listed → the app was never started.
│   $ pm2 start ecosystem.config.js
│
├─ LAYER 3: Is the process listening on the expected port?
│   $ ss -tlnp | grep 3000
│   Expected: LISTEN  0  128  0.0.0.0:3000  *:*  users:(("node",pid=12345,fd=18))
│   If nothing → the app started but failed to bind to port 3000.
│   Check: is another process using port 3000?
│   $ ss -tlnp | grep 3000
│   Or: $ lsof -i :3000
│
├─ LAYER 3: Is the process listening on the correct interface?
│   If the output shows "127.0.0.1:3000" instead of "0.0.0.0:3000":
│   The app is listening ONLY on localhost. It will reject connections
│   from the ALB (which connects via the instance's private IP).
│   Fix: change the app's listen() call from '127.0.0.1' to '0.0.0.0'.
│
├─ LAYER 1: Is the Security Group allowing inbound traffic on port 3000?
│   $ aws ec2 describe-security-groups \
│       --group-ids sg-app-prod \
│       --query 'SecurityGroups[0].IpPermissions[?FromPort==`3000`]'
│   The source must include the ALB's Security Group (sg-alb-prod).
│
└─ LAYER 2: Is there a firewall on the instance itself?
    $ sudo iptables -L -n
    $ sudo firewall-cmd --list-all  # (if firewalld is active)
    Look for REJECT or DROP rules on port 3000.
```

##### The Essential Port Diagnostic Commands:

```bash
# Show all listening TCP ports with process names
ss -tlnp
# Explanation:
#   -t = TCP only
#   -l = listening sockets only
#   -n = show port numbers, not service names
#   -p = show the process using each socket

# Alternative (older systems): netstat
netstat -tulnp
# Explanation:
#   -t = TCP, -u = UDP
#   -l = listening, -n = numeric
#   -p = PID/program name

# Find which process is using a specific port
lsof -i :3000
# Output:
# COMMAND   PID     USER   FD   TYPE  DEVICE  SIZE/OFF  NODE  NAME
# node    12345  ec2-user  18u  IPv4  123456      0t0   TCP   *:3000 (LISTEN)

# If the port is occupied by a zombie/stale process:
kill -9 $(lsof -ti :3000)
# Then restart your application.
```

#### Problem 4: "Process not running" / PM2 Shows "errored"

##### Diagnostic Flowchart:

```
PM2 shows "errored" or "stopped" status
│
├─ Step 1: Read the error logs FIRST
│   $ pm2 logs auth-api --err --lines 100
│   This is the single most important command. 90% of all
│   "app won't start" issues are answered in these logs.
│
├─ Common Error: "Error: Cannot find module './dist/server.js'"
│   The build output doesn't exist. You forgot to run 'npm run build'.
│   $ ls -la dist/
│   If empty or missing → $ npm run build
│
├─ Common Error: "EADDRINUSE: address already in use :::3000"
│   Another process is already using port 3000.
│   $ lsof -i :3000
│   Kill the conflicting process, then restart.
│
├─ Common Error: "ECONNREFUSED 127.0.0.1:5432"
│   The application cannot connect to the database.
│   → Jump to Section 13.6 (Dependency Issues).
│
├─ Common Error: "Error: secretsmanager is not authorized to perform"
│   The EC2 instance's IAM role does not have permission
│   to read from Secrets Manager.
│   Check the instance profile:
│   $ aws sts get-caller-identity
│   Verify the role has secretsmanager:GetSecretValue permission.
│
├─ Common Error: "JavaScript heap out of memory"
│   The Node.js process exceeded its default memory limit.
│   Add to ecosystem.config.js:
│   node_args: "--max-old-space-size=2048"
│   Or investigate the memory leak (see Section 11.8 Failure 2).
│
└─ Step 2: If logs show nothing useful, check system-level issues
    $ dmesg | tail -50        # Kernel messages (OOM kills, disk errors)
    $ journalctl -u pm2-ec2-user --since "1 hour ago"  # systemd logs for PM2
```

##### PM2 Deep Diagnostic Commands:

```bash
# Full process detail (memory, CPU, restarts, uptime, script path)
pm2 describe auth-api

# Show the last 20 restart timestamps (to detect crash loops)
pm2 show auth-api | grep -A 20 "restart"

# Real-time monitoring (CPU, memory, loop delay per worker)
pm2 monit

# Reset restart counter (after fixing the root cause)
pm2 reset auth-api

# Dump the full PM2 internal state (for support tickets)
pm2 prettylist

# Flush all log files (if disk is full from log accumulation)
pm2 flush

# View the ecosystem file PM2 is using (verify correct script path)
pm2 describe auth-api | grep "script"
```

---

### 13.4 Nginx Issues: "502 Bad Gateway" / "504 Gateway Timeout"

These are the most common user-facing errors and they always originate from Nginx, not from the application directly.

#### "502 Bad Gateway"

- **What it means:** Nginx received an invalid or no response from the upstream backend (the Node.js application). The application process is either crashed, not listening, or responded with something Nginx couldn't parse.

```bash
# Step 1: Check if the upstream process is alive
pm2 status
# If errored → fix the app (Section 13.3 Problem 4)

# Step 2: Test the upstream directly (bypass Nginx)
curl -s http://127.0.0.1:3000/health
# If "Connection refused" → app is not listening (Section 13.3 Problem 3)
# If returns valid JSON → app is fine, the issue is Nginx config

# Step 3: Check the Nginx error log
sudo tail -50 /var/log/nginx/error.log
# Look for: "connect() failed (111: Connection refused) while connecting to upstream"
# This confirms the app isn't running on the expected port.

# Step 4: Verify the Nginx upstream configuration
sudo grep -A 5 "proxy_pass" /etc/nginx/conf.d/dashboard.conf
# Verify the port matches the application's listening port.
```

#### "504 Gateway Timeout"

- **What it means:** Nginx forwarded the request to the upstream, but the upstream did not respond within the timeout window (default: 60 seconds). The application is alive but hanging—usually waiting on a database query, an external API call, or a deadlocked operation.

```bash
# Step 1: Check the application's response time
curl -w "\n%{time_total}s total\n" http://127.0.0.1:3000/health
# If >60 seconds → the endpoint is hanging

# Step 2: Identify what the application is waiting on
# Check active database connections
psql -h 127.0.0.1 -p 5432 -U dbadmin -c "SELECT pid, state, query, now() - query_start AS duration FROM pg_stat_activity WHERE state != 'idle' ORDER BY duration DESC;"
# Look for queries running for >30 seconds

# Step 3: Check if the app has open connections to external services
ss -tnp | grep $(pgrep -f "node")
# Look for connections in "ESTABLISHED" or "SYN_SENT" state to external IPs

# Step 4: Increase Nginx timeouts if the slow response is expected (e.g., report generation)
# In the Nginx location block:
#   proxy_connect_timeout 300;
#   proxy_send_timeout 300;
#   proxy_read_timeout 300;
```

---

### 13.5 Rollback Procedures

Use these when a change breaks production and you need to return to a known-good state quickly.

Rollback a PM2 application:

```bash
# Option 1: Git rollback (recent update)
cd ~/apps/myapp
git log --oneline -5
git checkout PREVIOUS_COMMIT_HASH
pm2 restart myapp-dev

# Option 2: Roll back to a known tag
git checkout v1.2.3
npm ci --production
pm2 restart myapp-dev
```

Rollback an Nginx config change:

```bash
# Before changes, keep a backup
sudo cp /etc/nginx/conf.d/myapp-dev.conf /etc/nginx/conf.d/myapp-dev.conf.bak

# If a reload breaks traffic, restore the backup
sudo cp /etc/nginx/conf.d/myapp-dev.conf.bak /etc/nginx/conf.d/myapp-dev.conf
sudo nginx -t
sudo systemctl reload nginx
```

Rollback an EC2 instance type change:

```bash
aws ec2 stop-instances --instance-ids i-YOURID
aws ec2 wait instance-stopped --instance-ids i-YOURID
aws ec2 modify-instance-attribute \
  --instance-id i-YOURID \
  --instance-type '{"Value": "t3.micro"}'
aws ec2 start-instances --instance-ids i-YOURID
```

### 13.6 Dependency Issues: "The App Runs But Something Is Broken"

The application process is running, the health endpoint returns 200, but specific features are failing. This is Layer 5—external dependency failures.

#### Database Connectivity Issues

```bash
# Test database connectivity from the application server
psql -h prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com -p 5432 -U dbadmin -d authdb -c "SELECT 1;"
# If "could not connect to server: Connection timed out" →
#   Check: Security Group sg-db-prod allows ingress from sg-app-prod on port 5432
#   Check: RDS instance is in the same VPC and the DB subnet group is correct
#   Check: RDS instance status is "available"

aws rds describe-db-instances \
    --db-instance-identifier prod-auth-db \
    --query 'DBInstances[0].[DBInstanceStatus, Endpoint.Address, Endpoint.Port]' \
    --output text

# If "FATAL: password authentication failed" →
#   The password has been rotated (Secrets Manager auto-rotation).
#   Retrieve the latest password:
aws secretsmanager get-secret-value \
    --secret-id prod/auth-db/master-credentials \
    --query 'SecretString' --output text | jq -r '.password'
#   Update the application's connection string and restart.

# If "FATAL: too many connections for role" →
#   Connection pool exhaustion. Check current connections:
psql -h localhost -p 5432 -U dbadmin -c "SELECT count(*) FROM pg_stat_activity;"
#   Compare to max_connections:
psql -h localhost -p 5432 -U dbadmin -c "SHOW max_connections;"
```

#### Redis Connectivity Issues

```bash
# Test Redis connectivity
redis-cli -h prod-redis.abcdefg.0001.use1.cache.amazonaws.com -p 6379 ping
# Expected: PONG
# If "Could not connect" → check Security Group and subnet configuration

# Check Redis memory usage
redis-cli -h prod-redis.abcdefg.0001.use1.cache.amazonaws.com INFO memory | grep used_memory_human
# If near the instance's max memory → eviction is occurring. Scale up the node.
```

#### S3 Access Issues

```bash
# Test S3 access from the instance
aws s3 ls s3://acme-corp-prod-assets/ --max-items 5
# If "AccessDenied" → the instance's IAM role does not have s3:ListBucket permission
# If "NoSuchBucket" → typo in the bucket name or wrong region

# Check the instance's IAM role
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Returns the role name. Then check what policies are attached:
aws iam list-attached-role-policies --role-name $(curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/)
```

---

### 13.7 System-Level Diagnostics: The Essential Command Reference

These commands should be committed to muscle memory. Every engineer troubleshooting a production Linux server will use them in the first 60 seconds.

#### CPU and Process Analysis

```bash
# Real-time process view sorted by CPU usage
top -c -o %CPU
# Press 'P' to sort by CPU, 'M' to sort by memory, 'q' to quit

# Snapshot of the top 10 processes by CPU
ps aux --sort=-%cpu | head -11

# Check system load averages (1min, 5min, 15min)
uptime
# Expected: load average: 0.50, 0.45, 0.40  (for a 2-core machine, <2.0 is healthy)
# If load average exceeds the number of CPU cores → system is overloaded
```

#### Memory Analysis

```bash
# System memory overview
free -h
# Key values:
#   total: total physical RAM
#   available: memory that can be used without swapping (THIS is the important number)
#   If "available" is <500 MiB, the system is under memory pressure.

# Identify the top memory consumers
ps aux --sort=-%mem | head -11

# Check for OOM (Out of Memory) kills in the kernel log
dmesg | grep -i "oom\|killed" | tail -20
# If entries found → a process was killed by the kernel due to memory exhaustion
```

#### Disk Space Analysis

```bash
# Check disk usage by filesystem
df -h
# CRITICAL: If any filesystem is >90% full, take immediate action.
# The root filesystem (/) at 100% will cause:
#   - Log writing failures
#   - Application crashes (cannot write temp files)
#   - Database corruption (if DB stores data on this filesystem)
#   - SSH failures (cannot write to /var/log/lastlog)

# Find the largest files and directories
du -h --max-depth=2 / | sort -rh | head -20

# Common culprits for disk space exhaustion:
# 1. Unrotated PM2 logs:      /var/log/pm2/
# 2. Old application builds:  /opt/apps/auth-api/node_modules/ (stale builds)
# 3. Package manager cache:   /var/cache/dnf/ or /var/cache/apt/
# 4. Core dumps:              /var/crash/ or /tmp/
# 5. Docker images (if used): /var/lib/docker/

# Quick fix for PM2 log accumulation:
pm2 flush
# Long-term fix: configure pm2-logrotate
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 7
```

#### Network Diagnostics

```bash
# Show all active network connections for a specific process
ss -tnp | grep node
# Output shows: state, local address:port, peer address:port, process
# Look for:
#   ESTABLISHED → active connections (to databases, Redis, external APIs)
#   CLOSE-WAIT  → connections the remote side closed but your app hasn't acknowledged
#                  (indicates a connection leak in the application)
#   TIME-WAIT   → recently closed connections (normal, but too many indicates churn)

# Count connections by state
ss -s
# Key: if "closed-wait" or "time-wait" counts are in the thousands, investigate.

# Test if a specific port is reachable from this machine
nc -zv database-hostname 5432 -w 5
# "succeeded" → port is reachable
# "timed out" → firewall/Security Group/NACL blocking
# "Connection refused" → port is reachable but nothing is listening

# DNS resolution check (when "hostname not found" errors occur)
dig +short prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com
# Expected: returns one or more IP addresses
# If empty → DNS resolution is failing. Check VPC DNS settings:
aws ec2 describe-vpc-attribute --vpc-id vpc-0abcd1234 --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id vpc-0abcd1234 --attribute enableDnsHostnames
```

---

### 13.8 The 60-Second Production Triage Runbook

When you receive a P1 alert at 2 AM and need to assess the situation in under 60 seconds, run these commands in this exact order:

```bash
# 1. Am I on the right machine? (5 seconds)
hostname
aws sts get-caller-identity --query 'Account' --output text

# 2. System overview: CPU, memory, disk (10 seconds)
uptime
free -h | grep Mem
df -h / | tail -1

# 3. Is the application running? (5 seconds)
pm2 status

# 4. Are there recent errors? (15 seconds)
pm2 logs auth-api --err --lines 30 --nostream

# 5. Is the port open and listening? (5 seconds)
ss -tlnp | grep 3000

# 6. Can the app reach the database? (10 seconds)
nc -zv prod-auth-db.abcdefg.us-east-1.rds.amazonaws.com 5432 -w 3

# 7. What does the health endpoint say? (5 seconds)
curl -s http://localhost:3000/health | jq .

# 8. Any kernel-level issues? (5 seconds)
dmesg | tail -10
```

After these 8 commands, you should have enough information to determine which layer the problem is in and begin targeted remediation. If the answer is still unclear, escalate with the output of these commands attached—it gives the next engineer a complete diagnostic baseline without repeating any work.

---

### 13.9 Real-World Troubleshooting War Stories

#### War Story 1: The Phantom Disk Full

- _The Symptom:_ Application returns 500 errors intermittently. PM2 shows the process is running. `pm2 logs` shows errors about "failed to write session file."
- _The Investigation:_ `df -h` shows `/` at 100%. `du -h --max-depth=2 / | sort -rh | head -10` reveals `/var/log/pm2/` contains 47 GiB of log files. PM2's `auth-api-out.log` has been growing unchecked for 8 months.
- _The Fix:_ `pm2 flush` to clear all log files immediately. Install `pm2-logrotate` to prevent recurrence. Set `max_size: 50M` and `retain: 7` to cap log files at 50 MiB with 7 rotations.
- _The Lesson:_ PM2 does not rotate logs by default. Log rotation must be explicitly configured on every instance, or logs will eventually consume all disk space.

#### War Story 2: The CLOSE_WAIT Connection Leak

- _The Symptom:_ The application becomes progressively slower over 72 hours, then suddenly crashes. PM2 restarts it. The cycle repeats exactly every 3 days.
- _The Investigation:_ `ss -s` shows 12,000 connections in `CLOSE_WAIT` state. The application makes HTTP requests to a third-party API but never calls `.destroy()` on the response stream when the API returns an error. The operating system's file descriptor limit (`ulimit -n`) is reached, and the Node.js process can no longer accept new connections.
- _The Fix:_ Update the HTTP client code to properly close response streams on error. Set socket timeouts (`timeout: 10000`). Add connection pool limits to the HTTP agent (`maxSockets: 50`). Set `keepAlive: true` with `keepAliveMsecs: 30000`.
- _The Lesson:_ Connection leaks are invisible until they become catastrophic. Monitor `ss -s` output as a custom CloudWatch metric.

#### War Story 3: The Silent Security Group Misconfiguration

- _The Symptom:_ After a "routine" infrastructure change (adding a new staging environment), the production API starts returning 502 errors for 30% of requests.
- _The Investigation:_ The engineer who created the staging environment accidentally modified `sg-app-prod` instead of creating `sg-app-staging`. They removed the ingress rule allowing traffic from `sg-alb-prod` on port 3000. The ALB could no longer reach 3 of the 5 ASG instances (the ones whose Security Group cache had been refreshed). `aws elbv2 describe-target-health` showed 3 targets as "unhealthy" with reason "Elb.InternalError."
- _The Fix:_ Re-added the ingress rule from `sg-alb-prod` on port 3000 to `sg-app-prod`. Health checks recovered within 30 seconds.
- _The Lesson:_ (1) Never modify production Security Groups directly. Use Infrastructure as Code (Terraform) with plan-and-apply workflows. (2) Security Group changes take effect immediately—there is no "undo." (3) The `describe-target-health` API is the first command to run when the ALB returns 502/503.

#### War Story 4: The Wrong Node.js Version

- _The Symptom:_ After deploying a new version, the application crashes immediately on startup with `SyntaxError: Unexpected token '?.'` (optional chaining operator).
- _The Investigation:_ The developer wrote code using optional chaining (`user?.address?.city`), which requires Node.js ≥ 14. The production server is running Node.js 12 (installed 3 years ago, never updated). The CI pipeline uses Node.js 22 (from the `.nvmrc` file), so all tests passed.
- _The Fix:_ Update Node.js on the production server to match the CI pipeline version. Add a `"engines": { "node": ">=22" }` field to `package.json` so that `npm ci` fails loudly if the wrong Node.js version is detected.
- _The Lesson:_ The CI/CD environment and the production environment must use the exact same Node.js version. Golden AMIs (pre-baked images) with a pinned Node.js version eliminate this class of error entirely.

## Chapter 14: Running Node.js on AWS at Scale

This chapter moves from basic deployment to production-grade architecture, covering load balancing, multi-AZ deployment, scaling strategies, and cost optimization for Node.js applications on AWS.

### Purpose

Translate a single-instance Node.js setup into a resilient, scalable architecture.

### What You Will Achieve

- Understand stateless architecture requirements
- Choose the right session and state storage strategy
- Apply AWS-native scaling patterns safely

### Step-by-Step Implementation

1. Review the stateless architecture principles (Section 14.1).
2. Choose a session strategy (Section 14.2).
3. Apply the scaling and cost controls that match your workload.

### Commands

All service provisioning commands are provided inline in the subsections.

### Validation

Confirm session storage works across multiple instances and survives restarts.

### Common Errors

- Storing session state in memory and breaking horizontal scaling.
- Opening Redis to the public internet.

### Pro Tips

- Keep auth/session flows simple and auditable.
- Prefer managed services (ElastiCache, RDS) for resilience.

### 14.1 Architecture: Stateless Services

**The Golden Rule:** In a scaled environment, session state must **never** be stored in application memory or on the local filesystem. All state must be externalized to a shared data store.

#### Why Stateless Matters

```text
┌─────────────────────┐             ┌─────────────────────┐
│    User Device      │             │    User Device      │
└─────────┬───────────┘             └─────────┬───────────┘
          │                                   │
          ▼                                   ▼
    ┌──────────────────────────────────────────┐
    │          Amazon Route 53 DNS               │
    └───────────────┬────────────────────────────┘
                    │
                    ▼
          ┌──────────────────────────┐
          │    AWS Certificate       │
          │    Manager (ACM)         │
          └──────────────┬───────────┘
                         │
                         ▼
          ┌──────────────────────────┐
          │   Application Load       │
          │   Balancer (ALB)         │
          │   Target Group: auth-sg │
          └──────────────┬───────────┘
                         │
      ┌──────────────────┴──────────────────┐
      ▼                                     ▼
┌───────────────────────────┐        ┌───────────────────────────┐
│    EC2 Instance 1         │        │    EC2 Instance 2         │
│ (Auto Scaling Group)      │        │ (Auto Scaling Group)      │
│                           │        │                           │
│  Process: pm2 start auth  │        │  Process: pm2 start auth  │
│                           │        │                           │
│  ✓ Session -> Redis       │        │  ✓ Session -> Redis       │
│  ✓ JWT -> Redis           │        │  ✓ JWT -> Redis           │
│                           │        │                           │
└───────────┬─────────────┘          └─────────────┬─────────────┘
            │                                      │
            └──────────┬───────────────────────────┘
                       │
                       ▼
            ┌──────────────────────────────┐
            │  Shared State Layer          │
            │  ✓ ElastiCache (Sessions)    │
            │  ✓ RDS (User Data)           │
            │  ✓ S3 (User Uploads)         │
            │  ✓ DynamoDB (Queues, etc.)   │
            └──────────────────────────────┘

```

**Key Architectural Principles:**

| Layer          | AWS Service | Purpose                           | Failure Impact                  | Recovery               |
| -------------- | ----------- | --------------------------------- | ------------------------------- | ---------------------- |
| DNS            | Route 53    | Global routing, health checks     | No access to service            | Automatic failover     |
| TLS            | ACM         | Managed SSL certificates          | 502/503 errors                  | Auto-renewal           |
| Load Balancing | ALB         | Distribute traffic, health checks | 503 errors, uneven load         | Auto-healing           |
| Application    | EC2 ASG     | Run Node.js processes             | App downtime                    | Auto-scaling/healing   |
| Sessions       | ElastiCache | Distributed session store         | Auth failures, logged out users | Cluster failover       |
| Database       | RDS         | User data persistence             | Data loss, system failure       | Read replicas, backups |
| Storage        | S3          | User files, assets                | Missing files                   | Versioning, CORS       |

### 14.2 Session Management Strategies

#### Option 1: ElastiCache Redis (Recommended)

**Setup:**

```bash
# 1. Create ElastiCache Redis cluster (Subnet Group in private subnets)
aws elasticache create-cache-cluster \
  --cache-cluster-id auth-redis-prod \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --engine-version 7.x \
  --num-cache-nodes 2 \
  --preferred-maintenance-window "tue:05:00-tue:07:00" \
  --port 6379 \
  --subnet-group-name redis-subnet-group \
  --security-group-ids sg-redis-cluster

# 2. Add Security Group ingress for ALB → Redis (port 6379)
```

**Node.js (Express):**

```javascript
// package.json
"dependencies": {
  "connect-redis": "^6.1.0",
  "express": "^4.18.2",
  "express-session": "^1.17.3",
  "ioredis": "^5.3.2"
}

// server.js (with ElastiCache)
const session = require("express-session");
const RedisStore = require("connect-redis")(session);
const Redis = require("ioredis");
const RedisClient = new Redis({
  host: process.env.REDIS_HOST || 'auth-redis-prod.abcd.clustercfg.euw1.cache.amazonaws.com',
  port: 6379,
  password: process.env.REDIS_PASSWORD
});

app.use(session({
  store: new RedisStore({ client: RedisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production' // true in prod
  }
}));
```

**Recovery/Failover:**

1. Redis cluster detects node failure via health checks.
2. Automatic failover promotes replica to primary (30-60 seconds).
3. `connect-redis` automatically reconnects to new primary.
4. Users may experience brief session loss during failover.
5. For <24 hour sessions, this is acceptable. For longer sessions, implement:
   - Sticky sessions on ALB (least preferred - bypasses auto-scaling)
   - Redis cluster failover automation (AWS handles this natively)

#### Option 2: JWT Tokens with Redis Validation (Better)

```javascript
// package.json
"dependencies": {
  "express": "^4.18.2",
  "jsonwebtoken": "^9.0.2",
  "ioredis": "^5.3.2"
}
```

```javascript
// server.js (JWT with Redis token blacklist)
const jwt = require("jsonwebtoken");
const Redis = require("ioredis");

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  password: process.env.REDIS_PASSWORD,
});

function issueToken(userId) {
  const payload = { sub: userId, jti: `${userId}-${Date.now()}` };
  return jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: "1h" });
}

async function verifyToken(token) {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  const isRevoked = await redis.get(`revoked:${decoded.jti}`);
  if (isRevoked) {
    throw new Error("Token revoked");
  }
  return decoded;
}

async function revokeToken(token) {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  const ttl = decoded.exp - Math.floor(Date.now() / 1000);
  await redis.setex(`revoked:${decoded.jti}`, ttl, "1");
}
```

_Validation:_

1. Log in and receive a JWT.
2. Call a protected endpoint from two different instances; both should succeed.
3. Call logout, then retry the protected endpoint with the same token; it should fail.

_Common Errors:_

- **Token still works after logout:** Redis revoke key not set or TTL calculation wrong.
- **Unauthorized on all requests:** JWT secret mismatched across instances.

## Production Readiness Checklist

Use this checklist before declaring any project production ready.

### Infrastructure Checklist

```
ACCOUNT AND SECURITY
[ ] Root account MFA is enabled
[ ] Root access keys are deleted
[ ] All IAM users have MFA enabled
[ ] IAM users have least-privilege permissions
[ ] CloudTrail is enabled (audit logging)
[ ] Billing alerts are configured

NETWORKING
[ ] Dedicated VPC created for this project (not default VPC)
[ ] Application servers are in private subnets
[ ] Database is in a private subnet (not publicly accessible)
[ ] Security groups follow least-privilege
[ ] SSH port 22 is not open to 0.0.0.0/0
[ ] HTTPS (port 443) is configured

COMPUTE
[ ] EC2 instances have descriptive names and tags
[ ] Instances are in an Auto Scaling Group or have a recovery plan
[ ] Application starts automatically on reboot (PM2 startup)
[ ] Health check endpoint exists (/health returns 200)

STORAGE AND DATABASE
[ ] S3 Block Public Access is enabled
[ ] S3 versioning is enabled for critical data
[ ] RDS automated backups are configured (min 7 days)
[ ] RDS deletion protection is enabled

DEPLOYMENT
[ ] Secrets are in Secrets Manager (not in source code or .env files)
[ ] CI/CD pipeline is configured
[ ] Rollback procedure is documented and tested
[ ] Deployment does not cause downtime

MONITORING
[ ] CloudWatch CPU alarms are configured
[ ] Application logs are accessible
[ ] Billing anomaly detection is configured
[ ] On-call contact is documented

ISOLATION (FOR SHARED ACCOUNTS)
[ ] New project has its own VPC
[ ] New project has its own Security Groups
[ ] New project uses a unique CIDR range
[ ] New project uses different ports from existing projects
[ ] No existing resources were modified
[ ] Pre-flight audit was completed and documented
```

### Go-Live Decision Matrix

| Status | Decision |
| --- | --- |
| All checkboxes checked | Safe to go live |
| 1-3 minor items unchecked (non-security) | Go live with a documented remediation plan |
| Any security item unchecked | Do not go live |
| Any isolation item unchecked | Do not go live (risk of impacting existing systems) |
