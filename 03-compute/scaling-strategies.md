# AWS Scaling Strategies: Auto Scaling & Load Balancing

**File Version:** 1.0  
**Last Updated:** May 2026  
**Audience:** Everyone — Beginners through Senior Architects  
**Prerequisites:** Read `03-compute/ec2-instances.md` (recommended)  
**Time to Read:** 50-70 minutes (complete), 25 min (beginner section)  
**Difficulty:** Beginner to Advanced  

---

## 📚 What You Will Learn

By reading this guide, you'll understand:

1. ✅ **Vertical Scaling** — Making one server bigger
2. ✅ **Horizontal Scaling** — Adding more servers
3. ✅ **Load Balancers** — Distributing traffic across servers
4. ✅ **Auto Scaling Groups** — Automatically add/remove servers
5. ✅ **Scaling Triggers** — When to scale based on metrics
6. ✅ **Scaling Policies** — Target tracking, step scaling, scheduled
7. ✅ **Load Balancer Types** — ALB, NLB, CLB differences
8. ✅ **Health Checks** — Detecting and replacing failed instances
9. ✅ **Zero-Downtime Deployments** — Rolling updates
10. ✅ **Connection Draining** — Graceful shutdown
11. ✅ **Scaling Architecture Patterns** — Production-grade designs
12. ✅ **Troubleshooting Scaling Issues** — Debugging

**After this guide, you will:**
- Scale from 1 to 1000+ instances automatically
- Handle traffic spikes without downtime
- Replace failed instances automatically
- Deploy updates without users noticing
- Design highly available systems
- Implement cost-effective auto-scaling

---

## 👥 Who Should Read This

**Read the whole document if you:**
- Building systems beyond single server
- Need high availability
- Expecting growth or traffic spikes
- Managing multiple instances
- Deploying applications at scale
- Designing infrastructure for teams

**Read just "Beginner Summary" if you:**
- Recently deployed first instance
- Want to understand scaling basics
- Will refer back for specific patterns

---

## 🎯 Beginner Summary: Scaling Explained Simply

```
SCALING = Making your system handle more users

TWO APPROACHES:

VERTICAL SCALING (Make server BIGGER):
├─ Current: t3.medium (2 vCPU, 4 GB RAM)
├─ Problem: Can't handle 1000 users
├─ Solution: Upgrade to m6i.2xlarge (8 vCPU, 32 GB RAM)
├─ Result: Now handles 1000 users
├─ Downtime: 2-5 minutes (stop, upgrade, start)
├─ Limits: Can only go so big (24 vCPU max)
└─ Use case: When you know max size

HORIZONTAL SCALING (Add more servers):
├─ Current: 1 × t3.medium handling 100 users
├─ Problem: Can't handle 1000 users
├─ Solution: Add 9 more servers = 10 × t3.medium
├─ Result: Now handles 1000 users (100 each)
├─ Downtime: ZERO (add servers while running)
├─ Limits: Can add 1000+ servers
└─ Use case: When demand is unpredictable

WHICH IS BETTER?
├─ Vertical: Simple, but has limits
├─ Horizontal: Complex, but unlimited growth
├─ Best practice: Combine both!
│  ├─ Vertical: Start with right size (m6i.large)
│  ├─ Horizontal: Add servers as traffic grows
│  └─ Result: Optimal performance + cost
```

### The Three Scenarios

```
SCENARIO 1: Static Known Load
├─ You know: Exactly 5,000 users
├─ Pattern: Same users, always online
├─ Solution: Pick right size (vertical scaling)
│  ├─ Don't buy too big (waste money)
│  ├─ Don't buy too small (users unhappy)
│  └─ Buy just right (optimal)
├─ Example: Internal tool with 50 employees
└─ Scaling: Rarely needed

SCENARIO 2: Predictable Growth
├─ You know: Start with 100 users, grow 20% per month
├─ Pattern: Steadily increasing
├─ Solution: Plan ahead (mix vertical + horizontal)
│  ├─ Month 1: 1 × m6i.large
│  ├─ Month 3: 2 × m6i.large
│  ├─ Month 6: 3 × m6i.xlarge
│  └─ Month 12: 5 × m6i.2xlarge
├─ Example: Startup with growing user base
└─ Scaling: Planned (scale every month)

SCENARIO 3: Unpredictable Traffic (MOST COMMON)
├─ You know: Peak is 10x baseline
├─ Pattern: Spiky (baseline, then huge spikes)
├─ Examples:
│  ├─ E-commerce: Baseline 100 users, Black Friday 10,000
│  ├─ News: Baseline 1,000, big story → 100,000
│  ├─ API: Consistent 5,000, daily 10x spike at 9am
│  └─ Live event: Baseline 0, event day 1,000,000
├─ Solution: Auto Scaling (automatic add/remove)
│  ├─ Baseline: 2 servers (always running)
│  ├─ Spike: Auto-add servers instantly
│  ├─ After spike: Auto-remove to save cost
│  └─ Users don't notice (seamless)
└─ Scaling: Automatic (done by system)

BEST PRACTICE:
├─ Start with scenario 1 (pick right size)
├─ Move to scenario 2 (plan growth)
├─ Implement scenario 3 (handle spikes)
└─ Result: Optimal performance + cost
```

---

## 🏗️ Architecture Overview

A fully scaled production system:

```
┌─────────────────────────────────────────────────────────┐
│                      INTERNET USERS                     │
│                   (Millions accessing                   │
│                    example.com)                         │
└────────────────────┬────────────────────────────────────┘
                     │ HTTPS Traffic (port 443)
                     ▼
        ┌──────────────────────────────┐
        │      Route53 (DNS)           │
        │  example.com → ALB IP        │
        │  Maps domain to load balancer│
        └────────────┬─────────────────┘
                     │
                     ▼
    ┌────────────────────────────────────────┐
    │   Application Load Balancer (ALB)      │
    │   ├─ Public IP: 54.123.45.67          │
    │   ├─ Listens on port 80/443            │
    │   ├─ Health checks instances           │
    │   ├─ Routes to healthy instances       │
    │   └─ Terminates SSL/TLS                │
    └────────────┬──────────────────────────┘
                 │
         ┌───────┴───────┬───────────┬────────┐
         │               │           │        │
         ▼               ▼           ▼        ▼
    ┌────────┐      ┌────────┐  ┌────────┐ ┌────────┐
    │ EC2-1  │      │ EC2-2  │  │ EC2-3  │ │ EC2-4  │
    │(10.0.1)│      │(10.0.2)│  │(10.0.3)│ │(10.0.4)│
    │:3000   │      │:3000   │  │:3000   │ │:3000   │
    └────────┘      └────────┘  └────────┘ └────────┘
         ▲               ▲           ▲        ▲
         │               │           │        │
         └───────────────┼───────────┼────────┘
                         │           │
            ┌────────────┴───────┬───┴──────────┐
            │                    │              │
            ▼                    ▼              ▼
       ┌─────────────────────────────────────────────┐
       │  Auto Scaling Group (ASG)                   │
       │  ├─ Min instances: 2                        │
       │  ├─ Max instances: 20                       │
       │  ├─ Desired: 4 (currently)                  │
       │  ├─ Spans AZ-a and AZ-b                     │
       │  └─ Automatically add/remove instances      │
       └─────────────────────────────────────────────┘
            │
    ┌───────┴──────────────────────────────────┐
    │  Scaling Policies                        │
    │  ├─ If CPU > 70% for 5 min: +1 instance │
    │  ├─ If CPU < 30% for 10 min: -1 instance│
    │  └─ Max one change per 5 minutes         │
    └───────┬──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │     RDS Database (separate)      │
    │     ├─ Multi-AZ (automatic failover)
    │     ├─ All instances share same DB
    │     └─ Read replicas for scaling │
    └──────────────────────────────────┘
```

---

## 📋 Part 1: Vertical Scaling (Beginner Level)

### 1.1 When to Use Vertical Scaling

```
GOOD FOR:
├─ Predictable growth (know max size)
├─ Single instance (not multi-instance)
├─ Acceptable downtime (2-5 minutes ok)
├─ Budget flexible (bigger = more expensive)
└─ Example: Internal tool → more users → upgrade server

NOT GOOD FOR:
├─ Unpredictable traffic (spikes)
├─ Zero-downtime requirement
├─ Cost-conscious (waste on unused capacity)
├─ Already at max instance size (24 vCPU limit)
└─ Example: E-commerce Black Friday (need horizontal)
```

### 1.2 Vertical Scaling Steps

```bash
CURRENT SETUP:
├─ Instance: i-0abc1234567890def
├─ Type: t3.medium (2 vCPU, 4 GB RAM)
├─ Running: Your Node.js app
├─ Public IP: 54.123.45.67

UPGRADE TO: m6i.large (2 vCPU, 8 GB RAM)

STEP 1: Stop instance (DOWNTIME STARTS)
aws ec2 stop-instances --instance-ids i-0abc1234567890def

Wait for state = stopped
aws ec2 wait instance-stopped --instance-ids i-0abc1234567890def

STEP 2: Change instance type
aws ec2 modify-instance-attribute \
  --instance-id i-0abc1234567890def \
  --instance-type "{\"Value\": \"m6i.large\"}"

STEP 3: Start instance (DOWNTIME ENDS)
aws ec2 start-instances --instance-ids i-0abc1234567890def

Wait for state = running
aws ec2 wait instance-running --instance-ids i-0abc1234567890def

STEP 4: Verify
ssh -i prod-key.pem ec2-user@54.123.45.67

Inside instance:
free -h
# Should show: 8 GB RAM (was 4 GB)

nproc
# Should show: 2 cores (same)

STEP 5: Restart application
pm2 restart myapp

TOTAL DOWNTIME: 2-5 minutes
COST INCREASE: t3.medium $30/month → m6i.large $70/month
```

### 1.3 Vertical Scaling Limitations

```
PROBLEM 1: Can only go so big
├─ Largest instance: 24 vCPU, 768 GB RAM (extremely expensive!)
├─ Cost: ~$30+/hour (thousands per month)
├─ If still not enough: Must split into multiple instances

PROBLEM 2: Downtime required
├─ Stop → Change type → Start = 2-5 minutes down
├─ Users can't access during this time
├─ Not acceptable for critical systems

PROBLEM 3: Single point of failure
├─ If instance fails: 100% downtime
├─ Even with bigger server (won't help if hardware dies)
├─ Need multiple servers for redundancy

SOLUTION: Use Horizontal Scaling instead
├─ More cost-effective
├─ No downtime needed
├─ Infinite scalability
└─ Industry standard for production
```

---

## 🔀 Part 2: Horizontal Scaling (Intermediate Level)

### 2.1 Horizontal Scaling Fundamentals

```
CONCEPT: Add more servers instead of making one bigger

BEFORE:
├─ 1 × t3.medium
├─ Handles: ~100 users
├─ Bottleneck: Server maxed out at 100
└─ Problem: Can't serve 1000 users

AFTER:
├─ 10 × t3.medium (in Auto Scaling Group)
├─ Handles: ~1000 users (100 each)
├─ Bottleneck: Now database becomes bottleneck
└─ Solution: Then add database replicas or upgrade

BENEFITS:
├─ No single point of failure (if 1 fails, 9 others continue)
├─ No downtime (add servers while running)
├─ Cost efficient (pay for what you use)
├─ Infinite scalability (add 100 more servers!)

REQUIREMENTS:
├─ Application must be stateless
│  ├─ Each request independent
│  ├─ No data stored in local memory (use database)
│  ├─ No server-to-server coordination
│  └─ Any instance can handle any request
│
├─ Load balancer (distributes traffic)
│  ├─ Receives all traffic
│  ├─ Decides which instance to send to
│  ├─ Monitors instance health
│  └─ Removes failed instances automatically
│
└─ Database (shared between all instances)
   ├─ Single database all instances connect to
   ├─ All data centralized
   └─ Instances are stateless
```

### 2.2 From Single to Horizontal

```
MIGRATION PATH:

PHASE 1: Single Server (MVP)
├─ 1 × t3.medium
├─ App + Database on same server
├─ Good for: 0-10,000 users
├─ Problems: Single point of failure, can't scale

PHASE 2: Separate Database
├─ EC2 instance for app
├─ RDS database separate
├─ Good for: 10,000-50,000 users
├─ Problem: Still single app instance

PHASE 3: Multi-Instance (Manual)
├─ 2 × EC2 instances
├─ 1 × Load Balancer
├─ 1 × RDS database
├─ Manually manage instances
├─ Good for: 50,000-500,000 users
├─ Problem: Manual scaling (do it yourself)

PHASE 4: Auto Scaling (Automated)
├─ 2-10 × EC2 instances (dynamic)
├─ 1 × Auto Scaling Group
├─ 1 × Load Balancer
├─ 1 × RDS database
├─ Automatic add/remove instances
├─ Good for: 500,000+ users, spiky traffic
├─ Best practice for production!

PHASE 5: Microservices (Advanced)
├─ Multiple services (API, worker, admin)
├─ Multiple load balancers
├─ Multiple databases
├─ Each service auto-scales independently
├─ Out of scope for this guide
```

### 2.3 Application Requirements for Horizontal Scaling

```
YOUR APP MUST BE STATELESS:

BAD (Stateful - won't scale):
├─ Session stored in memory
│  ├─ User logs in to Instance-1
│  ├─ Session saved in Instance-1 memory
│  ├─ Next request goes to Instance-2
│  ├─ Instance-2 doesn't have session!
│  └─ User gets "logged out" error
│
├─ Files stored on server
│  ├─ Upload file to Instance-1
│  ├─ File saved to /tmp/uploads on Instance-1
│  ├─ Next request goes to Instance-2
│  ├─ Instance-2 doesn't have file!
│  └─ "File not found" error
│
└─ Server-to-server coordination
   ├─ Instance-1 sends message to Instance-2
   ├─ Instance-2 crashes
   ├─ Message lost
   └─ Data corruption possible

GOOD (Stateless - scales perfectly):
├─ Session in database or cache
│  ├─ User logs in to Instance-1
│  ├─ Session saved in Redis
│  ├─ Next request goes to Instance-2
│  ├─ Instance-2 reads session from Redis
│  └─ User still logged in! ✓
│
├─ Files in S3
│  ├─ Upload file to Instance-1
│  ├─ File saved to S3 (not local)
│  ├─ Next request goes to Instance-2
│  ├─ Instance-2 reads from S3
│  └─ File found! ✓
│
└─ Tasks in message queue
   ├─ Instance-1 puts task in SQS queue
   ├─ Any instance can pick it up
   ├─ If instance crashes, task retried
   └─ No coordination needed! ✓
```

---

## 🔌 Part 3: Load Balancers (Intermediate Level)

### 3.1 What Is a Load Balancer?

A load balancer distributes incoming traffic across multiple servers.

```
ANALOGY: Restaurant with multiple servers

WITHOUT LOAD BALANCER:
├─ All customers go to Server-1
├─ Server-1 gets overloaded (long wait)
├─ Server-2 is idle (no customers)
└─ Inefficient!

WITH LOAD BALANCER:
├─ Customers arrive at host (load balancer)
├─ Host looks at each server's load
├─ "Server-1 has 5 customers, Server-2 has 2"
├─ Sends new customer to Server-2
├─ Load balanced! (both busy equally)
└─ Efficient!

BENEFITS:
├─ No single point of failure
├─ Better resource utilization
├─ Better user experience (not waiting)
├─ Easy to add/remove servers
```

### 3.2 Types of Load Balancers

```
AWS offers three types:

1. CLASSIC LOAD BALANCER (CLB) - OLD
   ├─ Layer 4 (transport) + Layer 7 (application)
   ├─ Legacy (don't use for new projects)
   ├─ Cost: ~$16/month
   └─ Recommendation: Don't use (use ALB instead)

2. APPLICATION LOAD BALANCER (ALB) - POPULAR
   ├─ Layer 7 (application layer)
   ├─ Can route based on:
   │  ├─ Path (/api/* → API servers, /static/* → cache)
   │  ├─ Domain (api.example.com → API, www.example.com → web)
   │  ├─ Host (mobile.example.com → mobile app)
   │  ├─ HTTP header (Accept: application/json → JSON API)
   │  └─ Query parameters (version=v2 → V2 servers)
   ├─ Cost: ~$16/month + $0.006 per LCU
   ├─ Use case: Most web applications
   └─ Recommendation: Use this! ✓

3. NETWORK LOAD BALANCER (NLB) - PERFORMANCE
   ├─ Layer 4 (transport layer)
   ├─ Ultra-high performance
   ├─ Handles millions of requests/second
   ├─ Cost: ~$16/month + $0.006 per LCU
   ├─ Use case: Real-time apps, gaming, IoT, non-HTTP
   └─ Recommendation: Use if you need extreme performance
```

### 3.3 Application Load Balancer (ALB) Setup

```bash
# CREATE ALB

aws elbv2 create-load-balancer \
  --name my-app-alb \
  --subnets subnet-1a subnet-1b \
  --security-groups sg-web \
  --scheme internet-facing

# Returns:
# LoadBalancerArn: arn:aws:elasticloadbalancing:...
# DNSName: my-app-alb-1234567890.ap-south-1.elb.amazonaws.com

# CREATE TARGET GROUP (where ALB sends traffic)

aws elbv2 create-target-group \
  --name my-app-targets \
  --protocol HTTP \
  --port 3000 \
  --vpc-id vpc-12345 \
  --health-check-protocol HTTP \
  --health-check-path / \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Returns:
# TargetGroupArn: arn:aws:elasticloadbalancing:...

# REGISTER TARGETS (add instances to target group)

aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-0abc1234567890def Id=i-1def4567890abcde

# CREATE LISTENER (what ALB listens on)

aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# TEST

curl http://my-app-alb-1234567890.ap-south-1.elb.amazonaws.com

# Should return: HTML from your app (from one of the instances)
```

### 3.4 ALB Health Checks

```
WHAT ARE HEALTH CHECKS?

ALB periodically pings instances:
├─ "Are you still alive?"
├─ If instance responds: "Yes, I'm healthy"
├─ ALB sends traffic to this instance
├─ If instance doesn't respond: "This instance is dead!"
├─ ALB stops sending traffic
├─ Auto Scaling Group replaces dead instance

CONFIGURATION:

Health Check Path: /health
├─ ALB sends: GET /health HTTP/1.1
├─ Instance must respond: 200 OK
├─ If response: 200, ALB marks healthy
├─ If response: 500, ALB marks unhealthy

Interval: 30 seconds
├─ ALB checks every 30 seconds
├─ If 2 consecutive checks fail → unhealthy
├─ If 2 consecutive checks succeed → healthy

Timeout: 5 seconds
├─ ALB waits 5 seconds for response
├─ If no response in 5 seconds → fail

Result:
├─ Instance must respond in < 5 seconds
├─ Instance must respond with 200 OK
├─ Or ALB marks unhealthy

Example: Failed instance recovery
├─ Instance starts having errors
├─ Health check: 500 error
├─ First failure: Still sends traffic
├─ Second failure (30 sec later): Mark unhealthy
├─ Stop sending traffic to this instance
├─ Auto Scaling Group: "Instance unhealthy, replace it"
├─ Launch new instance, old one terminates
├─ Users don't notice (traffic goes to other instances)
```

---

## 🔄 Part 4: Auto Scaling Groups (Intermediate to Advanced)

### 4.1 What Is an Auto Scaling Group?

```
AUTO SCALING GROUP (ASG) = Automated instance management

WHAT IT DOES:
├─ Maintains desired number of instances
│  ├─ You say: "I want 4 instances"
│  ├─ If 2 instances fail: Automatically launches 2 more
│  ├─ If 6 instances running: Automatically terminates 2
│  └─ Always maintains exactly 4!
│
├─ Launches from custom AMI
│  ├─ All instances identical
│  ├─ Launched from your custom AMI
│  └─ Fully configured (Node.js, app, PM2, etc)
│
├─ Can grow/shrink based on demand
│  ├─ Load increases: Add instances
│  ├─ Load decreases: Remove instances
│  ├─ Automatic based on metrics
│  └─ User configures scaling policies
│
└─ Integrated with load balancer
   ├─ Instances automatically registered
   ├─ Health checks determine load balancer routing
   └─ Failed instances replaced automatically
```

### 4.2 ASG Configuration

```bash
# CREATE LAUNCH TEMPLATE (what instances look like)

aws ec2 create-launch-template \
  --launch-template-name my-app-template \
  --version-description "Node.js app v1.0" \
  --launch-template-data '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.medium",
    "KeyName": "prod-key",
    "SecurityGroupIds": ["sg-web"],
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Name", "Value": "app-instance"}]
    }]
  }'

# CREATE AUTO SCALING GROUP

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-app-asg \
  --launch-template LaunchTemplateName=my-app-template,Version="$Latest" \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 4 \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --vpc-zone-identifier "subnet-1a,subnet-1b" \
  --target-group-arns arn:aws:elasticloadbalancing:...

# Meaning:
# ├─ Min: Always at least 2 instances running
# ├─ Max: Never more than 10 instances
# ├─ Desired: Try to keep exactly 4 instances
# ├─ Health: Use load balancer health checks
# ├─ Grace: Wait 300s after launch before health check
# └─ Zones: Span both AZs (high availability)

# VERIFY

aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-app-asg

# Shows:
# ├─ DesiredCapacity: 4
# ├─ MinSize: 2
# ├─ MaxSize: 10
# ├─ Instances: [i-abc, i-def, i-ghi, i-jkl] (4 running)
# └─ Status: OK
```

### 4.3 Scaling Policies

```
SCALING POLICIES = Rules for when to add/remove instances

POLICY 1: TARGET TRACKING (Recommended)
├─ What: "Keep CPU at 70%"
├─ How:
│  ├─ If CPU > 70%: Add instances
│  ├─ If CPU < 70%: Remove instances
│  └─ Automatically calculates how many
├─ Advantage: Simple, handles most cases
├─ Best for: Most applications

POLICY 2: STEP SCALING (Advanced)
├─ What: "If CPU > 80%, add 2 instances. If > 90%, add 5"
├─ How:
│  ├─ High CPU (80-90%): Add 2
│  ├─ Very high CPU (90%+): Add 5
│  └─ Multiple steps
├─ Advantage: Granular control
├─ Best for: Complex scaling

POLICY 3: SCHEDULED SCALING (Time-based)
├─ What: "At 9am, scale to 10 instances. At 6pm, scale to 2"
├─ How:
│  ├─ Known traffic pattern
│  ├─ Automatically scale at specific times
│  └─ No metric-based triggers
├─ Advantage: Precise for predictable patterns
├─ Best for: E-commerce peak hours, business hours
```

#### Creating Target Tracking Policy

```bash
# SCALE BASED ON CPU

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-app-asg \
  --policy-name scale-up-cpu \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 600
  }'

# Meaning:
# ├─ Target: Keep average CPU at 70%
# ├─ If CPU > 70%: Calculate how many instances needed to reach 70%
# ├─ ScaleOut cooldown: Wait 5 min before adding again
# └─ ScaleIn cooldown: Wait 10 min before removing

# SCALE BASED ON REQUEST COUNT

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-app-asg \
  --policy-name scale-by-requests \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 1000.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget"
    }
  }'

# Meaning:
# ├─ Target: 1000 requests per instance
# ├─ If 4 instances, allow 4000 req/s
# ├─ If traffic increases to 6000 req/s
# └─ ASG automatically adds 2 more instances
```

### 4.4 ASG Scaling Example

```
REAL-WORLD SCENARIO: E-commerce Black Friday

Morning (9am):
├─ Traffic: 1000 requests/second
├─ Current instances: 2
├─ CPU: 50% per instance
├─ Policy: "Keep CPU at 70%"
├─ Action: NONE (50% < 70%)
└─ Cost: Low

Afternoon (2pm):
├─ Traffic: 2000 requests/second
├─ Current instances: 2
├─ CPU: 95% per instance
├─ Policy: "Keep CPU at 70%"
├─ Action: Add instances!
│  ├─ Need: ~2000 * (95/70) = 2714 req/sec per instance
│  ├─ With 2 instances: Can handle 5428 req/sec
│  ├─ But 2000 < 5428, so 2 is enough
│  ├─ Wait, let me recalculate: 95% with 2 = not enough
│  ├─ Add 1 instance (now 3 total)
│  ├─ 2000 / 3 = 667 req/sec per instance
│  └─ CPU: ~667 * (95/2000) = ~31% (good!)
└─ Instances: 2 → 3

Evening (5pm - BLACK FRIDAY STARTS!):
├─ Traffic: 10,000 requests/second (spike!)
├─ Current instances: 3
├─ CPU: Would be 100% (maxed out!)
├─ Policy: "Keep CPU at 70%"
├─ Action: Add many instances!
│  ├─ Need: 10,000 req/sec / 1000 req/sec per instance ≈ 10
│  ├─ Launch: 10 - 3 = 7 new instances
│  └─ Cooldown: Wait 5 min
│
├─ Timeline:
│  ├─ 5:00pm: 10,000 req/sec hits, 3 instances (100% CPU!)
│  ├─ 5:00-5:02pm: Launching 7 new instances
│  ├─ 5:02pm: 5 instances online (users experiencing slow)
│  ├─ 5:03pm: 7 instances online (users see improvement)
│  ├─ 5:04pm: 10 instances online (CPU ~70%, fast!)
│  └─ Total downtime: ~4 minutes
│
└─ Instances: 3 → 10, Cost: 3× higher

Late night (11pm - TRAFFIC DROPS):
├─ Traffic: 1000 requests/second (normal)
├─ Current instances: 10
├─ CPU: 10% per instance (wasteful!)
├─ Policy: "Keep CPU at 70%"
├─ Action: Remove instances (scale down)
│  ├─ Need: 1000 / 1000 = 1 instance
│  ├─ But minimum is 2, so keep 2
│  ├─ Cooldown: Wait 10 min before removing again
│  └─ If still low, remove more
│
├─ Timeline:
│  ├─ 11:00pm: 10 instances, but only need 1-2
│  ├─ 11:10pm: Remove 8 instances (now 2)
│  ├─ 11:20pm: Could remove more, but at minimum (2)
│  └─ Instances stay at 2 (cost savings!)
│
└─ Instances: 10 → 2, Cost: Back to low

SUMMARY:
├─ Morning: 2 instances, $70/month
├─ Peak: 10 instances, $350/month (5x cost)
├─ Evening: 2 instances, $70/month
├─ Average: ~3 instances, ~$100/month
├─ Without auto-scaling: Need 10 all the time = $350/month
└─ Savings: $250/month (71% cheaper!)
```

---

## 🚀 Part 5: Zero-Downtime Deployments (Advanced)

### 5.1 Rolling Updates

```
PROBLEM: Deploying new app version causes downtime

OLD WAY (Downtime):
├─ Stop all instances (DOWNTIME!)
├─ Deploy new code
├─ Start all instances
├─ Result: 5-10 minutes downtime

NEW WAY (Zero downtime):
├─ Keep serving users
├─ Update instances one by one
├─ Users don't notice anything
```

### 5.2 Rolling Update Process

```
SETUP:
├─ ASG with 4 instances
├─ Load balancer routing traffic
├─ Current version: v1.0

DEPLOYMENT PROCESS:

Step 1: Create new AMI with v2.0
├─ Launch instance from v1.0 AMI
├─ Deploy v2.0 code
├─ Test it's working
├─ Create new AMI from this instance
└─ New AMI has v2.0

Step 2: Update launch template
├─ Update to point to v2.0 AMI
├─ But don't change running instances yet

Step 3: Start rolling update
├─ Tell ASG: "Update to new launch template"
├─ ASG gradually replaces instances

Timeline:
├─ T=0: 4 instances running v1.0
│       Load Balancer distributing traffic
│
├─ T=1min: Terminate Instance-1 (v1.0)
│          3 instances now handling traffic (slight increase)
│          Launch Instance-5 (v2.0)
│
├─ T=3min: Instance-5 booted, running v2.0
│          Health check passes
│          Load balancer routes new traffic to Instance-5
│          Now: 3 v1.0 + 1 v2.0
│
├─ T=4min: Terminate Instance-2 (v1.0)
│          3 running: 2 v1.0 + 1 v2.0
│          Launch Instance-6 (v2.0)
│
├─ T=8min: Instance-6 running
│          2 v1.0 + 2 v2.0
│
├─ T=12min: Terminate Instance-3 (v1.0)
│           Launch Instance-7 (v2.0)
│
├─ T=16min: Instance-7 running
│           1 v1.0 + 3 v2.0
│
├─ T=20min: Terminate Instance-4 (v1.0)
│           Launch Instance-8 (v2.0)
│
└─ T=24min: All 4 instances running v2.0
            Deployment complete!
            ZERO downtime! ✓

USER EXPERIENCE:
├─ No notice of deployment
├─ No connection drops
├─ No errors
├─ Seamless update
```

### 5.3 Connection Draining

```
CONNECTION DRAINING = Graceful shutdown

PROBLEM: Instance being replaced
├─ Instance running v1.0
├─ User connected, in middle of request
├─ ASG terminates instance abruptly
├─ User's request interrupted!
└─ Result: Bad experience

SOLUTION: Connection Draining
├─ ASG marks instance for termination
├─ Load balancer stops sending NEW requests
├─ Let existing connections finish
├─ Wait (default: 300 seconds)
├─ Then terminate
└─ Result: Graceful shutdown

Configuration:

aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=deregistration_delay.timeout_seconds,Value=300

# Meaning:
# ├─ When instance marked for termination
# ├─ Wait 300 seconds (5 minutes) max
# ├─ During this time:
# │  ├─ No NEW requests sent to instance
# │  ├─ Existing connections finish
# │  └─ Application does cleanup
# ├─ After 300s: Force terminate
# └─ Result: No abrupt disconnects

Best practice:
├─ Set to 60-300 seconds
├─ Long enough for requests to finish
├─ Not too long (deployment doesn't drag out)
```

---

## 🎯 Part 6: Production Architecture Patterns (Advanced)

### 6.1 Multi-AZ with Auto Scaling

```
ARCHITECTURE FOR HIGH AVAILABILITY:

┌────────────────────────────────────────┐
│ Application Load Balancer (Multi-AZ)   │
│ ├─ Public IP: 54.123.45.67            │
│ └─ Health checks: Every 30 seconds     │
└─┬───────────────────────────────┬──────┘
  │                               │
  │ AZ-a                          │ AZ-b
  │                               │
  ├─ Instance-1 (EC2)            ├─ Instance-3 (EC2)
  ├─ Instance-2 (EC2)            ├─ Instance-4 (EC2)
  └─ In ASG                       └─ In ASG
     │                               │
     ├─ Min: 2                       ├─ Min: 2
     ├─ Max: 20                      ├─ Max: 20
     └─ Desired: 4                   └─ Desired: 4

Auto Scaling Group spans both AZs!

FAILURE SCENARIOS:

Scenario 1: Instance fails in AZ-a
├─ Instance-1 becomes unhealthy
├─ Health check fails
├─ Load balancer removes Instance-1
├─ ASG notices: Only 3 instances
├─ ASG launches new Instance-5 (in AZ-a or AZ-b)
├─ Traffic continues uninterrupted
└─ Users don't notice!

Scenario 2: Entire AZ-a fails
├─ ALL instances in AZ-a fail
├─ Load balancer removes them
├─ Instance-3 and Instance-4 still working (AZ-b)
├─ ASG notices: Only 2 instances
├─ ASG launches new instances (preferentially in AZ-a)
├─ Traffic continues (slower, 2 instead of 4)
└─ Graceful degradation!

Scenario 3: Rolling deployment
├─ Terminating instances one by one
├─ Replacement instances launching
├─ At any point: 3-5 instances running
├─ Load balanced across AZs
└─ Zero downtime!
```

### 6.2 Multi-Tier Architecture with Scaling

```
COMPLETE PRODUCTION SETUP:

┌─────────────────────────────────────────┐
│ Route53 (DNS)                           │
│ example.com → ALB Public IP             │
└────────┬────────────────────────────────┘
         │
         ▼
    ┌─────────────┐
    │ ALB (Public)│
    │ Port 80/443 │
    └─────┬───────┘
          │
    ┌─────┴─────────────────┐
    │                       │
    ▼                       ▼
  AZ-a                    AZ-b
  ┌─────────┐             ┌─────────┐
  │ App ASG │             │ App ASG │
  │ 2-10    │             │ 2-10    │
  │ servers │             │ servers │
  └────┬────┘             └────┬────┘
       │                       │
    ┌──┴───────────────────────┴───┐
    │ (All connected to same DB)    │
    ▼                               ▼
 RDS Primary (AZ-a)         RDS Replica (AZ-b)
 ├─ Write queries           ├─ Read queries
 ├─ Master DB               ├─ Auto-replica
 └─ Multi-AZ replication    └─ Failover ready

Scaling Policies:
├─ App ASG: Scale on CPU > 70%
├─ RDS: Read-only replicas for read scaling
├─ Typical capacity: 4 app instances
├─ Peak capacity: 10-20 app instances
└─ Database: Doesn't scale, upgraded vertically

Cost:
├─ 4 × t3.medium EC2: $120/month
├─ ALB: $16/month
├─ RDS Multi-AZ: $100-200/month (depends on size)
├─ Total baseline: ~$250/month
├─ Peak (20 servers): ~$500/month
└─ Average: ~$300/month
```

---

## 🐛 Part 7: Troubleshooting Scaling Issues (Advanced)

### 7.1 Instances Not Launching

```
SYMPTOM: ASG desired=4 but only 2 instances running

Diagnosis:

1. Check ASG status:
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-app-asg

Look for:
├─ DesiredCapacity: 4
├─ Instances: [i-1, i-2] (only 2!)
└─ Suspended Processes: Any paused?

2. Check capacity available:
├─ Do you have EC2 capacity in your region?
├─ Are you hitting instance limits?
├─ Check: EC2 → Limits → (on-demand instances)

3. Check AMI:
aws ec2 describe-images --image-ids ami-xyz

├─ Does AMI exist?
├─ Is it in correct region?
├─ Is it available (not private/shared)?

4. Check launch template:
aws ec2 describe-launch-templates

├─ Does template exist?
├─ Is it pointing to correct AMI?
├─ Are security groups valid?

5. Check subnet capacity:
├─ Subnets full? (each subnet has limited IPs)
├─ Try launching in different subnets

SOLUTION:
├─ If capacity issue: Request limit increase
├─ If AMI problem: Fix/verify AMI
├─ If subnet issue: Use larger subnets
├─ If template issue: Fix launch template
```

### 7.2 Scaling Not Happening

```
SYMPTOM: CPU at 80% but instances not being added

Diagnosis:

1. Check scaling policy:
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-app-asg

Look for:
├─ Recent activities?
├─ Any errors?
├─ When was last scaling action?

2. Check cooldown:
├─ Scaling policy has cooldown
├─ Example: 5 min before scaling again
├─ If scaled 2 min ago: Wait 3 more min

3. Check if at max capacity:
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-app-asg

├─ DesiredCapacity: 10
├─ MaxSize: 10 (AT MAX!)
└─ Can't add more!

4. Check CPU metric:
aws cloudwatch get-metric-statistics \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Average

├─ Is CPU actually > 70%?
├─ Or is it just spiky?
├─ Scaling needs sustained high CPU

SOLUTION:
├─ If at max: Increase MaxSize
├─ If cooldown: Wait or lower cooldown
├─ If CPU low: Scaling working correctly!
├─ If CPU metric wrong: Check CloudWatch agent
```

---

## ✅ Best Practices Summary

### Scaling Design Principles

```
✓ PLANNING:
├─ Start small (1-2 instances)
├─ Monitor for 1-2 weeks
├─ See traffic pattern
├─ Then design scaling

✓ IMPLEMENTATION:
├─ Use custom AMI (fast launches)
├─ Use target tracking (simple)
├─ Span multiple AZs (high availability)
├─ Monitor health checks

✓ COST:
├─ Set reasonable min/max
├─ Min: At least 2 (for HA)
├─ Max: Don't set too high (budget)
├─ Monitor actual vs desired

✓ OPERATIONS:
├─ Test scaling manually first
├─ Verify health checks working
├─ Have rollback plan
├─ Document scaling policies
```

---

## 🎓 Conclusion

You now understand AWS scaling at a production level.

### Key Takeaways

1. **Vertical Scaling** — Bigger servers (has limits, downtime required)
2. **Horizontal Scaling** — More servers (unlimited, no downtime)
3. **Load Balancers** — Distribute traffic (ALB for most apps)
4. **Auto Scaling Groups** — Automatic add/remove (backbone of scaling)
5. **Scaling Policies** — Rules for when to add/remove
6. **Health Checks** — Detect and replace failed instances
7. **Zero-Downtime Deployments** — Rolling updates
8. **Multi-AZ Resilience** — Survive AZ failures
9. **Cost Optimization** — Pay for capacity used
10. **Production Patterns** — Multi-tier architecture

### Next Steps

1. **Launch ASG** — Create with 2-4 instances
2. **Monitor** — Track CPU and scaling actions for 1 week
3. **Tune policies** — Adjust thresholds based on real data
4. **Test failover** — Manually terminate instances, verify replacement
5. **Test deployment** — Practice zero-downtime update

---

**End of File**

You're ready to deploy and scale applications at production grade.

