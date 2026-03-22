# Cron Jobs Reference

## Crontab Syntax

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, 0=Sun, 7=Sun, or names: Mon,Tue...)
│ │ │ └──── Month (1-12 or names: Jan,Feb...)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

### Common Schedules
```bash
# Every minute
* * * * *

# Every 5 minutes
*/5 * * * *

# Every hour at :00
0 * * * *

# Daily at 2:30am
30 2 * * *

# Weekly — every Sunday at 4am
0 4 * * 0

# Monthly — 1st of month at midnight
0 0 1 * *

# Weekdays only at 9am
0 9 * * 1-5

# Every 6 hours
0 */6 * * *

# Twice daily (2am and 2pm)
0 2,14 * * *
```

---

## Managing Crontabs

```bash
# Edit current user's crontab
crontab -e

# Edit root's crontab
crontab -e -u root
sudo crontab -e

# List crontabs
crontab -l
crontab -l -u username

# Remove all crontabs for user
crontab -r

# System-wide cron files (preferred for services)
ls /etc/cron.d/         # per-job files with user field
ls /etc/cron.daily/     # run daily by run-parts
ls /etc/cron.weekly/    # run weekly
ls /etc/cron.monthly/   # run monthly
```

### System cron.d format (includes user field)
```bash
# /etc/cron.d/myapp — note the username column
30 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

---

## Logging Cron Output

```bash
# Redirect all output to log file
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Append with timestamp
0 2 * * * echo "=== $(date) ===" >> /var/log/backup.log && /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Suppress output entirely (silent cron)
0 2 * * * /usr/local/bin/backup.sh > /dev/null 2>&1

# Email output on failure only (via MAILTO)
MAILTO=you@example.com
0 2 * * * /usr/local/bin/backup.sh > /dev/null   # emails only on non-zero exit
```

---

## Debugging Cron Jobs

Cron runs in a minimal environment — most failures are due to missing PATH or env vars.

```bash
# Check if cron is running
systemctl status cron

# View cron execution log
grep CRON /var/log/syslog | tail -30
journalctl -u cron -n 50

# Test your script manually as root first
sudo bash /usr/local/bin/my-script.sh

# Simulate cron's minimal environment
env -i HOME=/root SHELL=/bin/bash PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin /usr/local/bin/my-script.sh
```

### Common Cron Failures & Fixes
| Problem | Symptom | Fix |
|---|---|---|
| Missing PATH | "command not found" in log | Use full paths: `/usr/bin/python3` not `python3` |
| Missing env vars | Script works manually, fails in cron | Export vars at top of script or set in crontab |
| Wrong permissions | Silent failure | `chmod +x /path/to/script` |
| Newline at end of crontab | Job never runs | Ensure crontab file ends with newline |
| Special chars (`%`) | Job fails silently | Escape with `\%` |

### Good Script Header for Cron
```bash
#!/bin/bash
set -euo pipefail        # exit on error, undefined vars, pipe failures
export PATH="/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin"
export HOME="/root"
LOG="/var/log/myjob.log"
exec >> "$LOG" 2>&1      # redirect all output to log
echo "=== $(date '+%Y-%m-%d %H:%M:%S') START ==="
```

---

## Essential Cron Job Templates

### Daily backup
```bash
# /etc/cron.d/daily-backup
30 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### Disk space alert
```bash
# /etc/cron.d/disk-alert
*/10 * * * * root /usr/local/bin/disk-alert.sh
```

### Renew SSL certificates
```bash
# /etc/cron.d/certbot
0 12 * * * root certbot renew --quiet --deploy-hook "systemctl reload nginx"
```

### Clear temp files
```bash
# /etc/cron.d/cleanup
0 4 * * 0 root find /tmp -type f -mtime +7 -delete && find /var/tmp -type f -mtime +30 -delete
```

### Rotate custom application logs
```bash
# /etc/cron.d/app-logs
0 0 * * * root find /var/log/myapp -name "*.log" -mtime +14 -exec gzip {} \; && find /var/log/myapp -name "*.log.gz" -mtime +30 -delete
```

### Database backup with retention
```bash
# /etc/cron.d/db-backup
0 3 * * * root mysqldump -u root -p'PASSWORD' mydb | gzip > /backups/db/mydb_$(date +\%Y\%m\%d).sql.gz && find /backups/db -name "*.sql.gz" -mtime +30 -delete
```

---

## Cron Alternatives

| Tool | When to use |
|---|---|
| **systemd timers** | Modern alternative — better logging, dependencies, retry logic |
| **anacron** | Runs missed jobs after downtime (good for VPS that gets rebooted) |
| **fcron** | Combines cron + anacron features |

### systemd timer example (preferred for services)
```bash
# /etc/systemd/system/mybackup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true      # run if missed (e.g. after reboot)
RandomizedDelaySec=10min  # avoid thundering herd

[Install]
WantedBy=timers.target
```

```bash
# /etc/systemd/system/mybackup.service
[Unit]
Description=Daily backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=root
```

```bash
systemctl enable --now mybackup.timer
systemctl list-timers     # view all timers + next run time
journalctl -u mybackup    # view logs
```
