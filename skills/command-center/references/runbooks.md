# Operational Runbooks

## Daily Health Check (5 min)

```bash
#!/bin/bash
# Run every morning

echo "=== PM2 Status ==="
pm2 status | grep -E "online|offline|stopped"

echo "\n=== Queue Depth ==="
for svc in "seo-intelligence" "geolab-links" "geolab-backlinks" "geolab-keywords" "geolab-writer"; do
  count=$(redis-cli -h 100.87.191.9 LLEN "bull:${svc}:wait" 2>/dev/null || echo "0")
  echo "$svc: $count pending jobs"
done

echo "\n=== Database ==="
psql -h 100.87.191.9 -U postgres -d openseo -c "SELECT COUNT(*) as active_connections FROM pg_stat_activity WHERE state='active';" 2>/dev/null

echo "\n=== Redis Memory ==="
redis-cli -h 100.87.191.9 INFO memory | grep used_memory_human

echo "\n=== Disk Space ==="
df -h / | tail -1 | awk '{print "Usage: " $5}'
```

## Service Deployment (zero-downtime)

```bash
#!/bin/bash
# Deploy new version of a service

SERVICE="geolab-links"
VERSION="v2.1.0"

echo "Deploying $SERVICE $VERSION..."

# 1. Pull code
cd /home/user/services/$SERVICE
git fetch origin
git checkout $VERSION
npm ci  # Install dependencies

# 2. Run tests
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed, aborting deployment"
  git checkout main
  exit 1
fi

# 3. Reload service (zero-downtime)
echo "Reloading service..."
pm2 reload $SERVICE --wait-ready

# 4. Health check
sleep 5
curl -f http://100.87.191.9:3002/health || exit 1

echo "✓ Deployment successful"
```

## Backup & Recovery (daily)

```bash
#!/bin/bash
# Backup PostgreSQL and Redis

DATE=$(date +%Y-%m-%d)
BACKUP_DIR="/backups/daily/$DATE"
mkdir -p $BACKUP_DIR

echo "Backing up PostgreSQL..."
pg_dump -h 100.87.191.9 -U postgres openseo | gzip > $BACKUP_DIR/openseo.sql.gz

echo "Backing up Redis..."
redis-cli -h 100.87.191.9 BGSAVE
cp /var/lib/redis/dump.rdb $BACKUP_DIR/redis-dump.rdb

echo "Uploading to Google Drive..."
rclone sync $BACKUP_DIR gdrive:/backups/daily/$DATE

echo "✓ Backup complete"
```

## Emergency Restart (all services)

```bash
#!/bin/bash
# Last resort: full system restart

echo "WARNING: Restarting all services!"
read -p "Continue? (y/n) " -n 1
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
  exit 1
fi

echo "Stopping all services..."
pm2 stop all
sleep 2

echo "Restarting databases..."
sudo systemctl restart postgresql
sudo systemctl restart redis-server
sleep 5

echo "Starting services..."
pm2 start all
pm2 save

sleep 5
echo "Health check:"
pm2 status
```

## Database Optimization (weekly)

```bash
#!/bin/bash
# Optimize PostgreSQL tables

echo "Running VACUUM..."
psql -h 100.87.191.9 -U postgres -d openseo -c "VACUUM ANALYZE;"

echo "Checking bloated tables..."
psql -h 100.87.191.9 -U postgres -d openseo -c "
  SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
  FROM pg_tables
  ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
  LIMIT 10;"

echo "✓ Optimization complete"
```

## Queue Cleanup (on-demand)

```bash
#!/bin/bash
# Clean stuck/old jobs from Redis

echo "Clearing failed jobs older than 7 days..."
redis-cli -h 100.87.191.9 EVAL "
local jobs = redis.call('LRANGE', KEYS[1], 0, -1)
local cutoff = tonumber(ARGV[1])
local removed = 0
for i, job in ipairs(jobs) do
  if tonumber(job) < cutoff then
    redis.call('LREM', KEYS[1], 0, job)
    removed = removed + 1
  end
end
return removed
" 1 "bull:geolab-links:failed" $(date -d '7 days ago' +%s)000

echo "Clearing completed jobs..."
redis-cli -h 100.87.191.9 DEL "bull:geolab-links:completed"

echo "✓ Queue cleanup complete"
```

## Escalation Contacts

**If service won't restart:**
→ Check PostgreSQL (`systemctl status postgresql`)
→ Check Redis (`systemctl status redis-server`)
→ Escalate to infrastructure team

**If database corrupted:**
→ Stop writes (pause all queues)
→ Restore from backup
→ Verify data integrity before resuming

**If queue growing unbounded:**
→ Pause queue
→ Inspect failing jobs
→ Fix root cause
→ Clear queue and restart

**If under DDoS/high load:**
→ Check Nginx logs: `tail -100 /var/log/nginx/access.log`
→ Enable rate limiting
→ Contact hosting provider
