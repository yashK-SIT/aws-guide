# AWS IAM: Users, Roles, Policies & Multi-Team Management

## Table of Contents
1. [IAM Fundamentals](#iam-fundamentals)
2. [Users vs Roles vs Policies](#users-vs-roles-vs-policies)
3. [Creating IAM Users](#creating-iam-users)
4. [IAM Roles](#iam-roles)
5. [IAM Policies](#iam-policies)
6. [Multi-Team Management](#multi-team-management)
7. [Cross-Account Access](#cross-account-access)
8. [Best Practices](#best-practices)

---

## IAM Fundamentals

### What is IAM?

IAM (Identity & Access Management) = Permission system for AWS.

Think of it as a company:
- **Users** = Employees (person)
- **Roles** = Job titles (temporary permissions)
- **Policies** = Job descriptions (specific permissions)
- **Groups** = Teams (collection of users with same permissions)

### Core Concepts

```
┌──────────────────────────────────────┐
│         AWS Resources                │
│  (EC2, S3, RDS, Lambda, etc.)       │
└──────────────────────────────────────┘
              ↑ Access controlled by
          ┌───────────────┐
          │  IAM Policy   │ (list of allowed/denied actions)
          └───────────────┘
              ↑ Assigned to
      ┌───────────┴────────────┐
      │                        │
  ┌────────────┐        ┌──────────┐
  │ IAM User   │        │ IAM Role │
  │(permanent) │        │(temporary)
  └────────────┘        └──────────┘
```

### Least Privilege Principle

**Golden Rule:** Give users/roles ONLY the permissions they need, nothing more.

```
❌ WRONG: Attach "AdministratorAccess" to every user
  → If one user compromised, attacker has full AWS access
  → Violates compliance requirements
  → Cannot audit who did what

✅ RIGHT: Create specific policy
  {
    "Effect": "Allow",
    "Action": [
      "ec2:StartInstances",
      "ec2:StopInstances",
      "ec2:DescribeInstances"
    ],
    "Resource": "arn:aws:ec2:*:*:instance/i-1234567890abcdef0"
  }
  → User can only start/stop specific instance
  → Cannot delete or access other resources
  → Easily auditable
```

---

## Users vs Roles vs Policies

### IAM User (Permanent Identity)

**What:** Long-term credentials for a person

**Use Case:** Developer, DevOps engineer, data analyst

**Characteristics:**
- Long-term access keys (stay valid until manually deleted)
- Console password (human can log in)
- MFA optional but recommended
- Lives in single AWS account

**Creation:**

```bash
aws iam create-user --user-name john.doe

# Set password for console access
aws iam create-login-profile \
  --user-name john.doe \
  --password 'TempPassword123!' \
  --password-reset-required

# Create access keys (for CLI/SDK)
aws iam create-access-key --user-name john.doe
```

**Pros:**
- Perfect for humans (console + CLI access)
- Persistent credentials
- Easy to audit (tracked by username)

**Cons:**
- Credentials must be managed
- If leaked, must manually revoke
- Not suitable for applications

### IAM Role (Temporary Identity)

**What:** Temporary credentials assumed by users/services

**Use Case:** Applications, Lambda, EC2 instances, cross-account access

**Characteristics:**
- Short-term credentials (1 hour default, up to 12 hours)
- No console password needed
- Automatically rotated
- Can be assumed by EC2, Lambda, users, cross-account principals

**Creation:**

```bash
aws iam create-role \
  --role-name MyAppRole \
  --assume-role-policy-document file://trust-policy.json

# Trust policy (who can assume this role)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Assume role from CLI:**

```bash
# Get temporary credentials
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyAppRole \
  --role-session-name MySession

# Response:
# {
#   "AssumedRoleUser": {...},
#   "Credentials": {
#     "AccessKeyId": "ASIAJ...",
#     "SecretAccessKey": "...",
#     "SessionToken": "...",
#     "Expiration": "2026-05-14T15:30:00Z"
#   }
# }

# Use credentials
export AWS_ACCESS_KEY_ID="ASIAJ..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

aws ec2 describe-instances  # Works with assumed role
```

**Pros:**
- Temporary credentials (safer if leaked)
- Automatic credential rotation
- Flexible (users, services, cross-account)
- Better for applications

**Cons:**
- Credentials must be refreshed
- More complex to understand

### IAM Policy (Permissions)

**What:** JSON document defining what actions are allowed/denied

**Structure:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*",
        "ec2:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowEC2Start",
      "Effect": "Allow",
      "Action": "ec2:StartInstances",
      "Resource": "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
    },
    {
      "Sid": "DenyDeleteEC2",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteVolume",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    }
  ]
}
```

**Components:**

- **Effect:** Allow or Deny
- **Action:** What operations (ec2:StartInstances, s3:GetObject, etc.)
- **Resource:** What things (specific ARN or *)
- **Condition:** When (optional - IP, time, tag, etc.)

**Attach to user/role:**

```bash
# Attach managed policy
aws iam attach-user-policy \
  --user-name john.doe \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Create and attach custom policy
aws iam put-user-policy \
  --user-name john.doe \
  --policy-name MyCustomPolicy \
  --policy-document file://custom-policy.json
```

---

## Creating IAM Users

### Step 1: Create User

```bash
aws iam create-user --user-name alice

# Verify
aws iam get-user --user-name alice
```

### Step 2: Attach Permission Policy

```bash
# Option A: Use AWS managed policy (easier)
aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::aws:policy/EC2ReadOnlyAccess

# Option B: Create custom policy (more control)
cat > ec2-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeNetworks",
        "ec2:DescribeImages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*"
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name alice \
  --policy-name EC2Manager \
  --policy-document file://ec2-policy.json
```

### Step 3: Create Console Password

```bash
aws iam create-login-profile \
  --user-name alice \
  --password 'TemporaryPassword123!' \
  --password-reset-required

# User must change password on first login
```

### Step 4: Create API Access Keys

```bash
aws iam create-access-key --user-name alice

# Response includes:
# AccessKeyId: AKIA...
# SecretAccessKey: ...

# Store securely (password manager, not email/Slack)
```

### Step 5: Configure MFA (Recommended)

```bash
# Create virtual MFA device
aws iam enable-mfa-device \
  --user-name alice \
  --serial-number arn:aws:iam::123456789012:mfa/alice \
  --authentication-code1 123456 \
  --authentication-code2 654321
# (6-digit codes from authenticator app)
```

---

## IAM Roles

### EC2 Instance Role

Allow EC2 instances to access AWS services without storing credentials.

**Create role:**

```bash
# Trust policy (EC2 service can assume this role)
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name MyEC2Role \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions
aws iam attach-role-policy \
  --role-name MyEC2Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Create instance profile (required for EC2)
aws iam create-instance-profile \
  --instance-profile-name MyEC2Profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name MyEC2Profile \
  --role-name MyEC2Role

# Launch EC2 with this role
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --iam-instance-profile Name=MyEC2Profile
```

**Inside EC2 instance, credentials automatically available:**

```bash
# No need to store credentials!
# AWS SDK automatically uses instance role

aws ec2 describe-instances  # Works!
aws s3 ls  # Works if role has S3 permissions!
```

### Lambda Execution Role

Allow Lambda functions to access AWS services.

```bash
# Create role for Lambda
aws iam create-role \
  --role-name LambdaExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach basic Lambda permissions
aws iam attach-role-policy \
  --role-name LambdaExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create custom permission (write to specific S3 bucket)
aws iam put-role-policy \
  --role-name LambdaExecutionRole \
  --policy-name S3Access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }]
  }'
```

---

## IAM Policies

### AWS Managed Policies

Pre-built policies by AWS (updated automatically):

```bash
# List all managed policies
aws iam list-policies

# Common useful policies:
arn:aws:iam::aws:policy/ReadOnlyAccess              # Everything, read-only
arn:aws:iam::aws:policy/PowerUserAccess             # Most things, no IAM changes
arn:aws:iam::aws:policy/EC2ReadOnlyAccess           # EC2 read-only
arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy # CloudWatch metrics
arn:aws:iam::aws:policy/AmazonS3FullAccess          # All S3 operations
```

### Customer Managed Policies

Your custom policies (version control, reusable):

```bash
# Create policy
aws iam create-policy \
  --policy-name EC2Operator \
  --policy-document file://policy.json

# Use policy
aws iam attach-user-policy \
  --user-name john \
  --policy-arn arn:aws:iam::123456789012:policy/EC2Operator

# Update policy
aws iam create-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/EC2Operator \
  --policy-document file://updated-policy.json \
  --set-as-default

# List policy versions
aws iam list-policy-versions \
  --policy-arn arn:aws:iam::123456789012:policy/EC2Operator

# Delete old versions (keep 5 max)
aws iam delete-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/EC2Operator \
  --version-id v2
```

### Inline Policies

Policies directly attached to user/role (not reusable):

```bash
# Attach inline policy to user
aws iam put-user-policy \
  --user-name john \
  --policy-name InlinePolicy \
  --policy-document file://policy.json

# List inline policies
aws iam list-user-policies --user-name john

# Delete inline policy
aws iam delete-user-policy \
  --user-name john \
  --policy-name InlinePolicy
```

### Condition Keys (Advanced)

Add conditions to policies for extra security:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24"]
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2026-05-01T00:00:00Z"
        }
      }
    }
  ]
}
```

---

## Multi-Team Management

### Team Structure

```
Organization
├── Finance Team
│   ├── Alice (read billing, cost analysis)
│   └── Bob (modify budgets, reserved instances)
├── Engineering Team
│   ├── Charlie (full EC2/RDS access)
│   └── David (read-only EC2)
└── Operations Team
    ├── Eve (all resources, audit logs)
    └── Frank (specific environments only)
```

### Create Groups

Groups = collections of users with same permissions.

```bash
# Create group for finance team
aws iam create-group --group-name FinanceTeam

# Add users to group
aws iam add-user-to-group \
  --group-name FinanceTeam \
  --user-name alice

aws iam add-user-to-group \
  --group-name FinanceTeam \
  --user-name bob

# Attach policy to group (all members get permission)
aws iam attach-group-policy \
  --group-name FinanceTeam \
  --policy-arn arn:aws:iam::aws:policy/BillingReadOnlyAccess

# List group members
aws iam get-group --group-name FinanceTeam
```

### Team Permission Examples

**Finance Team Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BillingReadOnly",
      "Effect": "Allow",
      "Action": [
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "budgets:ViewBudget",
        "aws-portal:ViewBilling"
      ],
      "Resource": "*"
    }
  ]
}
```

**Engineering Team Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2FullAccess",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {"aws:RequestedRegion": "us-east-1"}
      }
    },
    {
      "Sid": "RDSManagement",
      "Effect": "Allow",
      "Action": [
        "rds-db:connect",
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

**Operations Team Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FullAccess",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    },
    {
      "Sid": "PreventDeletion",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteUser",
        "iam:DeleteRole",
        "organizations:DeleteAccount"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Cross-Account Access

Allow users in one AWS account to access another account.

**Scenario:** Production account needs to access logging in security account.

### Setup Cross-Account Role

**In Security Account:**

```bash
# Create role that Production account can assume
aws iam create-role \
  --role-name CrossAccountLogsRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PROD-ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UniqueExternalId"
        }
      }
    }]
  }'

# Attach permissions
aws iam put-role-policy \
  --role-name CrossAccountLogsRole \
  --policy-name LogsAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:GetLogEvents"
      ],
      "Resource": "arn:aws:logs:*:SECURITY-ACCOUNT-ID:*"
    }]
  }'
```

### Assume Cross-Account Role

**In Production Account:**

```bash
# Create policy allowing users to assume role
aws iam put-user-policy \
  --user-name engineer \
  --policy-name AssumeLogsRole \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::SECURITY-ACCOUNT-ID:role/CrossAccountLogsRole"
    }]
  }'

# User assumes the role
aws sts assume-role \
  --role-arn arn:aws:iam::SECURITY-ACCOUNT-ID:role/CrossAccountLogsRole \
  --role-session-name ProductionSession \
  --external-id UniqueExternalId

# Get credentials and use them
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Now can access Security account logs
aws logs describe-log-groups --region us-east-1
```

---

## Best Practices

### 1. Use Roles for Applications, Users for Humans

| Use Case | Type | Why |
|----------|------|-----|
| Developer accessing AWS | User | Long-term, revokable |
| EC2 instance | Role | Automatic rotation, temporary |
| Lambda function | Role | No credentials to manage |
| Service integration | Role | Cross-account safe |

### 2. Enable MFA Everywhere

```bash
# Require MFA for all users
aws iam put-account-password-policy \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --require-numbers \
  --require-symbols \
  --minimum-password-length 14 \
  --password-reuse-prevention 5 \
  --max-password-age 90
```

### 3. Rotate Credentials

```bash
# Delete old access keys (if unused for 90 days)
aws iam list-access-keys --user-name john

# Delete old key
aws iam delete-access-key \
  --user-name john \
  --access-key-id AKIA...

# Create new key
aws iam create-access-key --user-name john
```

### 4. Audit IAM Changes

```bash
# Enable CloudTrail to log all IAM changes
aws cloudtrail create-trail \
  --name IAMTrail \
  --s3-bucket-name my-logs

# Query IAM changes
aws cloudtrail lookup-events \
  --event-name CreateUser \
  --max-results 10
```

### 5. Review Policies Regularly

```bash
# Find overly permissive policies
aws iam list-users
aws iam list-user-policies --user-name john
# Review each policy for least privilege
```

---

## Next Steps

1. **Create your first IAM user** → Practice in console
2. **Attach appropriate policies** → Start with AWS managed policies
3. **Enable MFA** → Add phone authenticator
4. **Create groups** → Organize users
5. **Audit permissions** → Use Access Analyzer
