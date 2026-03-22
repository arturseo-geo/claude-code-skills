# Backups Reference

## Backup Strategy — 3-2-1 Rule
- **3** copies of data
- **2** different storage media/locations
- **1** offsite (remote server, S3, Backblaze B2)

---

## rsync — File & Directory Backups

### Basic rsync backup
```bash
# Backup local directory to remote
rsync -avz --delete /var/www/ user@backup-server:/backups/www/

# Backup with exclusions
rsync -avz --delete \
  --exclude='*.log' \
  --exclude='node_modules/' \
  --exclude='.git/' \
  /var/www/ user@backup-server:/backups/www/

# Dry run first (always test)
rsync -avzn --delete /var/www/ user@backup-server:/backups/www/

# Bandwidth-limited backup (kB/s — avoids saturating connection)
rsync -avz --bwlimit=5000 /var/www/ user@backup-server:/backups/www/
```

### Timestamped incremental backups with rsync
```bash
#!/bin/bash
# /usr/local/bin/rsync-backup.sh

DATE=$(date +%Y-%m-%d_%H-%M)
DEST="/backups/snapshots/$DATE"
LATEST="/backups/latest"

rsync -avz --delete \
  --link-dest="$LATEST" \
  /var/www/ "$DEST/"

# Update symlink to latest
rm -f "$LATEST"
ln -s "$DEST" "$LATEST"

# Remove backups older than 30 days
find /backups/snapshots/ -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;

echo "Backup completed: $DEST"
```

---

## BorgBackup — Deduplicated, Encrypted Backups

Best for production — deduplication means even daily backups of large dirs stay small.

### Install & Initialise
```bash
apt install borgbackup -y

# Initialise encrypted repo (on remote or local)
borg init --encryption=repokey user@backup-server:/backups/borg-repo

# Save the passphrase securely — losing it = losing access to backups
```

### Create a Backup
```bash
#!/bin/bash
# /usr/local/bin/borg-backup.sh

export BORG_REPO='user@backup-server:/backups/borg-repo'
export BORG_PASSPHRASE='your-strong-passphrase'

# Create archive with timestamp
borg create \
  --verbose \
  --filter AME \
  --list \
  --stats \
  --show-rc \
  --compression lz4 \
  --exclude-caches \
  --exclude '/var/log/*' \
  --exclude '/var/cache/*' \
  --exclude '/tmp/*' \
  --exclude '/proc/*' \
  --exclude '/sys/*' \
  --exclude '/dev/*' \
  --exclude '/run/*' \
  ::'{hostname}-{now:%Y-%m-%dT%H:%M}' \
  /etc \
  /var/www \
  /home \
  /root

# Prune old archives
borg prune \
  --list \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  ::

# Compact freed space (Borg 1.2+)
borg compact ::

# Verify integrity
borg check ::
```

### Restore from Borg
```bash
# List archives
borg list user@backup-server:/backups/borg-repo

# List contents of an archive
borg list ::hostname-2026-03-20T02:00

# Restore specific directory
borg extract ::hostname-2026-03-20T02:00 var/www/html

# Full restore
cd /
borg extract --verbose ::hostname-2026-03-20T02:00
```

---

## Database Backups

### MySQL / MariaDB
```bash
# Single database
mysqldump -u root -p DATABASE_NAME > /backups/db/db_$(date +%Y%m%d).sql

# All databases
mysqldump -u root -p --all-databases > /backups/db/all_$(date +%Y%m%d).sql

# Compressed
mysqldump -u root -p DATABASE_NAME | gzip > /backups/db/db_$(date +%Y%m%d).sql.gz

# Restore
mysql -u root -p DATABASE_NAME < /backups/db/db_20260320.sql
gunzip < /backups/db/db_20260320.sql.gz | mysql -u root -p DATABASE_NAME
```

### PostgreSQL
```bash
# Single database
pg_dump DATABASE_NAME > /backups/db/db_$(date +%Y%m%d).sql
pg_dump -Fc DATABASE_NAME > /backups/db/db_$(date +%Y%m%d).dump  # custom format

# All databases
pg_dumpall > /backups/db/all_$(date +%Y%m%d).sql

# Restore
psql DATABASE_NAME < /backups/db/db_20260320.sql
pg_restore -d DATABASE_NAME /backups/db/db_20260320.dump
```

### SQLite
```bash
# Safe backup (uses SQLite's backup API — safe even during writes)
sqlite3 /path/to/database.db ".backup /backups/db/sqlite_$(date +%Y%m%d).db"

# Or with compression
sqlite3 /path/to/database.db ".dump" | gzip > /backups/db/sqlite_$(date +%Y%m%d).sql.gz
```

---

## rclone — Cloud Sync (S3, B2, Google Drive, and more)

### Install & Configure
```bash
# Install latest rclone
curl https://rclone.org/install.sh | bash

# Interactive setup for a remote
rclone config
# Follow prompts — supports 40+ cloud providers:
# - AWS S3, Backblaze B2, Google Drive, Google Cloud Storage
# - DigitalOcean Spaces, Wasabi, Cloudflare R2
# - SFTP, OneDrive, Dropbox
```

### rclone to Backblaze B2
```bash
# Upload backup directory
rclone copy /backups/db/ b2:your-bucket/db-backups/ --progress

# Sync (mirror — deletes files on remote not present locally)
rclone sync /backups/ b2:your-bucket/vps-backups/ --progress

# Verify files match
rclone check /backups/ b2:your-bucket/vps-backups/

# List remote contents
rclone ls b2:your-bucket/vps-backups/
rclone size b2:your-bucket/vps-backups/
```

### rclone to AWS S3
```bash
# Upload with server-side encryption
rclone copy /backups/db/ s3:your-bucket/db-backups/ \
  --s3-server-side-encryption AES256 --progress

# Sync with storage class
rclone sync /backups/ s3:your-bucket/vps-backups/ \
  --s3-storage-class STANDARD_IA --progress

# Upload single file
rclone copyto /backups/db/db_$(date +%Y%m%d).sql.gz s3:your-bucket/db/latest.sql.gz
```

### rclone to Google Drive
```bash
# After rclone config creates a "gdrive" remote:
rclone copy /backups/ gdrive:VPS-Backups/ --progress

# Sync specific folder
rclone sync /backups/db/ gdrive:VPS-Backups/db/ --progress

# Note: Google Drive uses OAuth tokens — for headless VPS:
# 1. Run "rclone authorize gdrive" on a machine with a browser
# 2. Copy the token JSON to the VPS rclone config
```

### rclone to Cloudflare R2 (S3-compatible, no egress fees)
```bash
# Configure as S3-compatible endpoint
# Provider: Cloudflare R2
# Endpoint: https://<account-id>.r2.cloudflarestorage.com
# Access key and secret from Cloudflare dashboard

rclone sync /backups/ r2:your-bucket/vps-backups/ --progress
```

### rclone Encryption (encrypt before upload)
```bash
# Create an encrypted remote wrapping another remote
# During "rclone config":
# Type: crypt
# Remote: b2:your-bucket/encrypted-backups
# Password: <strong passphrase>

rclone sync /backups/ crypt-remote: --progress
# Files are encrypted client-side before upload
```

### AWS CLI (S3) — alternative to rclone
```bash
aws s3 sync /backups/ s3://your-bucket/vps-backups/ --delete
aws s3 cp /backups/db/db_$(date +%Y%m%d).sql.gz s3://your-bucket/db/
```

---

## Automated Backup Cron Jobs

```bash
# Edit root's crontab
crontab -e -u root

# Daily DB backup at 2am
0 2 * * * /usr/local/bin/db-backup.sh >> /var/log/backups.log 2>&1

# Daily file backup at 3am
0 3 * * * /usr/local/bin/borg-backup.sh >> /var/log/backups.log 2>&1

# Weekly offsite sync at 4am on Sundays
0 4 * * 0 rclone sync /backups/ b2:your-bucket/ >> /var/log/rclone.log 2>&1

# Monthly backup verification (1st of month at 5am)
0 5 1 * * /usr/local/bin/backup-verify.sh >> /var/log/backup-verify.log 2>&1
```

---

## Backup Verification (Critical — Do This Monthly)

Never trust a backup you haven't restored:

### Database Restore Test
```bash
# MySQL — test restore to a temp database
mysql -u root -p -e "CREATE DATABASE test_restore;"
mysql -u root -p test_restore < /backups/db/db_latest.sql
mysql -u root -p test_restore -e "SHOW TABLES;"  # verify tables exist
mysql -u root -p -e "DROP DATABASE test_restore;"

# PostgreSQL — test restore
createdb test_restore
pg_restore -d test_restore /backups/db/db_latest.dump
psql test_restore -c "\dt"   # verify tables
dropdb test_restore
```

### Borg Integrity Check
```bash
export BORG_PASSPHRASE='your-passphrase'
borg check --verbose user@backup-server:/backups/borg-repo

# List and verify latest archive
borg list :: | tail -5
borg info ::$(borg list --short :: | tail -1)
```

### File Backup Verification
```bash
# Check backup file sizes (zero = failed backup)
ls -lh /backups/db/
find /backups/ -name "*.sql.gz" -size 0 -exec echo "EMPTY BACKUP: {}" \;

# Verify rclone remote matches local
rclone check /backups/ b2:your-bucket/vps-backups/ --one-way
```

### Automated Verification Script
```bash
#!/bin/bash
# /usr/local/bin/backup-verify.sh
set -euo pipefail
LOG="/var/log/backup-verify.log"
exec >> "$LOG" 2>&1
echo "=== $(date '+%Y-%m-%d %H:%M:%S') BACKUP VERIFICATION ==="

ERRORS=0

# Check latest backup file exists and is non-empty
LATEST=$(ls -t /backups/db/*.sql.gz 2>/dev/null | head -1)
if [ -z "$LATEST" ]; then
  echo "ERROR: No backup files found"
  ERRORS=$((ERRORS + 1))
elif [ ! -s "$LATEST" ]; then
  echo "ERROR: Latest backup is empty: $LATEST"
  ERRORS=$((ERRORS + 1))
else
  echo "OK: Latest backup: $LATEST ($(ls -lh "$LATEST" | awk '{print $5}'))"
fi

# Check backup age (alert if older than 48 hours)
if [ -n "$LATEST" ]; then
  AGE=$(( ($(date +%s) - $(stat -c %Y "$LATEST")) / 3600 ))
  if [ "$AGE" -gt 48 ]; then
    echo "ERROR: Latest backup is ${AGE}h old (>48h)"
    ERRORS=$((ERRORS + 1))
  else
    echo "OK: Backup age: ${AGE}h"
  fi
fi

# Check borg repo (if configured)
if [ -n "${BORG_REPO:-}" ]; then
  if borg check :: 2>&1; then
    echo "OK: Borg repo integrity check passed"
  else
    echo "ERROR: Borg repo integrity check failed"
    ERRORS=$((ERRORS + 1))
  fi
fi

echo "Verification complete: $ERRORS errors"
if [ "$ERRORS" -gt 0 ]; then
  echo "BACKUP VERIFICATION FAILED" | mail -s "BACKUP ALERT: $(hostname)" you@example.com
fi
```
