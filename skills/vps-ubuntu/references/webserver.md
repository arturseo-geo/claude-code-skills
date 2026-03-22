# Web Server Reference (Nginx, SSL, PM2)

## Nginx

### Install & Basic Commands
```bash
apt install nginx -y
systemctl enable --now nginx

nginx -t                        # test config — ALWAYS before reload
systemctl reload nginx          # apply config (graceful, no downtime)
systemctl restart nginx         # full restart (brief downtime)
nginx -s reload                 # alternative reload

# Logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### Reverse Proxy Config (Node.js / Python app)
`/etc/nginx/sites-available/myapp`:
```nginx
server {
    listen 80;
    server_name myapp.com www.myapp.com;

    # Redirect HTTP to HTTPS (after SSL setup)
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;

    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 90;
    }

    # Serve static files directly
    location /static/ {
        alias /var/www/myapp/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1000;

    # Rate limiting
    limit_req zone=api burst=20 nodelay;

    # Max upload size
    client_max_body_size 50M;
}
```

Enable site:
```bash
ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

### Nginx Rate Limiting
Add to `/etc/nginx/nginx.conf` inside `http {}`:
```nginx
# Define zones
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
limit_conn_zone $binary_remote_addr zone=conn:10m;

# In server block:
location /api/ {
    limit_req zone=api burst=20 nodelay;
    limit_conn conn 10;
}
location /login {
    limit_req zone=login burst=5 nodelay;
}
```

### Nginx Hardening
Add to `http {}` block in `/etc/nginx/nginx.conf`:
```nginx
server_tokens off;                    # hide Nginx version
client_body_timeout 12;
client_header_timeout 12;
send_timeout 10;
keepalive_timeout 15;
client_max_body_size 8m;             # adjust for your app

# Block common attack patterns
location ~* \.(git|svn|env|htaccess|htpasswd)$ { deny all; }
location ~ /\. { deny all; }
```

---

## SSL Certificates (Let's Encrypt / Certbot)

### Install & Get Certificate
```bash
apt install certbot python3-certbot-nginx -y

# Get cert + auto-configure Nginx
certbot --nginx -d myapp.com -d www.myapp.com

# Cert only (manual config)
certbot certonly --nginx -d myapp.com

# Wildcard cert (requires DNS challenge)
certbot certonly --manual --preferred-challenges dns -d "*.myapp.com"
```

### Manage Certificates
```bash
# List all certs
certbot certificates

# Renew all (usually handled by cron/systemd timer)
certbot renew

# Renew dry run
certbot renew --dry-run

# Renew specific cert
certbot renew --cert-name myapp.com

# Revoke cert
certbot revoke --cert-name myapp.com

# Delete cert
certbot delete --cert-name myapp.com
```

### Auto-renewal (Certbot installs this automatically)
```bash
# Check renewal timer
systemctl status certbot.timer
systemctl list-timers | grep certbot

# Or via cron — add if not present
echo "0 12 * * * root certbot renew --quiet --deploy-hook 'systemctl reload nginx'" > /etc/cron.d/certbot
```

### Check Certificate Expiry
```bash
# Check from command line
echo | openssl s_client -connect myapp.com:443 -servername myapp.com 2>/dev/null | openssl x509 -noout -dates

# Check all certs
certbot certificates

# Alert if expiry < 30 days
openssl s_client -connect myapp.com:443 -servername myapp.com < /dev/null 2>/dev/null | \
  openssl x509 -noout -checkend 2592000 || echo "CERT EXPIRING SOON!"
```

---

## PM2 — Node.js Process Manager

```bash
npm install -g pm2

# Start app
pm2 start app.js --name myapp
pm2 start ecosystem.config.js

# Process management
pm2 list                        # all processes
pm2 show myapp                  # details
pm2 restart myapp               # restart
pm2 reload myapp                # zero-downtime reload
pm2 stop myapp
pm2 delete myapp

# Logs
pm2 logs                        # all logs
pm2 logs myapp                  # app-specific
pm2 logs --lines 200            # last 200 lines
pm2 flush                       # clear all logs

# Auto-start on reboot
pm2 startup                     # generates systemd service
pm2 save                        # saves process list

# Monitoring
pm2 monit                       # real-time monitor
```

### PM2 Ecosystem Config (`ecosystem.config.js`)
```javascript
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'dist/index.js',
    instances: 'max',           // use all CPU cores
    exec_mode: 'cluster',       // cluster mode for load balancing
    max_memory_restart: '500M', // restart if memory exceeds 500MB
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: '/var/log/myapp/error.log',
    out_file: '/var/log/myapp/out.log',
    merge_logs: true,
    restart_delay: 3000,
    max_restarts: 10,
    watch: false                 // never watch in production
  }]
}
```

---

## systemd Services (for non-Node apps)

Create `/etc/systemd/system/myapp.service`:
```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/python3 app.py
Restart=always
RestartSec=5
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=myapp
Environment=NODE_ENV=production PORT=8000

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now myapp
systemctl status myapp
journalctl -u myapp -f          # follow logs
```
