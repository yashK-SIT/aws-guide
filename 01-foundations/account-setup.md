# AWS Account Setup & Security: Foundation for Everything

## Table of Contents
1. [Account Creation](#account-creation)
2. [Root User Lockdown](#root-user-lockdown)
3. [Billing Management](#billing-management)
4. [Multi-Account Strategy](#multi-account-strategy)
5. [Account Audit](#account-audit)

---

## Account Creation

### Step 1: Create AWS Account

Visit https://aws.amazon.com and click "Create an AWS Account"

**Required Information:**
- Email address (recoverable, unique per account)
- Password (15+ characters, mix of upper/lower/numbers/symbols)
- Account name (company/project name)
- Billing address
- Credit/debit card (AWS requires this; won't charge for free tier)
- Phone number (AWS calls for verification)

**What You Get:**
- 12-month free tier (750 hours EC2 t2.micro/month, etc.)
- 1 AWS account with root user access
- Automatic billing alerts setup

### Step 2: Verify Phone Number

AWS calls your phone to verify. Answer and enter the verification code shown on screen.

### Step 3: Choose Support Plan

Options:
- **Basic (Free)** - Recommended for learning. No cost, email support
- **Developer ($29/month)** - Business hours email support
- **Business ($100+/month)** - Production apps, chat support
- **Enterprise ($15k+/month)** - Mission-critical, TAM support

Start with **Basic**.

---

## Root User Lockdown

**CRITICAL:** Root user has unrestricted access. After account creation, LOCK IT DOWN immediately.

### Why Root User is Dangerous

```
Root user capabilities:
├── Delete entire AWS account
├── Close account and lose everything
├── Delete all infrastructure
├── Access all data
├── Modify billing settings
├── View all credentials
└── Cannot be audited or restricted
```

**Best practice:** Root user for emergencies only. Daily work uses IAM user.

### Step 1: Enable MFA on Root Account

MFA (Multi-Factor Authentication) = Phone confirms every login.

**In AWS Console:**

1. Click your account name (top right)
2. Select "My Security Credentials"
3. Click "MFA" section
4. "Assign MFA device"
5. Virtual MFA device (use phone authenticator app)
6. Scan QR code with:
   - Google Authenticator
   - Microsoft Authenticator
   - Authy
7. Enter 6-digit code from app
8. Complete setup

**Result:** Every root login requires phone confirmation.

### Step 2: Create Root User Backup Credentials

Store these securely (not in email/Slack):

```bash
# Document these details:
- AWS Account ID: 123456789012
- Root email: admin@company.com
- MFA ARN: arn:aws:iam::123456789012:mfa/root-account-mfa
- Password: (stored in password manager only, never written down)
```

Store in:
- Password manager (Bitwarden, 1Password, LastPass)
- Encrypted external drive (USB, not cloud storage)
- Safe deposit box (printed copy)

### Step 3: Create Root Credentials File (Emergency Only)

Keep printed copy in secure location (safe, locked drawer):

```
ROOT USER EMERGENCY CREDENTIALS
================================
Account ID: 123456789012
Email: admin@example.com
Password: [printed, never stored digitally]
MFA Device: Keep phone with authenticator app
Created: 2026-05-14
Last Updated: 2026-05-14

NEVER USE UNLESS:
- All IAM users compromised
- Billing changes needed
- Account recovery required

DELETE THIS AFTER 1 YEAR IF UNUSED
```

### Step 4: CloudTrail Setup (Audit Root Activity)

CloudTrail logs all AWS API calls (including root user).

**Steps:**

```bash
aws cloudtrail create-trail \
  --name RootActivityTrail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail

# Enable logging
aws cloudtrail start-logging \
  --trail-name RootActivityTrail
```

**Benefits:**
- See every root user action
- Detect unauthorized access
- Audit trail for compliance

---

## Billing Management

### Setup Billing Alerts

Prevent bill shock. AWS sends alerts when approaching budget.

**Via Console:**

1. Go to Billing Dashboard
2. Click "Billing Preferences"
3. Check "Receive Billing Alerts"
4. Save preferences
5. Go to CloudWatch → Alarms
6. Create alarm:
   - Metric: EstimatedCharges
   - Threshold: $50 (adjust to your budget)
   - Action: Send SNS email notification
7. Confirm email subscription

**Via CLI:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name BillingAlert-50USD \
  --alarm-description "Alert if monthly charges exceed $50" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 3600 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:BillingAlerts
```

### Monitor Costs

**Cost Explorer:**

1. Billing Dashboard
2. Click "Cost Explorer"
3. View spending by:
   - Service (EC2, RDS, etc.)
   - Time period (daily, monthly)
   - Region
   - Account

**Set Budgets:**

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget "{
    \"BudgetName\": \"Monthly-Budget\",
    \"BudgetLimit\": {\"Amount\": \"100\", \"Unit\": \"USD\"},
    \"TimeUnit\": \"MONTHLY\",
    \"BudgetType\": \"COST\"
  }"
```

### Cost Optimization Quick Wins

| Action | Savings | Difficulty |
|--------|---------|-----------|
| Use free tier resources | Save $50-200/month | Easy |
| Delete unused resources | Save $10-100/month | Easy |
| Reserved instances (1-yr) | Save 30% | Medium |
| Spot instances | Save 70% | Hard |
| S3 Intelligent-Tiering | Save 10-20% | Medium |
| Compress CloudFront | Save 20-40% | Easy |

---

## Multi-Account Strategy

For teams/organizations, use multiple AWS accounts:

```
Organization
├── Master Account (billing, consolidated view)
├── Production Account (prod infrastructure)
├── Staging Account (staging environment)
├── Development Account (dev environment)
└── Security Account (logging, monitoring)
```

### Why Multiple Accounts?

**Isolation:**
- Production failure doesn't affect staging
- Prevent accidental deletion of critical resources
- Sandbox for experimentation

**Cost Tracking:**
- Separate billing per environment
- Easy to identify expensive projects

**Security:**
- Limited blast radius if one account compromised
- Different access policies per account
- Audit isolation

### Setup AWS Organizations

**Steps:**

```bash
# Create organization
aws organizations create-organization \
  --feature-set ALL

# List accounts
aws organizations list-accounts

# Create new account
aws organizations create-account \
  --account-name "Production" \
  --email prod@example.com

# Get account ID
aws organizations list-accounts \
  --query "Accounts[?Name=='Production'].Id"
```

### Setup Cross-Account Access

Allow production account to access staging logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PROD-ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

### Cost Allocation Tags

Tag all resources for cost tracking:

```bash
# Tag all production EC2 instances
aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags Key=Environment,Value=Production Key=CostCenter,Value=Engineering

# View costs by tag
# (In Cost Explorer: Group by Tags)
```

---

## Account Audit

Verify account is properly secured:

### Security Audit Checklist

- [ ] MFA enabled on root user
- [ ] Root user credentials backed up securely
- [ ] IAM users created (not using root daily)
- [ ] CloudTrail logging enabled
- [ ] Billing alerts configured
- [ ] Account has trusted contacts
- [ ] No permanent root access keys (root user shouldn't have access keys)
- [ ] CloudWatch monitoring enabled
- [ ] VPC created (not using default VPC)
- [ ] All resources tagged appropriately

### Run AWS Access Analyzer

Identifies overly permissive access:

```bash
# Create analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name MyAnalyzer \
  --type ACCOUNT

# Run analysis
aws accessanalyzer validate-policy \
  --policy-document file://policy.json \
  --policy-type IDENTITY_POLICY
```

### Generate IAM Credential Report

Lists all users, access keys, passwords, MFA status:

```bash
# Generate report
aws iam generate-credential-report

# Get report
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d > credentials-report.csv

# Review in Excel for:
# - Users with active access keys
# - Users without MFA
# - Unused credentials
# - Password age
```

### AWS Trusted Advisor

Free security checks (with free tier):

```bash
aws support describe-trusted-advisor-checks \
  --query "checks[?name=='Security Groups - Specific Ports Unrestricted'].id"

aws support describe-trusted-advisor-check-result \
  --check-id "specific-id-from-above" \
  --language en
```

---

## Account Recovery & Troubleshooting

### Lost MFA Device

1. Sign in with root credentials (email + password)
2. AWS sends verification email
3. Verify email address
4. Remove old MFA device
5. Add new MFA device

### Account Compromised

**Immediate Actions:**

1. Change root password
2. Rotate all access keys
3. Review CloudTrail logs for unauthorized actions
4. Delete any unknown resources (EC2, RDS, etc.)
5. Check billing for unexpected charges
6. Contact AWS Support
7. File police report if data theft suspected

**AWS Support Contact:**
- https://console.aws.amazon.com/support/home
- Click "Create case"
- Choose "Account and billing"
- Severity: High

### Access Key Leaked

```bash
# If your access key is published (GitHub, etc.):

# 1. List access keys
aws iam list-access-keys

# 2. Delete compromised key
aws iam delete-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE

# 3. Create new access key
aws iam create-access-key \
  --user-name myuser

# 4. Update local AWS credentials file (~/.aws/credentials)
# 5. Deploy new credentials to servers
# 6. Check CloudTrail for unauthorized actions
```

---

## Next Steps

After securing your account:

1. **Create IAM Users** → Read: `01-foundations/iam-essentials.md`
2. **Set Up VPC** → Read: `02-networking/vpc-complete.md`
3. **Launch EC2** → Read: `03-compute/ec2-instances.md`
4. **Monitor Costs** → Read: `10-resilience/cost-optimization.md`
