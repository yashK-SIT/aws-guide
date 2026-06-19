# AWS EC2 Instances: Complete Deep Dive

**File Version:** 1.0  
**Last Updated:** May 2026  
**Audience:** Everyone — Beginners through Senior Architects  
**Prerequisites:** Read `02-networking/vpc-complete.md` (recommended)  
**Time to Read:** 50-70 minutes (complete), 25 min (beginner section)  
**Difficulty:** Beginner to Advanced  

---

## 📚 What You Will Learn

By reading this guide, you'll understand:

1. ✅ **EC2 Instance Lifecycle** — Launch, run, stop, terminate, terminate
2. ✅ **Instance Types Deep Dive** — Families, generations, optimization
3. ✅ **AMI (Amazon Machine Images)** — Pre-configured images, creation, sharing
4. ✅ **Instance Metadata** — How instances learn about themselves
5. ✅ **User Data** — Automation at launch time
6. ✅ **Elastic IPs** — Static public IPs for production
7. ✅ **Elastic Network Interfaces (ENI)** — Advanced networking
8. ✅ **Instance Storage Options** — EBS, Instance Store, EFS tradeoffs
9. ✅ **Performance Optimization** — CPU, memory, network tuning
10. ✅ **Cost Optimization** — Sizing, reserved instances, spot pricing
11. ✅ **Monitoring & Health** — EC2 status checks, CloudWatch
12. ✅ **Troubleshooting** — Common issues and solutions

**After this guide, you will:**
- Choose the perfect instance type for any workload
- Create and customize AMIs
- Understand instance internals
- Optimize for performance and cost
- Debug instance issues independently
- Design production-grade deployments

---

## 👥 Who Should Read This

**Read the whole document if you:**
- Deploying production workloads on EC2
- Optimizing instance costs
- Managing multiple instance types
- Creating custom AMIs
- Building infrastructure for teams
- Troubleshooting performance issues

**Read just "Beginner Summary" if you:**
- Recently deployed first instance
- Want to understand sizing better
- Will refer back for specific topics

---

## 🎯 Beginner Summary: EC2 Fundamentals

EC2 = Elastic Compute Cloud = Renting a computer in AWS.

```
ANALOGY: Renting a laptop

BUYING YOUR OWN LAPTOP:
├─ High upfront cost ($1000+)
├─ Stuck with that hardware
├─ Can't upgrade later without buying new
├─ Unused capacity is wasted money
└─ What if you need it for 1 month only?

RENTING FROM EC2:
├─ No upfront cost
├─ Pay by the hour (t3.medium = $0.04/hour)
├─ Can upgrade anytime (stop → change type → start)
├─ Can add more computers instantly
├─ Can delete when no longer needed
├─ Perfect for variable workloads
└─ What if you need it for exactly 1 month?

EC2 INSTANCE = Your rented computer
├─ Operating system: Ubuntu, Amazon Linux, Windows, etc.
├─ CPU: 1-96 vCPUs
├─ RAM: 0.5 GB - 768 GB
├─ Storage: 8 GB - 30+ TB
├─ Network: Up to 100 Gbps
└─ Price: $0.01 - $10+/hour

INSTANCE STATE:
├─ Launched: Getting ready
├─ Running: Active and working
├─ Stopped: Hibernated (cheaper, keep data)
├─ Terminated: Deleted forever
└─ Rebooting: Restarting (all data persists)
```

---

## 🏗️ Architecture Overview

A typical EC2 deployment:

```
┌─────────────────────────────────────────────────────────┐
│ AWS ACCOUNT                                             │
│  ┌───────────────────────────────────────────────────┐ │
│  │ VPC: 10.0.0.0/16                                 │ │
│  │ ┌─────────────────────────────────────────────┐  │ │
│  │ │ Public Subnet: 10.0.1.0/24                  │  │ │
│  │ │                                              │  │ │
│  │ │ ┌──────────────────────────────────────┐   │  │ │
│  │ │ │ EC2 Instance (t3.medium)             │   │  │ │
│  │ │ │ ├─ AMI: Ubuntu 24.04                 │   │  │ │
│  │ │ │ ├─ vCPU: 2                           │   │  │ │
│  │ │ │ ├─ RAM: 4 GB                         │   │  │ │
│  │ │ │ ├─ Storage: 30 GB (EBS gp3)          │   │  │ │
│  │ │ │ ├─ Public IP: 54.123.45.67           │   │  │ │
│  │ │ │ ├─ Private IP: 10.0.1.100            │   │  │ │
│  │ │ │ ├─ Security Group: web-sg            │   │  │ │
│  │ │ │ ├─ Key Pair: prod-key.pem            │   │  │ │
│  │ │ │ └─ Status: running                   │   │  │ │
│  │ │ │                                       │   │  │ │
│  │ │ │ PROCESSES RUNNING:                   │   │  │ │
│  │ │ │ ├─ Node.js app (port 3000)           │   │  │ │
│  │ │ │ ├─ Nginx (port 80/443)               │   │  │ │
│  │ │ │ ├─ PM2 (process manager)             │   │  │ │
│  │ │ │ └─ SSH server (port 22)              │   │  │ │
│  │ │ └──────────────────────────────────────┘   │  │ │
│  │ └─────────────────────────────────────────────┘  │ │
│  │ ┌─────────────────────────────────────────────┐  │ │
│  │ │ Private Subnet: 10.0.10.0/24               │  │ │
│  │ │ ┌──────────────────────────────────────┐   │  │ │
│  │ │ │ RDS Database (PostgreSQL)            │   │  │ │
│  │ │ │ ├─ Endpoint: prod-db.rds.amazonaws  │   │  │ │
│  │ │ │ ├─ Private IP: 10.0.10.50           │   │  │ │
│  │ │ │ └─ Only accessible from app tier    │   │  │ │
│  │ │ └──────────────────────────────────────┘   │  │ │
│  │ └─────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 📋 Part 1: EC2 Instance Lifecycle (Beginner Level)

### 1.1 Instance States

An EC2 instance goes through several states in its lifetime.

```
INSTANCE LIFECYCLE:

┌──────────────────────────────────────────────────┐
│                                                  │
│  [pending] → [running] → [stopped] → [terminated]
│                │            │
│                └─ [stopping]─┘
│                │            │
│                └─ [rebooting]
│
```

#### State 1: Pending

```
What's happening:
├─ You clicked "Launch Instance"
├─ AWS is allocating resources
├─ OS is loading
├─ Network is being configured
├─ Duration: 30-60 seconds

During pending:
├─ You CAN'T connect yet
├─ You CAN'T send traffic yet
├─ Instance not ready for use

What you see:
├─ AWS Console: Status = "pending" (yellow)
├─ Instance ID: Assigned (i-0abc1234567890def)
├─ Public IP: Not yet assigned
└─ State transition: pending → running
```

#### State 2: Running

```
What's happening:
├─ OS fully loaded
├─ Network ready
├─ Instance accepting traffic
├─ Billing ACTIVE (you're being charged)

During running:
├─ You CAN connect via SSH
├─ You CAN send traffic
├─ You CAN install software
├─ Instance fully operational

Billing:
├─ Charged per hour (even if idle)
├─ Even if your app uses 0% CPU
├─ Stopped instances are cheaper!
└─ Always stop if not in use
```

#### State 3: Stopped

```
What's happening:
├─ Instance powered down (hibernated)
├─ All data preserved (EBS volume remains)
├─ Public IP released
├─ Instance NOT accepting traffic

Benefits of stopping:
├─ Much cheaper (~10% of running cost)
├─ Keep your data
├─ Can restart quickly (1 minute)
├─ Perfect for dev/test instances

What remains:
├─ EBS volume with all your data
├─ Security groups, network config
├─ SSH key pair (still works)
└─ Private IP (might change on restart)

What's lost:
├─ Public IP (will be different when restarted)
├─ Connection to application
├─ In-memory data (RAM is cleared)
└─ Instance Store data (if applicable)
```

#### State 4: Terminated

```
What's happening:
├─ Instance deleted forever
├─ EBS volume deleted (if configured)
├─ Public IP released
├─ All data gone

After termination:
├─ CANNOT be recovered
├─ CANNOT be restarted
├─ Instance no longer appears in console
├─ Billing STOPS

Why terminate:
├─ No longer need the instance
├─ Cost optimization (not just stopping)
├─ Cleanup after project
├─ Free up resources

Warning:
├─ Termination is irreversible!
├─ Make sure you have backups
├─ Use termination protection (if critical)
└─ Better to stop than terminate
```

#### State 5: Rebooting

```
What's happening:
├─ Instance restarting (like Ctrl+Alt+Delete)
├─ OS shutdown + restart
├─ Network remains connected
├─ Takes 1-3 minutes

During reboot:
├─ Instance briefly unavailable
├─ Billing continues (instance still running!)
├─ SSH connections broken temporarily
├─ Applications must be restarted

What persists through reboot:
├─ All data (EBS volume)
├─ Private IP address
├─ Public IP address
├─ Security groups
└─ Network configuration

When to reboot:
├─ Kernel update (requires reboot)
├─ Memory leak (restart application)
├─ OS hanging or slow
├─ After SSH key update sometimes
```

### 1.2 Instance Lifecycle Commands

```bash
# Launch Instance
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name prod-key \
  --security-group-ids sg-12345 \
  --subnet-id subnet-abc123 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-server}]'

# Result:
# Instance ID: i-0abc1234567890def
# State: pending
# Public IP: Assigning...

# List Instances
aws ec2 describe-instances --instance-ids i-0abc1234567890def

# Result shows:
# State: running
# Public IP: 54.123.45.67
# Private IP: 10.0.1.100

# Stop Instance (keeps data)
aws ec2 stop-instances --instance-ids i-0abc1234567890def

# Start Instance (restart from stopped)
aws ec2 start-instances --instance-ids i-0abc1234567890def

# Reboot Instance (like Ctrl+Alt+Delete)
aws ec2 reboot-instances --instance-ids i-0abc1234567890def

# Terminate Instance (DELETE forever!)
aws ec2 terminate-instances --instance-ids i-0abc1234567890def

# In AWS Console:
# EC2 Dashboard → Instances → Right-click instance → Instance State
# Options: Start, Stop, Reboot, Terminate, Terminate and Delete Volume
```

---

## 🖥️ Part 2: Instance Types Deep Dive (Intermediate Level)

### 2.1 Instance Family Overview Revisited

In the fundamentals guide, we covered basic types. Now let's go deeper.

```
Instance Generation:
├─ Older: t2 (2013) - still available but outdated
├─ Current: t3, m5, c5, r6i (2019-2023)
├─ Latest: t4g, m7g, c7g (2023-2024, ARM processors)
└─ Within family: t3.micro, t3.small, t3.medium, t3.large, etc.

Each generation improves:
├─ Performance per dollar
├─ Energy efficiency
├─ Network throughput
├─ Storage speed (NVMe SSD)
└─ ARM support (newer generations)

Should you use older generations?
├─ t2 vs t3: Pick t3 (20-30% better performance, similar price)
├─ m5 vs m6i: Pick m6i (30-40% better, same price)
├─ Rule: Always use newest generation available!
```

### 2.2 Instance Type Families in Detail

#### T Family: Burstable (General Purpose - Variable Load)

```
T3 vs T4g:

T3 (Intel processor):
├─ Baseline CPU: 10-30% (burstable)
├─ Burst duration: ~1 hour (then throttles)
├─ Price: Lower
├─ Compatibility: Works everywhere
└─ Best for: Most people starting out

T4g (ARM-based Graviton processor):
├─ Baseline CPU: 10-30% (burstable)
├─ Burst duration: ~1 hour (then throttles)
├─ Price: 15-20% cheaper than T3
├─ Compatibility: Requires ARM-compatible OS/software
└─ Best for: Cost-conscious, modern applications

When to use T family:
├─ Low traffic or variable traffic
├─ Can tolerate occasional throttling
├─ Learning/development
├─ Internal tools
├─ Personal projects

When NOT to use T family:
├─ Consistent high traffic
├─ Can't tolerate throttling
├─ Production with SLA
└─ Need predictable performance

T Family Sizing:

t3.micro:
├─ vCPU: 1 (burstable)
├─ RAM: 1 GB
├─ Network: Up to 5 Gbps
├─ Cost: $0.0116/hour (~$8.47/month)
├─ Use case: Tiny personal projects
└─ Warning: RAM very limited!

t3.small:
├─ vCPU: 2 (burstable)
├─ RAM: 2 GB
├─ Network: Up to 5 Gbps
├─ Cost: $0.024/hour (~$17.52/month)
├─ Use case: Small hobby projects
└─ Still limited for production

t3.medium:
├─ vCPU: 2 (burstable)
├─ RAM: 4 GB
├─ Network: Up to 5 Gbps
├─ Cost: $0.0416/hour (~$30.37/month)
├─ Use case: Popular choice for startups
└─ Good starter instance for learning

t3.large:
├─ vCPU: 2 (burstable)
├─ RAM: 8 GB
├─ Network: Up to 5 Gbps
├─ Cost: $0.083/hour (~$60.59/month)
├─ Use case: Growing traffic, still burstable
└─ Upgrade path from t3.medium

Burst Credits:

How burst works:
├─ You earn credits while running below baseline
├─ Example: t3.medium baseline = 20% CPU
│  ├─ If you use 15% CPU → earning credits
│  ├─ If you use 20% CPU → neither earning nor using
│  └─ If you use 100% CPU → spending credits
│
├─ When you run out of credits → throttled to baseline
├─ One hour at 100% CPU uses ~3 hours of credits
└─ Typical: 3-4 hours of 100% per month before throttle

Burstable CPU Graph:

Time ──────────────────────────────────────────→

CPU %:
100% ┤                     ╱╲      ╭─ Spending credits
 80% ├                    ╱  ╲    ╱  (throttled after exhausted)
 60% ├                   ╱    ╲  ╱
 40% ├  ╭──────────╮    ╱      ╲╱   ╭──────────────
 20% ├──╯  earning ╰───╯ earning  ╰──╯ THROTTLED!
  0% └────────────────────────────────
       (below baseline) (above baseline) (out of credits)

Baseline = 20% for t3.medium
```

#### M Family: General Purpose (Balanced)

```
M5 vs M6i vs M7i:

M5 (2018, older):
├─ Balanced CPU/RAM ratio
├─ Price: High (older)
├─ Use case: Previous generation
└─ Recommendation: Don't use, pick M6i instead

M6i (2021, current):
├─ Balanced CPU/RAM ratio (same as M5)
├─ Price: Lower than M5, same/better performance
├─ Network: Better (up to 20 Gbps vs 10 Gbps)
├─ Storage: NVMe SSD (faster than M5)
└─ Recommendation: Use this for most production

M7i (2023, latest):
├─ Balanced CPU/RAM ratio (same as M5/M6i)
├─ Price: Similar to M6i (AI pricing, not much premium)
├─ Performance: 15-20% better than M6i
├─ Network: 12.5 Gbps
└─ Recommendation: Use if available in your region

When to use M family:
├─ General purpose workloads
├─ Web servers, APIs
├─ Application servers
├─ Enterprise software
├─ You don't know what else to choose → M!

M Family Sizing:

m6i.large:
├─ vCPU: 2
├─ RAM: 8 GB
├─ Cost: $0.096/hour (~$70/month)
├─ Use case: Production web server
└─ Sweet spot for small production

m6i.xlarge:
├─ vCPU: 4
├─ RAM: 16 GB
├─ Cost: $0.192/hour (~$140/month)
├─ Use case: Medium production app
└─ Growing startup workload

m6i.2xlarge:
├─ vCPU: 8
├─ RAM: 32 GB
├─ Cost: $0.384/hour (~$280/month)
├─ Use case: Large application
└─ Serving millions of requests

m6i.4xlarge:
├─ vCPU: 16
├─ RAM: 64 GB
├─ Cost: $0.768/hour (~$560/month)
├─ Use case: Very large application
└─ Before splitting into multiple servers
```

#### C Family: Compute Optimized (High CPU)

```
C5 vs C6i vs C7g:

C5 (2018, older):
├─ High CPU per core
├─ Lower RAM ratio than M
├─ Price: High
└─ Not recommended (older)

C6i (2021, current):
├─ High CPU per core
├─ Better performance than C5
├─ Price: Competitive with M6i
├─ Use when: Compute-heavy work
└─ Example: Video encoding, ML, calculations

C7g (2023, latest ARM):
├─ High CPU per core
├─ 15-20% better than C6i
├─ Price: 10-15% cheaper than C6i (due to ARM)
├─ Use when: Compute-heavy + cost conscious
└─ Requires ARM-compatible software

When to use C family:
├─ CPU-intensive work
├─ Video/image encoding
├─ Machine learning inference
├─ Data processing
├─ Financial calculations
├─ Typically NOT web servers

C Family Sizing:

c6i.large:
├─ vCPU: 2
├─ RAM: 4 GB (less than M6i.large!)
├─ Cost: $0.085/hour (~$62/month)
├─ RAM ratio: 2 GB per vCPU
└─ Use case: CPU-bound tasks

c6i.2xlarge:
├─ vCPU: 8
├─ RAM: 16 GB
├─ Cost: $0.34/hour (~$248/month)
├─ RAM ratio: 2 GB per vCPU (same)
└─ Use case: Parallel CPU-intensive work

Comparison: c6i.large vs m6i.large
├─ CPU: Both have 2 vCPU
├─ RAM: c6i has 4GB, m6i has 8GB
├─ Cost: c6i $62/mo, m6i $70/mo
├─ Use c6i if: CPU bound, don't need RAM
├─ Use m6i if: Balanced or RAM-heavy
```

#### R Family: Memory Optimized (High RAM)

```
R6i vs R7i:

R6i (2021):
├─ High RAM per core
├─ 5.3 GB RAM per vCPU
├─ Price: Premium for memory
└─ Use when: Memory-intensive

R7i (2023):
├─ High RAM per core (same ratio)
├─ Performance: 15% better CPU
├─ Price: Similar to R6i
└─ Use when: Memory-intensive + latest

When to use R family:
├─ In-memory databases (Redis clusters)
├─ Large data caches
├─ Big data processing (Spark)
├─ Real-time analytics
├─ Machine learning with huge models

R Family Sizing:

r6i.large:
├─ vCPU: 2
├─ RAM: 16 GB
├─ Cost: $0.252/hour (~$184/month)
├─ RAM ratio: 8 GB per vCPU
└─ Use case: Large Redis cache

r6i.xlarge:
├─ vCPU: 4
├─ RAM: 32 GB
├─ Cost: $0.504/hour (~$368/month)
├─ RAM ratio: 8 GB per vCPU
└─ Use case: Medium Spark cluster node

r6i.4xlarge:
├─ vCPU: 16
├─ RAM: 128 GB
├─ Cost: $2.016/hour (~$1472/month)
├─ RAM ratio: 8 GB per vCPU
└─ Use case: Large database or cache

Comparison: m6i.xlarge vs r6i.large
├─ Cost: m6i $140/mo vs r6i $184/mo
├─ m6i: 4 vCPU + 16 GB RAM
├─ r6i: 2 vCPU + 16 GB RAM
├─ r6i more expensive but fewer CPUs
├─ Use m6i if: Need CPU + RAM balance
├─ Use r6i if: RAM is the constraint
```

### 2.3 How to Choose Instance Type

```
DECISION TREE:

Q1: What's your budget per month?
├─ < $20: t3.micro or t3.small
├─ $20-50: t3.medium or t3.large
├─ $50-100: t3.large or m6i.large
├─ > $100: m6i.xlarge or larger
└─ Continue

Q2: What's your application type?
├─ Web server / API:
│  ├─ Predict: Balanced (CPU + RAM)
│  └─ Choose: m6i family
│
├─ Database / Cache:
│  ├─ Predict: High RAM
│  └─ Choose: r6i family
│
├─ Video processing / ML:
│  ├─ Predict: High CPU
│  └─ Choose: c6i family
│
├─ Learning / Dev / Test:
│  ├─ Predict: Variable
│  └─ Choose: t3/t4g family
│
└─ Uncertain:
   ├─ Predict: Start with general purpose
   └─ Choose: m6i family

Q3: Is traffic predictable?
├─ Yes (consistent): Use fixed instance (m/c/r family)
│  ├─ No need to pay for burstable guarantee
│  ├─ Better performance
│  └─ No throttling worries
│
└─ No (variable): Use burstable (t family)
   ├─ Pay less
   ├─ Scales to peaks
   └─ OK if occasional throttling

Q4: Is this production?
├─ Yes: Use current generation (m6i, not m5)
├─ No: Can use any generation
└─ Always prefer latest!

Q5: Do you need burstable CPU?
├─ Yes (learning, low traffic): t3 or t4g
├─ No (production, consistent): m/c/r family

QUICK DECISION MATRIX:

┌──────────────────┬─────────────────┐
│ Workload         │ Instance Type   │
├──────────────────┼─────────────────┤
│ Learning         │ t3.micro/small  │
│ Small startup    │ t3.medium       │
│ Web app (prod)   │ m6i.large       │
│ Database         │ r6i.large+      │
│ ML/Encoding      │ c6i.xlarge+     │
│ Scale app        │ m6i.2xlarge+    │
│ Personal project │ t4g.micro/small │
└──────────────────┴─────────────────┘
```

---

## 📦 Part 3: AMI (Amazon Machine Images) (Intermediate Level)

### 3.1 What Is an AMI?

An AMI is a pre-configured image (like a snapshot) of an operating system.

```
ANALOGY: Computer OS Installation

BUYING A LAPTOP:
├─ Computer comes with OS installed
├─ All drivers already loaded
├─ Some software pre-installed
├─ You take it home and customize

LAUNCHING EC2 FROM AMI:
├─ Instance gets OS installed (from AMI)
├─ All drivers already loaded
├─ Some software pre-installed (depends on AMI)
├─ You log in and customize
```

### 3.2 Popular AMIs

```
AWS PROVIDED AMIs:
(Free to use, maintained by AWS)

Amazon Linux 2 (AL2):
├─ AWS's custom OS (based on CentOS)
├─ Very lightweight (small, fast)
├─ Pre-optimized for AWS
├─ Package manager: yum
├─ Installation size: ~500 MB
├─ Free tier: YES ✓
├─ Use case: Default choice for AWS workloads
└─ Recommended: YES (especially if new to Linux)

Ubuntu (Canonical):
├─ Popular Linux distribution
├─ Larger community (StackOverflow answers)
├─ Package manager: apt
├─ Installation size: ~800 MB
├─ Free tier: YES ✓
├─ Use case: Developers familiar with Ubuntu
└─ Note: More third-party tools available

Ubuntu Server 24.04 LTS:
├─ Latest long-term support version
├─ LTS = Support for 5+ years
├─ Recommended for production
├─ Large community, well-documented
└─ Great choice for beginners (familiar interface)

Debian:
├─ Lightweight Linux distribution
├─ Minimal (very small)
├─ Package manager: apt
├─ Free tier: YES ✓
├─ Use case: Developers wanting minimal OS
└─ Note: Fewer pre-installed tools

Red Hat Enterprise Linux (RHEL):
├─ Enterprise Linux (older, stable versions)
├─ Commercial support available
├─ Package manager: yum/dnf
├─ Free tier: NO (costs per hour)
├─ Use case: Enterprise environments
└─ Note: Very expensive

Windows Server:
├─ Windows OS for servers
├─ RDP (Remote Desktop) instead of SSH
├─ Package manager: PowerShell
├─ Free tier: NO (expensive, ~$0.20/hour+)
├─ Use case: .NET applications, legacy Windows apps
└─ Note: Not recommended unless required

QUICK RECOMMENDATION:
├─ Beginner learning Linux: Ubuntu 24.04 LTS
├─ AWS-optimized workload: Amazon Linux 2
├─ Enterprise environment: RHEL or Windows
└─ Cost optimization: Amazon Linux 2 or Debian
```

### 3.3 Finding AMIs

```bash
# Search for Ubuntu AMIs
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal*" \
  --query 'Images[].[ImageId,Name,CreationDate]'

# Returns:
# ami-0c55b159cbfafe1f0 | ubuntu/images/hvm-ssd/ubuntu-focal-24.04-amd64-server-... | 2024-04-01T00:00:00.000Z

# Search for Amazon Linux 2
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'Images[].[ImageId,Name]'

# In AWS Console:
# EC2 Dashboard → Images → AMIs → Search public AMIs
# Filter by: Name (e.g., "ubuntu-focal")
# Sort by: Latest (most recent)
```

### 3.4 Creating Custom AMIs

Why create custom AMI?

```
SCENARIO 1: Manual Installation (Bad)
├─ Launch instance
├─ SSH and install Node.js
├─ Install PM2
├─ Install Nginx
├─ Clone code
├─ Start services
├─ Takes 10 minutes per instance
└─ Error-prone (manual steps)

SCENARIO 2: Custom AMI (Good)
├─ Create instance from base AMI
├─ Do all setup (manually, once)
├─ Create AMI from this instance
├─ Next time:
│  ├─ Launch from custom AMI
│  ├─ Already has Node.js, PM2, Nginx, code
│  └─ Takes 30 seconds (all pre-installed!)
└─ Repeatable and consistent

Benefits:
├─ Speed: Instant deployment
├─ Consistency: Same setup every time
├─ Reliability: Tested once, works always
├─ Scalability: Launch 100 identical instances
└─ Cost: Save time (faster TTM)
```

#### Creating a Custom AMI (Step-by-Step)

```bash
# STEP 1: Launch instance from base AMI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name prod-key

# STEP 2: Wait for instance to run
aws ec2 describe-instances --instance-ids i-0abc123 --query 'Reservations[0].Instances[0].State.Name'
# Wait for: running

# STEP 3: SSH and customize
ssh -i prod-key.pem ec2-user@54.123.45.67

# Inside instance:
sudo yum update -y                    # Update system
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install --lts                     # Install Node.js
npm install -g pm2                    # Install PM2
sudo yum install -y nginx             # Install Nginx
# ... more setup ...

# STEP 4: Stop instance (before creating AMI)
aws ec2 stop-instances --instance-ids i-0abc123

# Wait for stopped state
aws ec2 describe-instances --instance-ids i-0abc123 --query 'Reservations[0].Instances[0].State.Name'
# Wait for: stopped

# STEP 5: Create AMI from instance
aws ec2 create-image \
  --instance-id i-0abc123 \
  --name "my-nodejs-server" \
  --description "Node.js, PM2, Nginx pre-configured"

# Returns:
# ImageId: ami-xxxxxxxx

# STEP 6: Wait for AMI to be available (~5-10 minutes)
aws ec2 describe-images --image-ids ami-xxxxxxxx --query 'Images[0].State'
# Wait for: available

# STEP 7: Launch new instances from custom AMI (instant!)
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \  # Your custom AMI!
  --instance-type t3.medium \
  --key-name prod-key

# That's it! New instance has everything pre-configured!
```

#### AMI Naming Convention (Best Practice)

```
Recommended naming scheme:
my-app-v1.0.0-2024-05-14

Breakdown:
├─ my-app: Application name
├─ v1.0.0: Version number (semantic versioning)
├─ 2024-05-14: Date created
└─ Result: Sortable, descriptive, version traceable

Example:
├─ nodejs-api-v1.0.0-2024-05-14
├─ nodejs-api-v1.1.0-2024-05-21
├─ nodejs-api-v2.0.0-2024-06-01
└─ Easy to see which is newest!

Benefits:
├─ Can keep multiple versions
├─ Easy rollback (launch from old AMI)
├─ Clear what changed
├─ Track deployment history
```

### 3.5 AMI Best Practices

```
DO:
✓ Version your AMIs
✓ Document what's in each AMI
✓ Test AMI before using in production
✓ Keep old AMIs for rollback
✓ Copy AMIs to multiple regions (for DR)
✓ Use infrastructure-as-code (Packer/Terraform)
✓ Tag AMIs for organization
✓ Create read-only snapshots

DON'T:
✗ Put secrets in AMI (use IAM roles instead)
✗ Store passwords in AMI
✗ Keep thousands of unused AMIs
✗ Forget to delete old AMIs (costs money)
✗ Use someone's random AMI without review
✗ Manually customize instances (should be in AMI)
```

---

## 🔗 Part 4: Elastic IPs (Production Static IP) (Intermediate Level)

### 4.1 Why Elastic IPs Matter

```
PROBLEM: Public IP Changes

Scenario 1: Stop/Start Instance
├─ Instance running: Public IP 54.123.45.67
├─ You stop instance (save money)
├─ You restart instance
├─ New Public IP: 54.234.56.78 (DIFFERENT!)
├─ Users can't reach your old IP
├─ DNS takes time to update
└─ Service interrupted!

Scenario 2: Instance Fails and Auto-Scales
├─ Instance dies → Auto Scaling replaces it
├─ New instance gets NEW public IP
├─ DNS is outdated
├─ Traffic goes to old IP (dead instance!)
└─ Service down until DNS updates

SOLUTION: Elastic IP (Static IP)
├─ Remains same even if instance changes
├─ Stop/start instance: IP stays same
├─ Instance dies: Can assign to replacement
├─ Always points to right instance
└─ No DNS update delays
```

### 4.2 Elastic IP vs Public IP

```
COMPARISON:

Public IP:
├─ AWS assigns automatically
├─ Changes when instance stops/starts
├─ Free (included with instance)
├─ Good for: Temporary/dev instances
└─ Bad for: Production (unreliable)

Elastic IP:
├─ You explicitly allocate
├─ Stays same even after stop/start
├─ Costs ~$0.005/hour if NOT attached
├─ Free if attached to running instance
├─ Good for: Production (reliable)
└─ Bad for: Dev (extra cost if not used)

Cost analysis:
├─ Elastic IP: Free if attached
├─ Unused Elastic IP: $0.005/hour = $36/year
├─ Stop instance with public IP: Free
├─ Stop instance with elastic IP: Free
└─ VERDICT: Use Elastic IP for production
```

### 4.3 Using Elastic IPs

```bash
# STEP 1: Allocate Elastic IP
aws ec2 allocate-address

# Returns:
# PublicIp: 54.123.45.67
# AllocationId: eipalloc-12345678

# STEP 2: Associate with instance
aws ec2 associate-address \
  --instance-id i-0abc1234567890def \
  --public-ip 54.123.45.67

# STEP 3: Verify
aws ec2 describe-addresses --public-ips 54.123.45.67

# Returns:
# PublicIp: 54.123.45.67
# InstanceId: i-0abc1234567890def
# AssociationId: eipassoc-12345678
# NetworkInterfaceId: eni-12345678

# STEP 4: Update DNS to point to this IP
# (Do this in your domain registrar or Route53)

# To disassociate (but keep Elastic IP):
aws ec2 disassociate-address --association-id eipassoc-12345678

# To release Elastic IP (free it up):
aws ec2 release-address --public-ip 54.123.45.67

# In AWS Console:
# EC2 Dashboard → Elastic IPs
# Allocate, Associate, Disassociate, Release
```

### 4.4 Production Elastic IP Best Practices

```
MULTI-AZ WITH ELASTIC IPs:

Architecture:
├─ Elastic IP: 54.123.45.67 (points to active instance)
├─ Primary Instance (AZ-a): Has Elastic IP
├─ Secondary Instance (AZ-b): No IP (standby)
│
├─ If Primary fails:
│  ├─ Disassociate Elastic IP from failed instance
│  ├─ Associate Elastic IP with secondary instance
│  ├─ Elastic IP now points to secondary
│  └─ Traffic redirects automatically (< 1 minute)

Better: Use Load Balancer instead
├─ Route 53: Points to ALB
├─ ALB: Routes to healthy instances
├─ Automatic failover (no manual Elastic IP reassignment)
└─ Recommended for production ✓

Cost of Elastic IPs:
├─ Per running instance: Free (included)
├─ Per unattached Elastic IP: $0.005/hour (~$36/year)
├─ Typical production: 1-3 Elastic IPs
└─ Cost: Usually negligible
```

---

## 🔧 Part 5: Instance Metadata & User Data (Advanced)

### 5.1 Instance Metadata

Instance metadata is information about the instance itself.

```
What is metadata?
├─ Instance ID: i-0abc1234567890def
├─ Instance Type: t3.medium
├─ Availability Zone: ap-south-1a
├─ Public IP: 54.123.45.67
├─ Private IP: 10.0.1.100
├─ Security Groups: web-sg, app-sg
├─ IAM Role: MyEC2Role
├─ Tags: Name=prod-server, Env=production
└─ And more...

How to access metadata:
├─ From inside running instance
├─ Via special URL: http://169.254.169.254/latest/meta-data/
└─ Available only from inside instance
```

#### Accessing Metadata from Inside Instance

```bash
# SSH into instance, then:

# Get instance ID
curl http://169.254.169.254/latest/meta-data/instance-id
# Returns: i-0abc1234567890def

# Get instance type
curl http://169.254.169.254/latest/meta-data/instance-type
# Returns: t3.medium

# Get availability zone
curl http://169.254.169.254/latest/meta-data/placement/availability-zone
# Returns: ap-south-1a

# Get public IP
curl http://169.254.169.254/latest/meta-data/public-ipv4
# Returns: 54.123.45.67

# Get private IP
curl http://169.254.169.254/latest/meta-data/local-ipv4
# Returns: 10.0.1.100

# Get IAM role
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Returns: MyEC2Role

# Get IAM credentials (temporary tokens)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyEC2Role
# Returns: AccessKeyId, SecretAccessKey, Token (automatically rotating!)

# Get all metadata
curl http://169.254.169.254/latest/meta-data/
# Lists all available paths

# Get user data (script you provided at launch)
curl http://169.254.169.254/latest/user-data
# Returns: Your script output
```

#### Use Case: Instance Discovering Itself

```javascript
// Node.js example: Get metadata for logging

const http = require('http');

function getMetadata(path) {
  return new Promise((resolve, reject) => {
    http.get(`http://169.254.169.254/latest/meta-data/${path}`, (res) => {
      let data = '';
      res.on('data', (chunk) => { data += chunk; });
      res.on('end', () => { resolve(data); });
    }).on('error', reject);
  });
}

async function logInstanceInfo() {
  const instanceId = await getMetadata('instance-id');
  const az = await getMetadata('placement/availability-zone');
  const instanceType = await getMetadata('instance-type');
  
  console.log(`Instance ID: ${instanceId}`);
  console.log(`Availability Zone: ${az}`);
  console.log(`Instance Type: ${instanceType}`);
}

logInstanceInfo();

// Output:
// Instance ID: i-0abc1234567890def
// Availability Zone: ap-south-1a
// Instance Type: t3.medium
```

### 5.2 User Data (Automation at Launch)

User data is a script that runs when instance launches.

```
SCENARIO 1: Manual Setup (Bad)
├─ Launch instance
├─ SSH in
├─ Install software
├─ Configure services
├─ Start services
├─ Takes 10 minutes
└─ Error-prone

SCENARIO 2: User Data Script (Good)
├─ Create script to do everything
├─ Provide as user-data when launching
├─ Instance launches and runs script
├─ Takes 2 minutes (script runs automatically)
└─ Repeatable and consistent
```

#### User Data Script Example

```bash
#!/bin/bash
# This script runs as root when instance launches

set -e  # Exit if any command fails

# Update system
yum update -y

# Install Node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install --lts

# Install PM2
npm install -g pm2

# Install Nginx
yum install -y nginx

# Clone application code
cd /home/ec2-user
git clone https://github.com/username/myapp.git app
cd app

# Install dependencies
npm install

# Start with PM2
pm2 start app.js --name "myapp"
pm2 startup
pm2 save

# Configure Nginx
cp nginx.conf /etc/nginx/nginx.conf
systemctl start nginx
systemctl enable nginx

# Create startup script (so it runs on reboot)
echo "pm2 restart myapp" | sudo tee /usr/local/bin/start-app.sh
chmod +x /usr/local/bin/start-app.sh

# Log completion
echo "User data script completed at $(date)" > /tmp/user-data-complete.txt
```

#### Launching Instance with User Data

```bash
# Save script to file
cat > user-data.sh << 'EOF'
#!/bin/bash
set -e
yum update -y
yum install -y nodejs npm
npm install -g pm2
echo "Setup complete" > /tmp/setup-complete.txt
EOF

# Launch instance with user-data
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name prod-key \
  --user-data file://user-data.sh

# Or via console:
# EC2 Dashboard → Launch Instance
# Advanced Details → User Data
# Paste your script

# Check if script ran:
ssh -i prod-key.pem ec2-user@54.123.45.67
cat /tmp/setup-complete.txt
# Should show: Setup complete
```

#### User Data Best Practices

```
DO:
✓ Use #!/bin/bash at top
✓ Use set -e (exit on error)
✓ Log output: >> /tmp/user-data.log
✓ Keep scripts short (complex setup → use custom AMI)
✓ Test scripts locally first
✓ Use environment variables for config
✓ Handle idempotency (can run multiple times)

DON'T:
✗ Put secrets in user-data (visible in console!)
✗ Run long compilation (timeout risk)
✗ Assume specific working directory
✗ Use user-data for frequent updates
✗ Forget chmod +x for scripts
✗ Mix multiple script languages

Timeout:
├─ User-data has ~15 minute timeout
├─ Long tasks: Use custom AMI instead
├─ If needed: Background process + logging
```

---

## ⚡ Part 6: Performance Optimization (Advanced)

### 6.1 CPU Optimization

```
Choosing Right CPU:

Older generation (bad):
├─ t2: 2013 (10+ years old!)
├─ m5: 2019 (5 years old)
├─ Lower performance per dollar
└─ Avoid (always choose newest)

Current generation (good):
├─ t3: 2017-2023 (general purpose)
├─ m6i: 2021 (general purpose)
├─ c6i: 2021 (compute optimized)
├─ Better performance per dollar
└─ Choose these

Latest generation (best):
├─ t4g, m7g, c7i: 2023-2024
├─ 15-30% better than previous gen
├─ Same or lower price
├─ Newest features
└─ Choose if available in your region

CPU Monitoring:

Check CPU usage:
├─ aws ec2 get-console-output: Might show metrics
├─ CloudWatch: Best way to monitor
├─ Inside instance: top, htop, ps aux
└─ Example:
   top -bn1 | head -20
   Shows: CPU %, Memory %, Processes

If CPU frequently > 80%:
├─ Option 1: Vertical scale (bigger instance)
├─ Option 2: Horizontal scale (more instances)
├─ Option 3: Optimize application code
├─ Option 4: Use read replicas for DB

If CPU burstable throttling:
├─ You're using t family with sustained load
├─ Upgrade to fixed instance (m/c/r family)
├─ Example: t3.medium → m6i.large
```

### 6.2 Memory Optimization

```
Choosing Right Amount of RAM:

Application memory usage:
├─ Baseline: OS + system = ~1 GB
├─ Node.js: ~100-200 MB
├─ Python: ~50-100 MB
├─ Ruby: ~100-150 MB
├─ Cache: Variable (0-5+ GB)
└─ Total: Varies by application

Rule of thumb:
├─ Small app: 4 GB (t3.medium)
├─ Medium app: 8 GB (m6i.large)
├─ Large app: 16+ GB (m6i.xlarge or r6i)

RAM Monitoring:

Check memory usage:
├─ free -h: Shows total, used, free
├─ top: Shows memory per process
├─ CloudWatch: Monitor over time

Example:
$ free -h
              total        used        free
Mem:          3.8Gi       2.1Gi       1.7Gi
Swap:         1.0Gi       0.0B        1.0Gi

Interpretation:
├─ Total: 3.8 GB (t3.medium)
├─ Used: 2.1 GB (55% usage)
├─ Free: 1.7 GB (still comfortable)
├─ If > 90%: Memory pressure, scale up

Memory leak detection:
├─ Monitor daily: free -h | grep Mem
├─ Track pattern:
│  ├─ Day 1: Used 1.5 GB
│  ├─ Day 2: Used 2.0 GB
│  ├─ Day 3: Used 2.5 GB (growing!)
│  ├─ Problem: Memory leak in app
│  └─ Solution: Debug and fix app
│
├─ If growing: Restart application
│  └─ pm2 restart myapp (temporary fix)
│  └─ Find and fix memory leak (permanent)
```

### 6.3 Network Optimization

```
Network Performance:

Instance network bandwidth:
├─ t3: Up to 5 Gbps
├─ m6i: Up to 12.5 Gbps
├─ c6i: Up to 12.5 Gbps
├─ r6i: Up to 12.5 Gbps
├─ Larger instances: Up to 100+ Gbps
└─ For most apps: Not a constraint

Network optimization:

1. Use Enhanced Networking
   ├─ Older: No enhanced networking
   ├─ Current: SR-IOV (faster, lower latency)
   ├─ Latest: ENA (even faster)
   └─ Automatic on current instances

2. Place in same AZ (reduce latency)
   ├─ EC2 ↔ RDS: Same AZ = <1ms
   ├─ EC2 ↔ RDS: Different AZ = 1-5ms
   ├─ EC2 ↔ EC2: Same AZ = <0.5ms
   └─ Always keep related services same AZ

3. Use VPC Endpoints
   ├─ EC2 ↔ S3: VPC Endpoint = faster, free
   ├─ EC2 ↔ S3: Via IGW = slower, costs data transfer
   └─ Set up VPC endpoint for S3

4. Compress data
   ├─ Enable gzip compression in Nginx
   ├─ Reduces bandwidth 70-90%
   ├─ Example: 1 MB → 100 KB compressed
   └─ Nginx config:
      gzip on;
      gzip_types text/plain text/css text/javascript;
      gzip_min_length 1024;

Network monitoring:

Check network throughput:
├─ AWS Console: Instance → Monitoring → Network
├─ CloudWatch: NetworkIn, NetworkOut
├─ Inside instance: iftop, nethogs, ss

If network is bottleneck:
├─ Consider larger instance (more bandwidth)
├─ Or: Use CDN to offload traffic
├─ Or: Optimize application (compress, cache)
```

---

## 💰 Part 7: Cost Optimization (Advanced)

### 7.1 Instance Sizing for Cost

```
RIGHT-SIZING:

Over-provisioned (WASTEFUL):
├─ Choose m6i.4xlarge ($780/month)
├─ Use: 10% CPU, 20% RAM average
├─ Cost: $780/month for 10% utilization
├─ Waste: $700/month

Right-sized (OPTIMAL):
├─ Choose m6i.large ($70/month)
├─ Use: 70% CPU, 70% RAM average
├─ Cost: $70/month for 70% utilization
├─ Waste: $0 (optimal!)

Under-provisioned (RISKY):
├─ Choose t3.medium ($30/month)
├─ Use: 90% CPU, 90% RAM (throttling!)
├─ Cost: $30/month but experience throttling
├─ Problem: Slow application

BEST PRACTICE:
├─ Start small (t3.medium)
├─ Monitor for 1 week
├─ See avg CPU, RAM usage
├─ Upgrade only if needed (70-80% usage)
```

### 7.2 Reserved Instances vs On-Demand

```
ON-DEMAND (Pay per hour):
├─ t3.medium: $0.0416/hour
├─ 1 year cost: 365 × 24 × 0.0416 = $364/year
├─ Flexibility: Can stop/start anytime
├─ Good for: Variable workloads, learning
├─ Commitment: None

1-YEAR RESERVED INSTANCE:
├─ t3.medium: $0.028/hour (33% discount!)
├─ 1 year cost: 365 × 24 × 0.028 = $245/year
├─ Savings: $364 - $245 = $119/year (33%)
├─ Commitment: Must keep for 1 year
├─ Flexible: Can modify instance type (upgrade)

3-YEAR RESERVED INSTANCE:
├─ t3.medium: $0.022/hour (47% discount!)
├─ 3 year cost: 1095 × 24 × 0.022 = $577/year
├─ Savings: ~$287/year × 3 = $861 total
├─ Commitment: Must keep for 3 years
├─ Less flexible: Limited modifications

WHEN TO USE EACH:

On-Demand:
├─ Development/testing (need flexibility)
├─ Unpredictable traffic
├─ Temporary projects
├─ First few months (don't know what you need)

Reserved Instances:
├─ Baseline capacity (always needed)
├─ Production workloads (stable)
├─ Long-term projects
├─ After 3+ months (know your needs)

RECOMMENDATION:
├─ Months 1-3: On-demand (learn, adjust)
├─ After Month 3: Buy 1-year reserved (save 33%)
├─ If committed: 3-year reserved (save 47%)
```

### 7.3 Spot Instances (70% Discount!)

```
SPOT INSTANCES: Temporary, cheap capacity

How Spot works:
├─ AWS has excess capacity
├─ Sells at 70% discount
├─ But: Can be terminated with 2 minutes notice
├─ Price: Varies by demand (sometimes 90% cheaper!)

Spot vs On-Demand:

t3.medium pricing:
├─ On-demand: $0.0416/hour
├─ Spot: $0.0125/hour (70% cheaper!)
├─ Savings: $0.0291/hour
├─ 1 year: $255 savings!

When to use Spot:
├─ Batch processing (can restart)
├─ Machine learning training (resumable)
├─ Data analysis (can rerun)
├─ Development/testing (not production)

When NOT to use Spot:
├─ Production API servers (need stable)
├─ Databases (can't lose data)
├─ Real-time systems (can't tolerate interruption)
├─ User-facing services (can't interrupt users)

Spot Interruption:

If AWS needs capacity:
├─ 2-minute warning email
├─ Instance terminates
├─ Data lost (unless on EBS with snapshots)
├─ No charge for interrupted hour

Mitigation:
├─ Use Spot Fleet (multiple, auto-replace)
├─ Combine Spot + On-Demand
├─ Keep critical data in EBS snapshots
├─ Auto Scaling (replaces terminated instances)

RECOMMENDATION:
├─ Development: 100% Spot (save max)
├─ Testing: 100% Spot
├─ Production batch: 70% Spot + 30% On-Demand (mixed)
├─ Production API: 100% On-Demand (stable)
```

---

## 📊 Part 8: Monitoring & Health Checks (Advanced)

### 8.1 Instance Status Checks

EC2 has two types of status checks:

```
SYSTEM STATUS CHECK:
├─ Checks: Hardware, network, AWS infrastructure
├─ Examples of failures:
│  ├─ Physical server failure
│  ├─ Network connectivity problem
│  ├─ Power/cooling failure
│  └─ AWS platform issue
│
├─ Your responsibility: None (AWS handles)
├─ Recovery: Automatic (move to new hardware)
├─ Duration: Minutes to hours
└─ Impact: Complete downtime (can't fix)

INSTANCE STATUS CHECK:
├─ Checks: Operating system, application
├─ Examples of failures:
│  ├─ Kernel panic
│  ├─ Out of disk space
│  ├─ Out of memory
│  ├─ Misconfigured security group
│  └─ Application crashed
│
├─ Your responsibility: Yours to fix!
├─ Recovery: Manual or auto-scaling
├─ Duration: Minutes (once fixed)
└─ Impact: Only this instance

AWS Console showing status:
├─ Green checkmark: OK
├─ Red X: Failed
├─ Gray: Initializing or unknown

CloudWatch alarm:
├─ Automatic notification if status check fails
├─ Can trigger auto-recovery
├─ Can trigger scaling actions
```

### 8.2 CloudWatch Monitoring

```
KEY METRICS TO MONITOR:

CPU Utilization:
├─ What: Percentage of vCPU being used
├─ Alert: > 80% consistently (scale up)
├─ Alert: > 90% (immediate action needed!)
├─ View: CloudWatch → Metrics → EC2

Memory Usage:
├─ What: Percentage of RAM being used
├─ Alert: > 80% consistently
├─ Alert: Growing over time (memory leak)
├─ View: CloudWatch → Metrics → Custom metrics
        (requires agent installation)

Network In/Out:
├─ What: Bytes in/out per instance
├─ Alert: Sudden spike (DDoS? scan?)
├─ Use: Capacity planning
├─ View: CloudWatch → Metrics → EC2 → Network

Disk Space:
├─ What: Percentage of disk used
├─ Alert: > 80% full (will fill up!)
├─ Alert: > 90% full (emergency!)
├─ View: CloudWatch (requires custom metric)

Setting up monitoring:

1. Install CloudWatch agent
2. Configure to send metrics
3. Create CloudWatch alarms
4. Set SNS notifications

Example: Alert on high CPU

aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-instance \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:MyTopic

Result:
├─ If CPU > 80% for 10 minutes
├─ AWS sends SNS notification
├─ You get email/SMS alert
└─ Take action before performance degrades
```

---

## 🐛 Part 9: Troubleshooting Common Issues (Advanced)

### 9.1 Instance Won't Launch

```
ISSUE: Instance gets to "running" but can't connect

Checklist:

□ Is instance running?
  └─ AWS Console: Status should be "running" (green)

□ Is it in correct security group?
  └─ Check: Inbound rule allows port 22 (SSH)
  └─ Check: Source allows your IP (0.0.0.0/0 or specific)

□ Do you have correct SSH key?
  └─ Check: Using same key you selected at launch?
  └─ Check: Key has correct permissions (chmod 600)

□ Is security group allowing SSH?
  └─ EC2 Dashboard → Security Groups → Inbound
  └─ Should show: SSH (port 22) from 0.0.0.0/0 or your IP

□ Can you reach the instance at all?
  └─ ping 54.123.45.67
  └─ If timeout: SG blocking ICMP, or public IP wrong

□ SSH timeout (hangs forever)?
  └─ Problem: Security group not allowing SSH
  └─ Or: Instance in private subnet without bastion
  └─ Or: Wrong security group

SOLUTION STEPS:

1. Check instance has public IP:
   aws ec2 describe-instances --instance-ids i-0abc123 \
     --query 'Reservations[0].Instances[0].PublicIpAddress'
   
   If empty: Instance in private subnet or no Elastic IP

2. Check security group allows SSH:
   aws ec2 describe-security-groups --group-ids sg-12345

   Should show:
   - IpProtocol: tcp
   - FromPort: 22
   - CidrIp: 0.0.0.0/0 (or your IP)

3. Try with verbose SSH:
   ssh -v -i prod-key.pem ec2-user@54.123.45.67
   
   Look for: "Connecting to 54.123.45.67..." or timeout

4. Wait 60 seconds:
   └─ Instance needs time to fully boot
   └─ OS startup can take 30-60 seconds
```

### 9.2 Instance CPU/Memory Maxed Out

```
ISSUE: Instance using 95%+ CPU or RAM

Diagnosis:

1. SSH into instance:
   ssh -i prod-key.pem ec2-user@54.123.45.67

2. Check what's using resources:
   top -bn1 | head -30
   
   Shows:
   - CPU%: Which process using CPU
   - MEM%: Which process using memory
   - PID: Process ID

3. Check specific process:
   ps aux | grep node
   ps aux | grep nginx
   
   Shows: Command line, resources used

Common causes:

HIGH CPU:
├─ Application infinite loop
├─ Unoptimized database query
├─ Excessive CPU-bound work
├─ Solution: Fix application code

HIGH MEMORY:
├─ Memory leak in application
├─ Too many processes running
├─ Solution: Find memory leak, restart app

SOLUTIONS:

Immediate:
1. Restart application: pm2 restart myapp
2. Monitor if issue returns: top (watch for 5 min)

Short-term:
3. Upgrade instance: t3.medium → m6i.large
4. Lowers CPU% but doesn't fix root cause

Long-term:
5. Debug root cause
6. Optimize code/queries
7. Add caching (Redis)
8. Scale horizontally (multiple instances)
```

### 9.3 Disk Space Running Out

```
ISSUE: Disk 100% full, instance sluggish

Check disk usage:
df -h

Shows:
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda       30G   29G   1G   97%  /

The 1G remaining will fill up soon!

Find what's taking space:
du -sh /* 2>/dev/null | sort -rh

Shows largest directories:
2.0G    /var
1.5G    /home
800M    /opt

Check logs:
du -sh /var/log/*

Shows:
1.2G    /var/log/nginx
500M    /var/log/syslog
300M    /var/log/auth.log

Solutions:

IMMEDIATE (to prevent crash):

1. Clean old logs:
   sudo find /var/log -name "*.log" -type f -mtime +30 -delete
   (Delete logs older than 30 days)

2. Archive PM2 logs:
   pm2 flush
   (Clears PM2 logs)

3. Clean package cache:
   sudo yum clean all
   (For Amazon Linux)
   
   OR
   
   sudo apt clean
   (For Ubuntu)

LONG-TERM:

1. Enable log rotation:
   sudo nano /etc/logrotate.d/nginx
   
   Configure:
   daily
   rotate 14
   compress
   (Keeps 14 days, compresses older)

2. Increase disk size:
   Stop instance
   Modify EBS volume: 30GB → 50GB
   Start instance
   Resize filesystem: sudo resize2fs /dev/xvda

3. Monitor disk usage:
   CloudWatch alarm for > 80% used
   Auto-alert before it fills
```

---

## ✅ Best Practices Summary

### Design Principles

```
✓ LAUNCH:
├─ Choose right instance type (don't over-provision)
├─ Use custom AMI (faster, consistent)
├─ Set up monitoring from day 1
├─ Use Elastic IP for production

✓ OPERATE:
├─ Stop instances when not in use (cost savings)
├─ Monitor CPU, memory, disk daily
├─ Keep instance updated (yum/apt upgrades)
├─ Back up important data (EBS snapshots)

✓ SCALE:
├─ Start with right size (don't guess)
├─ Monitor before scaling
├─ Scale horizontally (multiple instances)
├─ Use Auto Scaling Groups for automation

✓ COST:
├─ Right-size for workload
├─ Reserved instances after 3 months
├─ Spot for non-critical work
├─ Stop unused instances

✓ SECURITY:
├─ Restrict security groups (least privilege)
├─ Use SSH keys, not passwords
├─ Keep OS updated
├─ No secrets in AMI (use IAM roles)
```

---

## 📋 Quick Reference Checklist

```
LAUNCHING PRODUCTION INSTANCE:

Instance:
□ Choose instance type (m6i.large? t3.medium?)
□ Choose AMI (Ubuntu 24.04? Amazon Linux 2?)
□ Create custom AMI (if needed)
□ Allocate Elastic IP
□ Assign to security group
□ Assign to VPC/subnet

Networking:
□ In private or public subnet? (correct for role)
□ Security group allows inbound (SSH, HTTP, HTTPS)
□ Elastic IP associated
□ DNS updated to point to Elastic IP

Configuration:
□ SSH key pair selected
□ User data script provided (if needed)
□ Monitoring enabled (CloudWatch)
□ Alarms configured (CPU, disk, memory)

Verification:
□ Instance running
□ Can SSH successfully
□ Application running
□ CloudWatch metrics visible
□ Health checks passing

Documentation:
□ Instance ID recorded
□ Elastic IP recorded
□ Security group documented
□ Purpose/function documented
□ Emergency contact documented

DEPLOYMENT READY: ✓
```

---

## 🎓 Conclusion

You now understand EC2 at a production level.

### Key Takeaways

1. **Instance Lifecycle** — pending → running → stopped → terminated
2. **Instance Types** — Choose based on workload (t/m/c/r families)
3. **AMI Creation** — Custom images for faster, consistent deployments
4. **Elastic IPs** — Static IPs for production reliability
5. **Instance Metadata** — Instances can discover themselves
6. **User Data** — Automation at launch time
7. **Performance** — Monitor CPU, memory, network
8. **Cost Optimization** — Right-size, use reserved/spot instances
9. **Monitoring** — CloudWatch alarms for proactive management
10. **Troubleshooting** — Systematic approach to debugging

### Next Steps

1. **Launch:** Create production-grade instance following checklist
2. **Monitor:** Watch metrics for 1 week
3. **Right-size:** Adjust instance type based on actual usage
4. **Optimize:** Purchase reserved instances for long-term cost savings
5. **Scale:** Use Auto Scaling Groups for resilience

---

**End of File**

You're ready to deploy and manage EC2 instances at scale.

