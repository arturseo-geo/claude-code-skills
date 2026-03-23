# Service Architecture & Operations

## Service Matrix

| Service | Port | Framework | Database | Cache | Purpose |
|---------|------|-----------|----------|-------|----------|
| seo-intelligence | 3001 | Node/Express | None (Redis snapshots) | Redis | Rank tracking, alerts |
| geolab-links | 3002 | Node/Express | PostgreSQL | Redis | Link profile, authority |
| geolab-backlinks | 4000 | Node/Express | PostgreSQL | Redis | Backlink discovery, gaps |
| geolab-keywords | 4001 | Node/Express | PostgreSQL | Redis | Keyword tracking, SERPs |
| geolab-writer | 4002 | Node/Express | PostgreSQL | Redis | Content generation, queue |

## Service Dependencies

```
seo-intelligence
  └─ Redis (snapshots)

geolab-links
  ├─ PostgreSQL (core data)
  ├─ Redis (BullMQ queue)
  └─ External APIs (CC backlinks, Serper SERP)

geolab-backlinks
  ├─ PostgreSQL (core data)
  ├─ Redis (job queue)
  └─ Common Crawl (data source)

geolab-keywords
  ├─ PostgreSQL (core data)
  ├─ Redis (queue)
  └─ Serper API (SERP data)

geolab-writer
  ├─ PostgreSQL (prompts, templates)
  ├─ Redis (BullMQ queue)
  ├─ Anthropic API (Claude)
  └─ WordPress REST API (publishing)
```

## Service Health Checks

### seo-intelligence (:3001)

**Test endpoint:**
```bash
curl http://100.87.191.9:3001/health
# Expected: {"status": "healthy"}
```

**Dependency check:**
- Redis snapshot cache (read-only, non-critical)
- If cache empty, rebuilds from last crawl

**Recovery:** Restart only if unresponsive
```bash
pm2 restart seo-intelligence
```

### geolab-links (:3002)

**Test endpoint:**
```bash
curl http://100.87.191.9:3002/api/health
```

**Dependencies:**
- PostgreSQL (required)
- Redis (required)
- External APIs (optional)

**If down:**
1. Check PostgreSQL: `psql -U postgres -d openseo -c "SELECT 1;"`
2. Check Redis: `redis-cli PING`
3. Restart service: `pm2 restart geolab-links`
4. Check logs: `pm2 logs geolab-links --err`

### geolab-backlinks (:4000)

**Test endpoint:**
```bash
curl http://100.87.191.9:4000/api/health
```

**Dependencies:**
- PostgreSQL (required)
- Redis (required)
- Common Crawl API (optional, for data collection)

**If down:**
1. Same as geolab-links (shared infrastructure)
2. Check CPU (crawling can spike)
3. Restart if stuck: `pm2 restart geolab-backlinks`

### geolab-keywords (:4001)

**Test endpoint:**
```bash
curl http://100.87.191.9:4001/api/health
```

**Dependencies:**
- PostgreSQL (required)
- Redis (required)
- Serper API (optional)

**If down:**
1. Check PostgreSQL connection
2. Monitor API quota (Serper has rate limits)
3. Restart: `pm2 restart geolab-keywords`

### geolab-writer (:4002)

**Test endpoint:**
```bash
curl http://100.87.191.9:4002/api/health
```

**Dependencies:**
- PostgreSQL (required)
- Redis/BullMQ (required)
- Anthropic API (required, has rate limits)
- WordPress API (for publishing)

**If down:**
1. Check API keys in .env
2. Check job queue: `redis-cli LRANGE "bull:geolab-writer:wait" 0 -1`
3. Clear stuck jobs: `redis-cli DEL "bull:geolab-writer:wait"`
4. Restart: `pm2 restart geolab-writer`

## PM2 Commands Reference

```bash
# Start all services
pm2 start ecosystem.config.js

# List all services with status
pm2 list
pm2 status

# Show detailed info
pm2 show geolab-links

# Monitor in real-time
pm2 monit

# View logs (last 50 lines)
pm2 logs service-name --lines 50
pm2 logs service-name --err  # Errors only
pm2 logs service-name --follow  # Stream in real-time

# Restart
pm2 restart service-name
pm2 restart all

# Reload (zero-downtime)
pm2 reload all

# Stop
pm2 stop service-name
pm2 stop all

# Save PM2 config
pm2 save
pm2 startup
```

## Port Availability

```bash
# Check if port in use
lsof -i :3001
lsof -i :3002
lsof -i :4000
lsof -i :4001
lsof -i :4002

# Kill process on port
kill -9 <PID>
```

## Nginx Reverse Proxy Config

**Location:** `/etc/nginx/sites-enabled/services`

**Pattern:**
```nginx
server {
    listen 100.87.191.9:3001;
    server_name api.seo-intelligence.local;
    
    location / {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Reload after change:**
```bash
nginx -t  # Test config
sudo systemctl reload nginx
```

## Service Environment Variables

Stored in `/home/user/.env` (one per service or shared):

```
DATABASE_URL=postgres://user:pass@100.87.191.9:5432/openseo
REDIS_URL=redis://100.87.191.9:6379
WORDPRESS_API_KEY=xxxx
ANTHROPIC_API_KEY=xxxx
SERPER_API_KEY=xxxx
```

**Apply changes:**
```bash
pm2 restart all
```
