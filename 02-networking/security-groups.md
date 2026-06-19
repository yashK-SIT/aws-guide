# Security Groups: Virtual Firewalls for AWS

## Table of Contents
1. [Security Groups Explained](#security-groups-explained)
2. [Inbound Rules](#inbound-rules)
3. [Outbound Rules](#outbound-rules)
4. [Best Practices](#best-practices)
5. [Common Configurations](#common-configurations)
6. [Troubleshooting](#troubleshooting)

---

## Security Groups Explained

### What is a Security Group?

Security Group = **Virtual firewall** that controls traffic to/from AWS resources (EC2, RDS, ELB, etc.).

```
Internet
   ↓
Route53 (DNS)
   ↓
Security Group (Firewall)
   ├─ Allow port 443 (HTTPS) from 0.0.0.0/0 (internet)
   ├─ Allow port 80 (HTTP) from 0.0.0.0/0 (internet)
   ├─ Allow port 22 (SSH) from 203.0.113.0/32 (your IP)
   ├─ Deny everything else by default
   ↓
EC2 Instance
   ├─ Nginx on port 443/80
   ├─ SSH on port 22
   ├─ Application on port 3000
```

### Key Characteristics

- **Stateful:** If inbound traffic allowed, response traffic automatically allowed (no outbound rule needed)
- **Default deny:** All inbound traffic blocked unless explicitly allowed
- **Default allow:** All outbound traffic allowed unless explicitly denied
- **No logs:** Security groups don't log traffic (use VPC Flow Logs for that)

### Security Group vs NACL

| Feature | Security Group | Network ACL |
|---------|---|---|
| **Level** | Instance-level | Subnet-level |
| **Stateful** | Yes (allow response automatically) | No (must allow both directions) |
| **Default** | Deny inbound, allow outbound | Allow all by default |
| **Apply to** | EC2, RDS, ELB, etc. | Entire subnet |
| **Rules** | Allow/Deny only | Allow/Deny explicitly |

---

## Inbound Rules

Inbound rules control traffic COMING IN to your resource.

### Rule Components

```
Protocol | Port | CIDR/Security Group | Description
---------|------|-------------------|-------------
TCP      | 22   | 0.0.0.0/0          | SSH access
TCP      | 80   | 0.0.0.0/0          | HTTP web traffic
TCP      | 443  | 0.0.0.0/0          | HTTPS web traffic
TCP      | 3000 | 10.0.1.0/24        | App port (internal only)
```

### Creating Inbound Rules

**Via CLI:**

```bash
# Allow HTTP from internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH from specific IP only (more secure)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.42/32

# Allow from another security group (EC2 to RDS)
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds-group \
  --protocol tcp \
  --port 5432 \
  --source-group sg-ec2-group
```

### Common Inbound Rules

**Web Server:**

```
Protocol | Port | Source | Purpose
---------|------|--------|----------
TCP      | 80   | 0.0.0.0/0 | HTTP
TCP      | 443  | 0.0.0.0/0 | HTTPS
TCP      | 22   | YOUR_IP/32 | SSH (admin only)
```

**Database Server (RDS):**

```
Protocol | Port | Source | Purpose
---------|------|--------|----------
TCP      | 5432 | sg-app-sg | PostgreSQL (from app servers only)
TCP      | 3306 | sg-app-sg | MySQL (from app servers only)
```

**Load Balancer:**

```
Protocol | Port | Source | Purpose
---------|------|--------|----------
TCP      | 80   | 0.0.0.0/0 | HTTP
TCP      | 443  | 0.0.0.0/0 | HTTPS
```

**Application Server (Internal):**

```
Protocol | Port | Source | Purpose
---------|------|--------|----------
TCP      | 3000 | sg-lb-sg | From load balancer only
TCP      | 22   | sg-bastion-sg | SSH from bastion host only
```

### Rule Precedence

If multiple rules match, AWS applies **ALL matching allow rules**:

```
Rule 1: Allow port 22 from 0.0.0.0/0
Rule 2: Allow port 22 from 203.0.113.42/32
→ Both apply, port 22 is open from anywhere

To restrict, use explicit deny (rare):
Rule 1: Allow port 22 from 0.0.0.0/0
Rule 2: Deny port 22 from 203.0.113.50/32
→ Port 22 open except from 203.0.113.50
```

---

## Outbound Rules

Outbound rules control traffic GOING OUT from your resource.

### Default Outbound Rule

By default, security groups allow ALL outbound traffic:

```
Protocol | Port | Destination
---------|------|-------------
ALL      | ALL  | 0.0.0.0/0
```

This means:
- EC2 can reach internet (download packages, APIs)
- EC2 can reach other AWS resources
- EC2 can reach databases

### Restrictive Outbound Rules (Advanced)

For high-security environments, restrict outbound:

```bash
# Remove default allow-all rule
aws ec2 revoke-security-group-egress \
  --group-id sg-0123456789abcdef0 \
  --protocol -1 \
  --port -1 \
  --cidr 0.0.0.0/0

# Allow only specific outbound (hardened)
aws ec2 authorize-security-group-egress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0  # HTTPS only

aws ec2 authorize-security-group-egress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 3306 \
  --cidr 10.0.1.50/32  # To specific database
```

**Result:** EC2 can ONLY:
- Reach internet on port 443 (HTTPS)
- Connect to database on port 3306
- Cannot download packages, connect to APIs on other ports, etc.

---

## Best Practices

### 1. Use Least Privilege

```
❌ WRONG:
Allow all traffic from 0.0.0.0/0
→ Anyone on internet can access everything

✅ RIGHT:
Allow port 80/443 from 0.0.0.0/0 (web traffic)
Allow port 22 from YOUR_IP/32 (SSH admin only)
Allow port 3000 from ALB security group only (app traffic)
```

### 2. Use Security Group References

Instead of IP addresses, reference other security groups:

```bash
# Bad: Hard-coded IPs
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 3000 \
  --cidr 10.0.1.50/32  # Brittle, changes break things

# Good: Reference security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 3000 \
  --source-group sg-alb
```

### 3. Organize by Tier

```
Backend Network:
├── sg-alb (Load Balancer)
│   ├─ Inbound: 80, 443 from 0.0.0.0/0
│   └─ Outbound: ALL to 0.0.0.0/0
│
├── sg-app (Application)
│   ├─ Inbound: 3000 from sg-alb
│   ├─ Inbound: 22 from sg-bastion
│   └─ Outbound: ALL to 0.0.0.0/0
│
├── sg-rds (Database)
│   ├─ Inbound: 5432 from sg-app
│   └─ Outbound: NONE (or minimal)
│
└── sg-bastion (Admin Access)
    ├─ Inbound: 22 from YOUR_IP/32
    └─ Outbound: 22 to sg-app, others
```

### 4. Document Rules

Add descriptions to every rule:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --description "HTTPS from internet - customer traffic"

# View with descriptions
aws ec2 describe-security-groups --group-ids sg-web \
  --query 'SecurityGroups[0].IpPermissions'
```

### 5. Review Regularly

```bash
# Find overly permissive rules (0.0.0.0/0)
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=*" \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].GroupId'

# Review each one
aws ec2 describe-security-groups --group-ids sg-xxx
```

---

## Common Configurations

### Web Server (Public Facing)

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 \
  --description "HTTP from internet"

aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp --port 443 --cidr 0.0.0.0/0 \
  --description "HTTPS from internet"

aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp --port 22 --cidr 203.0.113.42/32 \
  --description "SSH from admin IP only"
```

### Application Server (Private)

```bash
# Allow traffic from ALB only
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp --port 3000 --source-group sg-alb \
  --description "Application traffic from ALB"

# Allow SSH from bastion host
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp --port 22 --source-group sg-bastion \
  --description "SSH from bastion host"

# Outbound: allow all
# (default rule)
```

### Database (RDS)

```bash
# Allow from application tier only
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp --port 5432 --source-group sg-app \
  --description "PostgreSQL from app servers"

# Restrict outbound (no connections needed out)
aws ec2 revoke-security-group-egress \
  --group-id sg-rds \
  --protocol -1 --port -1 --cidr 0.0.0.0/0

# Allow minimal outbound if needed
aws ec2 authorize-security-group-egress \
  --group-id sg-rds \
  --protocol tcp --port 443 --cidr 0.0.0.0/0 \
  --description "AWS APIs (monitoring, backups)"
```

---

## Troubleshooting

### Can't SSH to Instance

```bash
# Check security group allows port 22
aws ec2 describe-security-groups --group-ids sg-xxx \
  --query 'SecurityGroups[0].IpPermissions[?FromPort==`22`]'

# Should see entry allowing port 22 from your IP (or 0.0.0.0/0)

# If missing, add rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp --port 22 \
  --cidr YOUR_IP/32

# If rule exists but still can't SSH:
# 1. Check network ACLs (might be blocking at subnet level)
# 2. Verify instance is running
# 3. Check instance has public IP/Elastic IP
```

### Can't Reach RDS from Application

```bash
# Check RDS security group allows port 5432
aws ec2 describe-security-groups --group-ids sg-rds \
  --query 'SecurityGroups[0].IpPermissions'

# Should show port 5432 from sg-app

# Check application security group is in RDS rule
aws ec2 describe-security-groups --group-ids sg-rds \
  --query 'SecurityGroups[0].IpPermissions[?FromPort==`5432`].UserIdGroupPairs'

# If missing, add rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp --port 5432 \
  --source-group sg-app
```

### Traffic Between Instances Blocked

```bash
# If Instance A can't reach Instance B:

# 1. Check Instance A outbound (should be all by default)
aws ec2 describe-security-groups --group-ids sg-a \
  --query 'SecurityGroups[0].IpPermissionsEgress'

# 2. Check Instance B inbound (should allow from sg-a)
aws ec2 describe-security-groups --group-ids sg-b \
  --query 'SecurityGroups[0].IpPermissions'

# If not allowed, add:
aws ec2 authorize-security-group-ingress \
  --group-id sg-b \
  --protocol tcp --port PORT \
  --source-group sg-a
```

---

## Advanced: VPC Flow Logs

See actual traffic (not just rules):

```bash
# Enable VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type NetworkInterface \
  --resource-ids eni-0123456789abcdef0 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs

# Query logs
aws logs tail /aws/vpc/flowlogs --follow

# Typical flow log line:
# 2 123456789012 eni-0123456 203.0.113.42 10.0.1.50 52382 443 6 1024 65536 ACCEPT 1234567890 1234567900 OK OK
# Fields: version account-id interface-id srcip dstip srcport dstport protocol bytes packets action
```

---

## Next Steps

1. Create security groups for your architecture2. Test inbound/outbound rules
3. Document each rule3. Review for least privilege
4. Set up monitoring with VPC Flow Logs
