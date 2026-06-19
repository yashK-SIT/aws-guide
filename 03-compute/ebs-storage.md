# EBS Storage: Persistent Block Storage for EC2

## Table of Contents
1. [EBS Basics](#ebs-basics)
2. [Volume Types](#volume-types)
3. [Creating & Managing Volumes](#creating--managing-volumes)
4. [Snapshots & Backups](#snapshots--backups)
5. [Performance Optimization](#performance-optimization)
6. [Cost Optimization](#cost-optimization)
7. [Troubleshooting](#troubleshooting)

---

## EBS Basics

### What is EBS?

EBS (Elastic Block Store) = Persistent storage for EC2 instances. Think of it as an external hard drive you can attach/detach from your server.

```
Without EBS (Ephemeral):
Instance store (temporary)
├─ Lost when instance stops
├─ Lost when instance fails
├─ Good for: Temporary cache, scratch space
└─ NOT suitable for important data

With EBS (Persistent):
EBS volume (permanent)
├─ Survives instance stops/reboots
├─ Backed up with snapshots
├─ Replicated across availability zones
└─ Perfect for: Databases, application data
```

### Key Characteristics

- **Persistence:** Data survives instance stop/start/failure
- **Replication:** Automatically replicated within AZ (high durability)
- **Snapshots:** Point-in-time backups to S3
- **Encryption:** At-rest encryption available
- **Resizing:** Can expand volume size (online, no downtime)
- **Cost:** Pay per GB per month (~$0.10/GB for gp3)

---

## Volume Types

### gp3 (General Purpose) - Recommended

**Best for:** Most workloads (databases, web servers, apps)

```
Price: ~$0.10/GB/month
IOPS: 3000-16000 (configurable)
Throughput: 125-1000 MB/s (configurable)
Latency: Low (~1ms)
Durability: 99.999%

Suitable for:
├─ Databases (MySQL, PostgreSQL)
├─ Web servers
├─ Development instances
└─ General applications
```

**Create:**

```bash
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --iops 3000 \
  --throughput 125 \
  --availability-zone us-east-1a \
  --encrypted
```

### io2 (High Performance) - For Databases

**Best for:** High-I/O databases, mission-critical apps

```
Price: ~$0.25/GB/month
IOPS: 100-64000 (configurable)
Throughput: Similar to gp3
Latency: Extremely low (<1ms)
Durability: 99.999%

Suitable for:
├─ Enterprise databases
├─ Real-time analytics
├─ High-frequency trading
└─ Critical transactional systems
```

### st1 (Throughput Optimized) - For Big Data

**Best for:** Large sequential reads/writes, big data, data warehouses

```
Price: ~$0.045/GB/month
IOPS: 500 max
Throughput: 500 MB/s
Latency: Higher than gp3
Durability: 99.9%

Suitable for:
├─ Data warehouses
├─ Big data processing
├─ Log processing
└─ Hadoop, Spark jobs
```

### sc1 (Cold Storage) - Rarely Used

**Best for:** Infrequent access, archive, backup

```
Price: ~$0.015/GB/month (cheapest)
IOPS: 250 max
Throughput: 250 MB/s
Latency: High
Durability: 99.9%

Suitable for:
├─ Backup storage
├─ Archive
├─ Disaster recovery
└─ Cold storage tiers
```

### Comparison Table

| Type | Price | IOPS | Throughput | Best For |
|------|-------|------|-----------|----------|
| **gp3** | $0.10 | 16k | 1000 MB/s | General purpose ✅ |
| **io2** | $0.25 | 64k | High | High-performance DB |
| **st1** | $0.045 | 500 | 500 MB/s | Big data |
| **sc1** | $0.015 | 250 | 250 MB/s | Archive |

---

## Creating & Managing Volumes

### Create Volume

```bash
# Create 100GB gp3 volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=data-volume}]'

# Response: Volume ID (vol-0123456789abcdef0)
```

### Attach to Instance

```bash
VOLUME_ID="vol-0123456789abcdef0"
INSTANCE_ID="i-0123456789abcdef0"

# Attach
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_ID \
  --device /dev/sdf

# Device name: /dev/sdf (or /dev/xvdf inside instance)
```

### Format & Mount (Inside Instance)

```bash
# SSH into instance
ssh -i key.pem ec2-user@instance-ip

# List available disks
lsblk
# Output:
# NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# nvme0n1 259:0    0  20G  0 disk
# └─nvme0n1p1  259:1    0  20G  0 part /
# nvme1n1 259:1    0 100G  0 disk              ← New volume

# Create filesystem
sudo mkfs.xfs /dev/nvme1n1

# Create mount point
sudo mkdir /data

# Mount
sudo mount /dev/nvme1n1 /data

# Verify
df -h
# Should show /data with 100G

# Make persistent (survives reboot)
echo '/dev/nvme1n1 /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab

# Verify
cat /etc/fstab
```

### Resize Volume (Online, No Downtime)

```bash
# On AWS side: increase volume size
aws ec2 modify-volume \
  --volume-id vol-0123456789abcdef0 \
  --size 200

# Inside instance: expand filesystem
sudo growpart /dev/nvme1n1 1

# Resize filesystem
sudo xfs_growfs /data

# Verify
df -h /data
# Should now show 200G
```

### Detach Volume

```bash
# Unmount from instance first!
sudo umount /data

# Then detach
aws ec2 detach-volume \
  --volume-id vol-0123456789abcdef0

# Can now attach to different instance
```

---

## Snapshots & Backups

### Create Snapshot

Snapshot = Point-in-time backup stored in S3

```bash
# Create snapshot
aws ec2 create-snapshot \
  --volume-id vol-0123456789abcdef0 \
  --description "Daily backup 2026-05-14" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Backup,Value=Daily}]'

# Response: Snapshot ID (snap-0123456789abcdef0)

# List snapshots
aws ec2 describe-snapshots --owner-ids self

# Monitor progress
aws ec2 describe-snapshots \
  --snapshot-ids snap-0123456789abcdef0 \
  --query 'Snapshots[0].[Progress,State]'
# Output: ["100%", "completed"]
```

### Automate Snapshots

```bash
#!/bin/bash
# daily-snapshot.sh - Run via cron

VOLUME_ID="vol-0123456789abcdef0"
DATE=$(date +%Y-%m-%d)

# Create snapshot
SNAPSHOT=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "Automated backup $DATE" \
  --query 'SnapshotId' --output text)

echo "Created snapshot: $SNAPSHOT"

# Keep only last 7 days
aws ec2 describe-snapshots \
  --owner-ids self \
  --filters "Name=volume-id,Values=$VOLUME_ID" \
  --query "sort_by(Snapshots, &StartTime)[:-7].SnapshotId" \
  --output text | xargs -I {} aws ec2 delete-snapshot --snapshot-id {}
```

**Add to crontab:**

```bash
0 2 * * * /usr/local/bin/daily-snapshot.sh
# Runs daily at 2 AM
```

### Restore from Snapshot

```bash
# Create new volume from snapshot
aws ec2 create-volume \
  --snapshot-id snap-0123456789abcdef0 \
  --availability-zone us-east-1a \
  --volume-type gp3

# Attach to instance
# (same process as creating new volume)
```

---

## Performance Optimization

### Monitor Volume Performance

```bash
# Check IOPS usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --start-time 2026-05-14T00:00:00Z \
  --end-time 2026-05-14T23:59:59Z \
  --period 3600 \
  --statistics Sum

# Check throughput
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadBytes \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --start-time 2026-05-14T00:00:00Z \
  --end-time 2026-05-14T23:59:59Z \
  --period 3600 \
  --statistics Sum
```

### Improve Performance

**Strategy 1: Use gp3 instead of gp2**

```bash
# gp2 → gp3 migration
# Create snapshot of gp2 volume
SNAPSHOT=$(aws ec2 create-snapshot \
  --volume-id vol-gp2 \
  --query 'SnapshotId' --output text)

# Create gp3 from snapshot
aws ec2 create-volume \
  --snapshot-id $SNAPSHOT \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --availability-zone us-east-1a

# Attach new gp3 volume
# Mount and copy data if needed
```

**Strategy 2: Increase IOPS**

```bash
# For gp3: increase IOPS (no downtime)
aws ec2 modify-volume \
  --volume-id vol-xxx \
  --iops 6000 \
  --throughput 250

# For high-performance: switch to io2
# (requires snapshot → new volume approach)
```

**Strategy 3: Use EBS-optimized Instances**

```bash
# Launch EBS-optimized instance
aws ec2 run-instances \
  --image-id ami-xxx \
  --instance-type m5.large \
  --ebs-optimized \
  # Guarantees consistent IOPS to EBS

# Benefits:
# - Dedicated network throughput to EBS
# - No network contention
# - Better for databases, high-traffic apps
```

---

## Cost Optimization

### Rightsizing

```
100GB gp3 @ $0.10/GB/month = $10/month

Too small?
200GB gp3 @ $0.10/GB/month = $20/month

Too large but rarely used?
100GB sc1 (cold storage) = $1.50/month

Decision tree:
├─ Frequently accessed? → gp3
├─ High IOPS needed? → io2
├─ Sequential large reads? → st1
└─ Rarely accessed? → sc1
```

### Cleanup Unattached Volumes

```bash
# Find unattached volumes
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[].{Id:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}' \
  --output table

# Delete unused volume (⚠️ irreversible)
aws ec2 delete-volume --volume-id vol-unused

# Or create snapshot first (backup)
aws ec2 create-snapshot \
  --volume-id vol-unused \
  --description "Backup before deletion"

# Wait for snapshot
# Then delete volume
```

### Compression for Snapshots

Snapshots are incremental (only new data backed up), which saves space:

```
Day 1: Create volume (100GB)
       Create snapshot: 100GB stored in S3
       Cost: ~$9.50 (100GB @ $0.095/GB)

Day 2: Modify 10GB of data
       Create snapshot: Only 10GB change stored
       Total cost: ~$10.45 (110GB total)

Day 30: Multiple snapshots
        Total: ~150GB stored
        Cost: ~$14.25/month
```

---

## Troubleshooting

### Can't Format Volume

```bash
# Error: "Device is busy"
lsof /dev/nvme1n1
# Kill any processes using device

# Error: "No space left on device"
df -i  # Check inode usage
# Inode exhaustion (many small files)
# May need to reformat or extend inode space

# Error: "Read-only file system"
sudo mount -o remount,rw /data
# Force read-write remount
```

### Snapshot Failed

```bash
# Check snapshot status
aws ec2 describe-snapshots --snapshot-ids snap-xxx \
  --query 'Snapshots[0].State'

# If failed: Try again
aws ec2 create-snapshot \
  --volume-id vol-xxx \
  --description "Retry"

# Check volume health
aws ec2 describe-volume-status \
  --volume-ids vol-xxx
```

### Volume Attached but Not Visible

```bash
# In instance:
lsblk
# If volume not showing: might be attaching still

# Wait a moment, then:
sudo partprobe  # Force Linux to re-read partition table

# Or restart instance
sudo shutdown -r now
```

---

## Best Practices

1. **Encrypt volumes:** Always enable encryption
2. **Backup regularly:** Snapshot at least daily
3. **Right-size:** Don't over-provision storage
4. **Monitor:** Watch IOPS/throughput metrics
5. **Tag volumes:** For cost tracking
6. **Delete unused:** Unattached volumes still cost money
7. **Use gp3:** Better value than gp2
8. **Test restores:** Verify snapshots work

---

## Next Steps

1. Create EBS volume for your EC2 instance
2. Set up daily snapshots
3. Monitor volume performance
4. Plan backup retention policy
5. Test restoration from snapshot
