# Resource Management and On-Demand Usage

Optimizing your cloud lab for cost efficiency and performance in resource-constrained environments.

## Overview

This guide helps you maximize the value of your DigitalOcean droplet (or similar) by providing flexible, on-demand usage patterns for your containerized development environment.

## Resource Requirements

### Minimum System Requirements
- **CPU**: 2 vCPUs
- **RAM**: 4GB
- **Storage**: 50GB SSD
- **Network**: Standard DigitalOcean networking

### Recommended System Requirements
- **CPU**: 4 vCPUs (for parallel builds)
- **RAM**: 8GB (for comfortable multi-service operation)
- **Storage**: 100GB SSD (for build artifacts and logs)

### Service Resource Usage

| Service | CPU (idle) | CPU (active) | RAM (idle) | RAM (active) | Storage |
|---------|------------|--------------|------------|--------------|---------|
| Traefik | 0.1% | 2% | 20MB | 50MB | 10MB |
| Jenkins | 5% | 50% | 500MB | 1.5GB | 2-5GB |
| Gitea | 2% | 10% | 100MB | 300MB | 1-3GB |
| PostgreSQL | 3% | 15% | 150MB | 400MB | 500MB-2GB |
| Semgrep Broker | 1% | 5% | 50MB | 100MB | 50MB |

## On-Demand Usage Patterns

### 1. Demo Preparation Mode
**Use case**: Preparing for customer demos
**Services**: Jenkins + Gitea only
**Duration**: 1-2 hours

```bash
# Start minimal services
docker-compose up -d jenkins gitea

# Verify services are running
docker-compose ps

# Expected resource usage: ~2GB RAM, 1-2 CPU cores
```

### 2. Active Demo Mode
**Use case**: Live customer demonstration
**Services**: All services (full experience)
**Duration**: 1-3 hours

```bash
# Start all services
docker-compose up -d

# Monitor resource usage during demo
watch -n 5 'docker stats --no-stream'

# Expected resource usage: ~3-4GB RAM, 2-3 CPU cores
```

### 3. Development Mode
**Use case**: Building reference architectures
**Services**: Jenkins + Gitea + PostgreSQL
**Duration**: 4-8 hours

```bash
# Start development services
docker-compose up -d jenkins gitea postgres

# Leave Semgrep broker and Traefik off to save resources
# Expected resource usage: ~2.5GB RAM, 1-2 CPU cores
```

### 4. Testing Mode
**Use case**: Semgrep integration testing
**Services**: Jenkins + Gitea + Semgrep Broker
**Duration**: 2-4 hours

```bash
# Start testing services
docker-compose up -d jenkins gitea semgrep-broker

# Skip database unless specifically needed
# Expected resource usage: ~2.2GB RAM, 1-2 CPU cores
```

### 5. Standby Mode
**Use case**: Preserving data while minimizing costs
**Services**: None running
**Duration**: Hours to days

```bash
# Stop all services (preserves data)
docker-compose down

# Expected resource usage: Minimal (base OS only)
```

## Cost Optimization Strategies

### 1. Scheduled Operations

#### Automatic Shutdown Script
```bash
#!/bin/bash
# auto-shutdown.sh - Schedule with cron for automatic cost savings

# Shutdown services at end of business day
SHUTDOWN_TIME="18:00"
CURRENT_TIME=$(date +%H:%M)

if [ "$CURRENT_TIME" = "$SHUTDOWN_TIME" ]; then
    echo "$(date): Shutting down lab services for cost savings"
    cd /path/to/cloud-lab/docker
    docker-compose down
    
    # Optional: Shutdown the entire droplet
    # sudo shutdown -h now
fi
```

#### Automatic Startup Script
```bash
#!/bin/bash
# auto-startup.sh - Start services when needed

# Startup services at beginning of business day
STARTUP_TIME="08:00"
CURRENT_TIME=$(date +%H:%M)

if [ "$CURRENT_TIME" = "$STARTUP_TIME" ]; then
    echo "$(date): Starting lab services for business day"
    cd /path/to/cloud-lab/docker
    docker-compose up -d jenkins gitea
fi
```

#### Cron Configuration
```bash
# Edit crontab
crontab -e

# Add scheduled operations
0 18 * * 1-5 /path/to/auto-shutdown.sh
0 8 * * 1-5 /path/to/auto-startup.sh

# Weekly cleanup (Sundays at 2 AM)
0 2 * * 0 /path/to/cleanup-logs.sh
```

### 2. Smart Service Selection

#### Context-Aware Startup
```bash
#!/bin/bash
# smart-start.sh - Start services based on intended use

usage() {
    echo "Usage: $0 [demo|dev|test|full|minimal]"
    echo "  demo    - Jenkins + Gitea (customer demos)"
    echo "  dev     - Jenkins + Gitea + PostgreSQL (development)"
    echo "  test    - Jenkins + Gitea + Semgrep Broker (testing)"
    echo "  full    - All services (complete experience)"
    echo "  minimal - Gitea only (quick access)"
}

case "$1" in
    demo)
        echo "Starting demo environment..."
        docker-compose up -d jenkins gitea
        ;;
    dev)
        echo "Starting development environment..."
        docker-compose up -d jenkins gitea postgres
        ;;
    test)
        echo "Starting testing environment..."
        docker-compose up -d jenkins gitea semgrep-broker
        ;;
    full)
        echo "Starting full environment..."
        docker-compose up -d
        ;;
    minimal)
        echo "Starting minimal environment..."
        docker-compose up -d gitea
        ;;
    *)
        usage
        exit 1
        ;;
esac

echo "Environment started. Use 'docker-compose ps' to verify."
```

### 3. Resource Monitoring

#### Real-time Monitoring
```bash
#!/bin/bash
# monitor-resources.sh - Track resource usage

echo "=== System Resources ==="
echo "CPU Usage:"
top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1

echo "Memory Usage:"
free -h | grep '^Mem:' | awk '{print $3 "/" $2 " (" $3/$2*100 "%)"}'

echo "Disk Usage:"
df -h / | tail -1 | awk '{print $3 "/" $2 " (" $5 ")"}'

echo -e "\n=== Docker Container Resources ==="
docker stats --no-stream --format "table {{.Name}}\\t{{.CPUPerc}}\\t{{.MemUsage}}\\t{{.MemPerc}}"

echo -e "\n=== Service Status ==="
docker-compose ps
```

#### Resource Alerts
```bash
#!/bin/bash
# resource-alerts.sh - Alert when resources are high

# Set thresholds
CPU_THRESHOLD=80
MEM_THRESHOLD=85
DISK_THRESHOLD=90

# Check CPU usage
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d' ' -f1)
if (( $(echo "$CPU_USAGE > $CPU_THRESHOLD" | bc -l) )); then
    echo "ALERT: High CPU usage: ${CPU_USAGE}%"
fi

# Check memory usage
MEM_USAGE=$(free | grep '^Mem:' | awk '{print ($3/$2)*100}')
if (( $(echo "$MEM_USAGE > $MEM_THRESHOLD" | bc -l) )); then
    echo "ALERT: High memory usage: ${MEM_USAGE}%"
fi

# Check disk usage
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | cut -d'%' -f1)
if [ "$DISK_USAGE" -gt "$DISK_THRESHOLD" ]; then
    echo "ALERT: High disk usage: ${DISK_USAGE}%"
fi
```

## Storage Management

### 1. Data Volume Optimization

#### Cleanup Scripts
```bash
#!/bin/bash
# cleanup-labs.sh - Clean up accumulated data

echo "=== Lab Cleanup Starting ==="

# Clean Jenkins build artifacts (keep last 10 builds)
find ./data/jenkins/jobs/*/builds/ -type d -name '[0-9]*' | \
    sort -V | head -n -10 | xargs rm -rf

# Clean Docker images and volumes
docker system prune -f
docker volume prune -f

# Clean Gitea logs (keep last 7 days)
find ./data/gitea/log/ -name "*.log" -mtime +7 -delete

# Clean PostgreSQL logs if present
find ./data/postgres/ -name "*.log" -mtime +7 -delete

echo "=== Cleanup Complete ==="
echo "Disk usage after cleanup:"
du -sh ./data/*
```

#### Backup Strategy
```bash
#!/bin/bash
# backup-critical-data.sh - Backup important data before cleanup

BACKUP_DIR="/backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Creating backup: $BACKUP_DIR"

# Backup critical Jenkins data
cp -r ./data/jenkins/jobs/ "$BACKUP_DIR/jenkins-jobs/"
cp ./data/jenkins/credentials.xml "$BACKUP_DIR/"

# Backup Gitea repositories
cp -r ./data/gitea/git/ "$BACKUP_DIR/gitea-repos/"
cp ./data/gitea/gitea.db "$BACKUP_DIR/"

# Backup configuration files
cp docker-compose.yml "$BACKUP_DIR/"
cp -r configs/ "$BACKUP_DIR/"

echo "Backup completed: $BACKUP_DIR"
```

### 2. Log Management

#### Log Rotation Configuration
```yaml
# docker-compose.yml - Add logging configuration
services:
  jenkins:
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"
        
  gitea:
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"
```

#### Manual Log Cleanup
```bash
#!/bin/bash
# cleanup-logs.sh - Manual log cleanup

echo "=== Cleaning Docker logs ==="
docker container prune -f
truncate -s 0 /var/lib/docker/containers/*/*-json.log

echo "=== Cleaning application logs ==="
find ./data/*/log/ -name "*.log" -mtime +7 -delete

echo "=== Log cleanup complete ==="
```

## Performance Tuning

### 1. Docker Optimization

#### Resource Limits
```yaml
# docker-compose.yml - Add resource limits
services:
  jenkins:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
          
  gitea:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 256M
```

#### Build Optimization
```bash
# Optimize Docker build context
echo ".git" >> .dockerignore
echo "logs/" >> .dockerignore
echo "backups/" >> .dockerignore
echo "*.tmp" >> .dockerignore
```

### 2. Application Tuning

#### Jenkins Optimization
```bash
# Set Jenkins JVM options in docker-compose.yml
services:
  jenkins:
    environment:
      - JAVA_OPTS=-Xmx1536m -Xms512m -XX:+UseG1GC
      - JENKINS_OPTS=--requestHeaderSize=16384
```

#### Gitea Optimization
```ini
# data/gitea/conf/app.ini
[repository]
ROOT = /data/git/repositories
ENABLE_PUSH_CREATE_USER = true
ENABLE_PUSH_CREATE_ORG = true

[database]
LOG_SQL = false

[log]
LEVEL = Warn
ROOT_PATH = /data/gitea/log
```

## Cost Monitoring

### 1. DigitalOcean Monitoring

#### Metrics to Track
- **CPU utilization**: Target < 70% average
- **Memory usage**: Target < 80% average  
- **Disk I/O**: Monitor for bottlenecks
- **Network usage**: Track API calls and builds

#### Billing Optimization
```bash
#!/bin/bash
# cost-analysis.sh - Track lab usage costs

# Get droplet info
doctl compute droplet list --format "Name,Memory,VCPUs,Disk,Price"

# Calculate monthly costs
DROPLET_COST=40  # $40/month for 4GB/2CPU droplet
HOURS_PER_MONTH=730
HOURS_USED_PER_DAY=8  # Estimated daily usage

MONTHLY_USAGE_COST=$(echo "scale=2; $DROPLET_COST * ($HOURS_USED_PER_DAY * 30) / $HOURS_PER_MONTH" | bc)

echo "Estimated monthly cost: $${MONTHLY_USAGE_COST}"
echo "Full month cost: $${DROPLET_COST}"
echo "Potential savings: $(echo "scale=2; $DROPLET_COST - $MONTHLY_USAGE_COST" | bc)"
```

### 2. Usage Analytics

#### Service Utilization Tracking
```bash
#!/bin/bash
# usage-analytics.sh - Track service usage patterns

LOG_FILE="/var/log/lab-usage.log"

# Log service starts/stops
track_usage() {
    local action="$1"
    local services="$2"
    echo "$(date '+%Y-%m-%d %H:%M:%S') $action $services" >> "$LOG_FILE"
}

# Hook into docker-compose commands
alias docker-compose-up='docker-compose up -d && track_usage "START" "$(docker-compose ps --services)"'
alias docker-compose-down='track_usage "STOP" "$(docker-compose ps --services)" && docker-compose down'

# Weekly usage report
generate_usage_report() {
    echo "=== Weekly Lab Usage Report ==="
    echo "Service start events:"
    grep "START" "$LOG_FILE" | tail -20
    
    echo -e "\nMost used services:"
    grep "START" "$LOG_FILE" | awk '{print $4}' | sort | uniq -c | sort -nr
}
```

## Troubleshooting Resource Issues

### 1. High Memory Usage
```bash
# Identify memory-hungry containers
docker stats --no-stream | sort -k 4 -hr

# Check for memory leaks
docker exec <container> cat /proc/meminfo

# Restart high-usage services
docker-compose restart jenkins
```

### 2. High CPU Usage
```bash
# Identify CPU-intensive processes
docker exec <container> top -n 1

# Check for runaway builds
docker-compose logs jenkins | grep -i "build.*running"

# Limit CPU usage
docker update --cpus="1.0" <container>
```

### 3. Disk Space Issues
```bash
# Check disk usage by container
docker system df

# Clean up build artifacts
docker builder prune -f

# Remove old images
docker image prune -a
```

## Best Practices Summary

### Daily Operations
1. **Start only needed services** for your current task
2. **Monitor resource usage** during active development
3. **Stop services** when stepping away for extended periods
4. **Check logs regularly** for errors or issues

### Weekly Maintenance
1. **Run cleanup scripts** to free disk space
2. **Backup critical data** before major changes
3. **Review usage patterns** and optimize accordingly
4. **Update containers** to latest versions

### Monthly Reviews
1. **Analyze cost vs. usage** patterns
2. **Evaluate droplet sizing** needs
3. **Review backup strategy** effectiveness
4. **Update documentation** based on learnings

---
*This completes the comprehensive cloud lab documentation. All guides work together to provide a complete reference for Solutions Engineers building hands-on Semgrep expertise.*