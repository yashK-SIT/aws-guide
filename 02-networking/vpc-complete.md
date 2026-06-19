# AWS VPC Complete: Virtual Private Cloud Deep Dive

**File Version:** 1.0  
**Last Updated:** May 2026  
**Audience:** Everyone — Beginners through Senior Architects  
**Prerequisites:** Read `01-foundations/aws-fundamentals.md` (recommended)  
**Time to Read:** 45-60 minutes (complete), 20 min (beginner section)  
**Difficulty:** Beginner to Advanced  

---

## 📚 What You Will Learn

By reading this guide, you'll understand:

1.  ✅ **CIDR Notation** — How IP addresses are divided and written
2.  ✅ **VPC Creation** — How to build your virtual network from scratch
3.  ✅ **Subnets** — Dividing VPC into smaller networks (public vs private)
4.  ✅ **Routing** — How traffic finds its way between subnets and internet
5.  ✅ **Internet Gateway** — Connection to the public internet
6.  ✅ **NAT Gateway** — Secure private subnet internet access
7.  ✅ **Security Groups** — Stateful firewall rules (per-instance)
8.  ✅ **Network ACLs** — Stateless firewall rules (per-subnet)
9.  ✅ **VPC Endpoints** — Private AWS service access (no internet needed)
10. ✅ **VPC Peering** — Connecting multiple VPCs
11. ✅ **Architecture Patterns** — Production VPC designs
12. ✅ **Troubleshooting** — Networking debugging methods

**After this guide, you will:**
- Understand how AWS networks actually work
- Design VPCs for security and scalability
- Troubleshoot connectivity issues independently
- Make informed architectural decisions
- Implement production-grade network architecture

---

## 👥 Who Should Read This

**Read the whole document if you:**
- Building production systems on AWS
- Need to understand network isolation
- Scaling beyond single subnet
- Implementing security best practices
- Troubleshooting connectivity problems
- Designing multi-tier architecture

**Read just "Beginner Summary" if you:**
- New to networking concepts
- Want 10-minute high-level overview
- Will dive deeper later

---

## 🎯 Beginner Summary: VPC in Plain English

Think of a VPC like **building your own office campus**:

```
AWS = ENTIRE CITY (global cloud)
└─ Region = BUSINESS DISTRICT (specific geographic location)
   └─ VPC = YOUR CAMPUS (walled-off private area)
      ├─ Building A = PUBLIC SUBNET (people visit here, faces street)
      ├─ Building B = PRIVATE SUBNET (internal only, hidden from street)
      ├─ Parking gate = INTERNET GATEWAY (entrance to main road)
      ├─ Security guard = SECURITY GROUP (controls who enters each building)
      ├─ Walls = NETWORK ACLs (controls what exits campus)
      └─ Internal roads = ROUTE TABLES (traffic directions)

Your Security Model:
├─ Outside world (Internet): Can't see inside campus
├─ Internet Gateway: 1 entrance/exit for authorized traffic
├─ Public Subnet: Receives/sends internet traffic
├─ Private Subnet: NEVER talks to internet (maximum security)
└─ Security Groups: Each building decides who enters
```

**The layers of protection:**

```
Internet (outside world)
    │
    ├─ Security Group: "Only HTTPS port 443? Allowed"
    │
    └─ Network ACL: "Only from these IPs? Allowed"
       │
       └─ Your EC2 instance: "Is this my security group? Yes → Process request"

Multiple layers = Multiple chances to block attacks
```

---

## 🏗️ Architecture Overview

Here's a typical production VPC:

```
┌──────────────────────────────────────────────────────────────────┐
│                      AWS REGION: ap-south-1                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  VPC: 10.0.0.0/16                          │  │
│  │                                                            │  │
│  │  AVAILABILITY ZONE A          AVAILABILITY ZONE B          │  │
│  │  ┌────────────────────┐      ┌────────────────────┐        │  │
│  │  │ PUBLIC SUBNET      │      │ PUBLIC SUBNET      │        │  │
│  │  │ 10.0.1.0/24        │      │ 10.0.2.0/24        │        │  │
│  │  │ ┌────────────────┐ │      │ ┌────────────────┐ │        │  │
│  │  │ │ EC2 Instance A │ │      │ │ EC2 Instance B │ │        │  │
│  │  │ │ (Web Server)   │ │      │ │ (Web Server)   │ │        │  │
│  │  │ │ 10.0.1.100     │ │      │ │ 10.0.2.100     │ │        │  │
│  │  │ └────────────────┘ │      │ └────────────────┘ │        │  │
│  │  └────────────────────┘      └────────────────────┘        │  │
│  │           ▲                             ▲                  │  │
│  │           │                             │                  │  │
│  │  ┌────────────────────────────────────────────────┐        │  │
│  │  │          LOAD BALANCER (ALB)                   │        │  │
│  │  │          Distributes traffic                   │        │  │
│  │  └────────────────────────────────────────────────┘        │  │
│  │           │                                                │  │
│  │  ┌────────▼────────────────────────────────────────┐       │  │
│  │  │    INTERNET GATEWAY (IGW)                       │       │  │
│  │  │    Connection to internet                       │       │  │
│  │  └────────┬────────────────────────────────────────┘       │  │
│  │           │                                                │  │
│  │  ┌────────────────────┐      ┌────────────────────┐        │  │
│  │  │ PRIVATE SUBNET     │      │ PRIVATE SUBNET     │        │  │
│  │  │ 10.0.10.0/24       │      │ 10.0.11.0/24       │        │  │
│  │  │ ┌────────────────┐ │      │ ┌────────────────┐ │        │  │
│  │  │ │ RDS Database   │ │      │ │ Redis Cache    │ │        │  │
│  │  │ │ (PostgreSQL)   │ │      │ │ (ElastiCache)  │ │        │  │
│  │  │ │ 10.0.10.50     │ │      │ │ 10.0.11.50     │ │        │  │
│  │  │ └────────────────┘ │      │ └────────────────┘ │        │  │
│  │  └────────────────────┘      └────────────────────┘        │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────┐      │  │
│  │  │ NAT GATEWAY (for private subnet internet access) │      │  │
│  │  │ Allows private subnets to reach internet         │      │  │
│  │  │ (downloading patches, contacting APIs)           │      │  │
│  │  └──────────────────────────────────────────────────┘      │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

Traffic Flow:
┌─────────────────────────────┐
│ Internet User               │
│ curl https://example.com    │
└────────┬────────────────────┘
         │
         ▼ (DNS resolves to ALB)
┌─────────────────────────────┐
│ HTTPS Traffic (port 443)    │ ← Security Group allows port 443
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Internet Gateway            │ ← Routes traffic into VPC
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Load Balancer               │ ← Decides which instance
│ (Route table: IGW allowed)  │
└────────┬────────────────────┘
         │
         ├──────────┬──────────┐
         ▼          ▼          ▼
    ┌────────┐  ┌────────┐
    │ EC2-A  │  │ EC2-B  │  (Private subnets: NOT SHOWN)
    │:443    │  │:443    │
    └────────┘  └────────┘
```

---

## 📋 Part 1: CIDR Notation (Beginner to Intermediate)

### Why This Matters

CIDR notation is how you write IP address ranges. Understanding it is essential for:
- Creating subnets
- Configuring security groups
- Troubleshooting connectivity
- Planning network growth

### 1.1 IP Address Basics

#### What Is an IP Address?

An IP address is a unique identifier for a device on a network.

```
Format: 4 numbers separated by dots
Example: 192.168.1.100

Each number:
├─ Range: 0-255 (8 bits of data)
├─ 255 represents 11111111 in binary
├─ 0 represents 00000000 in binary
└─ Total: 256 possibilities per number

Total combinations: 256 × 256 × 256 × 256 = 4,294,967,296 addresses
└─ That's about 4 billion addresses (IPv4)
```

#### IP Address Classes (Traditional)

Originally, IPs were divided into classes:

```
Class A: 1.0.0.0 to 126.255.255.255
├─ First number: 1-126
├─ Used for: Large networks
└─ Hosts per network: 16 million+

Class B: 128.0.0.0 to 191.255.255.255
├─ First two numbers define network
├─ Used for: Medium networks
└─ Hosts per network: 65,000+

Class C: 192.0.0.0 to 223.255.255.255
├─ First three numbers define network
├─ Used for: Small networks
└─ Hosts per network: 254

Private (NOT routable on internet):
├─ 10.0.0.0 to 10.255.255.255 (Class A private)
├─ 172.16.0.0 to 172.31.255.255 (Class B private)
└─ 192.168.0.0 to 192.168.255.255 (Class C private)
```

### 1.2 CIDR Notation Explained

CIDR = Classless Inter-Domain Routing. It replaces class-based addressing.

#### CIDR Format

```
10.0.0.0/16
│         └─ Prefix length (number of fixed bits)
└─ Base IP address

What does /16 mean?
├─ Total IP bits: 32 (8+8+8+8)
├─ Fixed bits: 16 (first two numbers)
├─ Variable bits: 32-16 = 16 (last two numbers can vary)
└─ Result: 2^16 = 65,536 possible IP addresses
```

#### CIDR Calculation Examples

```
10.0.0.0/16
├─ Fixed: 10.0 (first two numbers)
├─ Variable: x.x (last two numbers)
├─ Range: 10.0.0.0 to 10.0.255.255
├─ Total IPs: 65,536
└─ Usable for servers: 65,534 (minus network and broadcast)

10.0.0.0/24
├─ Fixed: 10.0.0 (first three numbers)
├─ Variable: x (last number)
├─ Range: 10.0.0.0 to 10.0.0.255
├─ Total IPs: 256
└─ Usable for servers: 254

10.0.1.0/28
├─ Fixed: 10.0.1 + first 4 bits of last number
├─ Variable: Last 4 bits
├─ Total IPs: 2^4 = 16
├─ Range: 10.0.1.0 to 10.0.1.15
└─ Usable for servers: 14
```

#### Common CIDR Prefixes

```
/8   = 256^3 = 16,777,216 addresses (entire Class A)
/16  = 256^2 = 65,536 addresses (typical VPC)
/20  = 4,096 addresses (typical subnet)
/24  = 256 addresses (typical small subnet)
/25  = 128 addresses
/28  = 16 addresses (very small, AWS uses for Lambda)
/32  = 1 address (single host)

Rule of thumb:
├─ Larger VPC: Use /16 (65k addresses)
├─ Each subnet: Use /24 (254 addresses per subnet)
├─ Very small: Use /28 (16 addresses)
└─ Single host: Use /32
```

#### CIDR Range Verification

How to verify if an IP is in a CIDR range:

```
Question: Is 10.0.5.100 in 10.0.0.0/16?

Step 1: Convert to binary
└─ 10.0.5.100 = 00001010.00000000.00000101.01100100
└─ 10.0.0.0   = 00001010.00000000.00000000.00000000

Step 2: Compare first /16 bits (first 16 bits)
├─ IP:    00001010.00000000 (rest doesn't matter)
├─ Range: 00001010.00000000 (first 16 bits)
└─ Match: YES ✓

Result: 10.0.5.100 IS in 10.0.0.0/16

Another example: Is 10.1.0.0 in 10.0.0.0/16?
├─ IP:    00001010.00000001 (first 16 bits)
├─ Range: 00001010.00000000 (first 16 bits)
└─ Match: NO ✗

Result: 10.1.0.0 is NOT in 10.0.0.0/16
```

### 1.3 CIDR Planning for VPC

#### VPC Size Planning

When creating a VPC, you assign a CIDR block. Choose wisely!

```
SCENARIO 1: Small startup (< 100 servers)
├─ VPC: 10.0.0.0/16 (65,536 addresses)
├─ Subnets: 10.0.1.0/24 (254 each)
├─ Reason: Easy to manage, room for growth
└─ Example:
   ├─ Public Subnet 1: 10.0.1.0/24 (AZ A)
   ├─ Public Subnet 2: 10.0.2.0/24 (AZ B)
   ├─ Private Subnet 1: 10.0.10.0/24 (AZ A)
   └─ Private Subnet 2: 10.0.11.0/24 (AZ B)

SCENARIO 2: Medium company (100-1000 servers)
├─ VPC: 10.0.0.0/16 (65,536 addresses)
├─ Subnets: 10.0.0.0/20 (4,096 each)
├─ Reason: Larger subnets for growth
└─ Example:
   ├─ Public: 10.0.1.0/20 (4,094 addresses)
   ├─ Private-App: 10.0.16.0/20 (4,094 addresses)
   ├─ Private-DB: 10.0.32.0/20 (4,094 addresses)
   └─ Private-Cache: 10.0.48.0/20 (4,094 addresses)

SCENARIO 3: Large company (1000+ servers)
├─ VPC: 10.0.0.0/15 (131,072 addresses)
├─ Subnets: 10.0.0.0/19 (8,192 each)
├─ Reason: Massive growth potential
└─ Multiple VPCs per environment
   ├─ Dev VPC: 10.0.0.0/16
   ├─ Staging VPC: 10.1.0.0/16
   ├─ Production VPC: 10.2.0.0/16
   └─ DR VPC: 10.3.0.0/16

SCENARIO 4: Multi-region organization
├─ Region 1: 10.0.0.0/8 (all subnets start with 10)
│  ├─ Dev: 10.0.0.0/16
│  ├─ Staging: 10.1.0.0/16
│  └─ Prod: 10.2.0.0/16
│
└─ Region 2: 172.16.0.0/8 (all subnets start with 172)
   ├─ Dev: 172.16.0.0/16
   ├─ Staging: 172.17.0.0/16
   └─ Prod: 172.18.0.0/16

Reason: Prevents IP overlap if VPCs are peered
```

#### CIDR Calculation Tool

Quick reference for common subnet sizes:

```
Prefix  Total    Usable   Use Case
───────────────────────────────────────
/16     65,536   65,534   Large VPC
/18     16,384   16,382   Large subnet
/20     4,096    4,094    Medium subnet
/22     1,024    1,022    Typical subnet
/24     256      254      Standard subnet (AWS default)
/25     128      126      Small subnet
/28     16       14       Tiny (AWS Lambda)
/32     1        1        Single host

AWS Reserved IPs (can't use):
└─ .0 = Network address (not usable)
└─ .1 = AWS router (reserved by AWS)
└─ .2 = DNS (reserved by AWS)
└─ .3 = Future use (reserved)
└─ .255 = Broadcast (not usable)

Example: 10.0.1.0/24
├─ .0 = Network address
├─ .1 = AWS router
├─ .2 = DNS
├─ .3 = Reserved
├─ .4 to .254 = Usable for instances (251 addresses)
└─ .255 = Broadcast

So /24 gives 256 addresses, but only 251 are usable!
```

---

## 🏗️ Part 2: VPC Architecture (Beginner to Intermediate)

### 2.1 VPC Fundamentals

#### What Is a VPC?

A VPC is your own isolated network within AWS.

```
AWS Global Infrastructure
    │
    ├─ 33 Regions worldwide
    │  │
    │  └─ ap-south-1 (Mumbai)
    │     │
    │     ├─ 3 Availability Zones (AZ)
    │     │
    │     └─ YOUR VPC ← YOU ARE HERE
    │        ├─ Isolated from other customers
    │        ├─ Only you can add/remove resources
    │        ├─ You control all networking
    │        └─ Firewall rules are yours
    │
    └─ Isolation = Security
       ├─ Other AWS customers can't see your VPC
       ├─ No network visibility across VPCs (by default)
       └─ You control everything inside
```

#### VPC Boundaries

```
DEFAULT: When you create AWS account
├─ AWS creates 1 default VPC per region
├─ Default VPC has 1 public subnet per AZ
├─ Default is OK for learning, NOT for production
└─ Best practice: Create custom VPC per project

CUSTOM: What we'll be creating
├─ Complete control
├─ Custom security model
├─ Custom CIDR ranges
├─ Better isolation
└─ Recommended for production
```

#### VPC Limits (What You Need to Know)

```
Per VPC:
├─ Subnets: Up to 200
├─ Route tables: Up to 200
├─ Security groups: Up to 500
├─ Network ACLs: Up to 200
└─ Elastic IPs: Up to 5 (soft limit, can increase)

Per Subnet:
├─ Network interfaces: Up to 350
└─ That's about 350 EC2 instances per subnet

What this means:
├─ You won't hit these limits in practice
├─ AWS is essentially unlimited for normal use
└─ Only extreme scale matters
```

### 2.2 Subnets: Dividing Your VPC

#### What Is a Subnet?

A subnet is a subdivision of your VPC. Subnets are how you achieve:
- **Isolation** (public vs private)
- **High Availability** (spread across AZs)
- **Scalability** (organize your resources)

```
Analogy:
├─ VPC = Entire office building
└─ Subnets = Individual floors
   ├─ Floor 1 (Public Subnet A): Reception, sales (faces outside)
   ├─ Floor 2 (Public Subnet B): Reception, sales (different building)
   ├─ Floor 3 (Private Subnet A): Engineering, R&D (internal only)
   └─ Floor 4 (Private Subnet B): Engineering, R&D (different building)
```

#### Public vs Private Subnets

```
PUBLIC SUBNET:
├─ Has route to Internet Gateway
├─ EC2 instances can talk to internet
├─ Can receive traffic from internet
├─ Used for: Web servers, load balancers, bastion hosts
├─ Risk: More exposed to attacks
└─ Must have Security Groups to protect

PRIVATE SUBNET:
├─ Does NOT have route to Internet Gateway
├─ EC2 instances CANNOT talk to internet (directly)
├─ Cannot receive traffic from internet
├─ Used for: Databases, internal services, cache
├─ Security: Hidden from internet
└─ Can still reach internet through NAT Gateway (if needed)

Decision Matrix:
┌────────────────────────────────────────────────┐
│ Should this resource be public or private?    │
├────────────────────────────────────────────────┤
│                                                │
│ Web Server    → PUBLIC  (users need to reach) │
│ API Server    → PUBLIC  (users call API)      │
│ Database      → PRIVATE (only app accesses)   │
│ Cache/Redis   → PRIVATE (only app accesses)   │
│ Message Queue → PRIVATE (only app accesses)   │
│ Load Balancer → PUBLIC  (users need to reach) │
│ Bastion Host  → PUBLIC  (admin SSH access)    │
│                                                │
└────────────────────────────────────────────────┘
```

#### Multi-AZ Subnet Architecture

For high availability, span subnets across AZs:

```
ap-south-1 (Region)
├─ AZ: ap-south-1a
│  ├─ Public Subnet: 10.0.1.0/24
│  │  └─ EC2 Instance A (Web Server)
│  └─ Private Subnet: 10.0.10.0/24
│     └─ RDS Database (Primary)
│
└─ AZ: ap-south-1b
   ├─ Public Subnet: 10.0.2.0/24
   │  └─ EC2 Instance B (Web Server)
   └─ Private Subnet: 10.0.11.0/24
      └─ RDS Database (Standby Replica)

Benefits:
├─ If AZ-a fails: AZ-b continues
├─ Load Balancer distributes between both
├─ Database replicates automatically
└─ Zero downtime failover

Load Balancer Health Checks:
├─ Checks EC2-A every 30 seconds
├─ If EC2-A fails: Routes all traffic to EC2-B
├─ If AZ-a fails completely: Everything routes to AZ-b
└─ Result: Users don't notice (automatic failover)
```

### 2.3 Internet Gateway

#### What Is an Internet Gateway?

An Internet Gateway (IGW) is how your VPC connects to the internet.

```
Without IGW:
├─ VPC completely isolated
├─ Can't reach internet
├─ Can't receive internet traffic
└─ Useless for public web servers

With IGW:
├─ VPC has 1 entrance/exit to internet
├─ Public subnets can reach internet
├─ Internet can reach public subnets
└─ Enables web servers
```

#### How IGW Works

```
Flow 1: Outbound (VPC → Internet)
├─ EC2 instance sends packet to 8.8.8.8
├─ Packet goes to route table
├─ Route table says: "8.8.8.8? → Internet Gateway"
├─ IGW receives packet
├─ IGW changes source IP from 10.0.1.100 to public IP (54.123.45.67)
├─ IGW sends to internet
├─ Response comes back to public IP
├─ IGW translates back to private IP (10.0.1.100)
└─ EC2 receives response

Flow 2: Inbound (Internet → VPC)
├─ User sends request to 54.123.45.67:443
├─ IGW receives on public IP
├─ IGW checks routing table
├─ Route table says: "54.123.45.67? → local VPC"
├─ IGW translates to private IP (10.0.1.100)
├─ Packet goes to EC2 instance
├─ EC2 security group checks: "Port 443? Allowed?"
├─ Yes → EC2 receives request
└─ EC2 responds back through IGW

Key Point: IGW does IP translation (NAT)
└─ Private IPs hidden from internet
└─ Only public IPs visible
└─ Good for security
```

#### Internet Gateway Rules

```
Per VPC:
├─ Only 1 IGW needed (it's not a bottleneck)
├─ Can attach to only 1 VPC
├─ Can detach and reattach
└─ Highly available (AWS manages redundancy)

Attachment:
├─ Create IGW: 1 minute
├─ Attach to VPC: 1 minute
└─ Route to subnet: Already done via route tables
```

---

## 🛣️ Part 3: Routing (Intermediate Level)

### 3.1 Route Tables Explained

#### What Is a Route Table?

A route table is a set of rules (called routes) that determine where network traffic goes.

```
Analogy: GPS Navigation
├─ Route table = Map with directions
├─ Each route = "If destination is X, go via Y"
├─ Destination = Where packet wants to go
├─ Target = Where to send it
└─ Match most specific first (if multiple routes match)

Example routes:
┌─────────────┬──────────────────────────────┐
│ Destination │ Target                       │
├─────────────┼──────────────────────────────┤
│ 10.0.0.0/16 │ local (stay in VPC)          │
│ 0.0.0.0/0   │ Internet Gateway (go out)    │
└─────────────┴──────────────────────────────┘

Logic:
├─ Packet destination: 10.0.1.100 (within VPC)
├─ Check route 1: 10.0.0.0/16? YES → Target: local
├─ Action: Keep inside VPC
│
├─ Packet destination: 8.8.8.8 (outside VPC)
├─ Check route 1: 10.0.0.0/16? NO
├─ Check route 2: 0.0.0.0/0? YES (matches everything)
├─ Target: Internet Gateway
└─ Action: Send to IGW, which sends to internet
```

#### Subnet Association

Route tables are associated with subnets:

```
Typical Setup:

VPC: 10.0.0.0/16
├─ Route Table: Public-RT
│  └─ Routes:
│     ├─ 10.0.0.0/16 → local
│     └─ 0.0.0.0/0 → Internet Gateway
│  └─ Associated Subnets:
│     ├─ 10.0.1.0/24 (Public Subnet A)
│     └─ 10.0.2.0/24 (Public Subnet B)
│
└─ Route Table: Private-RT
   └─ Routes:
      ├─ 10.0.0.0/16 → local
      └─ 0.0.0.0/0 → NAT Gateway (optional)
   └─ Associated Subnets:
      ├─ 10.0.10.0/24 (Private Subnet A)
      └─ 10.0.11.0/24 (Private Subnet B)

Result:
├─ Public subnets: Can reach internet via IGW
├─ Private subnets: Cannot reach internet (unless NAT configured)
└─ All subnets can reach each other (local route)
```

#### Route Priority (Most Specific Wins)

```
When multiple routes could match, AWS uses most specific:

Example Route Table:
┌──────────────┬─────────────┐
│ Destination  │ Target      │
├──────────────┼─────────────┤
│ 10.0.0.0/8   │ IGW         │
│ 10.0.0.0/16  │ Local       │
│ 10.0.1.0/24  │ EC2 (peered)│
│ 0.0.0.0/0    │ NAT         │
└──────────────┴─────────────┘

Packet to 10.0.1.5 (inside specific subnet):
├─ Match 10.0.0.0/8? YES (/8 = 8 bits fixed)
├─ Match 10.0.0.0/16? YES (/16 = 16 bits fixed)
├─ Match 10.0.1.0/24? YES (/24 = 24 bits fixed) ← MOST SPECIFIC!
├─ Match 0.0.0.0/0? YES (matches everything)
└─ Use: EC2 target (most specific = /24)

Rule: Always use MOST SPECIFIC (highest prefix)
└─ Allows you to override general rules with specific ones
```

### 3.2 NAT Gateway (For Private Subnets)

#### What Is NAT?

NAT = Network Address Translation. It lets private subnets reach the internet.

```
Scenario: Private database server needs to download security patches

Without NAT:
├─ Private subnet: Can't reach internet
├─ Result: Can't download patches
└─ Problem: Server stays vulnerable!

With NAT Gateway:
├─ Private subnet route: 0.0.0.0/0 → NAT Gateway
├─ NAT Gateway is in public subnet (has internet access)
├─ Private instance sends request to NAT
├─ NAT translates source IP to its own public IP
├─ Sends request to internet
├─ Response comes back to NAT
├─ NAT translates back to private IP
├─ Private instance receives response
└─ Result: Can download patches securely!
```

#### NAT Gateway vs NAT Instance

```
NAT GATEWAY (Recommended):
├─ Managed by AWS (you don't manage it)
├─ Highly available (scales automatically)
├─ No downtime maintenance
├─ Cost: ~$45/month per NAT
├─ Bandwidth: ~45 Gbps
└─ Use this (much better)

NAT INSTANCE:
├─ Manual EC2 instance running NAT software
├─ You manage it (updates, patches, monitoring)
├─ Manual failover if it fails
├─ Cost: EC2 instance cost (~$30/month) + bandwidth
├─ Bandwidth: Limited by instance type
└─ Don't use (obsolete)

Decision: Use NAT Gateway (worth the $15/month more)
```

#### NAT Gateway Architecture

```
CORRECT (Per-AZ NAT):
──────────────────────
ap-south-1a:
├─ Public Subnet: 10.0.1.0/24
│  └─ NAT Gateway-A (in public subnet)
│     ├─ Has Elastic IP: 54.123.45.67
│     └─ Highly available within AZ-a
│
└─ Private Subnet: 10.0.10.0/24
   ├─ Route: 0.0.0.0/0 → NAT Gateway-A
   └─ Instances can reach internet via NAT-A

ap-south-1b:
├─ Public Subnet: 10.0.2.0/24
│  └─ NAT Gateway-B (in public subnet)
│     ├─ Has Elastic IP: 54.123.45.68
│     └─ Highly available within AZ-b
│
└─ Private Subnet: 10.0.11.0/24
   ├─ Route: 0.0.0.0/0 → NAT Gateway-B
   └─ Instances can reach internet via NAT-B

Benefits:
├─ If NAT-A fails: NAT-B still works
├─ If AZ-a fails: AZ-b continues with NAT-B
├─ Resilient (no single point of failure)
└─ Recommended architecture

Cost per AZ: ~$45/month per NAT Gateway
Total: 2 NAT Gateways = ~$90/month
Worth it for high availability!
```

#### NAT Gateway Configuration

```
When to use NAT Gateway:
├─ Private database needs to download patches
├─ Private instances need to call external APIs
├─ Private Lambda functions need internet
├─ Any private resource needing outbound internet

When NOT to use:
├─ Private resource never needs internet access
├─ Can use VPC endpoints instead (free!)
└─ Example: EC2 accessing S3 → Use VPC endpoint, not NAT

Cost vs VPC Endpoints:
├─ NAT Gateway: $45/month per AZ (+ data transfer costs)
├─ VPC Endpoint: $7/month per endpoint (+ data transfer)
├─ VPC Endpoint: MUCH cheaper for AWS services!
└─ Use VPC endpoints when possible
```

---

## 🔐 Part 4: Security (Intermediate to Advanced)

### 4.1 Security Groups (Stateful Firewall)

#### What Is a Security Group?

A security group is a virtual firewall that controls inbound/outbound traffic for EC2 instances.

```
Analogy: Bouncer at nightclub
├─ Bouncer checks: "Is this person allowed?"
├─ If yes → Let them in
├─ If no → Block
├─ Security Group: Same logic for network traffic

Location: Attached directly to EC2 instance (or ENI)
Coverage: Protects individual instances
State: Stateful (remembers connections)
└─ If you let traffic IN, response automatically goes OUT
```

#### Security Group Rules

```
INBOUND RULES (Who can send data TO your instance):

Example:
┌──────┬──────┬─────────┬──────────────┐
│ Type │ Prot │ Port    │ Source       │
├──────┼──────┼─────────┼──────────────┤
│ HTTP │ TCP  │ 80      │ 0.0.0.0/0    │ (anyone)
│ HTTPS│ TCP  │ 443     │ 0.0.0.0/0    │ (anyone)
│ SSH  │ TCP  │ 22      │ 203.0.113.0/8│ (office only)
│ ALL  │ TCP  │ 3306    │ sg-xyz       │ (MySQL from app SG)
└──────┴──────┴─────────┴──────────────┘

Interpretation:
├─ Anyone can HTTP (port 80)
├─ Anyone can HTTPS (port 443)
├─ Only office IPs can SSH (port 22)
└─ Only instances in sg-xyz can connect to MySQL (port 3306)

OUTBOUND RULES (What data can your instance SEND):

Example (Default):
┌──────┬──────┬─────────┬──────────────┐
│ Type │ Prot │ Port    │ Destination  │
├──────┼──────┼─────────┼──────────────┤
│ ALL  │ ALL  │ ALL     │ 0.0.0.0/0    │ (anywhere)
└──────┴──────┴─────────┴──────────────┘

Interpretation:
├─ Your instance can send ANYTHING anywhere
├─ Usually keep default (allow all outbound)
└─ Restrict only if security requirement
```

#### Stateful Behavior (Important!)

```
STATEFUL = Connection remembered

Example: Web server listening on port 80

Inbound rule: Allow HTTP (port 80) from anywhere
Outbound rule: Allow all

Flow:
1. User sends: HTTP request to port 80
   ├─ Security group checks inbound rule
   ├─ Is it port 80? YES
   ├─ Allow: YES ✓
   └─ Packet passes through

2. Server sends: HTTP response (from port 80 to user's port 12345)
   ├─ Security group checks outbound rules
   ├─ Is it in allowed outbound? Need to check
   ├─ BUT: Security group remembers inbound connection
   ├─ It says: "I allowed port 80 in, so response must go out"
   ├─ Automatically allows response (stateful!)
   └─ Packet passes through

Result:
├─ No need to explicitly allow "port 80 response"
├─ Security group remembers and auto-allows
├─ Called "stateful" (keeps state of connections)
└─ Much easier than stateless firewalls (like NACLs)
```

#### Security Group Best Practices

```
PRINCIPLE: Least Privilege (allow minimum necessary)

BAD (Too Open):
├─ Allow SSH from 0.0.0.0/0 (anyone can SSH)
├─ Allow HTTP from 0.0.0.0/0 (even for internal service)
├─ Allow all ports
└─ Result: Huge attack surface

GOOD (Minimal):
├─ SSH: Allow from 203.0.113.0/8 only (your office)
├─ HTTP: Allow from Load Balancer security group only
├─ HTTPS: Allow from Load Balancer security group only
├─ Database: Allow port 3306 from app-server security group only
└─ Result: Minimal attack surface

Example: Multi-tier security

Load Balancer SG:
├─ Inbound: Port 80 from 0.0.0.0/0 (public)
├─ Inbound: Port 443 from 0.0.0.0/0 (public)
└─ Outbound: Allow all

Web Server SG:
├─ Inbound: Port 80 from ALB-SG only (not from internet!)
├─ Inbound: Port 443 from ALB-SG only
├─ Inbound: Port 22 from bastion-SG only (SSH access)
└─ Outbound: Port 3306 to database-SG

Database SG:
├─ Inbound: Port 3306 from web-server-SG only
└─ Outbound: None needed (mostly receives)

Result:
├─ Internet can only reach load balancer
├─ Can't directly reach web servers
├─ Can't directly reach database
├─ Only allowed inter-tier communication
└─ Very secure!
```

#### Security Group Chaining

You can reference other security groups:

```
Example: Allow traffic only from instances in another SG

Web Server SG:
├─ Inbound Rule:
│  ├─ Type: HTTPS
│  ├─ Port: 443
│  ├─ Source: app-server-sg (NOT an IP, but SG ID!)
│  └─ Meaning: "Allow HTTPS from instances with app-server-sg"
│
└─ When traffic comes:
   ├─ Check: Is source IP in app-server-sg? 
   ├─ If yes: Allow
   └─ If no: Block

Benefit:
├─ No need to know app server IPs
├─ Automatically works for new instances in app-server-sg
├─ Clean, scalable security
└─ Recommended for multi-tier architectures
```

### 4.2 Network ACLs (Stateless Firewall)

#### What Is a Network ACL?

A Network ACL is a firewall at the subnet level (vs instance level for Security Groups).

```
SECURITY GROUP vs NETWORK ACL:

Security Group:
├─ Level: Instance (per EC2)
├─ Type: Stateful
├─ Default: Allow all outbound
└─ Common: YES, usually sufficient

Network ACL:
├─ Level: Subnet (per subnet)
├─ Type: Stateless (must explicitly allow response!)
├─ Default: Allow all
└─ Common: Rarely needed (Security Groups sufficient)

When to use NACL:
├─ Blocking specific IPs at subnet level
├─ Advanced security requirements
├─ Rare in practice (Security Groups usually sufficient)
└─ Example: Block malicious IP from entire subnet
```

#### NACL Rules (Stateless)

```
STATELESS = No connection memory

Example: Allow port 80 inbound

Inbound rule: Allow HTTP port 80

Flow:
1. User sends: HTTP request to port 80
   ├─ NACL checks inbound rule
   ├─ Is it port 80? YES
   ├─ Allow: YES ✓
   └─ Packet passes through

2. Server sends: HTTP response (port 80 → port 12345)
   ├─ NACL checks OUTBOUND rules
   ├─ Is it explicitly allowed outbound?
   ├─ Stateless: NACL doesn't remember inbound!
   ├─ Need explicit rule: Allow port 12345 outbound
   └─ If missing: Block ✗

Need to configure:
├─ Inbound: Allow port 80
├─ Outbound: Allow port 80 (for responses)
├─ Outbound: Allow high ports (1024-65535 for responses)
└─ Verbose and error-prone!

VERDICT: Security Groups are much easier!
```

#### When To Use NACLs

```
Most common scenario using NACL:
└─ Blocking malicious IP from entire subnet

Example:
├─ Attack detected from 203.0.113.5
├─ Need to block at subnet level (not instance)
├─ Create NACL rule:
│  ├─ Deny all from 203.0.113.5
│  └─ Applied to entire subnet
│
└─ Result: 203.0.113.5 blocked from all instances in subnet

Other scenarios:
├─ Compliance requirement for subnet-level firewall
├─ Deny specific subnets from communicating
└─ Advanced network design (rare)

General recommendation:
├─ Use Security Groups for 95% of security needs
├─ Use NACLs for specific blocking requirements
└─ Security Groups are stateful and simpler!
```

---

## 🔗 Part 5: VPC Endpoints (Advanced)

### 5.1 Why VPC Endpoints Matter

#### Problem: Private Subnets Needing AWS Services

```
Scenario: Private database needs to backup to S3

Traditional solution (NAT Gateway):
├─ Private instance → Send data to S3
├─ No internet route in private subnet!
├─ Solution: Use NAT Gateway
│  ├─ Private instance sends to NAT
│  ├─ NAT forwards to S3 (goes over internet)
│  └─ Very inefficient (extra hop)
│
└─ Cost:
   ├─ NAT Gateway: $45/month
   ├─ Data transfer: $0.05 per GB OUT
   ├─ Example: 100 GB backup = 100 × $0.05 = $5
   └─ Total: $50/month for one backup!

Better solution (VPC Endpoint):
├─ Direct connection from VPC to S3 (no internet!)
├─ Private instance sends directly to S3
├─ Completely within AWS network
├─ Benefits:
│  ├─ No NAT Gateway needed (save $45/month!)
│  ├─ Lower latency (direct, not via internet)
│  ├─ Better security (traffic never leaves AWS)
│  ├─ Lower data transfer cost (sometimes free!)
│  └─ Faster backups
│
└─ Cost:
   ├─ VPC Endpoint: ~$7/month
   ├─ Data transfer: FREE within same region!
   ├─ Example: 100 GB backup = $0 (free!)
   └─ Total: $7/month (save $48/month!)
```

### 5.2 Types of VPC Endpoints

#### Gateway Endpoints

```
Services: S3, DynamoDB (mostly)

How it works:
├─ Add route to route table
├─ Route: "If destination is S3, use VPC endpoint"
├─ Traffic bypasses NAT, goes directly to S3
│
└─ Configuration:
   ├─ VPC → Endpoints → Create Endpoint
   ├─ Service: com.amazonaws.ap-south-1.s3
   ├─ VPC: Select your VPC
   ├─ Route table: Select private route table
   ├─ Endpoint added to route table automatically
   └─ Done! (No more configuration needed)

Usage:
├─ Private instance can now access S3
├─ Same as if it had internet (but it doesn't!)
└─ Transparent to your code

Example Route Table:
┌────────────┬──────────────────────┐
│ Destination│ Target               │
├────────────┼──────────────────────┤
│ 10.0.0.0/16│ local                │
│ 0.0.0.0/0  │ NAT Gateway          │
│ *.s3.*     │ pl-12345 (S3 prefix) │
└────────────┴──────────────────────┘

Key point: S3 endpoint added to route table automatically!
```

#### Interface Endpoints

```
Services: Most AWS services (EC2, SQS, SNS, Kinesis, etc.)

How it works:
├─ Creates ENI (Elastic Network Interface) in VPC
├─ Private IP assigned to ENI (e.g., 10.0.1.50)
├─ Private instance connects to this private IP
├─ ENI forwards traffic to AWS service
│
└─ Configuration:
   ├─ VPC → Endpoints → Create Endpoint
   ├─ Service: e.g., com.amazonaws.ap-south-1.ec2
   ├─ VPC: Select your VPC
   ├─ Subnets: Select which subnets (recommended: 1 per AZ)
   ├─ Security Group: Allow traffic to endpoint
   └─ DNS enabled: (optional) Creates private DNS entry

DNS Resolution (Optional):
├─ Enable "Private DNS name" option
├─ EC2 endpoint private DNS: ec2.ap-south-1.amazonaws.com
├─ Private instance can use standard AWS CLI
├─ Automatically resolves to endpoint (not internet)
└─ Seamless - your code doesn't need to change!

Example:
├─ Private instance wants to call EC2 API
├─ Instead of going to internet (10.0.1.100 → IGW → EC2 API)
├─ Goes to endpoint (10.0.1.100 → 10.0.1.50 → EC2 API)
├─ All within VPC!
└─ Benefits: Faster, more secure, cheaper
```

### 5.3 VPC Endpoint Best Practices

```
When to use VPC Endpoints:
├─ Private instances accessing AWS services ✓
├─ S3 backups from private instances ✓
├─ Private Lambda calling SQS ✓
├─ Private RDS accessing Secrets Manager ✓
└─ Almost any service access from private subnets ✓

Cost analysis:
├─ VPC Endpoint: ~$7/month per endpoint
├─ Data transfer: FREE (in same region) or cheap
├─ NAT Gateway: ~$45/month (only needed for non-AWS traffic now)
├─ Typical savings: $200-500/month at scale
└─ Highly recommended!

Comparison: NAT Gateway vs VPC Endpoints
┌────────────────┬──────────────┬────────────────┐
│ Scenario       │ NAT Gateway  │ VPC Endpoint   │
├────────────────┼──────────────┼────────────────┤
│ S3 backup      │ $45 + data   │ ~$7            │
│ EC2 API call   │ $45 + data   │ ~$7            │
│ External API   │ $45 + data   │ Need NAT ✗     │
│ Security       │ Medium       │ High ✓         │
│ Latency        │ Higher       │ Lower ✓        │
└────────────────┴──────────────┴────────────────┘

Recommendation:
├─ Use VPC Endpoints for all AWS services
├─ Use NAT Gateway for external APIs only
├─ Result: Significant cost savings!
```

---

## 🔗 Part 6: VPC Peering (Advanced)

### 6.1 What Is VPC Peering?

VPC Peering allows two VPCs to communicate as if they were the same network.

```
Use case: Multi-environment setup

Before Peering:
├─ Dev VPC: 10.0.0.0/16 (completely isolated)
├─ Prod VPC: 10.1.0.0/16 (completely isolated)
├─ Dev can't talk to Prod (by default)
├─ Problem: Can't share resources

After Peering:
├─ Dev VPC: 10.0.0.0/16 ←→ Prod VPC: 10.1.0.0/16
├─ Both VPCs connected
├─ Dev instances can access Prod resources
├─ Prod instances can access Dev resources
├─ Full network connectivity
└─ Private connection (not via internet!)
```

### 6.2 VPC Peering Setup

```
Step 1: Create Peering Connection
├─ VPC A (Dev) → Create Peering Connection
├─ Request VPC: Dev VPC (10.0.0.0/16)
├─ Peer VPC: Prod VPC (10.1.0.0/16)
├─ Status: Pending

Step 2: Accept Peering Connection
├─ VPC B (Prod) → Peering Connections
├─ Find pending connection
├─ Accept connection
├─ Status: Active

Step 3: Update Route Tables (Dev)
├─ Dev route table:
│  ├─ 10.0.0.0/16 → local (existing)
│  ├─ 10.1.0.0/16 → pcx-12345 (NEW - points to Prod VPC)
│  └─ 0.0.0.0/0 → IGW (existing)

Step 4: Update Route Tables (Prod)
├─ Prod route table:
│  ├─ 10.1.0.0/16 → local (existing)
│  ├─ 10.0.0.0/16 → pcx-12345 (NEW - points to Dev VPC)
│  └─ 0.0.0.0/0 → IGW (existing)

Step 5: Update Security Groups
├─ Dev instance security group:
│  └─ Allow 10.1.0.0/16 (Prod VPC) on needed ports
│
├─ Prod instance security group:
│  └─ Allow 10.0.0.0/16 (Dev VPC) on needed ports

Result:
├─ Dev and Prod VPCs connected
├─ Instances can communicate
├─ Still separate networks (isolation)
└─ Private connection (not via internet)
```

### 6.3 VPC Peering Limitations

```
Limitations:
├─ IP ranges can't overlap
│  └─ Dev: 10.0.0.0/16, Prod: 10.1.0.0/16 ✓ (different)
│  └─ Dev: 10.0.0.0/16, Prod: 10.0.1.0/16 ✗ (overlap!)
│
├─ Peering is 1-to-1 (A ↔ B only)
│  └─ If A peered to B, and B peered to C
│  └─ A cannot automatically reach C (need separate peering)
│  └─ Called "non-transitive"
│
├─ Traffic doesn't go through NAT
│  └─ Source IP remains same across peering
│  └─ Unlike internet traffic (translated by IGW)
│
└─ AWS Region can't peer (need different AWS service)
   └─ Same region peering only
   └─ Multi-region: Use Transit Gateway (different service)

For multi-region peering:
├─ Use different service: AWS Transit Gateway
├─ More complex setup
├─ Out of scope for this guide
└─ Advanced topic
```

---

## 🏛️ Part 7: Production VPC Architectures (Advanced)

### 7.1 3-Tier Production Architecture

This is the gold standard for most production applications.

```
ARCHITECTURE:

┌────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                       │
│                                                        │
│ ┌─────────────────────────────────────────────────┐    │
│ │ TIER 1: PUBLIC (Internet-facing)                │    │
│ │ Subnets: 10.0.1.0/24 (AZ-a), 10.0.2.0/24 (AZ-b) │    │
│ │                                                 │    │
│ │ ┌──────────────────────────────────────────┐    │    │
│ │ │ Application Load Balancer (ALB)          │    │    │
│ │ │ - Listens on ports 80/443                │    │    │
│ │ │ - Public IP: 54.123.45.67                │    │    │
│ │ │ - Health checks web servers              │    │    │
│ │ │ - Terminates SSL/TLS                     │    │    │
│ │ └──────────────────────────────────────────┘    │    │
│ │                                                 │    │
│ └─────────────────────────────────────────────────┘    │
│           ▲                           ▲                │
│           │ (Traffic from internet)   │                │
│ ┌─────────┴───────────────────────────┴────────────┐   │
│ │ TIER 2: PRIVATE APPLICATION (App servers)        │   │
│ │ Subnets: 10.0.10.0/24 (AZ-a), 10.0.11.0/24 (AZ-b)│   │
│ │                                                  │   │
│ │ ┌─────────────────────────────────────────┐      │   │
│ │ │ EC2 Instance (Web Server A)             │      │   │
│ │ │ - IP: 10.0.10.100                       │      │   │
│ │ │ - Runs Node.js / Python / Java          │      │   │
│ │ │ - Only receives traffic from ALB        │      │   │
│ │ │ - Talks to database layer               │      │   │
│ │ └─────────────────────────────────────────┘      │   │
│ │                                                  │   │
│ │ ┌─────────────────────────────────────────┐      │   │
│ │ │ EC2 Instance (Web Server B)             │      │   │
│ │ │ - IP: 10.0.11.100                       │      │   │
│ │ │ - Same config as Server A               │      │   │
│ │ └─────────────────────────────────────────┘      │   │
│ │                                                  │   │
│ └──────────────────────────────────────────────────┘   │
│           ▲                           ▲                │
│           │ (SQL queries, etc.)       │                │
│ ┌─────────┴───────────────────────────┴────────────┐   │
│ │ TIER 3: PRIVATE DATABASE (Data tier)             │   │
│ │ Subnets: 10.0.20.0/24 (AZ-a), 10.0.21.0/24 (AZ-b)│   │
│ │                                                  │   │
│ │ ┌─────────────────────────────────────────┐      │   │
│ │ │ RDS PostgreSQL (Primary)                │      │   │
│ │ │ - IP: 10.0.20.50                        │      │   │
│ │ │ - Only receives from app servers        │      │   │
│ │ │ - Replicated to secondary               │      │   │
│ │ └─────────────────────────────────────────┘      │   │
│ │                                                  │   │
│ │ ┌─────────────────────────────────────────┐      │   │
│ │ │ RDS PostgreSQL (Read Replica)           │      │   │
│ │ │ - IP: 10.0.21.50                        │      │   │
│ │ │ - Read-only copy (failover if primary)  │      │   │
│ │ └─────────────────────────────────────────┘      │   │
│ │                                                  │   │
│ │ ┌─────────────────────────────────────────┐      │   │
│ │ │ ElastiCache Redis (Cache)               │      │   │
│ │ │ - IP: 10.0.20.60                        │      │   │
│ │ │ - Session storage, query cache          │      │   │
│ │ │ - Speeds up app                         │      │   │
│ │ └─────────────────────────────────────────┘      │   │
│ │                                                  │   │
│ └──────────────────────────────────────────────────┘   │
│                                                        │
│ ┌──────────────────────────────────────────────┐       │
│ │ Internet Gateway                             │       │
│ │ (Connection to internet for public tier only)│       │
│ └──────────────────────────────────────────────┘       │
│                                                        │
│ ┌──────────────────────────────────────────────┐       │
│ │ NAT Gateway (in public subnet AZ-a)          │       │
│ │ (If private tier needs outbound internet)    │       │
│ └──────────────────────────────────────────────┘       │
│                                                        │
└────────────────────────────────────────────────────────┘

Security Layers:

Layer 1 (Public tier):
├─ ALB Security Group:
│  ├─ Allow port 80 from 0.0.0.0/0 (anyone)
│  ├─ Allow port 443 from 0.0.0.0/0 (anyone)
│  └─ Forward to app servers

Layer 2 (Private app tier):
├─ App Server Security Group:
│  ├─ Allow port 80 from ALB security group only
│  ├─ Allow port 443 from ALB security group only
│  ├─ Allow port 22 from bastion security group only (SSH)
│  └─ Allow all outbound (for API calls, patches)

Layer 3 (Private database tier):
├─ Database Security Group:
│  ├─ Allow port 3306 (MySQL) / 5432 (PostgreSQL) from app SG only
│  ├─ Allow port 6379 (Redis) from app SG only
│  └─ No inbound from public tier (extra security)

Traffic Flow:
├─ User sends HTTPS to 54.123.45.67 (ALB public IP)
├─ ALB receives and decrypts TLS
├─ ALB forwards to app server (10.0.10.100 or 10.0.11.100)
├─ App server processes request
├─ App server queries database (10.0.20.50 or 10.0.21.50)
├─ Database responds
├─ App server constructs response
├─ ALB sends response back to user (HTTPS)
└─ User receives encrypted response

Availability:
├─ Tier 1 (ALB): Spans 2 AZs (inherently HA)
├─ Tier 2 (App): 2 servers in different AZs
├─ Tier 3 (Database): RDS Multi-AZ (automatic failover)
└─ Result: 99.99% uptime (highly available)

Scalability:
├─ Add more app servers: Auto Scaling Group handles
├─ Scale database: RDS read replicas or upgrade instance
├─ Add cache: ElastiCache clusters
└─ Add storage: S3 for large files

Security:
├─ Internet only reaches ALB
├─ App servers hidden in private subnet
├─ Database hidden in private subnet
├─ Each tier has firewall (security group)
├─ Multiple attack surfaces to get through
└─ Defense in depth approach
```

### 7.2 Advanced: Multi-AZ with NAT

```
When private tier needs internet access:

┌──────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                     │
│                                                      │
│ ┌────────────────────┐         ┌──────────────────┐  │
│ │ AZ: ap-south-1a    │         │ AZ: ap-south-1b  │  │
│ │                    │         │                  │  │
│ │ PUBLIC SUBNET:     │         │ PUBLIC SUBNET:   │  │
│ │ 10.0.1.0/24        │         │ 10.0.2.0/24      │  │
│ │ ┌────────────────┐ │         │ ┌──────────────┐ │  │
│ │ │ NAT Gateway A  │ │         │ │ NAT Gateway B│ │  │
│ │ │ IP: 54.1.1.1   │ │         │ │ IP: 54.1.1.2 │ │  │
│ │ └────────────────┘ │         │ └──────────────┘ │  │
│ │       │            │         │       │          │  │
│ │       │            │         │       │          │  │
│ │ PRIVATE SUBNET:    │         │ PRIVATE SUBNET:  │  │
│ │ 10.0.10.0/24       │         │ 10.0.11.0/24     │  │
│ │ ┌────────────────┐ │         │ ┌──────────────┐ │  │
│ │ │ EC2 (App A)    │ │         │ │ EC2 (App B)  │ │  │
│ │ │ Route: 0.0.0.0/│ │         │ │ Route: 0.0.0/│ │  │
│ │ │ → NAT-A        │ │         │ │ → NAT-B      │ │  │
│ │ └────────────────┘ │         │ └──────────────┘ │  │
│ │                    │         │                  │  │
│ └────────────────────┘         └──────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘

Route Table (Private Subnet AZ-a):
├─ 10.0.0.0/16 → local
├─ 0.0.0.0/0 → NAT Gateway A

Route Table (Private Subnet AZ-b):
├─ 10.0.0.0/16 → local
├─ 0.0.0.0/0 → NAT Gateway B

Benefits:
├─ Each AZ has its own NAT (no single point of failure)
├─ If NAT-A fails: App-A still works (routes to NATA)
├─ If NAT-B fails: App-B still works (routes to NAT-B)
├─ If AZ-a fails: App-B handles all traffic
└─ Highly available internet access for private tier

Cost:
├─ NAT Gateway A: $45/month
├─ NAT Gateway B: $45/month
├─ Total: $90/month
└─ Worth it for high availability
```

---

## 🐛 Part 8: Troubleshooting Network Connectivity (Advanced)

### 8.1 Connectivity Troubleshooting Flowchart

```
PROBLEM: "Instance A can't reach Instance B"

FLOWCHART:

├─ STEP 1: Verify instances exist and are running
│  ├─ Check: Are both instances "running" (not stopped)?
│  ├─ If NO → Start instances
│  └─ If YES → Continue
│
├─ STEP 2: Verify instance IPs
│  ├─ Instance A: 10.0.1.100
│  ├─ Instance B: 10.0.2.100
│  └─ Are they in same VPC? → YES → Continue
│
├─ STEP 3: Check Security Groups
│  ├─ Instance A outbound rules: Allow traffic to Instance B?
│  │  └─ If NO (default) → Add rule: Allow all out
│  │  └─ If YES → Continue
│  │
│  ├─ Instance B inbound rules: Allow traffic from Instance A?
│  │  └─ If NO → Add rule: Allow from Instance A SG
│  │  └─ If YES → Continue
│  │
│  └─ Test: ssh from A to B
│     └─ If works → Done! (problem was SG)
│     └─ If not → Continue
│
├─ STEP 4: Check Network ACLs
│  ├─ Instance A subnet NACL outbound: Allow to Instance B?
│  │  └─ If NO → Add rule
│  │  └─ If YES → Continue
│  │
│  ├─ Instance B subnet NACL inbound: Allow from Instance A?
│  │  └─ If NO → Add rule
│  │  └─ If YES → Continue
│  │
│  └─ Test again
│     └─ If works → Done! (problem was NACL)
│     └─ If not → Continue
│
├─ STEP 5: Check Routing
│  ├─ Instance A subnet route table:
│  │  └─ Route to 10.0.2.0/24? → local
│  │  └─ If missing → Add route
│  │
│  ├─ Instance B subnet route table:
│  │  └─ Route to 10.0.1.0/24? → local
│  │  └─ If missing → Add route
│  │
│  └─ Test again
│     └─ If works → Done! (problem was routing)
│     └─ If not → Continue
│
├─ STEP 6: Cross-AZ issues
│  ├─ Instance A in AZ-a: 10.0.1.100 (ap-south-1a)
│  ├─ Instance B in AZ-b: 10.0.2.100 (ap-south-1b)
│  ├─ Problem: Different subnets in different AZs
│  │  └─ Solution: Everything already handled
│  │  └─ Check: Route tables have "local" for same VPC
│  │  └─ Should be automatic!
│  │
│  └─ If still not working: Check (a) and (b) above
│     └─ Is NACL rule limiting cross-AZ?
│     └─ Is SG rule too restrictive?
│
└─ STEP 7: Cross-VPC issues
   ├─ Instance A in VPC 1: 10.0.1.100
   ├─ Instance B in VPC 2: 10.1.1.100
   ├─ Problem: Different VPCs
   │  └─ Solution: VPC Peering required
   │  └─ Check: Is peering connection created and active?
   │  └─ Check: Route tables include peering routes?
   │  └─ Check: Security groups allow cross-VPC?
   │
   └─ If peering configured correctly, should work
      └─ Recheck all SG rules for VPC 2 peering CIDR
```

### 8.2 Debugging Commands

```bash
# SSH INTO INSTANCE AND RUN:

# 1. Check if other instance is reachable (ping)
ping 10.0.2.100
# Response: Bytes from 10.0.2.100: icmp_seq=1 ttl=254 time=1.5ms
# No response: No ICMP route or blocked

# 2. Check network interfaces
ip link show
# Shows: eth0 (network interface)
# Check: Is it UP and RUNNING?

# 3. Check IP configuration
ip addr show
# Shows: inet 10.0.1.100/24 (your IP)
# Check: IP is correct?

# 4. Check route table
ip route show
# Expected:
# 10.0.0.0/16 dev eth0 proto kernel scope link src 10.0.1.100
# default via 10.0.1.1 dev eth0

# 5. Test TCP port connectivity
nc -zv 10.0.2.100 3306  # MySQL
# Response: Connection successful
# Means: Port 3306 is open on target

# 6. Check security group
aws ec2 describe-security-groups --group-ids sg-12345
# Look for: Inbound rules
# Check: Does it allow traffic?

# 7. Check network ACL
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=subnet-abc123"
# Look for: Inbound/Outbound rules
# Check: Does it allow traffic?

# 8. Test with curl (HTTP)
curl -v http://10.0.2.100:3000
# Response: HTTP response (port open, app responding)
# Connection refused: Port not listening
# Timeout: Network blocked

# 9. Check local firewall (security group within OS)
sudo iptables -L -n
# AWS security groups are at instance level (not OS iptables)
# Usually: ACCEPT all (AWS SG is primary firewall)

# 10. Test DNS
nslookup example.com
# Response: Address: 54.123.45.67
# Can't resolve: DNS issue
```

### 8.3 Common Issues & Solutions

```
ISSUE 1: "Timeout" when connecting
──────────────────────────────────
Symptom: curl hangs, nc -zv times out
Cause: Network path blocked
Solution:
├─ Check Security Group (instance level)
├─ Check Network ACL (subnet level)
├─ Check Route table (routing)
├─ Check if app is listening on port
│  └─ Run: netstat -tlnp
│  └─ Should show: LISTEN 0.0.0.0:3000

ISSUE 2: "Connection refused"
──────────────────────────────
Symptom: curl gets "Connection refused"
Cause: Port not listening / application not running
Solution:
├─ Check application is running: ps aux | grep app
├─ Check PM2: pm2 status
├─ Check listening ports: netstat -tlnp
├─ Restart application: pm2 restart myapp

ISSUE 3: "Host unreachable"
────────────────────────────
Symptom: ping returns "Host unreachable"
Cause: No route to destination
Solution:
├─ Check route table: ip route show
├─ Check VPC peering (if cross-VPC): Is it configured?
├─ Check: Do both instances have routes to each other's subnets?

ISSUE 4: "Network unreachable"
───────────────────────────────
Symptom: ICMP error: "Network unreachable"
Cause: No route to destination network
Solution:
├─ Check route table: ip route show
├─ Must have route for destination CIDR
├─ Example:
│  ├─ Want to reach: 10.0.2.0/24
│  ├─ Route table should have: 10.0.2.0/24 via local
│  └─ If missing: Add route

ISSUE 5: Different subnets can't communicate
──────────────────────────────────────────────
Symptom: 10.0.1.0/24 can't reach 10.0.2.0/24
Cause: Usually NACL blocking or SG too restrictive
Solution:
├─ Check NACL rules in both subnets
├─ Check SG rules in both instances
├─ Example:
│  ├─ Instance A SG: Outbound - Allow all
│  ├─ Instance B SG: Inbound - Allow from Instance A SG
│  └─ Should work!

ISSUE 6: Cross-VPC connectivity not working
─────────────────────────────────────────────
Symptom: VPC A can't reach VPC B
Cause: Peering not configured or route missing
Solution:
├─ Check VPC Peering:
│  ├─ Is status "Active"?
│  ├─ Are both route tables updated?
│  └─ Example:
│     ├─ VPC A route table: 10.1.0.0/16 via pcx-12345
│     ├─ VPC B route table: 10.0.0.0/16 via pcx-12345
│
├─ Check Security Groups:
│  ├─ Instance A SG: Allow from VPC B CIDR
│  ├─ Instance B SG: Allow from VPC A CIDR
│
└─ If all above correct, should work!
```

---

## ✅ Best Practices Summary

### Design Principles

```
✓ SECURITY:
├─ Always use private subnets for databases
├─ Restrict security groups to minimum necessary
├─ Use VPC endpoints for AWS services (not NAT)
├─ Enable VPC Flow Logs for debugging

✓ SCALABILITY:
├─ Use /24 subnets (255 addresses, easily managed)
├─ Leave room for growth (start with /16 VPC)
├─ Plan for multiple AZs from start
├─ Use Auto Scaling across AZs

✓ HIGH AVAILABILITY:
├─ Always Multi-AZ for production
├─ NAT Gateway per AZ
├─ Database replication across AZs
├─ Load balancer spanning AZs

✓ COST OPTIMIZATION:
├─ Use VPC endpoints instead of NAT when possible
├─ Share resources where appropriate
├─ No extra charge for VPC itself (just resources)
└─ Monitor data transfer costs

✓ OPERATIONS:
├─ Document your VPC architecture
├─ Use consistent naming (prod-vpc, dev-vpc)
├─ Automate with Terraform/CloudFormation
├─ Monitor with VPC Flow Logs
```

---

## 📋 Quick Reference Checklist

Use this when creating a new VPC:

```
VPC CREATION CHECKLIST:
═════════════════════════

□ VPC
  □ Name: prod-vpc
  □ CIDR: 10.0.0.0/16
  □ DNS resolution: Enabled
  □ DNS hostnames: Enabled

□ Internet Gateway
  □ Create IGW
  □ Attach to VPC

□ Public Subnets (per AZ)
  □ Subnet name: public-subnet-1a
  □ CIDR: 10.0.1.0/24
  □ AZ: ap-south-1a
  □ Auto-assign public IP: Enabled

□ Private Subnets (per AZ)
  □ Subnet name: private-subnet-1a
  □ CIDR: 10.0.10.0/24
  □ AZ: ap-south-1a
  □ Auto-assign public IP: Disabled

□ NAT Gateways (if needed)
  □ Allocate Elastic IP
  □ Create NAT Gateway in public subnet (per AZ)
  □ Add route in private route table: 0.0.0.0/0 → NAT

□ Route Tables
  □ Public route table:
     ├─ 10.0.0.0/16 → local
     └─ 0.0.0.0/0 → Internet Gateway
  □ Private route table:
     ├─ 10.0.0.0/16 → local
     └─ 0.0.0.0/0 → NAT Gateway (optional)

□ Security Groups
  □ Web tier: Allow 80/443 from 0.0.0.0/0
  □ App tier: Allow 80/443 from web tier SG only
  □ Database tier: Allow 3306/5432 from app tier SG only

□ VPC Flow Logs (for debugging)
  □ Enable Flow Logs
  □ Send to CloudWatch Logs
  □ Helps with troubleshooting

DEPLOYMENT READY: ✓
```

---

## 🎓 Conclusion

You now understand AWS VPC at a production level.

### Key Takeaways

1. **CIDR notation** — How IP address ranges work (/16 = 65k IPs, /24 = 254 IPs)
2. **VPC architecture** — Isolated network with multiple layers
3. **Subnets** — Divide VPC into public and private sections
4. **Routing** — Route tables decide where traffic goes
5. **Internet Gateway** — Connection to the internet
6. **NAT Gateway** — Allow private subnets to reach internet securely
7. **Security Groups** — Stateful firewall at instance level
8. **Network ACLs** — Stateless firewall at subnet level
9. **VPC Endpoints** — Private AWS service access (cheaper than NAT!)
10. **VPC Peering** — Connect multiple VPCs

### Your Production VPC

With this knowledge, you can design:
- ✅ Highly available multi-tier architecture
- ✅ Secure networks with multiple layers
- ✅ Cost-effective using VPC endpoints
- ✅ Resilient to AZ failures
- ✅ Scalable for growth

### Next Steps

1. **Apply:** Create a VPC following the 3-tier architecture
2. **Test:** Deploy instances in public and private subnets
3. **Troubleshoot:** Use the debugging flowchart if connectivity fails
4. **Optimize:** Replace NAT with VPC endpoints where applicable
5. **Document:** Keep VPC diagram for reference

---

**End of File**

You're ready to build production-grade networks on AWS.

