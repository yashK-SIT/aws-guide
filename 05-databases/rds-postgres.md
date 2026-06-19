# AWS RDS PostgreSQL: Enterprise Database Setup & Operations

## Table of Contents
1. [Introduction and Architecture](#introduction-and-architecture)
2. [RDS Instance Creation](#rds-instance-creation)
3. [PostgreSQL Configuration](#postgresql-configuration)
4. [Connection and Access](#connection-and-access)
5. [Backup and Recovery](#backup-and-recovery)
6. [Multi-AZ High Availability](#multi-az-high-availability)
7. [Read Replicas](#read-replicas)
8. [Performance Tuning](#performance-tuning)
9. [Monitoring and Metrics](#monitoring-and-metrics)
10. [Security](#security)
11. [Scaling Strategies](#scaling-strategies)
12. [Cost Analysis](#cost-analysis)
13. [Troubleshooting](#troubleshooting)

---

## Introduction and Architecture

### What is AWS RDS?

Amazon Relational Database Service (RDS) is a managed database service that automates routine database administrative tasks like backups, patching, and replication. RDS supports PostgreSQL, MySQL, MariaDB, Oracle, and SQL Server.

**Why RDS for PostgreSQL on AWS?**

PostgreSQL on RDS provides:
- **Automated backups**: Daily automated backups with point-in-time recovery up to 35 days
- **High availability**: Multi-AZ deployments with synchronous replication and automatic failover
- **Scaling**: Easy vertical scaling (instance type changes) and horizontal scaling (read replicas)
- **Monitoring**: CloudWatch integration for metrics, logs, and alerting
- **Patching**: Automatic minor version patches and manual major version upgrades
- **Encryption**: Encryption at rest and in transit
- **Parameter groups**: Control PostgreSQL configuration without SSH access
- **Enhanced Monitoring**: Real-time OS metrics (CPU, memory, I/O, network)

### Typical Application Architecture

```
Application Tier (Nginx + App Servers on EC2)
         ↓
    AWS Security Group (allow port 5432)
         ↓
RDS PostgreSQL (Primary)
         ↓
  ├─ Automated Backups (S3)
  ├─ Multi-AZ Standby (synchronous replication)
  └─ Read Replicas (for scaling reads)
```

### RDS Pricing Components

```
1. Instance Pricing: per-instance per-hour
   - db.t3.micro: $0.017/hour (~$12/month)
   - db.t3.small: $0.034/hour (~$24/month)
   - db.t3.medium: $0.068/hour (~$50/month)
   - db.m5.large: $0.227/hour (~$166/month)
   - db.r5.large: $0.352/hour (~$257/month)

2. Storage Pricing: per GB per month
   - General Purpose (gp2): $0.115/GB/month
   - Provisioned IOPS: $0.13/GB/month + $0.10/IOPS/month
   - Magnetic (legacy): $0.10/GB/month

3. Backup Storage: per GB per month
   - Automated backups: Included in backup retention period
   - Manual snapshots: $0.095/GB/month

4. Data Transfer:
   - RDS to EC2 in same AZ: Free
   - RDS to EC2 in different AZ: $0.02/GB
   - RDS to internet: $0.09/GB (outbound)
```

### Backup Strategy Overview

```
Day 1                Day 2               Day 7           Day 35
  |                   |                   |                |
  +---Hour 1 ← Daily automated backup ← Daily backup ← Final backup
              (snapshot + transaction logs)
  
Point-in-time recovery within 35 days:
- Restore any point in last 35 days with second precision
- Based on snapshots + transaction log replay
- Creates new database instance (original unchanged)
```

---

## RDS Instance Creation

### AWS Management Console Setup

**Step 1: Create RDS Instance**

```
1. Navigate to AWS Console → RDS → Databases → Create Database
2. Choose Standard Create (not Easy Create for production)
3. Engine Options:
   - Engine: PostgreSQL
   - Version: 14.x or 15.x (latest stable for production)
   - Edition: PostgreSQL (not Aurora)
4. DB Cluster Identifier: myapp-db
5. Master Username: postgres
6. Master Password: [generate strong password 20+ characters]
   - AWS Secrets Manager integration: Enable (recommended)
```

**Step 2: DB Instance Configuration**

```
1. DB Instance Class:
   - Development: db.t3.micro or db.t3.small
   - Production: db.t3.medium minimum, db.m5.large for serious workloads
   
2. Storage:
   - Type: gp2 (General Purpose SSD)
   - Allocated storage: 20GB minimum for production
   - Enable storage autoscaling: YES (up to 1000GB)
   - IOPS: Automatic (gp2 manages IOPS automatically)

3. Availability & Durability:
   - Multi-AZ deployment: YES (for production)
     → Creates synchronous standby in another AZ
     → Automatic failover in case of failure
```

**Step 3: Connectivity**

```
1. Network & Security:
   - VPC: [Select your application VPC]
   - Subnet group: [Select or create DB subnet group]
   - Publicly accessible: NO (keep database private)
   - VPC Security Group: [Create or select]
   - Database port: 5432 (standard PostgreSQL port)

2. Database Authentication:
   - Enable IAM authentication: YES
     → Allows app to authenticate without storing passwords
     → Uses temporary credentials from IAM

3. Encryption:
   - Encryption at rest: YES
     → Uses AWS KMS key
   - Storage encryption key: [aws/rds]
   - Enable encryption in transit: YES
     → Connections to database are SSL/TLS
```

**Step 4: Database Configuration**

```
1. Database Name: applicationdb
   → Creates initial empty database
   → Application creates tables on first deployment

2. DB Parameter Group: [Select or create]
   → Controls PostgreSQL configuration (max_connections, etc.)
   → Changes take effect with reboot

3. DB Option Group: [Use default]
   → Advanced PostgreSQL extensions controlled here

4. Backup Settings:
   - Backup retention period: 7 days (production minimum)
   - Preferred backup window: 03:00-04:00 UTC
     → Time when backup doesn't impact production
   - Backup encryption: Use database encryption
```

**Step 5: Monitoring & Maintenance**

```
1. CloudWatch Monitoring:
   - Enable CloudWatch Logs Export: true
     → Logs types: postgresql (general logs)
   - Monitoring Granularity: 60 seconds

2. Backup & Maintenance:
   - Enable automated minor version upgrades: YES
   - Maintenance window: sun:04:00-sun:05:00 UTC
     → Window for patches, minor version upgrades

3. Enable deletion protection: YES (for production)
   → Prevents accidental deletion
```

### AWS CLI Creation

For infrastructure-as-code deployment:

```bash
#!/bin/bash

# Create RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.2 \
  --master-username postgres \
  --master-user-password 'YourRandomPassword12345!' \
  --allocated-storage 20 \
  --storage-type gp2 \
  --storage-encrypted \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --db-subnet-group-name myapp-db-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxx \
  --database-name applicationdb \
  --publicly-accessible false \
  --multi-az \
  --enable-cloudwatch-logs-exports postgresql \
  --delete-protection \
  --region us-east-1

# Wait for instance to be available (5-10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier myapp-db \
  --region us-east-1

echo "RDS instance created successfully"
```

### DB Subnet Group Configuration

RDS instances must be placed in a DB subnet group (multiple AZs):

```bash
# Create DB subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name myapp-db-subnet-group \
  --db-subnet-group-description "Subnet group for myapp database" \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy \
  --region us-east-1

# List subnet groups
aws rds describe-db-subnet-groups

# Output includes:
# DBSubnetGroupName: myapp-db-subnet-group
# Subnets: [two private subnets in different AZs]
```

### Security Group Configuration

Allow application to connect to database:

```bash
# Get VPC ID of your application security group
APP_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=app-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

# Get RDS security group ID
DB_SG_ID=$(aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" \
  --output text)

# Allow inbound PostgreSQL traffic from app to DB
aws ec2 authorize-security-group-ingress \
  --group-id "$DB_SG_ID" \
  --protocol tcp \
  --port 5432 \
  --source-security-group-id "$APP_SG_ID"

# Verify rule
aws ec2 describe-security-groups --group-ids "$DB_SG_ID"
```

---

## PostgreSQL Configuration

### Parameter Groups

Control PostgreSQL configuration without SSH access:

```bash
# Create custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name myapp-postgres-params \
  --db-parameter-group-family postgres15 \
  --description "Custom parameters for myapp" \
  --region us-east-1

# List available parameters
aws rds describe-db-parameters \
  --db-parameter-group-name myapp-postgres-params | head -50

# Common production parameters
```

**Important PostgreSQL Parameters**

```bash
# Function: Modify parameter (requires reboot on some parameters)
aws rds modify-db-parameter-group-parameters \
  --db-parameter-group-name myapp-postgres-params \
  --parameters "[
    {
      \"ParameterName\": \"max_connections\",
      \"ParameterValue\": \"500\",
      \"ApplyMethod\": \"immediate\"
    },
    {
      \"ParameterName\": \"shared_buffers\",
      \"ParameterValue\": \"262144\",
      \"ApplyMethod\": \"pending-reboot\"
    },
    {
      \"ParameterName\": \"effective_cache_size\",
      \"ParameterValue\": \"1048576\",
      \"ApplyMethod\": \"immediate\"
    },
    {
      \"ParameterName\": \"maintenance_work_mem\",
      \"ParameterValue\": \"65536\",
      \"ApplyMethod\": \"immediate\"
    }
  ]"
```

**Parameter Explanations**

```
max_connections (default: 100)
- Maximum concurrent connections to database
- Production: 200-500 (more if connection pooling)
- Too low: application gets "too many connections" error
- Too high: wastes memory
- Recommended formula: (RAM_GB × 2) + 250

shared_buffers (default: 25% of RAM)
- PostgreSQL buffer cache (similar to InnoDB buffer pool)
- Caches frequently accessed data in memory
- Production: 20-25% of instance RAM
- db.t3.medium (4GB) → 1GB (262144 × 8KB pages)

effective_cache_size (default: 50% of RAM)
- Tells query planner how much cache is available
- Influences query optimization decisions
- Should be ~50% of available system RAM

maintenance_work_mem (default: 64MB)
- Memory for maintenance operations (CREATE INDEX, ALTER TABLE)
- Production: 256-1024MB for faster index creation
- Doesn't impact regular queries, only maintenance

log_min_duration_statement (default: -1, disabled)
- Log queries taking longer than N milliseconds
- Production: 1000 (log queries > 1 second)
- Helps identify slow queries
```

### Application Database and User Setup

Create application database and user (cannot do via parameter groups):

```bash
# Connect to database from EC2 instance
# First, install PostgreSQL client
sudo yum install -y postgresql  # Amazon Linux 2
# or
sudo apt-get install -y postgresql-client  # Ubuntu

# Get RDS endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].Endpoint.Address" \
  --output text)

# Connect to database with master user
PGPASSWORD='YourPassword123!' psql -h "$RDS_ENDPOINT" -U postgres -d postgres << EOF

-- Create application database
CREATE DATABASE myapp_production;

-- Create application user (non-admin)
CREATE USER appuser WITH PASSWORD 'ApplicationPassword456!';

-- Grant privileges to user on database
GRANT CONNECT ON DATABASE myapp_production TO appuser;

-- Connect to application database
\c myapp_production

-- Grant usage on schema
GRANT USAGE ON SCHEMA public TO appuser;

-- Grant permissions on all tables
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;

-- Grant default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;

-- Grant sequence privileges (for auto-increment IDs)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO appuser;

EOF

echo "Database and user created successfully"
```

### Connection Pooling Configuration

PostgreSQL has limited connections (default 100), so use PgBouncer for connection pooling:

```bash
# Install PgBouncer on application EC2 instance
sudo yum install -y pgbouncer

# Create pgbouncer configuration
sudo tee /etc/pgbouncer/pgbouncer.ini > /dev/null << 'EOF'
[databases]
; Format: dbname = host=rds-endpoint port=5432 dbname=myapp_production
myapp = host=myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com port=5432 dbname=myapp_production user=appuser password=ApplicationPassword456!

[pgbouncer]
; Listen on localhost:6432
listen_port = 6432
listen_addr = 127.0.0.1

; Connection pooling mode
pool_mode = transaction  ; Each transaction gets a connection
; Alternatives:
; session = Connection per client session (more connections)
; statement = Connection per statement (best for pooling)

; Connection pool settings
max_client_conn = 1000        ; Max connections from applications
default_pool_size = 25        ; Connections per database
min_pool_size = 5             ; Minimum reserved connections
reserve_pool_size = 5         ; Emergency reserve connections
reserve_pool_timeout = 3

; Query timeouts
client_idle_timeout = 600     ; Disconnect idle clients after 10 min
client_login_timeout = 10
server_idle_timeout = 600
server_lifetime = 3600

; Logging
logfile = /var/log/pgbouncer/pgbouncer.log
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

; Daemon mode
daemonize = 1
pidfile = /var/run/pgbouncer/pgbouncer.pid
EOF

# Create pgbouncer user and set permissions
sudo useradd pgbouncer 2>/dev/null || true
sudo chown pgbouncer:pgbouncer /etc/pgbouncer/pgbouncer.ini
sudo chmod 600 /etc/pgbouncer/pgbouncer.ini

# Create log directory
sudo mkdir -p /var/log/pgbouncer
sudo chown pgbouncer:pgbouncer /var/log/pgbouncer

# Start pgbouncer
sudo systemctl start pgbouncer
sudo systemctl enable pgbouncer

# Test connection through pool
psql -h 127.0.0.1 -p 6432 -U appuser -d myapp << EOF
SELECT 1;
EOF

echo "Connection pooling configured"
```

Connection pooling benefits:
- RDS `max_connections=500` → 500 concurrent connections
- Without pooling: 1 connection per application, max ~10 apps
- With pooling: 500 connections shared across all applications
- Dramatically increases connection reuse

---

## Connection and Access

### Application Connection String

For Node.js application (using `pg` driver):

```javascript
// connection.js
const { Pool } = require('pg');

const pool = new Pool({
  // Connect through pgbouncer (on localhost:6432)
  host: 'localhost',
  port: 6432,
  database: 'myapp_production',
  user: 'appuser',
  password: process.env.DB_PASSWORD,  // From environment variable
  
  // Connection pool settings
  max: 20,                // Max connections in pool
  idleTimeoutMillis: 30000,  // Close idle after 30s
  connectionTimeoutMillis: 5000, // Timeout connection after 5s
});

// Query function
async function query(text, params) {
  const start = Date.now();
  try {
    const res = await pool.query(text, params);
    const duration = Date.now() - start;
    console.log('Executed query', { text, duration, rows: res.rowCount });
    return res;
  } catch (error) {
    console.error('Query error', { text, error });
    throw error;
  }
}

module.exports = { pool, query };
```

For Python application (using `psycopg2`):

```python
# database.py
import psycopg2
import psycopg2.pool
import os

# Create connection pool
connection_pool = psycopg2.pool.SimpleConnectionPool(
    1,  # Minimum connections
    20,  # Maximum connections
    host='localhost',
    port=6432,
    database='myapp_production',
    user='appuser',
    password=os.environ.get('DB_PASSWORD'),
    connect_timeout=5,
)

def get_connection():
    """Get connection from pool"""
    return connection_pool.getconn()

def return_connection(conn):
    """Return connection to pool"""
    connection_pool.putconn(conn)

def query(sql, params=None):
    """Execute query"""
    conn = get_connection()
    try:
        with conn.cursor() as cur:
            cur.execute(sql, params)
            if cur.description:  # SELECT query
                return cur.fetchall()
            else:  # INSERT/UPDATE/DELETE
                conn.commit()
                return cur.rowcount
    finally:
        return_connection(conn)
```

### IAM Database Authentication

Use temporary credentials instead of storing passwords:

```bash
# 1. Create IAM database authentication token (valid 15 minutes)
TOKEN=$(aws rds generate-db-auth-token \
  --hostname myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --region us-east-1 \
  --username appuser)

# 2. Connect using token (instead of password)
psql -h myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com \
  -U appuser \
  -d myapp_production \
  --password <<< "$TOKEN" \
  -c "SELECT 1;"

# 3. For applications: Generate token programmatically
```

Node.js with IAM authentication:

```javascript
// iam-auth.js
const AWS = require('aws-sdk');
const { Pool } = require('pg');

const signer = new AWS.RDS.Signer({
  region: 'us-east-1',
  hostname: 'myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com',
  port: 5432,
  username: 'appuser',
});

const pool = new Pool({
  host: 'myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com',
  port: 5432,
  database: 'myapp_production',
  user: 'appuser',
  password: signer.getAuthToken({
    username: 'appuser',
  }),
  ssl: 'require',
  max: 20,
});

// Token refreshes automatically every 15 minutes
```

Benefits of IAM auth:
- No passwords in environment variables
- Credentials rotate every 15 minutes
- Audit trail in CloudTrail
- Can revoke access immediately by removing IAM policy

### Troubleshooting Connection Issues

```bash
# Test connectivity from EC2 instance
telnet myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com 5432

# If fails:
# 1. Check security group allows port 5432 from app instance
# 2. Check DB instance is in "available" state
# 3. Check application is using correct password/IAM token

# Check RDS instance status
aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].DBInstanceStatus"
# Should return: "available"

# Check security group rules
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxx \
  --query "SecurityGroups[0].IpPermissions"

# If using IAM auth, ensure EC2 instance role has permission
aws iam get-role-policy \
  --role-name ec2-instance-role \
  --policy-name rds-auth-policy
```

---

## Backup and Recovery

### Automated Backups

RDS creates daily automated backups automatically:

```bash
# Check backup configuration
aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].[BackupRetentionPeriod,PreferredBackupWindow]"

# Output: [7, "03:00-04:00"]
# Means: Keep 7 days of backups, backup window 03:00-04:00 UTC

# List automated backups
aws rds describe-db-snapshots \
  --db-instance-identifier myapp-db \
  --snapshot-type automated \
  --query "DBSnapshots[].{SnapshotId:DBSnapshotIdentifier,CreatedTime:SnapshotCreateTime,Size:AllocatedStorage}"

# Output:
# SnapshotId: rds:myapp-db-2026-05-14-03-01
# CreatedTime: 2026-05-14T03:01:00Z
# Size: 20 (GB)
```

**How Automated Backups Work**

```
Daily Backup Process:
1. Full snapshot at 03:00 UTC each day (storage-intensive)
2. Transaction logs captured between snapshots
3. Point-in-time recovery via log replay

To restore to specific point:
1. Start from daily snapshot closest to desired time
2. Replay transaction logs up to desired point
3. Recovery process transparent to user (creates new DB instance)

Example: Recover to 2026-05-13 at 14:30:25 UTC
- Snapshot taken: 2026-05-13 at 03:00 UTC
- Restore from snapshot: Get DB state at 03:00 UTC
- Replay logs: Apply all transactions from 03:00 to 14:30:25
- Result: DB instance at exactly 2026-05-13 14:30:25 UTC
```

### Point-in-Time Recovery

Restore database to any point within backup retention period:

```bash
# Restore to specific point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier myapp-db \
  --target-db-instance-identifier myapp-db-recovered \
  --restore-time 2026-05-13T14:30:25Z \
  --region us-east-1

# Restore to latest restorable time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier myapp-db \
  --target-db-instance-identifier myapp-db-recovered \
  --use-latest-restorable-time \
  --region us-east-1

# Monitor restore progress
aws rds describe-db-instances \
  --db-instance-identifier myapp-db-recovered \
  --query "DBInstances[0].DBInstanceStatus"
# Output: creating → backing-up → available (takes 5-10 minutes)

# After recovery succeeds:
# 1. Verify recovered data
# 2. Test application connection
# 3. Update connection string to recovered instance
# 4. Delete original instance if corruption confirmed
# 5. Rename recovered instance back to original name
```

### Manual Snapshots

Create manual snapshots for long-term retention:

```bash
# Create manual snapshot before major changes
aws rds create-db-snapshot \
  --db-instance-identifier myapp-db \
  --db-snapshot-identifier myapp-db-pre-upgrade-2026-05-14 \
  --region us-east-1

# Monitor snapshot creation
aws rds describe-db-snapshots \
  --db-snapshot-identifier myapp-db-pre-upgrade-2026-05-14 \
  --query "DBSnapshots[0].[Status,SnapshotCreateTime,AllocatedStorage]"

# Snapshots retained until manually deleted
# Cost: $0.095/GB/month for snapshot storage

# Restore from manual snapshot (point-in-time recovery)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier myapp-db-restored \
  --db-snapshot-identifier myapp-db-pre-upgrade-2026-05-14 \
  --region us-east-1

# List all snapshots (manual + automated)
aws rds describe-db-snapshots \
  --db-instance-identifier myapp-db

# Delete manual snapshot (not needed anymore)
aws rds delete-db-snapshot \
  --db-snapshot-identifier myapp-db-pre-upgrade-2026-05-14
```

### Snapshot to S3 Export

Export database to S3 for long-term archival or analysis:

```bash
# Export snapshot to Parquet files in S3
aws rds start-export-task \
  --export-task-identifier myapp-db-export-2026-05-14 \
  --source-arn arn:aws:rds:us-east-1:123456789012:snapshot:myapp-db-snapshot-id \
  --s3-bucket-name my-backup-bucket \
  --s3-prefix "database-exports/" \
  --iam-role-arn arn:aws:iam::123456789012:role/rds-export-role \
  --export-only myapp_production \
  --region us-east-1

# Monitor export progress
aws rds describe-export-tasks \
  --export-task-identifier myapp-db-export-2026-05-14 \
  --query "ExportTasks[0].[Status,PercentProgress,SnapshotTime]"

# Once complete, files in S3: s3://my-backup-bucket/database-exports/
# Can analyze with Athena SQL queries
```

### Backup Best Practices

**Backup Configuration**

```bash
# Recommended settings for production
aws rds modify-db-instance \
  --db-instance-identifier myapp-db \
  --backup-retention-period 30 \
  --preferred-backup-window "03:00-04:00" \
  --copy-tags-to-snapshot \
  --delete-automated-backups false \
  --apply-immediately false \
  --region us-east-1
```

**Backup Retention Policy**

```
Development:
- Backup retention: 7 days (automatic cleanup)
- Cost: ~$2-3/month for 20GB database

Staging:
- Backup retention: 14 days
- Cost: ~$4-6/month for 20GB database

Production:
- Backup retention: 30 days (regulatory requirement often)
- Manual snapshots: Keep 1 monthly for 1 year
- Cost: ~$9-12/month for 20GB database

Formula: Backup storage cost = (Daily backup size) × (retention days) × ($0.095/GB/month) ÷ 30 days

For 20GB database with 30-day retention:
(20GB) × (30) × ($0.095) ÷ 30 = ~$19/month for backup storage
```

**Backup Testing**

```bash
#!/bin/bash
# test-backup.sh - Verify backups work by restoring periodically

# Every Monday: Restore from latest backup to test DB
RESTORE_DB="myapp-db-test"

# Get latest automated snapshot
SNAPSHOT=$(aws rds describe-db-snapshots \
  --db-instance-identifier myapp-db \
  --snapshot-type automated \
  --query "sort_by(DBSnapshots, &SnapshotCreateTime)[-1].DBSnapshotIdentifier" \
  --output text)

# Delete existing test DB if present
aws rds delete-db-instance \
  --db-instance-identifier "$RESTORE_DB" \
  --skip-final-snapshot 2>/dev/null || true

sleep 10

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier "$RESTORE_DB" \
  --db-snapshot-identifier "$SNAPSHOT" \
  --region us-east-1

# Wait for restore to complete (10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier "$RESTORE_DB" \
  --region us-east-1

# Test connectivity and basic queries
ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier "$RESTORE_DB" \
  --query "DBInstances[0].Endpoint.Address" \
  --output text)

PGPASSWORD="$DB_PASSWORD" psql -h "$ENDPOINT" -U appuser -d myapp_production << EOF
-- Test basic queries
SELECT count(*) FROM information_schema.tables WHERE table_schema='public';
SELECT version();
EOF

echo "Backup test completed successfully"
```

---

## Multi-AZ High Availability

Multi-AZ deployments maintain a synchronous standby replica in a different Availability Zone with automatic failover.

### How Multi-AZ Works

```
Primary AZ                          Standby AZ
┌─────────────────────┐            ┌──────────────────┐
│ myapp-db (Primary)  │            │ myapp-db         │
│                     │            │ (Standby Replica)│
│ - Accepts writes    │            │ - Synchronous    │
│ - Accepts reads     │            │ - Read-only      │
│                     │            │                  │
│ Transaction Log ←───┼────────────→ Replay logs      │
│ Commits to standby  │   TCP       │                  │
└─────────────────────┘            └──────────────────┘
         ↑
    Application connects to single endpoint:
    myapp-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com
    
    If primary fails:
    1. RDS detects failure (90 seconds)
    2. Promotion vote among cluster
    3. Standby promoted to primary
    4. DNS updated (30-60 seconds)
    5. Application connection drops + retries
    6. Application reconnects to new primary
    
    Estimated failover time: 2-3 minutes with no data loss
```

### Enable Multi-AZ

```bash
# Create with Multi-AZ enabled
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --multi-az \
  ...

# Or convert existing single-AZ to Multi-AZ
aws rds modify-db-instance \
  --db-instance-identifier myapp-db \
  --multi-az \
  --apply-immediately false
  # apply-immediately: false = apply during next maintenance window

# Verify Multi-AZ status
aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].[MultiAZ,DBInstanceStatus,SecondaryAvailabilityZone]"

# Output:
# [true, "available", "us-east-1b"]
```

### Manual Failover Testing

Test failover without losing data:

```bash
# Initiate manual failover
aws rds reboot-db-instance \
  --db-instance-identifier myapp-db \
  --force-failover

# Monitor failover progress
watch -n 10 'aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].[DBInstanceStatus,AvailabilityZone,SecondaryAvailabilityZone]"'

# Expected output during failover:
# [rebooting, us-east-1a, us-east-1b]
# → 
# [available, us-east-1b, us-east-1a]  # Roles switched!

# Check application logs for connection drops
tail -50 /var/log/app.log | grep -i "connection\|error"
```

### Cost Implications

```
Multi-AZ pricing:
- Primary instance: db.t3.medium = $50/month
- Standby replica: db.t3.medium = $0 (standby doesn't cost)
- Bandwidth (AZ-to-AZ replication): $0.02/GB

Total: ~$50/month (vs $25/month for single-AZ)

ROI Analysis:
- Database downtime cost: $10,000/hour (estimated for e-commerce)
- Failover prevents: ~4 hours of downtime/year
- Cost prevented: $40,000/year
- Multi-AZ cost increase: $300/year
- ROI: 13,300% (extremely worth it for production)
```

---

## Read Replicas

Read replicas scale database read workload across multiple instances. Unlike Multi-AZ standbys (which don't serve queries), read replicas accept SELECT queries.

### When to Use Read Replicas

**Scenario 1: Read-Heavy Application**

```
Without read replicas:
- 1000 total queries/second
- 900 read queries/second (90%)
- 100 write queries/second (10%)
- Single db.m5.xlarge handles workload: $2.67/hour

With read replicas:
- Primary: write queries only (100/sec)
- Read Replica 1: read queries (300/sec)
- Read Replica 2: read queries (300/sec)
- Read Replica 3: read queries (300/sec)
- Three t3.large instances: $0.30/hour each = $0.90/hour
- Much cheaper + better isolation
```

**Scenario 2: Reporting without impacting transactions**

```
Production use case:
- Application queries: 95% small, fast queries
- Reporting: 1-2 expensive queries/hour
- Reports scan large tables, use lots of CPU

Solution:
- Primary DB: Application transactions
- Read Replica: Run reporting queries
- Reporting can't affect application performance
```

### Creating Read Replicas

```bash
# Create read replica (same region, different AZ)
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-db-read-1 \
  --source-db-instance-identifier myapp-db \
  --db-instance-class db.t3.medium \
  --storage-type gp2 \
  --region us-east-1

# Create read replica in different region (for disaster recovery)
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-db-read-dr \
  --source-db-instance-identifier myapp-db \
  --db-instance-class db.t3.medium \
  --source-region us-east-1 \
  --region us-west-2

# Monitor replica creation (takes 5-10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier myapp-db-read-1 \
  --region us-east-1

# Get read replica endpoint
aws rds describe-db-instances \
  --db-instance-identifier myapp-db-read-1 \
  --query "DBInstances[0].Endpoint.Address"
# Output: myapp-db-read-1.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com
```

### Application Configuration for Read Replicas

**Strategy 1: Read Replica for Specific Queries**

```javascript
// connection.js
const { Pool } = require('pg');

const primaryPool = new Pool({
  host: 'myapp-db.xxx.us-east-1.rds.amazonaws.com',
  database: 'myapp_production',
  user: 'appuser',
  max: 20,
});

const replicaPool = new Pool({
  host: 'myapp-db-read-1.xxx.us-east-1.rds.amazonaws.com',
  database: 'myapp_production',
  user: 'appuser',
  max: 20,
});

// Use primary for writes
async function insertUser(name, email) {
  return primaryPool.query(
    'INSERT INTO users (name, email) VALUES ($1, $2)',
    [name, email]
  );
}

// Use replica for heavy reads
async function generateReport() {
  const result = await replicaPool.query(
    'SELECT DATE_TRUNC(\'month\', created_at) as month, COUNT(*) as count ' +
    'FROM users GROUP BY month ORDER BY month'
  );
  return result.rows;
}

// Use primary for fast reads (consistency important)
async function getUser(userId) {
  return primaryPool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
  );
}

module.exports = { insertUser, generateReport, getUser };
```

**Strategy 2: Write Primary, Read Replica (with eventual consistency)**

```javascript
// For eventual consistency applications
// (like analytics, recommendations, caches)

async function getUserStats(userId) {
  // Small delay after write to allow replication
  const stats = await replicaPool.query(
    'SELECT COUNT(*) as total_orders FROM orders WHERE user_id = $1',
    [userId]
  );
  return stats.rows[0];
}

async function createOrder(userId, items) {
  // Write to primary
  const result = await primaryPool.query(
    'INSERT INTO orders (user_id, items) VALUES ($1, $2) RETURNING id',
    [userId, JSON.stringify(items)]
  );
  
  // For next request, stats come from replica
  // (may be 100ms-500ms behind, which is acceptable for analytics)
  return result.rows[0];
}
```

### Read Replica Replication Lag

Track replication delay:

```bash
# Check replica lag
aws rds describe-db-instances \
  --db-instance-identifier myapp-db-read-1 \
  --query "DBInstances[0].ReplicationLag"
# Output: 0 (seconds behind primary)

# Monitor lag continuously
watch -n 1 'aws rds describe-db-instances \
  --db-instance-identifier myapp-db-read-1 \
  --query "DBInstances[0].ReplicationLag"'

# If lag increases:
# 1. Check primary write volume
# 2. Check replica instance class (may need upgrade)
# 3. Check network connectivity between instances
```

During heavy writes, replication lag increases (usually < 100ms):

```
Write Volume vs Lag:

5000 writes/second
└─ Lag: ~50ms (normal)

10000 writes/second
└─ Lag: ~100ms (acceptable)

20000 writes/second
└─ Lag: ~500ms (concerning)
└─ Solution: Upgrade replica instance class

50000 writes/second
└─ Lag: ~2 seconds (problematic)
└─ Solution: Add more read replicas, spread load
```

### Promoting Read Replica to Primary

Convert read replica to standalone database (for failover or split workload):

```bash
# Promote read replica to standalone instance
aws rds promote-read-replica \
  --db-instance-identifier myapp-db-read-1

# This:
# 1. Stops replication from primary
# 2. Makes it a standalone database
# 3. Enables backups and Multi-AZ
# 4. Allows writes
# 5. Breaks connection to primary (irreversible)

# Monitor promotion
aws rds wait db-instance-available \
  --db-instance-identifier myapp-db-read-1

# Update application to point to new endpoint if needed
```

Promotion use cases:
- **Disaster recovery**: If primary fails, promote regional read replica
- **Production split**: Run analytics workload on separate database
- **Development**: Clone production data to dev environment

### Read Replica Costs

```
Primary database: db.m5.large = $0.227/hour = $166/month

Read Replicas:
- Same region, different AZ: db.t3.medium = $0.068/hour = $50/month each
- Different region: db.t3.medium = $0.068/hour + ~$15 data transfer/month

3 Read Replicas (same region):
- Cost: 3 × $50 = $150/month
- Total: $166 (primary) + $150 (replicas) = $316/month
- ROI: 100x scaling at 2x cost (excellent value)

Cost optimization:
- Use smaller instance class for replicas (t3.small if possible)
- Use Spot instances for analytics replicas (70% savings, interruption acceptable)
- Delete replicas when not needed (e.g., daily reporting replica)
```

---

## Performance Tuning

### Query Performance Analysis

Identify slow queries using PostgreSQL logs:

```bash
# Enable slow query logging (1+ second queries)
aws rds modify-db-parameter-group-parameters \
  --db-parameter-group-name myapp-postgres-params \
  --parameters "[{
    \"ParameterName\": \"log_min_duration_statement\",
    \"ParameterValue\": \"1000\",
    \"ApplyMethod\": \"immediate\"
  }]"

# Connect to database and view slow queries
PGPASSWORD="$DB_PASSWORD" psql -h myapp-db.xxx.us-east-1.rds.amazonaws.com \
  -U appuser -d myapp_production << EOF
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
WHERE mean_time > 1000  -- Queries averaging > 1 second
ORDER BY mean_time DESC
LIMIT 10;

-- Output:
-- query                                          | calls | total_time | mean_time
-- SELECT * FROM users WHERE email = $1           | 10000 | 85000     | 8.5
-- SELECT * FROM orders WHERE user_id = $1        | 5000  | 35000     | 7.0
EOF
```

### Index Strategy

Proper indexing dramatically improves query performance:

```bash
# Connect and view table structure
PGPASSWORD="$DB_PASSWORD" psql -h myapp-db.xxx.us-east-1.rds.amazonaws.com \
  -U appuser -d myapp_production << EOF

-- View table structure
\d users

-- Check existing indexes
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE tablename='users'
ORDER BY tablename;

-- Create indexes for common queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite index for frequently used together columns
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- Monitor index size and usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

EOF
```

### Query Optimization Example

```sql
-- BEFORE: N+1 query (slow)
-- Find all users and their recent orders
SELECT id, email FROM users;
-- (in application code, loop through users)
SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 5;
-- (runs N times, N = number of users)

-- AFTER: Single query with JOIN (fast)
SELECT 
  u.id, 
  u.email, 
  json_agg(
    json_build_object('id', o.id, 'total', o.total)
    ORDER BY o.created_at DESC
  ) FILTER (WHERE o.id IS NOT NULL) as recent_orders
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.email
LIMIT 100;

-- Explain plan to verify performance
EXPLAIN ANALYZE SELECT ...;

-- Output shows:
-- Seq Scan on users  (too slow, need index)
-- vs
-- Index Scan on idx_users_email (fast)
```

### Connection Pool Tuning

```bash
# Monitor connection pool usage
PGPASSWORD="$DB_PASSWORD" psql -h myapp-db.xxx.us-east-1.rds.amazonaws.com \
  -U postgres -d postgres << EOF

-- View current connections
SELECT pid, usename, application_name, state
FROM pg_stat_activity
WHERE datname = 'myapp_production'
ORDER BY query_start DESC;

-- Count connections by state
SELECT state, COUNT(*) as count
FROM pg_stat_activity
WHERE datname = 'myapp_production'
GROUP BY state;

EOF

# If connection pool is exhausted:
# 1. Increase pgbouncer pool_size
# 2. Increase RDS max_connections parameter
# 3. Reduce connection timeout (close idle connections faster)
# 4. Add more application instances with their own pools
```

### Memory and Cache Configuration

```bash
# Optimal configuration for db.m5.large (8 GB RAM)
aws rds modify-db-parameter-group-parameters \
  --db-parameter-group-name myapp-postgres-params \
  --parameters "[
    {
      \"ParameterName\": \"shared_buffers\",
      \"ParameterValue\": \"2097152\",
      \"ApplyMethod\": \"pending-reboot\"
    },
    {
      \"ParameterName\": \"effective_cache_size\",
      \"ParameterValue\": \"4194304\",
      \"ApplyMethod\": \"immediate\"
    },
    {
      \"ParameterName\": \"maintenance_work_mem\",
      \"ParameterValue\": \"524288\",
      \"ApplyMethod\": \"immediate\"
    },
    {
      \"ParameterName\": \"work_mem\",
      \"ParameterValue\": \"16384\",
      \"ApplyMethod\": \"immediate\"
    }
  ]"

# Reboot to apply pending-reboot changes
aws rds reboot-db-instance --db-instance-identifier myapp-db
```

---

## Monitoring and Metrics

### CloudWatch Metrics

Monitor database health in real-time:

```bash
# Get CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-db \
  --start-time 2026-05-14T00:00:00Z \
  --end-time 2026-05-14T23:59:59Z \
  --period 3600 \
  --statistics Average,Maximum

# Get database connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-db \
  --start-time 2026-05-14T00:00:00Z \
  --end-time 2026-05-14T23:59:59Z \
  --period 3600 \
  --statistics Average,Maximum

# Important metrics to monitor:
# - CPUUtilization: Should be < 80% average
# - DatabaseConnections: Monitor trends
# - DiskQueueDepth: I/O wait queue
# - NetworkReceiveThroughput: Incoming data
# - NetworkTransmitThroughput: Outgoing data
# - ReadLatency: Database read response time
# - WriteLatency: Database write response time
# - ReadThroughput: MB/sec read
# - WriteThroughput: MB/sec write
```

### CloudWatch Alarms

Alert on abnormal conditions:

```bash
# Alarm: High CPU utilization (> 80% for 5 minutes)
aws cloudwatch put-metric-alarm \
  --alarm-name myapp-db-cpu-high \
  --alarm-description "Alert if RDS CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-db \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:database-alerts

# Alarm: Database connections too high
aws cloudwatch put-metric-alarm \
  --alarm-name myapp-db-connections-high \
  --alarm-description "Alert if connections > 400" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 400 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-db \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:database-alerts

# Alarm: Disk queue depth (I/O bottleneck)
aws cloudwatch put-metric-alarm \
  --alarm-name myapp-db-io-high \
  --alarm-description "Alert if disk queue > 2" \
  --metric-name DiskQueueDepth \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 2 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-db \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:database-alerts
```

### Custom Dashboards

```bash
# Create dashboard with key metrics
aws cloudwatch put-dashboard \
  --dashboard-name MyAppDatabase \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/RDS", "CPUUtilization", {"stat": "Average"}],
            [".", "DatabaseConnections", {"stat": "Average"}],
            [".", "ReadLatency", {"stat": "Average"}],
            [".", "WriteLatency", {"stat": "Average"}],
            [".", "DiskQueueDepth", {"stat": "Average"}]
          ],
          "period": 300,
          "stat": "Average",
          "region": "us-east-1",
          "title": "Database Performance"
        }
      }
    ]
  }'
```

### Enhanced Monitoring

Get OS-level metrics (CPU, memory, I/O):

```bash
# Enable Enhanced Monitoring
aws rds modify-db-instance \
  --db-instance-identifier myapp-db \
  --enable-cloudwatch-logs-exports postgresql \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role

# View enhanced monitoring data in CloudWatch Logs
# Log group: /aws/rds/instance/myapp-db/instance
# Streams show OS-level metrics per second
```

---

## Security

### Authentication and Authorization

```bash
# IAM authentication (recommended for EC2)
# EC2 role needs policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds:us-east-1:123456789012:db:myapp-db"
    }
  ]
}

# Password authentication (for CI/CD, external access)
# Store password in AWS Secrets Manager
aws secretsmanager create-secret \
  --name rds/myapp/db-password \
  --secret-string '{"username":"appuser","password":"RandomPassword123!"}'

# Application retrieves password from Secrets Manager
# (automatic rotation every 30 days possible)
```

### VPC Security

Isolate database in private subnets:

```bash
# Database security group: Allow access only from app
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \  # Database security group
  --protocol tcp \
  --port 5432 \
  --source-security-group-id sg-yyyyyyyy  # App security group

# Do NOT allow:
# - 0.0.0.0/0 (internet)
# - Any public access

# Remove publicly accessible flag
aws rds modify-db-instance \
  --db-instance-identifier myapp-db \
  --no-publicly-accessible
```

### Encryption

Ensure encryption enabled:

```bash
# At-rest encryption (required for production)
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012

# In-transit encryption (SSL)
# All connections to RDS use SSL by default
# Force SSL-only connections:

PGPASSWORD="$DB_PASSWORD" psql -h myapp-db.xxx.us-east-1.rds.amazonaws.com \
  -U postgres -d postgres << EOF

-- Require SSL for all connections
ALTER SYSTEM SET ssl = on;
SELECT pg_reload_conf();

EOF
```

### Audit Logging

Track administrative changes:

```bash
# Enable PostgreSQL audit logging
aws rds modify-db-parameter-group-parameters \
  --db-parameter-group-name myapp-postgres-params \
  --parameters "[
    {
      \"ParameterName\": \"pgaudit.log\",
      \"ParameterValue\": \"ALL\",
      \"ApplyMethod\": \"pending-reboot\"
    }
  ]"

# View audit logs
PGPASSWORD="$DB_PASSWORD" psql -h myapp-db.xxx.us-east-1.rds.amazonaws.com \
  -U postgres -d postgres << EOF
SELECT * FROM pg_audit_log;
EOF
```

---

## Scaling Strategies

### Vertical Scaling (Bigger Instance)

Upgrade instance class for more CPU/RAM:

```bash
# Modify instance class (causes brief downtime with Multi-AZ)
aws rds modify-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.m5.xlarge \
  --apply-immediately false \
  --multi-az  # No downtime if already Multi-AZ

# With Multi-AZ:
# 1. Apply changes to standby first
# 2. Failover to standby (now new class)
# 3. Primary becomes standby and gets upgraded
# 4. Total downtime: 1-2 minutes
```

When to scale vertically:
- CPU utilization consistently > 80%
- Memory usage > 85%
- Need better query performance for specific application

### Horizontal Scaling (Read Replicas)

Add read replicas for read-heavy workloads:

```bash
# Create read replica for analytics workload
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-db-analytics \
  --source-db-instance-identifier myapp-db \
  --db-instance-class db.t3.large

# Application reads analytics queries from replica
# Primary focuses on transactional writes
```

### Sharding (Application-Level)

For extremely large databases, shard by customer or region:

```
Unsharded: 1TB database
Queries/second: 5000
Problem: Single database bottleneck

Sharded across 4 instances (by customer ID):
- Shard 1: Customers A-D (250GB)
- Shard 2: Customers E-H (250GB)
- Shard 3: Customers I-L (250GB)
- Shard 4: Customers M-Z (250GB)

Queries/second: 5000 ÷ 4 = 1250 per shard
Much easier to manage, scale independently

Tradeoff: Application complexity for cross-shard queries
```

---

## Cost Analysis

### Cost Breakdown

```
Small Production Database (db.t3.medium, 20GB):

Instance: $50/month
  └─ db.t3.medium: $0.068/hour × 730 hours

Storage: $2.30/month
  └─ 20GB × $0.115/GB/month

Backups: $1.90/month
  └─ 7 days retention ÷ 30 days × 20GB × $0.095/GB/month

Data Transfer: $0-10/month
  └─ Usually free (same AZ), or ~$0.02/GB if cross-AZ

Total: ~$55-65/month

Medium Production Database (db.m5.large, 100GB):

Instance: $166/month
  └─ db.m5.large: $0.227/hour × 730 hours

Storage: $11.50/month
  └─ 100GB × $0.115/GB/month

Backups: $9.50/month
  └─ 30 days retention ÷ 30 days × 100GB × $0.095/GB/month

With 2 Read Replicas (db.t3.large each): +$100/month
  └─ 2 × $50/month

Total: ~$287/month
```

### Cost Optimization

**1. Right-Size Instance**

```
Over-provisioned: db.m5.2xlarge = $0.908/hour = $663/month
- Using 20% of capacity
- Cost: $663/month

Right-sized: db.t3.medium = $0.068/hour = $50/month
- Using 80% of capacity
- Savings: $613/month annually = $7,356
```

**2. Use gp2 Storage**

```
Provisioned IOPS (io1):
- Base: $0.13/GB/month
- IOPS: $0.10/IOPS/month
- 100GB with 3000 IOPS: $13 + $300 = $313/month

General Purpose (gp2):
- Cost: $0.115/GB/month
- Auto-IOPS based on usage
- 100GB: $11.50/month
- Savings: 96%!
```

**3. Reduce Backup Retention**

```
Development: 7 days = $1.90/month
Staging: 14 days = $3.80/month
Production: 30 days = $9.50/month

If willing to reduce to 7 days for dev: Save $8.50/month × 12 = $102/year
```

**4. Consolidate Environments**

```
Separate instances (dev, staging, prod):
- Dev: db.t3.micro = $12/month
- Staging: db.t3.small = $24/month
- Prod: db.t3.medium = $50/month
- Total: $86/month

Consolidated (same instance with different databases):
- Single: db.t3.medium = $50/month
- Savings: 42% = $36/month annually = $432
- Trade-off: Workload interference possible (not recommended)
```

---

## Troubleshooting

### Connection Issues

```bash
# "too many connections"
aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].DBParameterGroups[0].ParameterApplyStatus"

# Solution: Increase max_connections parameter
aws rds modify-db-parameter-group-parameters \
  --db-parameter-group-name myapp-postgres-params \
  --parameters "[{
    \"ParameterName\": \"max_connections\",
    \"ParameterValue\": \"800\",
    \"ApplyMethod\": \"pending-reboot\"
  }]"

# Reboot to apply
aws rds reboot-db-instance --db-instance-identifier myapp-db
```

### Performance Degradation

```bash
# Check CPU utilization spike
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-db \
  --start-time 2026-05-14T12:00:00Z \
  --end-time 2026-05-14T14:00:00Z \
  --period 300 \
  --statistics Average,Maximum

# If high, check slow queries
PGPASSWORD="$DB_PASSWORD" psql -h myapp-db.xxx.us-east-1.rds.amazonaws.com \
  -U appuser -d myapp_production << EOF
SELECT query, calls, mean_time FROM pg_stat_statements
ORDER BY mean_time DESC LIMIT 5;
EOF

# Optimize slow query or add index
```

### Storage Full

```bash
# Check storage usage
aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].AllocatedStorage"

# Increase storage (automatic with autoscaling)
aws rds modify-db-instance \
  --db-instance-identifier myapp-db \
  --allocated-storage 100 \
  --apply-immediately false
  # No downtime for gp2 storage expansion
```

### Backup Failures

```bash
# Check backup status
aws rds describe-db-instances \
  --db-instance-identifier myapp-db \
  --query "DBInstances[0].LatestRestorableTime"

# If backups failing:
# 1. Ensure sufficient storage (backups need temp space)
# 2. Check backup window isn't during high load
# 3. Verify IAM role has S3 permissions for backup
```

---

## Summary

AWS RDS PostgreSQL provides:
- **Automated backups** with point-in-time recovery
- **High availability** via Multi-AZ failover
- **Scalability** through read replicas and instance upgrades
- **Managed operations** (patching, backups, monitoring)
- **Security** with encryption and IAM authentication
- **Cost efficiency** with precise resource billing

For production applications, minimum recommendations:
- **Instance**: db.t3.medium or larger
- **Multi-AZ**: Enabled for failover
- **Backups**: 30-day retention
- **Monitoring**: CloudWatch alarms on CPU, connections, latency
- **Security**: VPC isolation, encryption enabled, IAM auth
- **Read Replicas**: For analytics/reporting workloads
