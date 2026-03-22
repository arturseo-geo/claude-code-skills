# Infrastructure — Building SEO Tools

## Node.js Stack

- **Runtime**: Node.js 20 LTS on Ubuntu
- **Framework**: Express.js for API endpoints
- **HTML parsing**: cheerio
- **Process manager**: PM2
- **Database**: SQLite (via better-sqlite3 or sqlite3)

---

## Proxy Rotation (Webshare)

### Setup

Webshare provides residential proxies. Use their rotating proxy endpoint:

```
http://USERNAME:PASSWORD@p.webshare.io:80
```

### https-proxy-agent

**CRITICAL:** Use version 5, NOT version 7+.

```bash
npm install https-proxy-agent@5
```

Version 7+ is ESM-only (`import` syntax). It breaks `require()` in CommonJS projects
with `ERR_REQUIRE_ESM`. This is not fixable without converting your entire project
to ESM modules.

```javascript
// CORRECT — v5 with require()
const { HttpsProxyAgent } = require('https-proxy-agent');
const agent = new HttpsProxyAgent('http://user:pass@p.webshare.io:80');

const res = await fetch(url, { agent });
```

### When to Use Proxies

| Task | Proxy needed? |
|---|---|
| Google Autocomplete | Yes — rate limited from datacenter IPs |
| YouTube Autocomplete | Yes — same as Google |
| Serper.dev API | No — API key handles auth |
| Common Crawl CDX | No — no rate limiting |
| Page fetching (audit) | Usually no — unless scraping many pages quickly |
| Google PSI API | No — API handles auth |

---

## SQLite Caching

### Separate Tables for Different Cache Types

**CRITICAL:** SERP cache and trend cache use different column names.
Do not mix them in one table or use the wrong column name.

```sql
-- SERP results cache
CREATE TABLE IF NOT EXISTS serp_cache (
  keyword TEXT PRIMARY KEY,
  results TEXT NOT NULL,
  cached_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Trend / autocomplete cache
CREATE TABLE IF NOT EXISTS trend_cache (
  key TEXT PRIMARY KEY,
  data TEXT NOT NULL,
  cached_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Backlink cache
CREATE TABLE IF NOT EXISTS backlink_cache (
  domain TEXT PRIMARY KEY,
  links TEXT NOT NULL,
  cached_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

The `serp_cache` uses `keyword` as the primary key column.
The `trend_cache` uses `key` as the primary key column.
Mixing these up causes silent failures — queries return zero rows.

### Cache Expiry

```javascript
function isCacheValid(cachedAt, maxAgeHours = 24) {
  const age = Date.now() - new Date(cachedAt).getTime();
  return age < maxAgeHours * 60 * 60 * 1000;
}
```

Recommended cache durations:
- SERP results: 24 hours
- Autocomplete suggestions: 7 days
- Backlink data: 30 days
- Core Web Vitals: 7 days
- GSC data: 1 day

---

## PM2 Deployment

### Ecosystem Config

```javascript
// ecosystem.config.cjs
module.exports = {
  apps: [{
    name: 'seo-tool',
    script: 'server.js',
    cwd: '/var/www/seo-tool',
    env: {
      NODE_ENV: 'production',
      PORT: 3100
    },
    // Cron jobs for scheduled tasks
    cron_restart: '0 3 * * *', // restart daily at 3 AM
    max_memory_restart: '500M',
    error_file: '/var/log/pm2/seo-tool-error.log',
    out_file: '/var/log/pm2/seo-tool-out.log'
  }]
};
```

### Environment Variables

PM2 loads `.env` at startup via dotenv. Ensure dotenv is loaded at the top of your
entry point:

```javascript
require('dotenv').config();
// Now process.env.SERPER_API_KEY, GROQ_API_KEY, etc. are available
```

**Known issue:** If you add new environment variables to `.env`, you must restart
the PM2 process for them to take effect. `pm2 reload` may not pick up new vars —
use `pm2 restart` instead.

### Useful PM2 Commands

```bash
pm2 start ecosystem.config.cjs
pm2 list
pm2 logs seo-tool
pm2 restart seo-tool
pm2 stop seo-tool
pm2 delete seo-tool
pm2 save          # persist across reboots
pm2 startup       # generate startup script
```

---

## Nginx Reverse Proxy

### Basic Config with Auth

```nginx
server {
    listen 443 ssl;
    server_name seo.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/seo.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/seo.yourdomain.com/privkey.pem;

    # Prevent search engine indexing
    add_header X-Robots-Tag "noindex, nofollow" always;

    # Basic auth
    auth_basic "SEO Dashboard";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://127.0.0.1:3100;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### robots.txt

```
User-agent: *
Disallow: /
```

SEO tools should not be indexed. Add both the `X-Robots-Tag` header AND
`robots.txt` Disallow for belt-and-suspenders protection.

### SSL with Let's Encrypt

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d seo.yourdomain.com
```

Auto-renewal is set up by default. Verify:
```bash
certbot renew --dry-run
```

### Basic Auth Setup

```bash
apt install apache2-utils
htpasswd -c /etc/nginx/.htpasswd admin
# Enter password when prompted
```

---

## GSC Service Account Setup

### Step-by-Step

1. **Create GCP project**: [console.cloud.google.com](https://console.cloud.google.com)
2. **Enable Search Console API**: APIs & Services → Library → Search Console API
3. **Create service account**: IAM & Admin → Service Accounts → Create
4. **Download JSON key**: Service account → Keys → Add Key → JSON
5. **Copy to server**: `scp credentials.json user@server:/var/www/seo-tool/`
6. **Add to .env**:
   ```
   GOOGLE_APPLICATION_CREDENTIALS=/var/www/seo-tool/credentials.json
   ```
7. **Grant access in GSC**: Search Console → Settings → Users and permissions → Add user → service account email → Full permission

### Verify Connection

```javascript
const { google } = require('googleapis');
const auth = new google.auth.GoogleAuth({
  scopes: ['https://www.googleapis.com/auth/webmasters.readonly']
});
const searchconsole = google.searchconsole({ version: 'v1', auth });

// List properties to verify access
const sites = await searchconsole.sites.list();
console.log('Accessible properties:', sites.data.siteEntry?.map(s => s.siteUrl));
```

---

## Serper.dev

### Pricing

| Plan | Searches/month | Price |
|---|---|---|
| Free | 2,500 | $0 |
| Basic | 50,000 | $50/mo |
| Standard | 100,000 | $100/mo |

### API Usage

```javascript
// Standard search
const res = await fetch('https://google.serper.dev/search', {
  method: 'POST',
  headers: {
    'X-API-KEY': process.env.SERPER_API_KEY,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    q: 'seo tools 2026',
    gl: 'us',         // country
    hl: 'en',         // language
    num: 10            // results count
  })
});
const data = await res.json();
```

### Available Endpoints

| Endpoint | URL | Credits per search |
|---|---|---|
| Web search | `google.serper.dev/search` | 1 |
| News | `google.serper.dev/news` | 1 |
| Images | `google.serper.dev/images` | 1 |
| Videos | `google.serper.dev/videos` | 1 |
| Autocomplete | `google.serper.dev/autocomplete` | 1 |
| Shopping | `google.serper.dev/shopping` | 1 |

---

## Groq

### Setup

Free tier — no credit card required.

```bash
npm install groq-sdk
```

```
# .env
GROQ_API_KEY=gsk_xxxxxxxxxxxxx
```

### Models

| Model | Speed | Use case |
|---|---|---|
| llama-3.1-8b-instant | Fastest | Keyword clustering, audit fixes, sentiment |
| llama-3.1-70b-versatile | Slower | Complex analysis, content generation |

### Rate Limits (Free Tier)

- 30 requests/minute
- 14,400 tokens/minute
- 500,000 tokens/day

More than sufficient for SEO tasks. If you hit limits, add a simple queue:

```javascript
const queue = [];
let processing = false;

async function queueGroqRequest(prompt) {
  return new Promise((resolve, reject) => {
    queue.push({ prompt, resolve, reject });
    processQueue();
  });
}

async function processQueue() {
  if (processing || queue.length === 0) return;
  processing = true;
  const { prompt, resolve, reject } = queue.shift();
  try {
    const result = await callGroq(prompt);
    resolve(result);
  } catch (e) {
    reject(e);
  }
  processing = false;
  if (queue.length > 0) setTimeout(processQueue, 2000); // 2s between requests
}
```

---

## .env Template

```bash
# Serper.dev
SERPER_API_KEY=your_serper_api_key

# Groq
GROQ_API_KEY=gsk_your_groq_key

# Google Search Console
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json

# Webshare Proxy
PROXY_URL=http://user:pass@p.webshare.io:80

# DataForSEO (optional)
DATAFORSEO_LOGIN=your_login
DATAFORSEO_PASSWORD=your_password

# App
PORT=3100
NODE_ENV=production
```
