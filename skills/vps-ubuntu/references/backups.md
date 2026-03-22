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

---

## Upload Backups to Cloud (Offsite)

### Backblaze B2 / AWS S3 with rclone
```bash
apt install rclone -y
rclone config    # interactive setup for B2/S3

# Upload backup
rclone copy /backups/db/ b2:your-bucket/db-backups/ --progress

# Sync (mirror)
rclone sync /backups/ b2:your-bucket/vps-backups/ --progress

# Verify
rclone check /backups/ b2:your-bucket/vps-backups/
```

### AWS CLI (S3)
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
```

---

## Backup Verification (Critical — Do This Monthly)

Never trust a backup you haven't restored:
```bash
# Test restore DB to a temp database
mysql -u root -p -e "CREATE DATABASE test_restore;"
mysql -u root -p test_restore < /backups/db/db_latest.sql
mysql -u root -p test_restore -e "SHOW TABLES;"  # verify tables exist
mysql -u root -p -e "DROP DATABASE test_restore;"

# Verify borg backup integrity
export BORG_PASSPHRASE='your-passphrase'
borg check --verbose user@backup-server:/backups/borg-repo

# Check backup file sizes (zero = failed backup)
ls -lh /backups/db/
find /backups/ -name "*.sql.gz" -size 0 -exec echo "EMPTY BACKUP: {}" \;
```
