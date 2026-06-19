# PM2: Node.js Process Manager & Clustering

## Table of Contents
1. [PM2 Basics](#pm2-basics)
2. [Installation & Setup](#installation--setup)
3. [Process Management](#process-management)
4. [Clustering](#clustering)
5. [Monitoring](#monitoring)
6. [Auto-Restart & Logs](#auto-restart--logs)
7. [Production Deployment](#production-deployment)

---

## PM2 Basics

### What is PM2?

PM2 = Process manager for Node.js that:
- Keeps app running 24/7 (auto-restart on crash)
- Manages multiple processes
- Enables clustering (multi-core usage)
- Provides monitoring & logs
- Auto-starts on server reboot
- Enables zero-downtime deployments

```
Without PM2:
Node.js app crashes
→ App is offline
→ Users get errors
→ Manual restart needed

With PM2:
Node.js app crashes
→ PM2 detects immediately
→ PM2 auto-restarts app
→ App online in seconds
→ You get alerted
```

---

## Installation & Setup

### Install PM2 Globally

```bash
# Install
sudo npm install -g pm2

# Verify
pm2 --version
# Output: 5.x.x

# Update (get latest)
sudo npm install -g pm2@latest
```

### Create Application

```javascript
// app.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({ status: 'OK', pid: process.pid });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

```bash
# npm start defined in package.json
{
  "scripts": {
    "start": "node app.js"
  }
}
```

---

## Process Management

### Start Application with PM2

```bash
# Simple start
pm2 start app.js

# Start with npm script
pm2 start npm --name "myapp" -- start

# Start with specific port
pm2 start app.js -- --port 3000

# Start with environment variables
pm2 start app.js --env production
```

### List Processes

```bash
pm2 list

# Output:
# ┌─────┬────────┬──────────┬──────┬────────┬─────────┐
# │ id  │ name   │ mode     │ ↺    │ status │ cpu %   │
# ├─────┼────────┼──────────┼──────┼────────┼─────────┤
# │ 0   │ myapp  │ fork     │ 0    │ online │ 0.5%    │
# └─────┴────────┴──────────┴──────┴────────┴─────────┘
```

### Stop/Restart/Delete

```bash
# Stop specific process
pm2 stop 0                    # by ID
pm2 stop myapp              # by name
pm2 stop all                # all processes

# Restart
pm2 restart 0
pm2 restart myapp
pm2 restart all

# Delete from PM2 (stops & removes)
pm2 delete 0
pm2 delete myapp
pm2 delete all

# Reload (graceful restart, no downtime)
pm2 reload myapp
pm2 reload all
```

### Process Status

```bash
# Detailed info
pm2 info myapp

# Shows:
# ├─ process id: 12345
# ├─ status: online
# ├─ uptime: 2d 5h 30m
# ├─ CPU: 0.5%
# ├─ Memory: 45MB
# ├─ Restarts: 2
# └─ Last restart: 2 days ago
```

---

## Clustering

### Enable Cluster Mode (Multi-Core)

Node.js runs on single core by default. PM2 cluster mode uses all cores:

```bash
# Start in cluster mode (auto-detect cores)
pm2 start app.js -i max
# "max" = number of CPU cores

# Start with specific number of instances
pm2 start app.js -i 4
# Starts 4 instances of app

# View cluster
pm2 list

# Output (4 instances):
# ┌─────┬────────┬──────────┬──────┬────────┬──────────┐
# │ id  │ name   │ mode     │ ↺    │ status │ cpu %    │
# ├─────┼────────┼──────────┼──────┼────────┼──────────┤
# │ 0   │ myapp  │ cluster  │ 0    │ online │ 1.2%     │
# │ 1   │ myapp  │ cluster  │ 0    │ online │ 1.1%     │
# │ 2   │ myapp  │ cluster  │ 0    │ online │ 1.0%     │
# │ 3   │ myapp  │ cluster  │ 0    │ online │ 1.3%     │
# └─────┴────────┴──────────┴──────┴────────┴──────────┘
```

### How Clustering Works

```
Port 3000 (Nginx listens here)
    ↓
PM2 Load Balancer (distributes requests)
    ├─ Instance 0 (Node.js process, core 1)
    ├─ Instance 1 (Node.js process, core 2)
    ├─ Instance 2 (Node.js process, core 3)
    └─ Instance 3 (Node.js process, core 4)

All instances share port 3000 (kernel load balancing)
```

### Ecosystem File (Advanced Config)

```bash
# Create ecosystem.config.js
cat > ecosystem.config.js << 'EOF'
module.exports = {
  apps: [
    {
      name: "myapp",
      script: "app.js",
      instances: "max",          // Use all CPU cores
      exec_mode: "cluster",      // Cluster mode
      env: {
        NODE_ENV: "production",
        PORT: 3000
      },
      error_file: "./logs/error.log",
      out_file: "./logs/out.log",
      log_file: "./logs/combined.log",
      log_date_format: "YYYY-MM-DD HH:mm:ss",
      watch: false,              // Don't watch files
      ignore_watch: ["node_modules", "logs"],
      max_memory_restart: "500M", // Restart if > 500MB
      max_restarts: 10,
      min_uptime: "10s",
      listen_timeout: 10000,
      kill_timeout: 5000
    }
  ],
  deploy: {
    production: {
      user: "ec2-user",
      host: "52.123.456.789",
      key: "~/.ssh/key.pem",
      ref: "origin/main",
      repo: "git@github.com:user/repo.git",
      path: "/var/www/myapp",
      "post-deploy": "npm install && pm2 reload ecosystem.config.js --env production"
    }
  }
};
EOF

# Start using ecosystem file
pm2 start ecosystem.config.js

# Reload (graceful restart)
pm2 reload ecosystem.config.js
```

---

## Monitoring

### Real-Time Monitoring

```bash
# Dashboard
pm2 monit

# Shows real-time:
# ├─ CPU %
# ├─ Memory MB
# ├─ Request count
# └─ Error rate

# (Press 'q' to exit)
```

### Logs

```bash
# View logs (tail -f style)
pm2 logs

# View specific process logs
pm2 logs myapp

# View only error logs
pm2 logs myapp --err

# View last 100 lines
pm2 logs myapp --lines 100

# Clear logs
pm2 flush

# Log file locations
ls -la ~/.pm2/logs/
# Contains: myapp-out.log, myapp-error.log
```

### Memory Limit & Auto-Restart

```bash
# Restart if exceeds 500MB
pm2 start app.js --max-memory-restart 500M

# Or in ecosystem file:
{
  max_memory_restart: "500M"
}
```

---

## Auto-Restart & Logs

### Auto-Start on Server Reboot

```bash
# Generate startup script
pm2 startup

# Output:
# [PM2] Init script generated at /etc/init.d/pm2-init.sh
# [PM2] To make PM2 auto-start on boot, run:
# pm2 save

# Save current process list
pm2 save

# Verify
pm2 startup --user ec2-user

# Now when server reboots:
# 1. PM2 starts automatically
# 2. All processes restart
# 3. App is back online
```

### Log Rotation

Prevent log files from consuming all disk space:

```bash
# Install logrotate package
sudo yum install logrotate -y

# Create logrotate config
sudo tee /etc/logrotate.d/pm2 > /dev/null << EOF
~/.pm2/logs/*.log {
  daily
  rotate 10
  compress
  delaycompress
  notifempty
  missingok
  postrotate
    pm2 flush
  endscript
}
EOF

# Test
sudo logrotate -d /etc/logrotate.d/pm2

# Rotate immediately
sudo logrotate -f /etc/logrotate.d/pm2
```

### Monitor with CloudWatch

```bash
# Send logs to CloudWatch
npm install pm2-logrotate

pm2 install pm2-logrotate

# Configure
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 10

# Or send logs to CloudWatch via agent (see CloudWatch guide)
```

---

## Production Deployment

### Zero-Downtime Reload

```bash
# Graceful reload (no requests lost)
pm2 reload myapp

# Or all apps
pm2 reload all

# Process:
# 1. PM2 starts new instance of app
# 2. Waits for new instance to listen on port
# 3. Kills old instance
# 4. Nginx already load-balancing between them
# 5. Result: No downtime, no dropped requests
```

### Rolling Update (Multiple Instances)

```bash
#!/bin/bash
# rolling-update.sh - Update app with zero downtime

for i in {0..3}; do
  echo "Restarting instance $i..."
  
  # Restart single instance
  pm2 restart myapp --only $i
  
  # Wait for it to come online
  sleep 5
  
  # Verify health
  curl -f http://localhost:3000/health || exit 1
done

echo "Rolling update complete"
```

### Health Check Endpoint

```javascript
// app.js
app.get('/health', (req, res) => {
  res.json({
    status: 'UP',
    uptime: process.uptime(),
    timestamp: new Date()
  });
});
```

---

## Best Practices

1. **Always use cluster mode:** pm2 start app.js -i max
2. **Use ecosystem file:** For complex configs
3. **Enable auto-startup:** pm2 startup && pm2 save
4. **Set memory limits:** Restart if exceeds threshold
5. **Monitor logs:** Regular log rotation
6. **Health checks:** Implement /health endpoint
7. **Graceful restarts:** Use pm2 reload for zero-downtime
8. **Error handling:** Catch unhandled exceptions

### Common Issues

**App keeps restarting:**

```bash
# Check logs
pm2 logs myapp --err

# If infinite restart loop:
pm2 stop myapp
# Fix the code
pm2 restart myapp
```

**Memory leak:**

```bash
# Monitor memory growth
pm2 monit

# If growing: set max_memory_restart
pm2 delete myapp
pm2 start app.js --max-memory-restart 300M
```

---

## Next Steps

1. Install PM2 on your EC2 instance
2. Start your Node.js app with PM2
3. Test auto-restart (kill process, PM2 restarts it)
4. Enable auto-startup on reboot
5. Set up clustering for multi-core usage
6. Configure log rotation
