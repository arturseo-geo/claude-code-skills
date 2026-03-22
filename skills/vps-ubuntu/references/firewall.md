# Firewall & Network Reference

## UFW (Uncomplicated Firewall) — Primary Tool

### Initial Setup (fresh server)
```bash
# Default deny all incoming, allow all outgoing
ufw default deny incoming
ufw default allow outgoing

# Allow essential services BEFORE enabling (or you'll lock yourself out)
ufw allow 22/tcp          # SSH — do this FIRST
ufw allow 80/tcp          # HTTP
ufw allow 443/tcp         # HTTPS

# Enable
ufw enable
ufw status verbose
```

### Common Rules
```bash
# Allow specific port
ufw allow 3000/tcp

# Allow port range
ufw allow 6000:6007/tcp

# Allow from specific IP only
ufw allow from 203.0.113.10 to any port 22

# Allow from subnet
ufw allow from 192.168.1.0/24

# Deny specific IP
ufw deny from 45.33.32.156

# Delete a rule (find line number first)
ufw status numbered
ufw delete 3

# Allow service by name
ufw allow 'Nginx Full'    # HTTP + HTTPS
ufw allow 'OpenSSH'
```

### Rate Limiting (brute force protection)
```bash
# Limit SSH connection attempts (6 in 30 seconds triggers block)
ufw limit ssh

# Custom rate limiting with iptables (more granular)
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
```

### UFW Logging
```bash
ufw logging on          # enable logging
ufw logging medium      # low | medium | high | full
tail -f /var/log/ufw.log
```

---

## Port Management

### Check What's Listening
```bash
ss -tulpn                          # all listening ports + process
ss -tulpn | grep LISTEN            # only active listeners
netstat -tlnp                      # alternative (install net-tools)
lsof -i -P -n | grep LISTEN        # with process names
```

### Check If Port Is Open From Outside
```bash
# From your local machine:
nc -zv SERVER_IP 22
nmap -p 22,80,443 SERVER_IP

# From the server itself:
curl -s https://portchecker.co/check?port=80
```

### Close a Port (if a service is listening unexpectedly)
```bash
# Find what's using the port
lsof -i :PORT
fuser PORT/tcp

# Stop the service
systemctl stop SERVICE_NAME
# Then block via UFW
ufw deny PORT/tcp
```

---

## iptables (Advanced / When UFW Isn't Enough)

```bash
# View rules
iptables -L -n -v --line-numbers

# Allow established connections (crucial — always set this)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT   # allow loopback

# Block an IP
iptables -A INPUT -s ATTACKER_IP -j DROP

# Save rules (persist across reboots)
apt install iptables-persistent
netfilter-persistent save

# View saved rules
cat /etc/iptables/rules.v4
```

---

## fail2ban — Intrusion Prevention

### Install & Configure
```bash
apt install fail2ban -y
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
bantime  = 3600          # ban for 1 hour (use -1 for permanent)
findtime  = 600          # within 10 minutes
maxretry = 5             # after 5 failures
banaction = ufw          # use UFW to ban (not iptables directly)

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
maxretry = 3

[nginx-http-auth]
enabled = true

[nginx-limit-req]
enabled = true
```

### Managing fail2ban
```bash
systemctl enable --now fail2ban

# Check status
fail2ban-client status
fail2ban-client status sshd

# Unban an IP
fail2ban-client set sshd unbanip 1.2.3.4

# Check your own IP isn't banned
fail2ban-client get sshd banip

# View banned IPs
iptables -L f2b-sshd -n
```

---

## Network Monitoring & Diagnostics

```bash
# Real-time bandwidth per interface
apt install nload -y && nload

# Real-time connections
watch -n1 ss -tulpn

# Traffic by process
apt install nethogs -y && nethogs

# DNS lookup
dig domain.com
host domain.com

# Trace route
mtr SERVER_IP    # better than traceroute

# Check if IP is in blocklists
curl https://api.abuseipdb.com/api/v2/check?ipAddress=IP_HERE \
  -H "Key: YOUR_API_KEY" -H "Accept: application/json"
```
