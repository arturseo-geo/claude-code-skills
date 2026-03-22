# Security Hardening Reference

## SSH Hardening

Edit `/etc/ssh/sshd_config`:
```bash
# Disable root login
 PermitRootLogin no

# Disable password auth (use keys only)
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Disable unused auth methods
 PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

# Restrict to specific users
AllowUsers deploy youruser

# Change default port (security through obscurity — update UFW too)
Port 2222

# Limit auth attempts
MaxAuthTries 3
MaxSessions 5

# Disconnect idle sessions after 15 min
ClientAliveInterval 300
ClientAliveCountMax 3

# Disable X11 forwarding (unless needed)
X11Forwarding no

# Use strong algorithms only (2026 recommendations)
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,diffie-hellman-group-exchange-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
```

**Always test before restarting:**
```bash
sshd -t                         # validate config
systemctl reload ssh            # reload (safer than restart)
```

### Add SSH Key for a New User
```bash
# On LOCAL machine — generate key (Ed25519 preferred over RSA)
ssh-keygen -t ed25519 -C "server-name-2026"

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Or manually
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

### SSH Key Rotation
```bash
# Generate new key pair
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new -C "rotated-$(date +%Y%m)"

# Add new key to server
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub user@server

# Test new key works
ssh -i ~/.ssh/id_ed25519_new user@server "echo 'New key works'"

# Remove old key from server's authorized_keys
ssh user@server "sed -i '/old-key-comment/d' ~/.ssh/authorized_keys"
```

---

## User & Privilege Management

```bash
# Create deploy user (non-root for apps)
adduser deploy
usermod -aG sudo deploy

# Create service account (no login shell)
adduser --system --no-create-home --shell /bin/false myservice

# Check sudoers
visudo                          # always use visudo — validates syntax

# List users with sudo access
grep -Po '^sudo.+:\K.*$' /etc/group

# Check who has logged in recently
last -n 30
lastb | head -20                # failed login attempts

# Lock a compromised account
passwd -l username
usermod --expiredate 1 username # disable account

# Audit all user accounts with login shells
awk -F: '$7 !~ /(nologin|false)/ {print $1, $7}' /etc/passwd
```

---

## Automatic Security Updates

```bash
apt install unattended-upgrades -y
dpkg-reconfigure --priority=low unattended-upgrades
```

Edit `/etc/apt/apt.conf.d/50unattended-upgrades`:
```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";   // set true if OK to reboot
Unattended-Upgrade::Mail "you@example.com";
```

Edit `/etc/apt/apt.conf.d/20auto-upgrades`:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

```bash
# Test it
unattended-upgrades --dry-run --debug
# View log
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## Kernel & Sysctl Hardening

Edit `/etc/sysctl.d/99-hardening.conf`:
```ini
# Disable IP forwarding (unless this is a router/VPN server)
net.ipv4.ip_forward = 0

# Prevent IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignore ICMP broadcast requests (smurf attacks)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP errors
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Log suspicious packets
net.ipv4.conf.all.log_martians = 1

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Protect against SYN flood attacks
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# Randomise virtual address space (ASLR)
kernel.randomize_va_space = 2

# Restrict kernel logs to root only
kernel.dmesg_restrict = 1

# Prevent core dumps leaking sensitive info
fs.suid_dumpable = 0

# Restrict ptrace (prevent process snooping)
kernel.yama.ptrace_scope = 2

# Restrict unprivileged user namespaces (mitigates container escape vectors)
kernel.unprivileged_userns_clone = 0
```

```bash
sysctl -p /etc/sysctl.d/99-hardening.conf   # apply immediately
```

---

## Intrusion Detection

### rkhunter (Rootkit Hunter)
```bash
apt install rkhunter -y
rkhunter --update
rkhunter --propupd          # baseline scan (run after clean install)
rkhunter --check --skip-keypress --report-warnings-only

# Set up daily scan via cron
echo "0 3 * * * root rkhunter --check --skip-keypress --report-warnings-only 2>&1 | mail -s 'rkhunter report' you@example.com" > /etc/cron.d/rkhunter
```

#### rkhunter Configuration
Edit `/etc/rkhunter.conf` for common tuning:
```bash
# Reduce false positives after package updates
PKGMGR=DPKG
UPDATE_MIRRORS=1
MIRRORS_MODE=0
WEB_CMD=""

# Whitelist known safe items (adjust per server)
ALLOWHIDDENDIR=/etc/.java
ALLOW_SSH_ROOT_USER=no
ALLOW_SSH_PROT_V1=0
```

```bash
# Update baseline after system changes (apt upgrade, new packages)
rkhunter --propupd

# View detailed report
cat /var/log/rkhunter.log | grep Warning
```

### chkrootkit
```bash
apt install chkrootkit -y
chkrootkit

# Run weekly
echo "0 4 * * 0 root chkrootkit 2>&1 | mail -s 'chkrootkit report' you@example.com" > /etc/cron.d/chkrootkit

# Common false positives — check these manually if flagged
# - Suckit rootkit: often false positive on modern kernels
# - Hidden processes: can be transient kernel threads
```

### ClamAV (Antivirus)
```bash
apt install clamav clamav-daemon -y
systemctl stop clamav-freshclam
freshclam                   # update signatures
systemctl start clamav-freshclam

# Scan a directory
clamscan -r /home --infected --remove

# Scan with logging and summary
clamscan -r /var/www --infected --log=/var/log/clamav/scan.log --exclude-dir=/var/www/.git

# Schedule weekly scan
echo "0 2 * * 0 root clamscan -r /var/www /home --infected --log=/var/log/clamav/weekly-scan.log 2>&1" > /etc/cron.d/clamav
```

#### ClamAV On-Access Scanning (real-time file monitoring)
```bash
# Enable the clamonacc daemon for real-time scanning
cat >> /etc/clamav/clamd.conf << EOF
OnAccessIncludePath /var/www
OnAccessIncludePath /home
OnAccessExcludeUname clamav
OnAccessPrevention yes
EOF

systemctl restart clamav-daemon
clamonacc                   # start on-access scanner
```

#### ClamAV Signature Management
```bash
# Check signature freshness
sigtool --info /var/lib/clamav/daily.cvd

# Force update
systemctl stop clamav-freshclam
freshclam --verbose
systemctl start clamav-freshclam

# Add custom signatures (e.g., for web shells)
# Place .ndb or .ldb files in /var/lib/clamav/
```

### auditd (System Call Auditing)
```bash
apt install auditd audispd-plugins -y
systemctl enable --now auditd

# Watch file changes (detect tampering)
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/shadow -p wa -k shadow_changes
auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config
auditctl -w /etc/sudoers -p wa -k sudoers
auditctl -w /etc/crontab -p wa -k crontab_changes

# Monitor command execution by non-root users
auditctl -a always,exit -F arch=b64 -S execve -F uid!=0 -k user_commands

# Monitor network connections
auditctl -a always,exit -F arch=b64 -S connect -S accept -k network_connections

# View audit log
ausearch -k passwd_changes
ausearch -k user_commands --start today
aureport --summary
aureport --auth             # authentication report
aureport --login            # login attempts
aureport --file --summary   # file access summary

# Make rules persistent: /etc/audit/rules.d/audit.rules
# Copy current rules
auditctl -l > /etc/audit/rules.d/audit.rules
```

#### auditd Log Management
```bash
# Configure log rotation in /etc/audit/auditd.conf
max_log_file = 50           # MB per log file
num_logs = 10               # number of rotated logs
max_log_file_action = ROTATE

# Search audit logs by time range
ausearch --start '03/20/2026' '00:00:00' --end '03/21/2026' '23:59:59'

# Generate comprehensive daily report
aureport --start today --summary
```

---

## File Integrity & Permissions Audit

```bash
# Find world-writable files (potential security risk)
find / -xdev -type f -perm -0002 -not -path "/proc/*" 2>/dev/null

# Find SUID/SGID binaries (can escalate privileges)
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null

# Find files with no owner (orphaned)
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null

# Check for recently modified system files (last 24h)
find /etc /bin /sbin /usr/bin /usr/sbin -newer /tmp -type f 2>/dev/null

# Correct key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub
```

### AIDE (Advanced Intrusion Detection Environment)
```bash
apt install aide -y

# Initialize database (takes a while — baseline of all files)
aideinit
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Check for changes
aide --check

# Update database after legitimate changes
aide --update
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Schedule daily integrity check
echo "0 5 * * * root aide --check 2>&1 | mail -s 'AIDE report' you@example.com" > /etc/cron.d/aide
```

---

## Security Audit Checklist

Run monthly or after any incident:
- [ ] `ufw status verbose` — firewall active and rules correct?
- [ ] `fail2ban-client status` — running? ban counts reasonable?
- [ ] `last -n 50` — any unexpected logins?
- [ ] `grep "Failed password" /var/log/auth.log | wc -l` — brute force activity?
- [ ] `apt list --upgradable` — security patches pending?
- [ ] `rkhunter --check` — clean?
- [ ] `chkrootkit` — clean?
- [ ] `clamscan -r /var/www --infected` — no malware?
- [ ] `aureport --auth --summary` — audit log reviewed?
- [ ] `systemctl list-units --failed` — any crashed services?
- [ ] `ls -la /root/.ssh/authorized_keys` — only expected keys?
- [ ] `cat /etc/passwd | grep -v nologin | grep -v false` — unexpected login shells?
- [ ] SSL cert expiry: `certbot certificates`
- [ ] Docker images: `docker images --filter "dangling=true"` — stale images?
- [ ] Open ports: `ss -tulpn` — any unexpected listeners?
