# Command Center Skill

## When This Skill Activates

This skill loads automatically when you mention:
- system status, process status, PM2 status
- restart service, debug job, stuck job
- BullMQ, job queue, queue inspection, orchestrator logs
- deployment status, health check, system health
- service down, service restart
- Redis queue, PostgreSQL check, Nginx reload, cache clear
- pipeline status, approval queue
- what's running, is everything healthy, why is X down
- check logs, Redis memory, database connections, queue backlog
- job retry, clear queue, stalled workers
- Any operational status or troubleshooting of services, databases, caching, or job queues

## VPS Services Managed

**Location:** 100.87.191.9 (Tailscale)

**5 Production Services:**
1. **seo-intelligence** (:3001) — Rank tracking, alerts, competitor data
2. **geolab-links** (:3002) — Link profile analysis (requires PostgreSQL + Redis)
3. **geolab-backlinks** (:4000) — Backlink discovery and competitor gap
4. **geolab-keywords** (:4001) — Keyword clustering, SERP features
5. **geolab-writer** (:4002) — Content generation, publishing pipeline

**Supporting Infrastructure:**
- PostgreSQL (5432) — geolab-links database
- Redis (6379) — BullMQ job queue, caching
- Nginx (80/443) — Load balancing, SSL termination
- PM2 — Process management, auto-restart

## Health Check Command

```bash
#!/bin/bash
# Full system health check

echo "=== PM2 Process Status ==="
pm2 status

echo "\n=== Redis Connectivity ==="
redis-cli -h 100.87.191.9 -p 6379 PING

echo "\n=== PostgreSQL Connections ==="
psql -h 100.87.191.9 -U postgres -d openseo -c \
  "SELECT datname, count(*) as connections FROM pg_stat_activity GROUP BY datname;"

echo "\n=== Nginx Status ==="
sudo systemctl status nginx

echo "\n=== Disk Usage ==="
df -h / | awk 'NR==2 {print "Used: " $3 " / Total: " $2 " (" $5 ")"}'

echo "\n=== Memory Usage ==="
free -h | awk 'NR==2 {print "Used: " $3 " / Total: " $2 " (" int($3/$2*100) "%)"}'

echo "\n=== Queue Status ==="
redis-cli -h 100.87.191.9 -p 6379 INFO stats

echo "\n=== Active Jobs ==="
redis-cli -h 100.87.191.9 -p 6379 KEYS 'bull:*' | wc -l
```

## Service Operations

### Check Service Status

```bash
# SSH to VPS
ssh root@100.87.191.9

# List all PM2 services
pm2 status

# Check specific service
pm2 logs seo-intelligence --lines 50  # Last 50 lines
pm2 show seo-intelligence  # Full details
```

### Restart Service

```bash
# Restart single service
pm2 restart seo-intelligence

# Restart all services
pm2 restart all

# Reload (zero-downtime restart)
pm2 reload all
```

### View Service Logs

```bash
# Last 100 lines of specific service
pm2 logs geolab-links --lines 100

# Stream logs in real-time
pm2 logs seo-intelligence --lines 50 --follow

# Check error logs
pm2 logs geolab-backlinks --err --lines 50
```

### Check Service Dependencies

```bash
# Services that require PostgreSQL
# → geolab-links, geolab-backlinks, geolab-keywords

# Services that require Redis
# → geolab-links (BullMQ queue), geolab-writer (job queue)

# Services that are read-only
# → seo-intelligence (uses cached snapshots)
```

## BullMQ Queue Operations

### Inspect Queue Depth

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  LRANGE "bull:seo-intelligence:wait" 0 -1 | wc -l

# All queue states
redis-cli -h 100.87.191.9 -p 6379 \
  INFO stats

# Stalled jobs
redis-cli -h 100.87.191.9 -p 6379 \
  LRANGE "bull:geolab-links:failed" 0 -1
```

### Retry Failed Jobs

```bash
# List failed jobs
redis-cli -h 100.87.191.9 -p 6379 \
  LRANGE "bull:geolab-writer:failed" 0 -1

# Move job from failed to waiting (retry)
redis-cli -h 100.87.191.9 -p 6379 \
  RPOPLPUSH "bull:geolab-writer:failed" "bull:geolab-writer:wait"

# Clear failed queue entirely
redis-cli -h 100.87.191.9 -p 6379 \
  DEL "bull:geolab-writer:failed"
```

### Clear Queue

```bash
# Stop processing (pause queue)
redis-cli -h 100.87.191.9 -p 6379 \
  SET "bull:geolab-links:paused" "1"

# Clear all waiting jobs
redis-cli -h 100.87.191.9 -p 6379 \
  DEL "bull:geolab-links:wait"

# Resume processing
redis-cli -h 100.87.191.9 -p 6379 \
  DEL "bull:geolab-links:paused"
```

## Database Operations

### PostgreSQL Connections

```bash
# Check connection count
psql -h 100.87.191.9 -U postgres -d openseo -c \
  "SELECT datname, count(*) as connections FROM pg_stat_activity GROUP BY datname;"

# Kill idle connections
psql -h 100.87.191.9 -U postgres -d openseo -c \
  "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='openseo' AND state='idle';"

# Check table sizes
psql -h 100.87.191.9 -U postgres -d openseo -c \
  "SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) 
   FROM pg_tables ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;"
```

### Redis Memory

```bash
# Check Redis memory usage
redis-cli -h 100.87.191.9 -p 6379 INFO memory

# Clear cache if needed
redis-cli -h 100.87.191.9 -p 6379 FLUSHALL

# Check key memory by pattern
redis-cli -h 100.87.191.9 -p 6379 \
  DEBUG OBJECT key_name
```

## Nginx Cache Management

```bash
# Clear Nginx FastCGI cache
ssh root@100.87.191.9 \
  'rm -rf /var/cache/nginx/fastcgi/* && systemctl reload nginx'

# Verify cache cleared
ssh root@100.87.191.9 \
  'du -sh /var/cache/nginx/fastcgi/'
```

## Emergency Procedures

### Service Down (no response)

1. Check PM2 status:
   ```bash
   pm2 status
   ```

2. If status shows "stopped", restart:
   ```bash
   pm2 restart service-name
   ```

3. Check logs for errors:
   ```bash
   pm2 logs service-name --err --lines 100
   ```

4. If restart fails, check dependencies:
   - PostgreSQL running? (geolab-links, backlinks, keywords)
   - Redis running? (geolab-links, writer)
   - Nginx responsive? (reverse proxy)

5. Escalate if:
   - Database connection fails
   - Redis crashes
   - Nginx won't reload

### Queue Backlog (1,000+ pending jobs)

1. Check queue health:
   ```bash
   redis-cli LRANGE "bull:service:wait" 0 -1 | wc -l
   ```

2. Check if workers alive:
   ```bash
   pm2 logs service-name | grep "processing"
   ```

3. If no processing, restart workers:
   ```bash
   pm2 restart service-name
   ```

4. If queue still stuck:
   ```bash
   redis-cli DEL "bull:service:wait"
   ```

### High Memory Usage (>80%)

1. Check Redis memory:
   ```bash
   redis-cli INFO memory
   ```

2. If Redis >80%:
   ```bash
   redis-cli FLUSHALL  # Clear all cache (warning: temporary)
   ```

3. Check PostgreSQL:
   ```bash
   psql -c "SELECT pid, usename, memory_mb FROM pg_stat_activity ORDER BY memory_mb DESC;"
   ```

4. Kill idle connections:
   ```bash
   psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle';"
   ```

## See Also

- **services.md** — Detailed service configs, endpoints, dependencies
- **queues.md** — BullMQ operations, job inspection, troubleshooting
- **runbooks.md** — Step-by-step operational procedures
