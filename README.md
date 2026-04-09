# Automated VPS Backup to AWS S3 (Bash)

This project is a simple and practical backup system for a Linux VPS.

It automatically:

* Creates backups of important directories
* Encrypts them using GPG (AES256)
* Uploads them securely to AWS S3
* Sends an email notification after completion


---

## Why this project?

Backups are often ignored until something breaks.

This project helps you:

* Automate backups
* Keep them secure
* Store them safely off-server

---

## 📦 Features

* Automated backups using Bash
* Compression using `tar`
* Encryption using GPG (AES256)
* Upload to AWS S3
* Email notifications
* Cron-based scheduling
* Log tracking

---

## ⚙️ Requirements

* Linux VPS (Ubuntu / AlmaLinux / etc.)
* AWS account
* AWS CLI
* GPG
* Mail utility (Postfix / mail)

---

## 🛠️ Setup Guide

### 1. Create an S3 bucket

* Go to AWS S3 → Create bucket
* Enable:

  * Block public access
  * Versioning
  * Default encryption (SSE-S3)

> Note: Bucket name must be globally unique

---

### 2. Create IAM user (least privilege)

Create a user with only S3 access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

Save:

* Access Key
* Secret Key

---

### 3. Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Configure:

```bash
aws configure
```

---

### 4. Install GPG

```bash
sudo apt install gpg -y
```

Create passphrase file:

```bash
gpg --gen-random --armor 1 32 > /root/.backup_passphrase
chmod 600 /root/.backup_passphrase
```

---

### 5. Create backup script

Create directory:

```bash
sudo mkdir -p /opt/vps_backup
sudo nano /opt/vps_backup/backup.sh
```

Paste this complete script:

```bash
#!/bin/bash
# ============================================================
# VPS Automated Backup to S3 — Project 3
# ============================================================

# --- Config ---
S3_BUCKET="<your-bucket-name>"
S3_REGION="ap-south-1"
BACKUP_ROOT="/tmp/vps_backup_staging"
LOG_FILE="/var/log/vps_backup/backup.log"
PASSPHRASE_FILE="/root/.backup_passphrase"
RETENTION_DAYS=30
EMAIL="<youremail>"
HOSTNAME_SHORT="<hostname>"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M")

# --- Directories to back up ---
# change according to your setup
BACKUP_DIRS=(
  "/etc/nginx"
  "/opt/testapp"
  "/opt/monitorapp"
  "/opt/vps_monitor"
  "/etc/letsencrypt"
  "/etc/postfix"
  "/etc/cron.d"
  "/etc/systemd/system"
)

# --- Setup ---
mkdir -p "$BACKUP_ROOT" "$(dirname $LOG_FILE)"
START_TIME=$(date +%s)
ERRORS=0

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

log "========== BACKUP STARTED: $TIMESTAMP =========="

# --- Create tarball ---
TARBALL="$BACKUP_ROOT/${HOSTNAME_SHORT}_${TIMESTAMP}.tar.gz"
log "Creating archive: $TARBALL"

tar -czf "$TARBALL" \
  --ignore-failed-read \
  --exclude="/etc/letsencrypt/archive" \
  "${BACKUP_DIRS[@]}" 2>>"$LOG_FILE" || { log "ERROR: tar failed"; ERRORS=$((ERRORS+1)); }

TARBALL_SIZE=$(du -sh "$TARBALL" 2>/dev/null | cut -f1)
log "Archive size: $TARBALL_SIZE"

# --- Encrypt ---
ENCRYPTED="${TARBALL}.gpg"
log "Encrypting archive..."
gpg --batch --yes \
    --passphrase-file "$PASSPHRASE_FILE" \
    --symmetric --cipher-algo AES256 \
    -o "$ENCRYPTED" "$TARBALL" 2>>"$LOG_FILE" \
  || { log "ERROR: Encryption failed"; ERRORS=$((ERRORS+1)); }

rm -f "$TARBALL"   # remove unencrypted version immediately

# --- Upload to S3 ---
S3_KEY="backups/${TIMESTAMP}/${HOSTNAME_SHORT}_${TIMESTAMP}.tar.gz.gpg"
log "Uploading to s3://${S3_BUCKET}/${S3_KEY} ..."

aws s3 cp "$ENCRYPTED" "s3://${S3_BUCKET}/${S3_KEY}" \
    --region "$S3_REGION" \
    --storage-class STANDARD_IA \
    2>>"$LOG_FILE" \
  && log "Upload successful" \
  || { log "ERROR: S3 upload failed"; ERRORS=$((ERRORS+1)); }

rm -f "$ENCRYPTED"

# --- Verify: list recent backups ---
RECENT=$(aws s3 ls "s3://${S3_BUCKET}/backups/" --region "$S3_REGION" 2>/dev/null | tail -5)

# --- Duration ---
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

# --- Email notification ---
STATUS="SUCCESS"
[ "$ERRORS" -gt 0 ] && STATUS="FAILED ($ERRORS errors)"

mail -s "[${HOSTNAME_SHORT}] Backup ${STATUS} — ${TIMESTAMP}" "$EMAIL" <<EOF
VPS Backup Report
=================
Host:      $HOSTNAME_SHORT
Time:      $(date)
Status:    $STATUS
Duration:  ${DURATION}s
Size:      $TARBALL_SIZE
S3 path:   s3://${S3_BUCKET}/${S3_KEY}

Recent backups in S3:
$RECENT

Full log: $LOG_FILE
EOF

log "========== BACKUP FINISHED: $STATUS in ${DURATION}s =========="

# Exit non-zero if there were errors (for cron error detection)
[ "$ERRORS" -eq 0 ]
```

Set permissions:

```bash
sudo chmod 700 /opt/vps_backup/backup.sh
```

---

### 6. Test the script

```bash
sudo /opt/vps_backup/backup.sh
```

Check:

* Backup created
* File uploaded to S3
* Email received

---

### 7. Automate with cron

```bash
sudo crontab -e
```

Example (run daily at 2 AM):

```bash
0 2 * * * /opt/vps_backup/backup.sh >> /var/log/vps_backup/cron.log 2>&1
```

---

## 🔐 Security Notes

* Backups are encrypted before upload
* Use IAM with minimal permissions
* Secure credentials:

```bash
chmod 600 ~/.aws/credentials
chmod 600 /root/.backup_passphrase
```

---

## ♻️ Restore Test (Important)

Always test your backups.

Download from S3:

```bash
aws s3 cp s3://your-bucket/backups/your-file.gpg /tmp/restore.gpg
```

Decrypt and list contents:

```bash
gpg --batch --passphrase-file /root/.backup_passphrase \
  -d /tmp/restore.gpg | tar -tzv | head
```

---

## 🧠 What you will learn

* Bash scripting for automation
* AWS S3 and IAM basics
* Encryption using GPG
* Cron jobs
* Backup and restore workflow

---

## 💡 Notes

* Replace placeholders like `your-bucket-name`
* Do not share your credentials or passphrase
* Always verify restore — backups alone are not enough

---

## 🙌 Contribution

Feel free to improve or extend this project.

---
