# Elastic IP: Static IP Addresses for AWS

## Table of Contents
1. [What is Elastic IP](#what-is-elastic-ip)
2. [Allocating Elastic IP](#allocating-elastic-ip)
3. [Associating with Resources](#associating-with-resources)
4. [Cost Considerations](#cost-considerations)
5. [Best Practices](#best-practices)
6. [Troubleshooting](#troubleshooting)

---

## What is Elastic IP?

### Regular Public IP vs Elastic IP

| Feature | Public IP | Elastic IP |
|---------|---|---|
| **Persistence** | Changes on instance stop/start | Never changes |
| **Ownership** | AWS owns it | Your account owns it |
| **Availability** | Automatic, can't choose | You allocate it |
| **Cost** | Free | Free if associated, $0.005/hour if not |
| **Use Case** | Development, temporary | Production, DNS records |

### When to Use Elastic IP

**Use Elastic IP when:**
- Setting up DNS records (need static IP)
- Running production website (IP should never change)
- Need predictable server address for clients
- Using Route53 for domain mapping

**Don't need Elastic IP when:**
- Development/testing only
- Load balancer handles traffic (ALB/NLB)
- Using CloudFront (no direct server connection)
- Private instance (no internet access)

### Architecture Example

```
Without Elastic IP (Fragile):
Domain → myapp.com
  ↓ (DNS points to IP)
EC2 Instance: 52.123.456.789 (might change!)
  → Instance stops/starts
  → Instance reboots
  → AWS changes the IP
  → DNS outdated, site down

With Elastic IP (Stable):
Domain → myapp.com
  ↓ (DNS points to Elastic IP)
Elastic IP: 52.123.456.789 (permanent!)
  ↓ (associated with)
EC2 Instance: might have different public IP, but doesn't matter
  → Instance stops/starts
  → Instance reboots
  → IP NEVER changes, DNS always works
```

---

## Allocating Elastic IP

### Via AWS Console

1. EC2 Dashboard
2. Left menu → "Elastic IPs"
3. Click "Allocate Elastic IP address"
4. Network Border Group: (leave as default)
5. Public IPv4 address pool: Amazon pool
6. Click "Allocate"

**Result:** You now own a permanent IP address (52.xxx.xxx.xxx)

### Via AWS CLI

```bash
# Allocate Elastic IP
aws ec2 allocate-address \
  --domain vpc

# Response:
# {
#   "PublicIp": "52.123.456.789",
#   "AllocationId": "eipalloc-0123456789abcdef0",
#   "Domain": "vpc"
# }

# Save the AllocationId (needed later)
ALLOCATION_ID="eipalloc-0123456789abcdef0"

# List all Elastic IPs
aws ec2 describe-addresses

# Get specific details
aws ec2 describe-addresses \
  --allocation-ids eipalloc-0123456789abcdef0
```

### Allocate Multiple

```bash
# For high-availability setup (multiple servers)
for i in {1..3}; do
  aws ec2 allocate-address --domain vpc
done

# List all
aws ec2 describe-addresses \
  --query "Addresses[].PublicIp"

# Output:
# 52.123.456.789
# 52.234.567.890
# 52.345.678.901
```

---

## Associating with Resources

### Associate with EC2 Instance

**Via CLI:**

```bash
# Get instance ID
INSTANCE_ID="i-0123456789abcdef0"
ALLOCATION_ID="eipalloc-0123456789abcdef0"

# Associate Elastic IP with instance
aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $ALLOCATION_ID

# Response:
# {
#   "AssociationId": "eipassoc-0123456789abcdef0"
# }

# Verify
aws ec2 describe-addresses --allocation-ids $ALLOCATION_ID

# Output shows:
# InstanceId: i-0123456789abcdef0
# AssociationId: eipassoc-0123456789abcdef0
# PublicIp: 52.123.456.789
# AssociationStatus: associated
```

**Via Console:**

1. Elastic IPs page
2. Select Elastic IP
3. Click "Associate Elastic IP address"
4. Instance: Select your instance
5. Network interface: (auto-selected)
6. Click "Associate"

### Associate with Network Interface

```bash
# Get network interface ID
INSTANCE_ID="i-0123456789abcdef0"
ENI_ID=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].NetworkInterfaces[0].NetworkInterfaceId" \
  --output text)

# Associate with specific network interface
aws ec2 associate-address \
  --network-interface-id $ENI_ID \
  --allocation-id eipalloc-0123456789abcdef0 \
  --private-ip-address 10.0.1.50
```

### Reassociate Elastic IP

If replacing a failed server:

```bash
# Disassociate from old instance
aws ec2 disassociate-address \
  --association-id eipassoc-old

# Associate with new instance
aws ec2 associate-address \
  --instance-id i-new \
  --allocation-id eipalloc-0123456789abcdef0

# Elastic IP now points to new instance
# DNS records automatically work with new server
```

---

## Cost Considerations

### Elastic IP Charges

**Scenario 1: Associated with Running Instance**

```
Cost: FREE
Elastic IP: 52.123.456.789
Instance: i-0123456789abcdef0 (running)
Status: Associated and in use
```

**Scenario 2: Associated with Stopped Instance**

```
Cost: $0.005/hour = ~$3.65/month
Elastic IP: 52.123.456.789
Instance: i-0123456789abcdef0 (stopped)
Status: Associated but not in use
```

⚠️ **Surprise cost:** If you allocate Elastic IPs and don't use them, they cost money!

**Scenario 3: Not Associated**

```
Cost: $0.005/hour = ~$3.65/month
Elastic IP: 52.123.456.789
Instance: (none)
Status: Allocated but unused
```

### Optimization

```bash
# List unused Elastic IPs (not associated)
aws ec2 describe-addresses \
  --filters "Name=association-id,Values=None" \
  --query "Addresses[].PublicIp"

# Release unused IPs to save costs
aws ec2 release-address \
  --allocation-id eipalloc-unused

# Set up billing alert for Elastic IPs
aws cloudwatch put-metric-alarm \
  --alarm-name UnusedElasticIPs \
  --metric-name UnassociatedAddressCount \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 86400 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold
```

---

## Best Practices

### 1. Use for Production Only

```
❌ DON'T:
- Allocate Elastic IP for development
- Leave unused Elastic IPs lying around
- Allocate 10 IPs but only use 1

✅ DO:
- Allocate only what you need
- Release unused IPs immediately
- Use Elastic IP for production/DNS only
- Auto-cleanup unused IPs monthly
```

### 2. Document Your IPs

```bash
# Add tags to track usage
aws ec2 create-tags \
  --resources eipalloc-0123456789abcdef0 \
  --tags \
    Key=Name,Value=prod-web-server-1 \
    Key=Environment,Value=Production \
    Key=Owner,Value=devops-team \
    Key=CostCenter,Value=engineering

# List with tags
aws ec2 describe-addresses \
  --query "Addresses[].[PublicIp,Tags[?Key=='Name'].Value|[0],Tags[?Key=='Environment'].Value|[0]]" \
  --output table
```

### 3. Plan IP Allocation

```
Before launching:
├─ How many instances need static IPs? (only public ones)
├─ Development: Do they need Elastic IP? (usually no)
├─ Staging: Maybe 1-2 for testing
├─ Production: 1 per web server
└─ Total needed: Calculate and allocate at start

Example for scaling:
Current: 3 web servers = 3 Elastic IPs
Allocate: 5 Elastic IPs (buffer for 2 more)
Don't allocate: 10 (waste of money)
```

### 4. Use with Route53

```bash
# Create hosted zone
aws route53 create-hosted-zone \
  --name myapp.com \
  --caller-reference "unique-id"

# Create A record pointing to Elastic IP
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC456 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "myapp.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "52.123.456.789"}]
      }
    }]
  }'

# If need to change server:
# 1. Associate Elastic IP with new instance
# 2. Route53 automatically serves new instance (TTL delay)
```

### 5. Monitor Elastic IP Health

```bash
# Create script to check Elastic IPs
#!/bin/bash
echo "=== Elastic IP Status ==="
aws ec2 describe-addresses --query "Addresses[].[PublicIp,Tags[?Key=='Name'].Value|[0],AssociationId]" --output table

# Run daily to audit
0 9 * * * /usr/local/bin/check-eips.sh

# Check for unassociated
UNASSOCIATED=$(aws ec2 describe-addresses \
  --filters "Name=association-id,Values=None" \
  --query "length(Addresses[])")

if [ "$UNASSOCIATED" -gt 0 ]; then
  echo "WARNING: $UNASSOCIATED unused Elastic IPs costing money!"
fi
```

---

## Troubleshooting

### Can't Associate Elastic IP

**Error: "Resource.AlreadyAssociated"**

```
Cause: Elastic IP already associated with another instance
Solution: Disassociate first, then re-associate

aws ec2 disassociate-address \
  --association-id eipassoc-old

aws ec2 associate-address \
  --instance-id i-new \
  --allocation-id eipalloc-xxx
```

**Error: "InvalidInstanceID.NotFound"**

```
Cause: Instance doesn't exist
Solution: Verify instance ID is correct

aws ec2 describe-instances --instance-ids i-xxx
# Should return instance details, not empty
```

### Elastic IP Not Reachable

```bash
# Check if associated
aws ec2 describe-addresses --allocation-ids eipalloc-xxx
# Should show: AssociationId (if associated)

# Check instance is running
aws ec2 describe-instances --instance-ids i-xxx \
  --query "Reservations[0].Instances[0].State.Name"
# Should show: running

# Check security group allows inbound traffic
aws ec2 describe-security-groups --group-ids sg-xxx \
  --query "SecurityGroups[0].IpPermissions[].[FromPort,ToPort,IpRanges[0].CidrIp]"
# Should show open ports

# Test connectivity
ping 52.123.456.789 -c 1  # Check ICMP
ssh -v ec2-user@52.123.456.789  # Check SSH
curl http://52.123.456.789  # Check HTTP
```

### Elastic IP Changed (Shouldn't Happen)

```bash
# Check if still associated
aws ec2 describe-addresses --allocation-ids eipalloc-xxx

# If shows different instance/association:
# Likely someone disassociated and associated with another instance

# Check CloudTrail for who changed it
aws cloudtrail lookup-events \
  --event-name DisassociateAddress \
  --max-results 10
```

---

## Migration: Elastic IP to Load Balancer

As you scale, migrate from Elastic IP to Application Load Balancer:

```
Stage 1 (Single Server):
Domain: myapp.com → Elastic IP → EC2 instance

Stage 2 (Multiple Servers):
Domain: myapp.com → ALB → EC2 instances (in ASG)
(No Elastic IP needed; ALB handles traffic distribution)
```

### Migration Steps

```bash
# 1. Create ALB
aws elbv2 create-load-balancer \
  --name myapp-alb \
  --subnets subnet-1a subnet-1b

# 2. Point domain to ALB
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC456 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "myapp.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "myapp-alb-123.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'

# 3. Release Elastic IP (free up money!)
aws ec2 release-address --allocation-id eipalloc-xxx
```

---

## Next Steps

1. Allocate Elastic IP for your production servers
2. Associate with running instances
3. Point your domain (Route53) to Elastic IP
4. Set up billing alert for unused IPs
5. Test: Verify domain resolves to IP
6. When scaling: Migrate to ALB (stop using Elastic IP)
