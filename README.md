# IMAP Mail → S3 weekly backup (using PGP Encryption)

Architecture: EventBridge Scheduler starts a stopped EC2 instance once a week →
the instance syncs new mail over IMAP → zips, GPG-encrypts, and streams the
archive to S3 → the instance shuts itself down. Total cost: roughly **$2–2.50/month**.

Every `PLACEHOLDER_LIKE_THIS` below needs to be replaced with your own values.

**How the files get created:** every file below is created directly in
whatever terminal you're working in, using `cat > filename << 'EOF'` followed
by the file's content and a line with just `EOF`. Note the **quotes around
`EOF`** — this matters. With quotes, bash treats everything between the two
`EOF` markers as pure literal text. Without them, bash tries to expand every
`$VARIABLE`, `${...}`, and `$(...)` in the content immediately as you paste
it — which `backup.sh` has a lot of — silently corrupting the script. Always
paste with the quoted form below.

This also means nothing needs to be uploaded or transferred separately: run
the heredoc commands directly wherever you need the file (your local
terminal or CloudShell for the `.json` files, and an EC2 Instance Connect /
SSH session on the instance itself for `backup.sh` and friends in step 7).

---

## 1. Generate an app password

Most modern mail providers let you create an app password. For security reasons, avoid using your main (root) credentials. The instructions below are for mailbox.org, but the process is similar for other providers.

In mailbox.org: **Settings → Security → Email App-Passwords** → create one with
IMAP access. You'll paste this into SSM in step 4.

## 2. Generate a PGP keypair (on your own computer — never on AWS)

```bash
gpg --full-generate-key
# choose RSA and RSA, 4096 bits
# use the email you'll reference as GPG_RECIPIENT in backup.sh

# Export the PUBLIC key -- this is not secret, it goes on the EC2 instance:
gpg --export -a "you@example.com" > mailbox-backup-public.asc

# Export and safely store the PRIVATE key -- this is the ONLY thing that can
# ever decrypt your backups. Put it in a password manager and/or a printed
# paper backup. It must NEVER touch AWS, S3, or the EC2 instance.
gpg --export-secret-keys -a "you@example.com" > private-key-backup.asc
```

**If you lose the private key, every backup becomes permanently unreadable.**
There is no recovery path — that's the trade-off for "no one else can ever
open this," including you, if the key is gone.

## 3. Create the S3 bucket and apply the lifecycle rule

```bash
aws s3api create-bucket --bucket your-backup-bucket-name --region eu-central-1 --create-bucket-configuration LocationConstraint=eu-central-1
```

```bash
cat > s3-lifecycle-policy.json << 'EOF'
{
  "Rules": [
    {
      "ID": "ExpireBackupsAfter30Days",
      "Filter": { "Prefix": "mailbox-backup-" },
      "Status": "Enabled",
      "Expiration": {
        "Days": 30
      }
    }
  ]
}
EOF
```

```bash
aws s3api put-bucket-lifecycle-configuration --bucket your-backup-bucket-name --lifecycle-configuration file://s3-lifecycle-policy.json
```

S3 lifecycle rules work off object age, not "keep the N most recent" — there's
no native concept of that, and no concept of "only delete this if a newer
backup exists." A 30-day window keeps roughly the last 4 weekly backups,
trading a bit more storage for a month-long safety margin: if the pipeline
ever breaks silently (a failed login, the schedule not firing, the instance
not starting), you have real time to notice and fix it before your last
backup ages out and gets deleted regardless of whether a replacement exists.

## 4. Store the IMAP server app password in SSM Parameter Store

```bash
aws ssm put-parameter --name /mailbackup/mail-app-password --value "the-app-password-from-step-1" --type SecureString --region eu-central-1
```

Standard SecureString parameters have no storage charge. Note the KMS key ID
it used (default is the `aws/ssm` AWS-managed key, which has no monthly fee)
— you'll need its ARN in step 5.

## 4b. Set up Resend for email notifications

Sign up at [resend.com](https://resend.com), verify a sending domain under
**Domains** (or use their `onboarding@resend.dev` test sender, which can
only send to the email address you signed up with — fine for testing, not
for real use), then grab an API key under **API Keys**.

```bash
aws ssm put-parameter --name /mailbackup/resend-api-key --value "re_your_actual_key" --type SecureString --region eu-central-1
```

The free tier (3,000 emails/month) comfortably covers one email a week, so
this adds $0 to the monthly cost.

## 5. Create the EC2 instance role

Create the policy file — fill in `ACCOUNT_ID`, `SSM_KMS_KEY_ID`, and your
bucket name before pasting:

```bash
cat > ec2-instance-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadMailbackupParameters",
      "Effect": "Allow",
      "Action": "ssm:GetParameter",
      "Resource": "arn:aws:ssm:eu-central-1:ACCOUNT_ID:parameter/mailbackup/*"
    },
    {
      "Sid": "DecryptSecureStringParameter",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:eu-central-1:ACCOUNT_ID:key/SSM_KMS_KEY_ID"
    },
    {
      "Sid": "UploadEncryptedBackups",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::your-backup-bucket-name/*"
    }
  ]
}
EOF
```

Then create the role and attach it:

```bash
aws iam create-role --role-name mailbox-backup-instance-role --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

```bash
aws iam put-role-policy --role-name mailbox-backup-instance-role --policy-name mailbox-backup-permissions --policy-document file://ec2-instance-role-policy.json
```

```bash
aws iam create-instance-profile --instance-profile-name mailbox-backup-profile
```

```bash
aws iam add-role-to-instance-profile --instance-profile-name mailbox-backup-profile --role-name mailbox-backup-instance-role
```

## 6. Launch the EC2 instance

- AMI: Ubuntu 24.04 LTS, **arm64** (Graviton — cheaper than x86)
- Instance type: `t4g.small`
- Region: `eu-central-1` (close to mailbox.org, keeps data in the EU)
- EBS volume: 30GB gp3 — sized for a confirmed 6GB mailbox: OS (~4GB) +
  persistent Maildir cache (~6GB) + temporary zip and its encrypted copy
  existing simultaneously during the encrypt step (~6GB each) ≈ 22GB peak,
  plus headroom. If your mailbox grows substantially, resize the volume
  again later — it's non-destructive (see the troubleshooting notes on
  `growpart`/`resize2fs` from setup).
- IAM instance profile: `mailbox-backup-profile`
- Only allow your own IP to connect (other inbound connections to be rejected)
- Under "Advanced → Shutdown behavior," confirm it's set to **Stop** (this is
  the default for EBS-backed instances, but worth double-checking)

## 7. Set up the instance itself

Connect via EC2 Instance Connect (or SSH), then:

```bash
sudo apt update && sudo apt install -y isync gnupg2 zip unzip curl

# awscli isn't reliably available via apt on Ubuntu cloud images, and where
# it is, it's the old v1 CLI, which doesn't support commands used later in
# this guide (aws scheduler create-schedule). Install v2 directly instead
# (aarch64 build, matching the Graviton t4g.small instance type):
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

sudo useradd -m mailbackup
sudo -u mailbackup mkdir -p /home/mailbackup/mail/mailbox
```

Create the mbsync config — fill in your real email address on
the `User` line before pasting:

```bash
sudo -u mailbackup tee /home/mailbackup/.mbsyncrc > /dev/null << 'EOF'
# mbsync (isync) config for backing up via IMAP.
# Install to /home/mailbackup/.mbsyncrc on the EC2 instance.
#
# The password is never stored in this file or on disk in plaintext -- PassCmd
# fetches it fresh from SSM Parameter Store on every run, using the EC2
# instance's IAM role (no static AWS credentials involved).

IMAPAccount mailbox
Host YOUR_IMAP_SERVER_HERE
Port 993
User your-email@yourdomain.com
PassCmd "aws ssm get-parameter --name /mailbackup/mail-app-password --with-decryption --query Parameter.Value --output text --region eu-central-1"
SSLType IMAPS
CertificateFile /etc/ssl/certs/ca-certificates.crt

IMAPStore mailbox-remote
Account mailbox

MaildirStore mailbox-local
Path ~/mail/mailbox/
Inbox ~/mail/mailbox/INBOX
SubFolders Verbatim

Channel mailbox
Master :mailbox-remote:
Slave :mailbox-local:
Patterns *
Create Slave
# Pull only: new messages and flag changes flow server -> local, never the
# reverse, so this backup can never modify or delete anything on your mail server.
Sync Pull
# Never physically remove a message locally, even if it gets deleted or
# expunged on the server. A message deleted upstream just ends up flagged
# as trashed in its local filename -- the content itself is always kept.
# This is what makes it a *backup* rather than a live mirror.
Expunge None
SyncState *
EOF
```

Create the backup script — fill in `BUCKET`, `GPG_RECIPIENT`, `REGION`,
`EMAIL_FROM`, and `EMAIL_TO` in the configuration block before pasting:

```bash
sudo tee /home/mailbackup/backup.sh > /dev/null << 'EOF'
#!/usr/bin/env bash
#
# Weekly IMAP Server -> S3 encrypted backup.
# Runs once per boot via the mailbox-backup.service systemd unit.
# On success OR failure, the instance shuts itself down (EC2's default
# instance-initiated-shutdown behavior is "stop", not "terminate", so the
# EBS volume -- and the local Maildir cache used for incremental syncs --
# is preserved for next week's run).

set -euo pipefail

# ---- Configuration: replace these placeholders ----------------------------
REGION="eu-central-1"
BUCKET="your-backup-bucket-name"          # no s3:// prefix, just the name
GPG_RECIPIENT="you@example.com"           # the email on your PGP keypair
BACKUP_USER="mailbackup"
MAILDIR="/home/${BACKUP_USER}/mail/mailbox"
MBSYNC_CONF="/home/${BACKUP_USER}/.mbsyncrc"
EMAIL_FROM="backups@yourdomain.com"       # must be on a domain verified in Resend
EMAIL_TO="you@example.com"
RESEND_PARAM_NAME="/mailbackup/resend-api-key"
# -----------------------------------------------------------------------------

DATE="$(date +%F)"
ARCHIVE_NAME="mailbox-backup-${DATE}.zip.gpg"
TMP_ZIP="/tmp/mailbox-backup-${DATE}.zip"
TMP_GPG="${TMP_ZIP}.gpg"
LOGFILE="/tmp/mailbox-backup-${DATE}.log"

# Capture everything printed from here on -- this script's own log lines
# plus mbsync/zip/gpg/aws output -- so the full run can be emailed as an
# attachment. Truncates rather than appends, so each run's log stays scoped
# to that run instead of growing across repeated same-day retries.
exec > >(tee "${LOGFILE}") 2>&1

log() {
  logger -t mailbox-backup "$1"
  echo "$(date -Is) $1"
}

# Fetched once at startup. Guarded with `|| echo ""` so a broken email
# config can never fail the actual backup -- notifications are best-effort.
RESEND_API_KEY="$(aws ssm get-parameter --name "${RESEND_PARAM_NAME}" --with-decryption --query Parameter.Value --output text --region "${REGION}" 2>/dev/null || echo "")"

send_email() {
  local status="$1"   # SUCCESS or FAILED
  local subject="Email backup ${status} - ${DATE}"
  local body_text="Weekly email backup finished with status: ${status}. Full run log is attached."
  local log_b64
  local response
  local http_code
  local http_body
  local payload_file="/tmp/mailbox-backup-${DATE}-email-payload.json"

  if [ -z "${RESEND_API_KEY}" ]; then
    log "WARNING: no Resend API key available, skipping notification email."
    return 0
  fi

  log_b64="$(base64 -w0 "${LOGFILE}" 2>/dev/null || echo "")"

  # Write the JSON payload to a file rather than passing it as a curl
  # argument -- printf is a bash builtin, so it isn't subject to the OS's
  # argument-length limit the way an external command like curl is. A large
  # base64'd log embedded directly in a -d "..." argument can otherwise hit
  # "Argument list too long" (E2BIG).
  printf '{"from":"%s","to":["%s"],"subject":"%s","text":"%s","attachments":[{"filename":"backup-log-%s.txt","content":"%s"}]}' \
    "${EMAIL_FROM}" "${EMAIL_TO}" "${subject}" "${body_text}" "${DATE}" "${log_b64}" > "${payload_file}"

  # -w appends the HTTP status code on its own trailing line so we can tell
  # a genuinely accepted request (200) apart from one Resend rejected (401
  # bad key, 422 unverified domain, etc.) -- plain curl exit code alone
  # can't tell these apart, since the HTTP request itself still "succeeds"
  # even when the API rejects it.
  response="$(curl -s -w '\n%{http_code}' -X POST 'https://api.resend.com/emails' \
    -H "Authorization: Bearer ${RESEND_API_KEY}" \
    -H 'Content-Type: application/json' \
    --data "@${payload_file}" 2>&1)" || true

  rm -f "${payload_file}"

  http_code="${response##*$'\n'}"
  http_body="${response%$'\n'*}"

  if [ "${http_code}" = "200" ]; then
    log "Notification email sent (${status})."
  else
    log "WARNING: notification email failed (HTTP ${http_code}): ${http_body}"
  fi
}

cleanup_and_shutdown() {
  rm -f "${TMP_ZIP}" "${TMP_GPG}" "${LOGFILE}" "/tmp/mailbox-backup-${DATE}-email-payload.json" 2>/dev/null || true
  if [ "${SKIP_SHUTDOWN:-0}" = "1" ]; then
    log "SKIP_SHUTDOWN=1 set -- leaving the instance running for debugging."
    return 0
  fi
  log "Shutting down instance."
  sudo shutdown -h now
}

# No matter what happens below, email a failure notice and stop the
# instance -- we never want to pay for an idle box because a step failed
# partway through, and we never want a silent failure to go unnoticed.
trap 'log "ERROR: backup run failed."; send_email FAILED; cleanup_and_shutdown' ERR

log "Starting weekly email backup."

# 1 & 2. Incremental IMAP sync (only pulls new/changed messages since last
#    run; deletions on the server are never propagated as real deletions
#    locally -- see the mbsyncrc comments) and build the archive to a temp
#    file. Skippable with SKIP_SYNC=1 if today's zip already exists, so you
#    can re-test the encrypt/upload/notify steps without re-syncing and
#    re-zipping a large mailbox every time.
if [ "${SKIP_SYNC:-0}" = "1" ] && [ -f "${TMP_ZIP}" ]; then
  log "SKIP_SYNC=1 set and ${TMP_ZIP} already exists -- reusing it, skipping IMAP sync and zip creation."
else
  #    -V prints progress per mailbox/message so long syncs aren't silent.
  mbsync -V -c "${MBSYNC_CONF}" -a
  log "IMAP sync complete."

  #    Dropping zip's -q prints one line per file as it's added, so long
  #    runs aren't silent.
  cd "$(dirname "${MAILDIR}")"
  zip -r "${TMP_ZIP}" "$(basename "${MAILDIR}")"
fi

# 3. Verify the zip's internal integrity (CRC check on every file) before
#    we ever encrypt or upload it. This is the step most likely to catch
#    real corruption, since it's checking the actual archive contents.
if ! zip -T "${TMP_ZIP}"; then
  log "ERROR: zip integrity check failed -- archive is corrupt."
  exit 1
fi
log "Zip integrity check passed."

# 4. Encrypt with your PGP public key. Only your offline private key can
#    ever decrypt this -- the instance never has that capability.
gpg --batch --yes --trust-model always --encrypt --recipient "${GPG_RECIPIENT}" \
  --output "${TMP_GPG}" "${TMP_ZIP}"

# 5. Sanity-check the encrypted file for truncation, without needing the
#    private key. (Note: `gpg --list-packets` is NOT a valid check here --
#    for an encrypted message it always attempts real decryption and always
#    fails with "No secret key" on this box by design, regardless of
#    whether the file is actually fine.) Encryption adds only small framing
#    overhead and doesn't shrink already-compressed data, so a healthy
#    output should be roughly the same size as the input, never drastically
#    smaller -- a big gap is what a truncated write actually looks like.
ZIP_SIZE=$(stat -c%s "${TMP_ZIP}")
GPG_SIZE=$(stat -c%s "${TMP_GPG}")
log "Source zip: ${ZIP_SIZE} bytes. Encrypted output: ${GPG_SIZE} bytes."
if [ ! -s "${TMP_GPG}" ] || [ "${GPG_SIZE}" -lt "$(( ZIP_SIZE * 95 / 100 ))" ]; then
  log "ERROR: encrypted archive size looks inconsistent with the source -- possible truncation."
  exit 1
fi
log "Encrypted archive size looks consistent with the source."

# 6. Upload, then clean up the local temp files -- nothing unencrypted or
#    encrypted stays behind on disk between runs.
aws s3 cp "${TMP_GPG}" "s3://${BUCKET}/${ARCHIVE_NAME}" --region "${REGION}"
log "Uploaded ${ARCHIVE_NAME} to s3://${BUCKET}/."
rm -f "${TMP_ZIP}" "${TMP_GPG}"

log "Backup run complete."
send_email SUCCESS
cleanup_and_shutdown
EOF
```

Create the systemd unit:

```bash
sudo tee /etc/systemd/system/mailbox-backup.service > /dev/null << 'EOF'
[Unit]
Description=email encrypted backup to S3
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=mailbackup
ExecStart=/home/mailbackup/backup.sh

[Install]
WantedBy=multi-user.target
EOF
```

Now finish the setup:

```bash
sudo chmod +x /home/mailbackup/backup.sh
sudo chown mailbackup:mailbackup /home/mailbackup/.mbsyncrc /home/mailbackup/backup.sh
```

For the GPG public key: on your own computer, run `cat mailbox-backup-public.asc`
and copy its full output (it's plain text, starting with
`-----BEGIN PGP PUBLIC KEY BLOCK-----`), then on the instance:

```bash
sudo -u mailbackup tee /home/mailbackup/mailbox-backup-public.asc > /dev/null << 'EOF'
-- paste the public key block here --
EOF

sudo -u mailbackup gpg --import /home/mailbackup/mailbox-backup-public.asc
```

Let the mailbackup user shut the machine down without a password prompt:

```bash
echo "mailbackup ALL=(root) NOPASSWD: /sbin/shutdown" | sudo tee /etc/sudoers.d/mailbackup-shutdown
```

Test it manually FIRST, with `SKIP_SHUTDOWN=1` so a failure doesn't power
off the instance while you're still debugging it:

```bash
sudo -u mailbackup SKIP_SHUTDOWN=1 /home/mailbackup/backup.sh
```

Repeat that (fixing whatever it complains about) until it completes with
"Backup run complete." Only once it succeeds cleanly, enable the systemd
service — this is what makes future boots (including the real weekly
schedule) run it automatically:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mailbox-backup.service
```

Optional final check: run it once more WITHOUT `SKIP_SHUTDOWN` to confirm
the real shutdown behavior works too, since the schedule will rely on it:

```bash
sudo -u mailbackup /home/mailbackup/backup.sh
```

Check your S3 bucket for the resulting `mailbox-backup-<date>.zip.gpg` file.
Once it works, the instance should already be shutting itself down at the end
of the script — confirm it actually stops, then move on.

## 8. Create the weekly schedule

Create the two policy files and the schedule target — fill in `ACCOUNT_ID`
and your instance ID in both `scheduler-permission-policy.json` and
`scheduler-target.json` before pasting:

```bash
cat > scheduler-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "scheduler.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

```bash
cat > scheduler-permission-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:StartInstances",
      "Resource": "arn:aws:ec2:eu-central-1:ACCOUNT_ID:instance/i-XXXXXXXXXXXXXXXXX"
    }
  ]
}
EOF
```

```bash
cat > scheduler-target.json << 'EOF'
{
  "Arn": "arn:aws:scheduler:::aws-sdk:ec2:startInstances",
  "RoleArn": "arn:aws:iam::ACCOUNT_ID:role/mailbox-backup-scheduler-role",
  "Input": "{\"InstanceIds\":[\"i-XXXXXXXXXXXXXXXXX\"]}"
}
EOF
```

Then create the role and the schedule itself:

```bash
aws iam create-role --role-name mailbox-backup-scheduler-role --assume-role-policy-document file://scheduler-trust-policy.json
```

```bash
aws iam put-role-policy --role-name mailbox-backup-scheduler-role --policy-name start-instance --policy-document file://scheduler-permission-policy.json
```

```bash
aws scheduler create-schedule --name weekly-mailbox-backup --schedule-expression "cron(0 3 ? * SUN *)" --schedule-expression-timezone "Asia/Hong_Kong" --flexible-time-window '{"Mode":"OFF"}' --target file://scheduler-target.json
```

Adjust the day/time/timezone to whatever suits you — this runs every Sunday
at 3am Hong Kong time.

## Corruption checks

Every run automatically verifies the zip's internal CRC integrity before
encrypting it (catching corruption in the source content), and afterward
compares the encrypted file's size against the source zip's to catch
truncated writes (catching corruption introduced during encryption or from
running out of disk space mid-write). Both are logged (`Zip integrity check
passed.` / `Encrypted archive size looks consistent with the source.`) and
either one failing aborts the run without uploading a bad file.

Neither check can decrypt the archive to prove the content is fully
restorable — that would require the private key to live on the instance,
which defeats the point. (An earlier version of this script tried to use
`gpg --list-packets` for the post-encryption check, but that command
actually attempts real decryption for an encrypted message and always fails
with "No secret key" here by design — it could never have passed, on any
run, ever. The size comparison above is the correct replacement.) The only
way to prove an archive fully restores is the manual test below; worth
repeating every few months, not just once.

## Restoring a backup

On any machine with your **private** key imported:

```bash
aws s3 cp s3://your-backup-bucket-name/mailbox-backup-2026-09-07.zip.gpg .
gpg --import private-key-backup.asc   # only needed once per machine
gpg --decrypt mailbox-backup-2026-09-07.zip.gpg > restored.zip
unzip restored.zip
```

You'll be prompted for your private key's passphrase. The result is a plain
Maildir folder you can point any mail client at, or re-import via `imapsync`
into a fresh mailbox.
