# IMAP Mail → S3 weekly backup (using PGP Encryption)

Architecture: EventBridge Scheduler starts a stopped EC2 instance once a week →
the instance syncs mail over IMAP for one or more inboxes → zips, GPG-encrypts,
and streams each one to S3 → the instance shuts itself down. Cost is roughly
**$3/month for one ~6GB inbox, plus about $0.55/month per additional inbox**
— most of the cost is fixed regardless of account count; see the cost recap
near the bottom for why.

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

## 1. Generate an app password for each inbox

Most modern mail providers let you create an app password — for security
reasons, avoid using your main account credentials for this. The exact menu
differs by provider; for mailbox.org it's **Settings → Security → Email
App-Passwords** → create one with IMAP access.

If you're backing up more than one inbox, generate a separate app password
for each one now — you'll store them all in SSM in step 4.

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
open this," including you, if the key is gone. One keypair covers every
inbox you back up; there's no need to generate more than one.

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
backup exists." A 30-day window keeps roughly the last 4 weekly backups **per
inbox**, trading a bit more storage for a month-long safety margin: if the
pipeline ever breaks silently (a failed login, the schedule not firing, the
instance not starting), you have real time to notice and fix it before your
last backup ages out and gets deleted regardless of whether a replacement
exists.

## 4. Store each inbox's app password in SSM Parameter Store

One parameter per inbox, following the naming pattern the script expects
(`/mailbackup/<short-name>-app-password`):

```bash
aws ssm put-parameter --name /mailbackup/personal-app-password --value "the-app-password-from-step-1" --type SecureString --region eu-central-1
```

Repeat for each additional inbox, e.g. `/mailbackup/work-app-password`, with
its own app password. Standard SecureString parameters have no storage
charge regardless of how many you create. Note the KMS key ID used (default
is the `aws/ssm` AWS-managed key, which has no monthly fee) — you'll need
its ARN in step 5.

## 4b. Set up Resend for email notifications

Sign up at [resend.com](https://resend.com), verify a sending domain under
**Domains** (or use their `onboarding@resend.dev` test sender, which can
only send to the email address you signed up with — fine for testing, not
for real use), then grab an API key under **API Keys**.

```bash
aws ssm put-parameter --name /mailbackup/resend-api-key --value "re_your_actual_key" --type SecureString --region eu-central-1
```

The free tier (3,000 emails/month) comfortably covers one email a week
regardless of how many inboxes you're backing up, so this adds $0 to the
monthly cost.

## 5. Create the EC2 instance role

Create the policy file — fill in `ACCOUNT_ID`, `SSM_KMS_KEY_ID`, and your
bucket name before pasting. The SSM permission below is a wildcard over the
whole `/mailbackup/*` path, so it already covers every inbox's password
parameter — adding more inboxes later never requires touching this policy:

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
- Region: `eu-central-1`, or pick one closer to your mail provider's servers
  / matching your own data-residency preference — this mostly affects sync
  latency, not cost
- EBS volume: 30GB gp3, sized for OS (~4GB) + a full IMAP resync (~6GB) +
  the temporary zip and its encrypted copy existing simultaneously during
  the encrypt step (~6GB each) ≈ 22GB peak, plus headroom, for a ~6GB inbox.
  **With multiple inboxes, this does NOT scale per account** — accounts are
  processed one at a time, and each account's temp files and Maildir are
  cleaned up before the next one starts (see backup.sh below), so peak disk
  usage is bounded by whichever single inbox is largest, not the sum of all
  of them. Size the volume for your single biggest inbox, regardless of how
  many total inboxes you're backing up. This space is transient, not a
  persistent cache — nothing survives on disk between runs. Resize the
  volume later if your largest inbox grows; it's non-destructive (see the
  troubleshooting notes on `growpart`/`resize2fs` from setup).
- IAM instance profile: `mailbox-backup-profile`
- Security group: restrict inbound access (SSH / EC2 Instance Connect) to
  just your own IP address rather than leaving it open to the internet
  (`0.0.0.0/0`). This instance only needs to be reachable by you for setup
  and debugging — nothing else needs to connect to it inbound at all, since
  the backup itself only makes outbound connections (to mailbox.org's IMAP
  server, S3, SSM, and Resend).
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
```

Create the backup script — fill in `IMAP_HOST`, `BUCKET`, `GPG_RECIPIENT`,
`REGION`, `EMAIL_FROM`, `EMAIL_TO`, and the `ACCOUNTS` list in the
configuration block before pasting. The `ACCOUNTS` format and how to add
more inboxes is explained right after this block, and again in the
comments inside the script itself:

```bash
sudo tee /home/mailbackup/backup.sh > /dev/null << 'EOF'
#!/usr/bin/env bash
#
# Weekly IMAP -> S3 encrypted backup. Supports one or more inboxes on the
# same IMAP provider -- see the ACCOUNTS list below.
# Runs once per boot via the mailbox-backup.service systemd unit.
# On success OR failure, the instance shuts itself down (EC2's default
# instance-initiated-shutdown behavior is "stop", not "terminate", so the
# EBS volume is preserved between runs -- though each account's Maildir
# gets wiped clean after its own successful upload; see backup_one_account
# below).

set -euo pipefail

# ---- Configuration: replace these placeholders -----------------------------
REGION="eu-central-1"
BUCKET="your-backup-bucket-name"          # no s3:// prefix, just the name
GPG_RECIPIENT="you@example.com"           # the email on your PGP keypair
BACKUP_USER="mailbackup"
IMAP_HOST="imap.yourprovider.com"
IMAP_PORT=993
EMAIL_FROM="backups@yourdomain.com"       # must be on a domain verified in Resend
EMAIL_TO="you@example.com"
RESEND_PARAM_NAME="/mailbackup/resend-api-key"

# One entry per inbox to back up, all on the IMAP_HOST above:
#   "short-name:imap-username:ssm-password-parameter-name"
# short-name is used in S3 filenames, log lines, and as mbsync's internal
# channel name -- keep it short, lowercase, with no spaces or colons in it.
# Each account needs its own app password stored in SSM first, e.g.:
#   aws ssm put-parameter --name /mailbackup/personal-app-password \
#     --value "the-app-password" --type SecureString --region eu-central-1
# To add or remove an inbox, just edit this list -- nothing else in this
# script, and no IAM change, is needed (the instance role already grants
# read access to the whole /mailbackup/* parameter path).
ACCOUNTS=(
  "personal:you@yourdomain.com:/mailbackup/personal-app-password"
)
# -----------------------------------------------------------------------------

DATE="$(date +%F)"
LOGFILE="/tmp/mailbox-backup-${DATE}.log"
MBSYNC_CONF="/tmp/mailbackup-generated.mbsyncrc"

# Capture everything printed from here on -- this script's own log lines
# plus mbsync/zip/gpg/aws output for every account -- so the full run can
# be emailed as one attachment. Truncates rather than appends, so each
# run's log stays scoped to that run instead of growing across repeated
# same-day retries.
exec > >(tee "${LOGFILE}") 2>&1

log() {
  logger -t mailbox-backup "$1"
  echo "$(date -Is) $1"
}

# Fetched once at startup. Guarded with `|| echo ""` so a broken email
# config can never fail the actual backup -- notifications are best-effort.
RESEND_API_KEY="$(aws ssm get-parameter --name "${RESEND_PARAM_NAME}" --with-decryption --query Parameter.Value --output text --region "${REGION}" 2>/dev/null || echo "")"

SUCCEEDED_ACCOUNTS=()
FAILED_ACCOUNTS=()

send_email() {
  local status="$1"   # SUCCESS or FAILED
  local subject="Mail backup ${status} - ${DATE}"
  local body_text="Weekly mail backup finished with status: ${status}. Succeeded: ${SUCCEEDED_ACCOUNTS[*]:-none}. Failed: ${FAILED_ACCOUNTS[*]:-none}. Full run log is attached."
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
  if [ "${SKIP_SHUTDOWN:-0}" = "1" ]; then
    log "SKIP_SHUTDOWN=1 set -- leaving the instance and its temp files as-is for debugging."
    return 0
  fi
  # Temp zip/gpg files are only cleaned up here (not immediately after each
  # account's upload) so that SKIP_SYNC can still reuse them across debug
  # retries. Each account's plaintext Maildir, though, is always wiped
  # immediately after its own successful upload inside backup_one_account,
  # regardless of debug mode -- that's the core security property and
  # shouldn't depend on a debug flag.
  local account acct_name
  for account in "${ACCOUNTS[@]}"; do
    IFS=':' read -r acct_name _ _ <<< "${account}"
    rm -f "/tmp/mailbox-backup-${acct_name}-${DATE}.zip" "/tmp/mailbox-backup-${acct_name}-${DATE}.zip.gpg" 2>/dev/null || true
  done
  rm -f "${LOGFILE}" "${MBSYNC_CONF}" "/tmp/mailbox-backup-${DATE}-email-payload.json" 2>/dev/null || true
  log "Shutting down instance."
  sudo shutdown -h now
}

# Safety net for genuinely unexpected failures outside the per-account loop
# below (e.g. something going wrong while generating the mbsync config).
# Per-account failures are caught explicitly inside backup_one_account
# instead, so one bad inbox never aborts the whole run -- see the comment
# on backup_one_account for why that requires checking each command
# explicitly rather than leaving it bare.
trap 'log "ERROR: backup run failed unexpectedly."; send_email FAILED; cleanup_and_shutdown' ERR

log "Starting weekly mail backup for ${#ACCOUNTS[@]} account(s)."

# Generates one shared mbsync config with one Channel per account, so
# adding an inbox is just an ACCOUNTS edit above plus one new SSM
# parameter -- nothing here needs hand-maintaining separately.
generate_mbsyncrc() {
  : > "${MBSYNC_CONF}"
  local account acct_name acct_user acct_param
  for account in "${ACCOUNTS[@]}"; do
    IFS=':' read -r acct_name acct_user acct_param <<< "${account}"
    cat >> "${MBSYNC_CONF}" << MBBLOCK
IMAPAccount ${acct_name}
Host ${IMAP_HOST}
Port ${IMAP_PORT}
User ${acct_user}
PassCmd "aws ssm get-parameter --name ${acct_param} --with-decryption --query Parameter.Value --output text --region ${REGION}"
SSLType IMAPS
CertificateFile /etc/ssl/certs/ca-certificates.crt

IMAPStore ${acct_name}-remote
Account ${acct_name}

MaildirStore ${acct_name}-local
Path ~/mail/${acct_name}/
Inbox ~/mail/${acct_name}/INBOX
SubFolders Verbatim

Channel ${acct_name}
Master :${acct_name}-remote:
Slave :${acct_name}-local:
Patterns *
Create Slave
# Pull only: new messages and flag changes flow server -> local, never the
# reverse, so this backup can never modify or delete anything on the server.
Sync Pull
# Never physically remove a message locally, even if deleted or expunged
# on the server -- it just ends up flagged as trashed in its local
# filename, content intact. This is what makes it a *backup*, not a mirror.
Expunge None
SyncState *

MBBLOCK
  done
}
generate_mbsyncrc
log "Generated mbsync config for: $(for a in "${ACCOUNTS[@]}"; do printf '%s ' "${a%%:*}"; done)"

# Runs the full pipeline for one account: sync, zip, verify, encrypt,
# verify, upload, wipe. Every risky command below is wrapped as an
# if-condition (`if ! cmd; then ...; return 1; fi`) rather than left bare --
# under `set -e`, a bare failing command anywhere in this function would
# abort the ENTIRE script, killing the whole multi-account run over one bad
# inbox, whereas a command used as an if-condition is specifically exempt
# from that. Returns 1 on failure so the caller's loop can log it, move on
# to the next account, and still report an accurate overall status by email
# at the end.
backup_one_account() {
  local acct_name="$1"
  local maildir="/home/${BACKUP_USER}/mail/${acct_name}"
  local archive_name="mailbox-backup-${acct_name}-${DATE}.zip.gpg"
  local tmp_zip="/tmp/mailbox-backup-${acct_name}-${DATE}.zip"
  local tmp_gpg="${tmp_zip}.gpg"
  local zip_size gpg_size

  mkdir -p "${maildir}"

  # 1 & 2. Full IMAP sync and build the archive to a temp file. Every run
  #    does a full fresh download rather than an incremental one, since the
  #    Maildir is wiped after each successful upload below -- trading a
  #    somewhat longer sync for never leaving plaintext mail on disk at
  #    rest. Skippable with SKIP_SYNC=1 if today's zip already exists, so
  #    you can re-test the encrypt/upload/notify steps without re-syncing
  #    and re-zipping a large mailbox every time.
  if [ "${SKIP_SYNC:-0}" = "1" ] && [ -f "${tmp_zip}" ]; then
    log "[${acct_name}] SKIP_SYNC=1 set and ${tmp_zip} already exists -- reusing it, skipping IMAP sync and zip creation."
  else
    #    -V prints progress per mailbox/message so long syncs aren't silent.
    if ! mbsync -V -c "${MBSYNC_CONF}" "${acct_name}"; then
      log "[${acct_name}] ERROR: IMAP sync failed."
      return 1
    fi
    log "[${acct_name}] IMAP sync complete."

    #    Dropping zip's -q prints one line per file as it's added, so long
    #    runs aren't silent.
    if ! (cd "$(dirname "${maildir}")" && zip -r "${tmp_zip}" "$(basename "${maildir}")"); then
      log "[${acct_name}] ERROR: zip creation failed."
      return 1
    fi
  fi

  # 3. Verify the zip's internal integrity (CRC check on every file) before
  #    we ever encrypt or upload it. This is the step most likely to catch
  #    real corruption, since it's checking the actual archive contents.
  if ! zip -T "${tmp_zip}"; then
    log "[${acct_name}] ERROR: zip integrity check failed -- archive is corrupt."
    return 1
  fi
  log "[${acct_name}] Zip integrity check passed."

  # 4. Encrypt with your PGP public key. Only your offline private key can
  #    ever decrypt this -- the instance never has that capability.
  if ! gpg --batch --yes --trust-model always --encrypt --recipient "${GPG_RECIPIENT}" \
    --output "${tmp_gpg}" "${tmp_zip}"; then
    log "[${acct_name}] ERROR: gpg encryption failed."
    return 1
  fi

  # 5. Sanity-check the encrypted file for truncation, without needing the
  #    private key. (`gpg --list-packets` is NOT a valid check here -- for
  #    an encrypted message it always attempts real decryption and always
  #    fails with "No secret key" on this box by design, regardless of
  #    whether the file is actually fine.) Encryption adds only small
  #    framing overhead and doesn't shrink already-compressed data, so a
  #    healthy output should be roughly the same size as the input, never
  #    drastically smaller -- a big gap is what a truncated write looks like.
  zip_size=$(stat -c%s "${tmp_zip}")
  gpg_size=$(stat -c%s "${tmp_gpg}")
  log "[${acct_name}] Source zip: ${zip_size} bytes. Encrypted output: ${gpg_size} bytes."
  if [ ! -s "${tmp_gpg}" ] || [ "${gpg_size}" -lt "$(( zip_size * 95 / 100 ))" ]; then
    log "[${acct_name}] ERROR: encrypted archive size looks inconsistent with the source -- possible truncation."
    return 1
  fi
  log "[${acct_name}] Encrypted archive size looks consistent with the source."

  # 6. Upload, then wipe this account's plaintext mail from disk
  #    immediately -- unconditionally, even in SKIP_SHUTDOWN debug mode,
  #    since this is the core security property (no plaintext mail at rest)
  #    and shouldn't depend on a debug flag.
  if ! aws s3 cp "${tmp_gpg}" "s3://${BUCKET}/${archive_name}" --region "${REGION}"; then
    log "[${acct_name}] ERROR: upload failed."
    return 1
  fi
  log "[${acct_name}] Uploaded ${archive_name} to s3://${BUCKET}/."
  rm -rf "${maildir}"
  mkdir -p "${maildir}"
  log "[${acct_name}] Wiped local Maildir -- no plaintext mail left on disk until next run."

  return 0
}

for account in "${ACCOUNTS[@]}"; do
  IFS=':' read -r ACCT_NAME _ _ <<< "${account}"
  log "=== Starting backup for account: ${ACCT_NAME} ==="
  if backup_one_account "${ACCT_NAME}"; then
    SUCCEEDED_ACCOUNTS+=("${ACCT_NAME}")
    log "=== ${ACCT_NAME}: done ==="
  else
    FAILED_ACCOUNTS+=("${ACCT_NAME}")
    log "=== ${ACCT_NAME}: FAILED, continuing with any remaining accounts ==="
  fi
done

if [ ${#FAILED_ACCOUNTS[@]} -eq 0 ]; then
  OVERALL_STATUS="SUCCESS"
else
  OVERALL_STATUS="FAILED"
fi

log "Backup run complete. Succeeded: ${SUCCEEDED_ACCOUNTS[*]:-none}. Failed: ${FAILED_ACCOUNTS[*]:-none}."
send_email "${OVERALL_STATUS}"
cleanup_and_shutdown
EOF
```

### Adding another inbox

Everything needed to add or remove an inbox lives in the `ACCOUNTS` array at
the top of `backup.sh` — nothing else in this repo needs to change. To back
up two inboxes on the same provider:

```bash
ACCOUNTS=(
  "personal:you@yourdomain.com:/mailbackup/personal-app-password"
  "work:you@company-domain.com:/mailbackup/work-app-password"
)
```

Then store the second inbox's app password the same way as step 4:

```bash
aws ssm put-parameter --name /mailbackup/work-app-password --value "work-inbox-app-password" --type SecureString --region eu-central-1
```

No IAM change, no second `.mbsyncrc`, no second copy of `backup.sh` — the
script generates its own mbsync config from this list at the start of every
run, and processes each inbox one at a time, uploading it as its own
separate S3 object (`mailbox-backup-<short-name>-<date>.zip.gpg`). If one
inbox's account details are wrong or its login fails, that inbox is reported
as failed in the summary email while the others still back up normally.

Now finish the setup:

```bash
sudo chmod +x /home/mailbackup/backup.sh
sudo chown mailbackup:mailbackup /home/mailbackup/backup.sh
```

Create the systemd unit:

```bash
sudo tee /etc/systemd/system/mailbox-backup.service > /dev/null << 'EOF'
[Unit]
Description=Encrypted mail backup to S3
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

Check your S3 bucket for the resulting `mailbox-backup-<short-name>-<date>.zip.gpg`
file(s), one per inbox. Once it works, the instance should already be
shutting itself down at the end of the script — confirm it actually stops,
then move on.

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
aws scheduler create-schedule --name weekly-mailbox-backup --schedule-expression "cron(0 3 ? * SUN *)" --schedule-expression-timezone "UTC" --flexible-time-window '{"Mode":"OFF"}' --target file://scheduler-target.json
```

Adjust the day/time/timezone to whatever suits you — this example runs
every Sunday at 3am UTC.

## Corruption checks

Every run automatically verifies each inbox's zip integrity (CRC check)
before encrypting it, and afterward compares the encrypted file's size
against the source zip's to catch truncated writes (e.g. from running out
of disk space mid-write). Both are logged per account and either one
failing aborts just that account's backup — see "Adding another inbox"
above for how failures are isolated per inbox.

Neither check can decrypt the archive to prove the content is fully
restorable — that would require the private key to live on the instance,
which defeats the point. (`gpg --list-packets` is deliberately NOT used for
this: for an encrypted message it always attempts real decryption and
always fails with "No secret key" on this box by design, regardless of
whether the file is actually fine — it could never pass, on any run, ever.
The size comparison in the script is the correct approach instead.) The
only way to prove an archive fully restores is the manual test below;
worth repeating every few months, not just once, and for each inbox.

## Restoring a backup

On any machine with your **private** key imported:

```bash
aws s3 cp s3://your-backup-bucket-name/mailbox-backup-personal-2026-09-07.zip.gpg .
gpg --import private-key-backup.asc   # only needed once per machine
gpg --decrypt mailbox-backup-personal-2026-09-07.zip.gpg > restored.zip
unzip restored.zip
```

You'll be prompted for your private key's passphrase. The result is a plain
Maildir folder you can point any mail client at, or re-import via `imapsync`
into a fresh mailbox. If you're backing up multiple inboxes, each one has
its own dated file in the bucket (`mailbox-backup-<short-name>-<date>.zip.gpg`)
and restores independently of the others.

## Cost recap (weekly cadence)

| Item | Monthly cost |
|---|---|
| EC2 (t4g.small, ~30–40 min **per inbox** per week, since accounts run sequentially) | ~$0.05/inbox |
| EBS (30GB gp3 — fixed, sized for your single largest inbox, not per account) | ~$2.40 |
| S3 Standard (last ~4 weekly backups, ~6GB each, per inbox) | ~$0.55/inbox |
| SSM Parameter Store, KMS (aws/ssm key), EventBridge Scheduler | $0 |
| **Total for one ~6GB inbox** | **~$3.00/month** |
| **Each additional ~6GB inbox** | **+~$0.60/month** |

- The EBS volume and the fixed AWS services stay flat regardless of how many inboxes you back up — they're driven by your single largest inbox and by the fact that there's only one scheduled boot a week, neither of which changes as you add accounts. 

- EC2 compute time and S3 storage, both scale with account count: EC2 because accounts are processed one after another rather than in parallel, so total runtime (and therefore cost) adds up per inbox — 2 inboxes at ~30 min each is roughly an hour a week, ~4 hours a month, still only a few cents at these instance sizes; and S3 because every inbox's backups persist in the bucket simultaneously and long-term, unlike the transient EBS scratch space. 

- If your largest inbox is bigger than ~6GB, resize the EBS volume to match that one inbox — the number of other, smaller inboxes alongside it doesn't matter.
