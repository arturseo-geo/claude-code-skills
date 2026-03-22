---
name: vps-ubuntu
description: >
  Manage, secure, monitor, and maintain Ubuntu VPS servers. Use this skill
  whenever the user mentions VPS, Ubuntu server, Linux server, firewall, UFW,
  iptables, fail2ban, SSH hardening, security hardening, server security,
  intrusion detection, rootkit detection, rkhunter, chkrootkit, ClamAV, auditd,
  server backups, rsync backup, borgbackup, automated backups, cron jobs,
  crontab, scheduled tasks, disk space, df, du, disk usage, out of disk,
  memory usage, swap, RAM, server monitoring, htop, netstat, ss, open ports,
  system health, server performance, log rotation, logwatch, unattended upgrades,
  automatic updates, server hardening, sysctl, kernel parameters, brute force
  protection, Nginx, Apache, SSL certificates, Certbot, Let's Encrypt, PM2,
  systemd services, process management, server deployment, or any task involving
  managing a Linux/Ubuntu VPS or dedicated server.
---

# VPS Ubuntu Management Skill

This skill covers the full lifecycle of Ubuntu VPS management.
Load the relevant reference file for detailed commands:

| Topic | Reference file | Load when... |
|---|---|---|
| Firewall & network | `references/firewall.md` | UFW, iptables, open ports, rate limiting |
| Security hardening | `references/security.md` | SSH, fail2ban, kernel params, auditing |
| Backups | `references/backups.md` | rsync, Borg, automated backup scripts |
| Monitoring & alerts | `references/monitoring.md` | disk, memory, CPU, log analysis |
| Cron jobs | `references/cron.md` | crontab syntax, scheduling, debugging |
| Web server | `references/webserver.md` | Nginx, SSL, PM2, reverse proxy |

---

## Quick Diagnostics — Run These First

When a user reports a server problem, start with:

```bash
# System overview
uptime && free -h && df -h

# Top processes
ps aux --sort=-%cpu | head -15

# Network connections
ss -tulpn

# Recent errors
journalctl -p err -n 50 --no-pager

# Failed systemd services
systemctl --failed

# Last logins and failed SSH attempts
last -n 20
grep "Failed password" /var/log/auth.log | tail -20
```

---

## Server Health Scorecard

Before doing anything else on a VPS, run this mental checklist:

- [ ] **Disk**: `df -h` — any partition >80% full?
- [ ] **Memory**: `free -h` — swap in use? High?
- [ ] **Load**: `uptime` — load average > number of CPUs?
- [ ] **Firewall**: `ufw status` — active and rules correct?
- [ ] **Updates**: `apt list --upgradable 2>/dev/null | wc -l` — security updates pending?
- [ ] **Failed services**: `systemctl --failed` — anything crashed?
- [ ] **Auth log**: `grep "Failed password" /var/log/auth.log | wc -l` — brute force?
- [ ] **Backups**: last backup timestamp — recent and verified?
- [ ] **SSL certs**: `certbot certificates` — expiry within 30 days?

---

## Emergency Procedures

### Server is unreachable / SSH locked out
1. Use VPS provider's web console (DigitalOcean, Hetzner, Vultr all have one)
2. Check if UFW blocked your IP: `ufw status` → `ufw allow from YOUR_IP`
3. fail2ban may have banned you: `fail2ban-client status sshd` → `fail2ban-client set sshd unbanip YOUR_IP`
4. Check SSH service: `systemctl status ssh`

### Disk 100% full — server broken
```bash
# Find what's eating space
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /var/log/* | sort -rh | head -10

# Emergency cleanup
journalctl --vacuum-size=100M        # trim systemd logs
apt clean                            # clear apt cache
docker system prune -f 2>/dev/null   # clear Docker if installed
find /tmp -mtime +7 -delete          # old temp files

# Rotate logs immediately
logrotate -f /etc/logrotate.conf
```

### Server under attack / high load
```bash
# See what's hammering the server
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20

# Block an IP immediately
ufw deny from ATTACKER_IP

# Find CPU-hungry processes
ps aux --sort=-%cpu | head -10

# Kill runaway process
kill -9 PID
```

### Memory exhausted / OOM killer
```bash
# Check OOM kill history
dmesg | grep -i "killed process"
grep -i "out of memory" /var/log/syslog | tail -20

# See what's using RAM
ps aux --sort=-%mem | head -15
free -h

# Add emergency swap (if none exists)
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

---

## Safety Rules — Always Follow

Before running any destructive command, state it and ask for confirmation:
- `rm -rf` anything — STOP, confirm path and scope
- `ufw reset` — will drop ALL firewall rules
- `iptables -F` — flushes all rules, server becomes open
- Editing `/etc/ssh/sshd_config` — test before reload or you risk lockout
- `reboot` / `shutdown` — confirm timing with user
- Dropping a database — always confirm and verify backup exists first

Always test SSH config before restarting:
```bash
sshd -t   # test config validity — ALWAYS run before: systemctl restart ssh
```

---

## Tailscale SSH Connection (Project Setup)

This VPS connects via Tailscale. Claude Code SSHs through the Tailscale IP directly.

### Keep-Alive Configuration (prevents drop after inactivity)

**Local `~/.ssh/config`** — prevents client-side timeout:
```
Host vps-tailscale
    HostName 100.x.x.x
    User youruser
    ServerAliveInterval 60
    ServerAliveCountMax 10
    TCPKeepAlive yes
    ConnectTimeout 30
```

**VPS `/etc/ssh/sshd_config`** — prevents server-side timeout:
```
ClientAliveInterval 120
ClientAliveCountMax 10
TCPKeepAlive yes
```
Always run `sshd -t && systemctl reload ssh` after editing.

**Why it drops**: VPS providers (Hetzner, DO, Vultr) NAT-timeout idle UDP connections
in 60–90 seconds, which silently kills the WireGuard tunnel Tailscale uses.
`ServerAliveInterval 60` keeps SSH traffic flowing through the tunnel, preventing the drop.

### Reconnect Command (if session drops anyway)
```bash
tailscale status           # verify tunnel is up
ssh vps-tailscale          # reconnect using config alias
```
