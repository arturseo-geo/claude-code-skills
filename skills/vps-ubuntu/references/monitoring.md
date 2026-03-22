# Monitoring, Disk & Memory Reference

## Disk Space Management

### Check Usage
```bash
df -h                          # disk usage by filesystem
df -h /var /home /             # specific mountpoints
du -sh /*  2>/dev/null         # top-level directory sizes
du -sh /var/log/* | sort -rh   # log sizes sorted
du -sh /var/www/* | sort -rh   # web app sizes

# Find largest files
find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -20

# Find files changed in last 24 hours (useful for finding log runaway)
find /var/log -newer /tmp -type f -exec ls -lh {} \; 2>/dev/null
```

### Free Up Disk Space
```bash
# Clean apt cache
apt clean && apt autoremove -y

# Clean systemd journal (keep last 7 days or max 100MB)
journalctl --vacuum-time=7d
journalctl --vacuum-size=100M

# Find and remove old kernels
dpkg --list | grep linux-image   # list installed kernels
apt autoremove --purge           # removes old kernels automatically

# Remove Docker junk
docker system prune -af          # removes stopped containers, unused images, volumes

# Find and delete old log files
find /var/log -name "*.gz" -mtime +30 -delete
find /var/log -name "*.1" -mtime +14 -delete

# Truncate (don't delete) an actively-written log
> /var/log/some-large.log        # truncates without breaking the process
```

### Disk Alerts — Setup
Add to cron (`crontab -e`):
```bash
# Alert when any disk partition exceeds 80%
*/10 * * * * df -h | awk '0+$5 >= 80 {print}' | grep -v "^$" | mail -s "DISK ALERT $(hostname)" you@example.com
```

Or use this script `/usr/local/bin/disk-alert.sh`:
```bash
#!/bin/bash
THRESHOLD=80
EMAIL="you@example.com"

df -H | grep -vE '^Filesystem|tmpfs|cdrom' | awk '{print $5 " " $6}' | while read output; do
  usage=$(echo $output | awk '{print $1}' | tr -d '%')
  partition=$(echo $output | awk '{print $2}')
  if [ $usage -ge $THRESHOLD ]; then
    echo "ALERT: $partition is ${usage}% full on $(hostname)" | \
      mail -s "Disk Space Alert: $(hostname)" $EMAIL
  fi
done
```

---

## Memory Management

### Check Usage
```bash
free -h                        # RAM + swap summary
free -h -s 2                   # refresh every 2 seconds
cat /proc/meminfo              # detailed breakdown
vmstat -s                      # virtual memory stats

# Top memory consumers
ps aux --sort=-%mem | head -15
ps aux --sort=-%mem | awk 'NR>1{sum += $4} END {print "Total %MEM:", sum}'

# Memory per process
pmap PID                       # memory map of specific process
```

### Swap Management
```bash
# Check swap
swapon --show
cat /proc/swaps

# Add a swapfile (useful on RAM-constrained VPS)
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make permanent — add to /etc/fstab
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Tune swappiness (default 60 — lower = prefer RAM, higher = use swap earlier)
sysctl vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.d/99-swap.conf

# Remove a swapfile
swapoff /swapfile
rm /swapfile
```

### OOM (Out of Memory) Events
```bash
# Check if OOM killer has fired
dmesg | grep -i "out of memory"
dmesg | grep -i "killed process"
grep -i "oom" /var/log/syslog | tail -20

# Protect a critical process from OOM killer
echo -1000 > /proc/$(pgrep nginx)/oom_score_adj    # -1000 = never kill
echo 1000 > /proc/$(pgrep java)/oom_score_adj      # 1000 = kill first
```

---

## CPU & Load Monitoring

```bash
# Current load
uptime
cat /proc/loadavg             # raw: 1min 5min 15min

# CPU usage
top                           # interactive
htop                          # better interactive (apt install htop)
mpstat 1 5                    # 5 samples, 1 second apart

# Number of CPUs (to compare against load average)
nproc
cat /proc/cpuinfo | grep "processor" | wc -l

# Process tree
pstree -p

# See what a process is doing
strace -p PID                  # system calls (verbose)
lsof -p PID                   # open files/connections
```

---

## Log Monitoring & Analysis

### Key Log Locations
| Log | Path | What it covers |
|---|---|---|
| Auth / SSH | `/var/log/auth.log` | Logins, sudo, SSH |
| Syslog | `/var/log/syslog` | General system events |
| Kernel | `/var/log/kern.log` | Kernel messages |
| Apt/dpkg | `/var/log/dpkg.log` | Package changes |
| Nginx | `/var/log/nginx/access.log` | HTTP requests |
| Nginx errors | `/var/log/nginx/error.log` | HTTP errors |
| MySQL | `/var/log/mysql/error.log` | DB errors |
| Mail | `/var/log/mail.log` | Email delivery |

### Useful Log Commands
```bash
# Follow a log in real time
tail -f /var/log/syslog
tail -f /var/log/nginx/access.log

# Follow multiple logs at once
tail -f /var/log/syslog /var/log/auth.log

# Search logs
grep "error" /var/log/nginx/error.log | tail -50
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Systemd journal (more powerful)
journalctl -f                          # follow all
journalctl -u nginx -f                 # follow specific service
journalctl -p err --since "1 hour ago" # errors in last hour
journalctl --since "2026-03-20" --until "2026-03-21"
```

### logwatch — Daily Log Summary Emails
```bash
apt install logwatch -y

# Configure
cat > /etc/logwatch/conf/logwatch.conf << EOF
Output = mail
MailTo = you@example.com
MailFrom = logwatch@$(hostname)
Detail = Med
Range = yesterday
EOF

# Test
logwatch --output stdout --detail Med

# Auto-run daily (already in /etc/cron.daily/ after install)
```

### Log Rotation
```bash
# Check config
cat /etc/logrotate.conf
ls /etc/logrotate.d/

# Example app config /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data www-data
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}

# Test logrotate config
logrotate -d /etc/logrotate.d/myapp    # dry run
logrotate -f /etc/logrotate.conf       # force immediate rotation
```

---

## Alerting — Email, Slack, and Webhook Notifications

### Email Alerts with msmtp (lightweight SMTP relay)
```bash
apt install msmtp msmtp-mta -y

# Configure /etc/msmtprc
cat > /etc/msmtprc << 'EOF'
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        default
host           smtp.gmail.com
port           587
from           alerts@yourdomain.com
user           alerts@yourdomain.com
password       your-app-password
EOF
chmod 600 /etc/msmtprc

# Test
echo "Test alert from $(hostname)" | mail -s "Test Alert" you@example.com
```

### Slack Webhook Alerts
```bash
#!/bin/bash
# /usr/local/bin/slack-alert.sh
# Usage: slack-alert.sh "Alert message here"

WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
HOSTNAME=$(hostname)
MESSAGE="${1:-No message provided}"

curl -s -X POST "$WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  -d "{
    \"text\": \"*[$HOSTNAME]* $MESSAGE\",
    \"username\": \"VPS Alert\",
    \"icon_emoji\": \":warning:\"
  }"
```

### Telegram Bot Alerts
```bash
#!/bin/bash
# /usr/local/bin/telegram-alert.sh
# Usage: telegram-alert.sh "Alert message here"

BOT_TOKEN="YOUR_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"
MESSAGE="[$(hostname)] ${1:-No message provided}"

curl -s "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -d "chat_id=${CHAT_ID}" \
  -d "text=${MESSAGE}" \
  -d "parse_mode=Markdown"
```

### Generic Webhook Alert
```bash
#!/bin/bash
# /usr/local/bin/webhook-alert.sh
# Works with Ntfy, Gotify, Pushover, Discord, etc.

WEBHOOK_URL="${WEBHOOK_URL:-https://ntfy.sh/your-topic}"
MESSAGE="${1:-Alert from $(hostname)}"

curl -s -X POST "$WEBHOOK_URL" \
  -H "Title: VPS Alert" \
  -H "Priority: high" \
  -d "$MESSAGE"
```

---

## Health Check Endpoints

### HTTP Health Check Script
Create an endpoint for uptime monitoring services (UptimeRobot, Hetrix, etc.):

```bash
#!/bin/bash
# /usr/local/bin/health-endpoint.sh
# Serve via Nginx or run as a simple Python HTTP check

# Check all critical services
HEALTHY=true
CHECKS=""

# Nginx running
if ! systemctl is-active --quiet nginx; then
  HEALTHY=false
  CHECKS="$CHECKS nginx:DOWN"
else
  CHECKS="$CHECKS nginx:OK"
fi

# Database accessible
if ! pg_isready -q 2>/dev/null && ! mysqladmin ping -s 2>/dev/null; then
  HEALTHY=false
  CHECKS="$CHECKS db:DOWN"
else
  CHECKS="$CHECKS db:OK"
fi

# Disk not full (>95%)
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$DISK_USAGE" -gt 95 ]; then
  HEALTHY=false
  CHECKS="$CHECKS disk:CRITICAL(${DISK_USAGE}%)"
else
  CHECKS="$CHECKS disk:OK(${DISK_USAGE}%)"
fi

# Memory not exhausted (>95%)
MEM_USAGE=$(free | awk '/^Mem:/ {printf "%.0f", $3/$2*100}')
if [ "$MEM_USAGE" -gt 95 ]; then
  HEALTHY=false
  CHECKS="$CHECKS mem:CRITICAL(${MEM_USAGE}%)"
else
  CHECKS="$CHECKS mem:OK(${MEM_USAGE}%)"
fi

if [ "$HEALTHY" = true ]; then
  echo "OK |$CHECKS"
  exit 0
else
  echo "UNHEALTHY |$CHECKS"
  exit 1
fi
```

### Nginx Health Check Location
```nginx
# Add to your Nginx server block
location /health {
    access_log off;
    allow 127.0.0.1;          # restrict to localhost
    allow 100.0.0.0/8;        # allow Tailscale
    deny all;
    default_type text/plain;
    return 200 "OK\n";
}

# Or proxy to health check script
location /health {
    access_log off;
    content_by_lua_block {
        local handle = io.popen("/usr/local/bin/health-endpoint.sh")
        local result = handle:read("*a")
        handle:close()
        ngx.say(result)
    }
}
```

---

## Monitoring Stack Options

### Lightweight (single VPS)
- **Netdata** — beautiful real-time dashboard, zero config: `bash <(curl -Ss https://my-netdata.io/kickstart.sh)`
- **Glances** — terminal dashboard: `apt install glances -y && glances`
- **logwatch** — daily email digest (see above)
- **Monit** — process supervisor with built-in alerts

### Monit — Process Monitoring with Auto-Restart
```bash
apt install monit -y

# Configure /etc/monit/conf.d/myapp
check process nginx with pidfile /var/run/nginx.pid
  start program = "/bin/systemctl start nginx"
  stop program = "/bin/systemctl stop nginx"
  if failed host 127.0.0.1 port 80 protocol http then restart
  if 5 restarts within 5 cycles then alert

check system $HOST
  if loadavg (5min) > 4 then alert
  if memory usage > 90% then alert
  if cpu usage > 95% for 10 cycles then alert

set mailserver smtp.gmail.com port 587
  username "alerts@yourdomain.com" password "app-password"
  using tls
set alert you@example.com

# Enable web interface (optional)
set httpd port 2812
  allow admin:secret

systemctl enable --now monit
monit status              # check all monitored services
monit summary             # brief overview
```

### Custom Alert Script
`/usr/local/bin/health-check.sh` — run every 5 min via cron:
```bash
#!/bin/bash
ALERT_CMD="/usr/local/bin/slack-alert.sh"  # or telegram-alert.sh, or mail
HOST=$(hostname)
ALERTS=""

# CPU load > number of cores
LOAD=$(cat /proc/loadavg | awk '{print $1}')
CORES=$(nproc)
if (( $(echo "$LOAD > $CORES" | bc -l) )); then
  ALERTS="${ALERTS}HIGH LOAD: $LOAD (${CORES} cores)\n"
fi

# RAM usage > 90%
MEM_USED=$(free | awk '/^Mem:/ {printf "%.0f", $3/$2*100}')
if [ "$MEM_USED" -gt 90 ]; then
  ALERTS="${ALERTS}HIGH MEMORY: ${MEM_USED}% used\n"
fi

# Disk > 85%
while read -r line; do
  USAGE=$(echo "$line" | awk '{print $5}' | tr -d '%')
  MOUNT=$(echo "$line" | awk '{print $6}')
  if [ "$USAGE" -gt 85 ]; then
    ALERTS="${ALERTS}DISK FULL: $MOUNT at ${USAGE}%\n"
  fi
done < <(df -h | grep -vE '^Filesystem|tmpfs')

# Check critical services
for SVC in nginx ssh; do
  if ! systemctl is-active --quiet "$SVC"; then
    ALERTS="${ALERTS}SERVICE DOWN: $SVC\n"
  fi
done

# Send alert if anything triggered
if [ -n "$ALERTS" ]; then
  $ALERT_CMD "$(echo -e "$ALERTS")"
fi
```

```bash
chmod +x /usr/local/bin/health-check.sh
echo "*/5 * * * * root /usr/local/bin/health-check.sh" > /etc/cron.d/health-check
```
