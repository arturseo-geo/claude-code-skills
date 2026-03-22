---
name: vps-ubuntu
description: >
  Manage, secure, monitor, and maintain Ubuntu VPS servers. Use this skill
  whenever the user mentions VPS, Ubuntu server, Linux server, firewall, UFW,
  iptables, fail2ban, SSH hardening, security hardening, server security,
  intrusion detection, rootkit detection, rkhunter, chkrootkit, ClamAV, auditd,
  server backups, rsync backup, borgbackup, automated backups, rclone, cron jobs,
  crontab, scheduled tasks, disk space, df, du, disk usage, out of disk,
  memory usage, swap, RAM, server monitoring, htop, netstat, ss, open ports,
  system health, server performance, log rotation, logwatch, unattended upgrades,
  automatic updates, server hardening, sysctl, kernel parameters, brute force
  protection, Nginx, Apache, SSL certificates, Certbot, Let's Encrypt, PM2,
  systemd services, process management, server deployment, Docker, containers,
  Docker Compose, Tailscale, WireGuard, VPN, server provisioning, or any task
  involving managing a Linux/Ubuntu VPS or dedicated server.
---

# VPS Ubuntu Management Skill

This skill covers the full lifecycle of Ubuntu VPS management.
Load the relevant reference file for detailed commands:

| Topic | Reference file | Load when... |
|---|---|---|
| Firewall & network | `references/firewall.md` | UFW, iptables, open ports, rate limiting |
| Security hardening | `references/security.md` | SSH, fail2ban, kernel params, auditing, rootkits, ClamAV |
| Backups | `references/backups.md` | rsync, Borg, rclone, cloud sync, backup testing |
| Monitoring & alerts | `references/monitoring.md` | disk, memory, CPU, log analysis, alerting, health checks |
| Cron jobs | `references/cron.md` | crontab syntax, scheduling, debugging, systemd timers |
| Web server | `references/webserver.md` | Nginx, SSL, PM2, reverse proxy, compression, rate limiting |
| Containers | `references/containers.md` | Docker, Docker Compose, networking, image cleanup |

---

## Server Provisioning — Fresh Ubuntu VPS

Run these steps immediately after first SSH login to a new server:

```bash
# 1. Update system
apt update && apt upgrade -y

# 2. Set hostname and timezone
hostnamectl set-hostname myserver
timedatectl set-timezone UTC   # or your preferred timezone

# 3. Create deploy user (never run apps as root)
adduser deploy
usermod -aG sudo deploy

# 4. Copy SSH key to deploy user
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh && chmod 600 /home/deploy/.ssh/authorized_keys

# 5. Firewall — allow SSH before enabling
ufw default deny incoming && ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable

# 6. Harden SSH (see references/security.md for full config)
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sshd -t && systemctl reload ssh

# 7. Install essentials
apt install -y curl wget git unzip htop tmux fail2ban ufw \
  certbot python3-certbot-nginx logwatch unattended-upgrades

# 8. Enable automatic security updates
dpkg-reconfigure --priority=low unattended-upgrades

# 9. Set up fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
systemctl enable --now fail2ban
```

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
- [ ] **Docker**: `docker system df` — disk consumed by images/volumes?
- [ ] **Tailscale**: `tailscale status` — connected to mesh?

---

## Emergency Procedures

### Server is unreachable / SSH locked out
1. Use VPS provider's web console (DigitalOcean, Hetzner, Vultr all have one)
2. Check if UFW blocked your IP: `ufw status` then `ufw allow from YOUR_IP`
3. fail2ban may have banned you: `fail2ban-client status sshd` then `fail2ban-client set sshd unbanip YOUR_IP`
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
- `docker system prune -a` — removes all unused images, not just dangling

Always test SSH config before restarting:
```bash
sshd -t   # test config validity — ALWAYS run before: systemctl restart ssh
```

---

## Tailscale VPN Setup & SSH Connection

### Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --ssh          # enable Tailscale SSH (optional)
tailscale status            # verify connection
tailscale ip -4             # get Tailscale IPv4 address
```

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
in 60-90 seconds, which silently kills the WireGuard tunnel Tailscale uses.
`ServerAliveInterval 60` keeps SSH traffic flowing through the tunnel, preventing the drop.

### Reconnect Command (if session drops anyway)
```bash
tailscale status           # verify tunnel is up
ssh vps-tailscale          # reconnect using config alias
```

### WireGuard (manual alternative to Tailscale)
```bash
apt install wireguard -y

# Generate keys
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key
chmod 600 /etc/wireguard/private.key

# Config: /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <server-private-key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.0.0.2/32

# Enable
systemctl enable --now wg-quick@wg0
ufw allow 51820/udp
```

---

## Unattended Upgrades

```bash
apt install unattended-upgrades -y
dpkg-reconfigure --priority=low unattended-upgrades

# Verify enabled
cat /etc/apt/apt.conf.d/20auto-upgrades
# Should show:
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";

# Check what will be upgraded
unattended-upgrades --dry-run --debug

# View upgrade log
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## Log Rotation Quick Reference

```bash
# Check existing configs
ls /etc/logrotate.d/

# Test config (dry run)
logrotate -d /etc/logrotate.d/myapp

# Force immediate rotation
logrotate -f /etc/logrotate.conf

# Example: /etc/logrotate.d/myapp
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
```
