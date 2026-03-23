# BullMQ Queue Operations

## Queue Basics

Each service (except seo-intelligence) has a BullMQ queue in Redis:

```
bull:geolab-links:wait      # Pending jobs
bull:geolab-links:active    # Currently processing
bull:geolab-links:completed # Finished
bull:geolab-links:failed    # Errors
bull:geolab-links:stalled   # Timeout/crash
```

## Inspect Queue State

### Queue Depth (how many jobs waiting)

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  LLEN "bull:geolab-links:wait"
```

### See All Queues

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  KEYS "bull:*" | sort
```

### Count Jobs by State

```bash
redis-cli -h 100.87.191.9 -p 6379 EVAL "
local states = {'wait', 'active', 'completed', 'failed', 'stalled'}
for _, state in ipairs(states) do
  redis.call('ECHO', state .. ': ' .. redis.call('LLEN', KEYS[1] .. ':' .. state))
end
" 1 "bull:geolab-links"
```

### Active Jobs (currently running)

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  LRANGE "bull:geolab-links:active" 0 -1 | head -5
```

## Job Inspection

### View Job Details

```bash
# Get job from queue
redis-cli -h 100.87.191.9 -p 6379 \
  HGETALL "bull:geolab-links:job_id_123"

# Example output:
# id: "123"
# data: "{\"url\":\"example.com\",\"depth\":2}"
# progress: "50"
# state: "active"
# attemptsMade: "1"
```

### Failed Job Details

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  LINDEX "bull:geolab-links:failed" 0 | jq '.'

# Shows:
# - Job ID
# - Error message
# - Stack trace
# - Retry count
# - Last attempt timestamp
```

## Queue Management

### Pause Queue (stop processing)

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  SET "bull:geolab-links:paused" "1"
```

**Effect:** Workers stop picking up new jobs (graceful shutdown)

### Resume Queue

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  DEL "bull:geolab-links:paused"
```

### Clear Waiting Jobs

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  DEL "bull:geolab-links:wait"
```

**Warning:** This removes all pending jobs (data loss)

### Clear Failed Jobs

```bash
redis-cli -h 100.87.191.9 -p 6379 \
  DEL "bull:geolab-links:failed"
```

### Retry Failed Job

```bash
# Move from failed to waiting
redis-cli -h 100.87.191.9 -p 6379 \
  RPOPLPUSH "bull:geolab-links:failed" "bull:geolab-links:wait"
```

### Bulk Retry Failed Jobs

```bash
# Retry all failed jobs
redis-cli -h 100.87.191.9 -p 6379 EVAL "
local failed = redis.call('LRANGE', KEYS[1], 0, -1)
for _, job in ipairs(failed) do
  redis.call('RPUSH', KEYS[2], job)
end
return table.getn(failed)
" 2 "bull:geolab-links:failed" "bull:geolab-links:wait"
```

## Performance Tuning

### Check Worker Concurrency

```bash
# View service config for worker count
pm2 show geolab-links | grep -i concurrency
```

**Default:** 5 concurrent workers per service

### Increase Concurrency (if queue backlogging)

```bash
# Edit ecosystem.config.js
vim ecosystem.config.js

# Change:
# max_memory_restart: "100M",
# instances: "max",  # Use all CPU cores
# exec_mode: "cluster",

pm2 restart geolab-links
```

### Monitor Queue Performance

```bash
# Average job processing time
redis-cli -h 100.87.191.9 -p 6379 \
  HGETALL "bull:geolab-links:stats" | grep avg_time

# Processing rate (jobs/second)
redis-cli -h 100.87.191.9 -p 6379 \
  INFO stats | grep "instantaneous_ops_per_sec"
```

## Troubleshooting

### Queue Stuck (no processing)

**Symptoms:** Queue depth increasing, no completed jobs

**Diagnosis:**
```bash
# Check if workers alive
pm2 logs geolab-links | grep "processing\|completed"

# Check if paused
redis-cli GET "bull:geolab-links:paused"

# Check active jobs
redis-cli LLEN "bull:geolab-links:active"
```

**Fix:**
```bash
# Resume if paused
redis-cli DEL "bull:geolab-links:paused"

# Restart workers
pm2 restart geolab-links

# Force clear stuck jobs
redis-cli DEL "bull:geolab-links:active"
```

### Jobs Timing Out

**Symptoms:** Jobs in "stalled" state, max retries exceeded

**Causes:**
- Job takes too long (timeout threshold)
- Worker crash during processing
- External API timeout (Serper, CC, etc.)

**Fix:**
1. Check logs: `pm2 logs geolab-links --err`
2. Increase timeout in config (if legitimate)
3. Retry: Move from stalled to wait

### Memory Leak (queue growing indefinitely)

**Diagnosis:**
```bash
redis-cli INFO memory | grep used_memory
redis-cli LLEN "bull:geolab-links:completed"  # Unbounded?
```

**Fix:**
```bash
# Archive old completed jobs
redis-cli LTRIM "bull:geolab-links:completed" 0 1000

# Or clear entirely
redis-cli DEL "bull:geolab-links:completed"
```
