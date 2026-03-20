# WebDAV backup vs OneDrive (rclone): why you don’t see the file, and how to make it appear

## 1) What your current setup is doing

You have a **local WebDAV server** running in Docker:

- Container: `ghcr.io/hacdias/webdav`
- Storage: `./data` on the host → `/data` in the container
- User `will` is mapped to the subdirectory `/data/will` (per your `config.yml`)
- Your All API Hub WebDAV backup ended up here on the host:

`/opt/allapihub-webdav/data/will/all-api-hub-backup/all-api-hub-1-0.json`

That means: **All API Hub successfully uploaded a backup to your WebDAV server**, and the WebDAV server wrote it to local disk.

## 2) Why it won’t automatically show up in OneDrive via rclone

**WebDAV and OneDrive are separate storage systems.**

- Your WebDAV server stores files in **local disk** (`/opt/allapihub-webdav/data/...`).
- Your rclone “onedrive” remote stores files in **OneDrive cloud**.
- Unless you run **a sync/mount/serve step**, OneDrive will never see what gets written into `/opt/allapihub-webdav/data`.

So the missing piece is not All API Hub—it’s that there’s **no link** between:
- “WebDAV server folder” ↔ “OneDrive remote folder”

## 3) How All API Hub decides where to write the WebDAV file (repo evidence)

All API Hub treats the WebDAV URL as either:

- a **full file URL** (ends in `.json`), or
- a **directory URL** (doesn’t end in `.json`), in which case it auto-creates a deterministic path:

`all-api-hub-backup/all-api-hub-1-0.json`

Code reference: `src/services/webdav/webdavService.ts`

- `PROGRAM_NAME = "all-api-hub"`
- `BACKUP_FOLDER_NAME = "all-api-hub-backup"`
- `CONFIG_VERSION = "1-0"`
- `ensureFilename(url)` builds:
  `.../<BACKUP_FOLDER_NAME>/<PROGRAM_NAME>-<CONFIG_VERSION>.json`

Docs reference: `docs/docs/en/webdav-sync.md` (and `docs/docs/webdav-sync.md`) explains the same default path behavior.

## 4) Pick one approach to “make it appear in OneDrive”

### Option A — Periodically snapshot the current WebDAV backup into OneDrive (daily) + keep only the latest 3 copies (recommended)

**Goal:** You keep using WebDAV for multi-device sync (single canonical file), *and* you get a separate OneDrive “time machine” with retention.

**Important detail:** All API Hub WebDAV auto-sync typically overwrites the same path:

- Local file (from your container bind mount):
  - `/opt/allapihub-webdav/data/will/all-api-hub-backup/all-api-hub-1-0.json`

So to keep history, your OneDrive backup should be a **daily copy with a date in the filename**, e.g.:

- `.../all-api-hub-1-0-2026-03-20.json`

#### A.1 Snapshot script (upload daily + retain last 3)

Create a script on the server that runs the WebDAV container (example: `/opt/scripts/allapihub-webdav-snapshot.sh`).

This version is configured for your decisions:
- OneDrive destination: `Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub/webdav-snapshots`
- **One snapshot per day** (same filename per day → overwrites if rerun)
- Retain **latest 3** snapshots
- Date stamping is forced to **Asia/Taipei** to avoid “wrong day” around midnight if the server timezone differs

```bash
#!/usr/bin/env bash
set -euo pipefail

# Cron/systemd may have a minimal PATH; ensure `rclone` is found.
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Local file written by your WebDAV container
SRC_FILE="/opt/allapihub-webdav/data/will/all-api-hub-backup/all-api-hub-1-0.json"

# Your rclone remote + destination folder in OneDrive
# NOTE: rclone paths use forward slashes.
RCLONE_REMOTE="Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub/webdav-snapshots"

# Filename prefix for snapshots
PREFIX="all-api-hub-1-0"

# Use ISO date so lexicographic order == chronological order
# Force Taipei timezone for consistent naming
STAMP="$(TZ=Asia/Taipei date +%F)"
DEST_FILE="${PREFIX}-${STAMP}.json"

# Avoid overlapping runs
LOCK_FILE="/tmp/allapihub-webdav-snapshot.lock"
exec 9>"${LOCK_FILE}"
if ! flock -n 9; then
  echo "Another snapshot job is running; exiting." >&2
  exit 0
fi

if [[ ! -f "${SRC_FILE}" ]]; then
  echo "Source file not found: ${SRC_FILE}" >&2
  exit 1
fi

# 1) Upload snapshot (idempotent for same day)
rclone mkdir "${RCLONE_REMOTE}" >/dev/null 2>&1 || true
rclone copyto "${SRC_FILE}" "${RCLONE_REMOTE}/${DEST_FILE}" -P

# 2) Retain only the newest 3 snapshots (by filename ordering)
mapfile -t files < <(
  rclone lsf "${RCLONE_REMOTE}" \
    --files-only \
    --include "${PREFIX}-*.json" \
  | sort
)

count="${#files[@]}"
keep=3

if (( count > keep )); then
  delete_count=$((count - keep))
  for ((i=0; i<delete_count; i++)); do
    f="${files[$i]}"
    echo "Purging old snapshot: ${f}"
    rclone deletefile "${RCLONE_REMOTE}/${f}"
  done
fi

echo "Done. Total snapshots now: ${#files[@]} (kept last ${keep})."
```

Make it executable:

```bash
sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/allapihub-webdav-snapshot.sh
sudo chmod +x /opt/scripts/allapihub-webdav-snapshot.sh
```

**Important:** run this as the same Linux user that has your `rclone` config for `Onedrive-Yahooforsub-Tao` (often your non-root user, e.g. `will`).

Test it once manually:

```bash
/opt/scripts/allapihub-webdav-snapshot.sh
rclone lsf "Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub/webdav-snapshots" --max-depth 1
```

#### A.2 Schedule it daily (Asia/Taipei 04:44)

Yes: the correct pattern is **cron (or systemd timer) → run the snapshot script → script uploads + purges**.

**Cron (simple):**

- If your server timezone is already Asia/Taipei, use:

```cron
44 4 * * * /opt/scripts/allapihub-webdav-snapshot.sh >> "$HOME/allapihub-webdav-snapshot.log" 2>&1
```

- If your server timezone is *not* Asia/Taipei, use `CRON_TZ` to force Taipei scheduling:

```cron
CRON_TZ=Asia/Taipei
44 4 * * * /opt/scripts/allapihub-webdav-snapshot.sh >> "$HOME/allapihub-webdav-snapshot.log" 2>&1
```

Edit your user crontab (recommended: run as `will` so it uses `~/.config/rclone/rclone.conf`):

```bash
crontab -e
```

**systemd timer (recommended for servers):**

Create `/etc/systemd/system/allapihub-webdav-snapshot.service`:

```ini
[Unit]
Description=Snapshot All API Hub WebDAV backup into OneDrive (rclone)

[Service]
Type=oneshot
ExecStart=/opt/scripts/allapihub-webdav-snapshot.sh
```

Create `/etc/systemd/system/allapihub-webdav-snapshot.timer`:

```ini
[Unit]
Description=Daily OneDrive snapshot for All API Hub WebDAV

[Timer]
OnCalendar=*-*-* 04:44:00
Timezone=Asia/Taipei
Persistent=true
RandomizedDelaySec=10m

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now allapihub-webdav-snapshot.timer
sudo systemctl list-timers --all | grep allapihub
```

#### A.3 Quick verification commands

```bash
# Local exists
ls -lah /opt/allapihub-webdav/data/will/all-api-hub-backup/

# Remote snapshots
rclone lsf "Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub/webdav-snapshots" --max-depth 1
```

Pros
- WebDAV sync remains stable (single canonical file) for all devices.
- You get offsite history in OneDrive with deterministic retention.
- Works even if the extension overwrites the same WebDAV JSON repeatedly.

Cons
- Daily backups are “point-in-time” (not continuous).

Notes
- If you previously used a Windows path like `Scripts-ssh-ssl-keys\Allapihub`, switch to **forward slashes** in rclone paths.
- If you prefer “keep last 3 *days*” instead of “keep last 3 *copies*”, we can simplify retention with `rclone delete --min-age`.

Scheduling
- `cron` (quick)
- `systemd timer` (more robust)

### Option B — Mount OneDrive into the WebDAV data directory (near real-time)

**Concept:** rclone mounts OneDrive as a filesystem; WebDAV writes “locally”, but it’s actually writing into OneDrive.

Pros
- Files “appear” in OneDrive quickly.
- No extra copy job needed.

Cons
- More moving parts: FUSE mount + caching.
- Requires careful rclone mount flags (`--vfs-cache-mode writes` is usually required).
- If the mount goes down, WebDAV writes fail.

Sketch

```bash
mkdir -p /mnt/onedrive-allapihub
rclone mount \
  "Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub" \
  /mnt/onedrive-allapihub \
  --vfs-cache-mode writes \
  --allow-other \
  --dir-cache-time 1m

# Then point WebDAV server directory at /mnt/onedrive-allapihub
# (you’d update docker-compose volumes / container directory accordingly)
```

### Option C — Replace the WebDAV container with `rclone serve webdav` (OneDrive-backed WebDAV)

**Concept:** rclone exposes your OneDrive remote over WebDAV directly.

Pros
- Fewer layers: browser → WebDAV → OneDrive.
- No extra sync step.

Cons
- Different behavior from `hacdias/webdav` (auth, performance characteristics).
- Still need to secure the endpoint (TLS/reverse proxy).

Example

```bash
rclone serve webdav \
  "Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub" \
  --addr 127.0.0.1:6065 \
  --user will \
  --pass 'your-password' \
  --read-only=false
```

## 5) Debug checklist (fast triage)

### Confirm WebDAV upload really happened

The presence of `/opt/allapihub-webdav/data/will/all-api-hub-backup/all-api-hub-1-0.json` already indicates it did.

### Confirm rclone is looking at the right remote path

```bash
rclone about "Onedrive-Yahooforsub-Tao:"
rclone lsf "Onedrive-Yahooforsub-Tao:" --max-depth 2
rclone lsf "Onedrive-Yahooforsub-Tao:Scripts-ssh-ssl-keys/Allapihub" --max-depth 5
```

### Confirm you’re syncing the *correct local folder*

Your backup is in:
- `/opt/allapihub-webdav/data/will/all-api-hub-backup/`

Not in:
- the repo directory (the All API Hub source tree)
- whatever path you might have used previously for scripts/keys

### Confirm permissions won’t block the sync

Your file is `root:root` owned, but likely world-readable (`-rw-r--r--`), so `rclone` as user `will` can read it. If you later run into issues, fix ownership:

```bash
sudo chown -R will:will /opt/allapihub-webdav/data/will/all-api-hub-backup
```

## 6) Security / operational notes

- You mapped `127.0.0.1:6065:6065`, which is good (not publicly exposed).
- If you later expose it via a reverse proxy, use HTTPS and consider restricting by IP or adding additional auth at the proxy layer.
