# Rsync Backups with systemd Timers (Incremental + Pruning)

Create dependable **incremental backups** with `rsync` using **hard links** (space-efficient snapshots), run them on a **systemd timer**, and keep only the latest **daily/weekly/monthly** sets. Safe to copy/paste.

## Prerequisites
- A sudo-capable user
- `rsync` installed (`sudo apt install rsync` on Debian/Ubuntu)
- Backup target:
  - **Local disk** mounted at `/mnt/backup` **or**
  - **Remote SSH** account like `backup@backup.example` with key auth
- Optional: see [/guides/systemd-service/](/guides/systemd-service/) if you want to customize units

---

## 1) Prepare folders, SSH key (if remote), and exclude list
```bash
# Create a dedicated directory for backup config/scripts
sudo install -d -m 0755 /etc/backup
sudo install -d -m 0750 /var/log/backup

# (Remote only) Generate a deploy key for rsync over SSH and copy to backup host
sudo ssh-keygen -t ed25519 -a 100 -f /etc/backup/backup_ed25519 -N ''
# On the backup server, append the public key to the backup user's authorized_keys:
#   sudo -u backup mkdir -p ~backup/.ssh && cat /etc/backup/backup_ed25519.pub | ssh backup@backup.example 'cat >> ~/.ssh/authorized_keys'

# Create a sane exclude list (edit freely)
sudo tee /etc/backup/rsync-excludes.txt >/dev/null <<'EOF'
# Volatile / not needed
/proc/
/sys/
/dev/
/run/
/tmp/
/mnt/backup/
/var/tmp/
/var/cache/
/var/lib/systemd/coredump/

# Big, optional caches
$HOME/.cache/
*.iso
*.img
EOF
```

> [!NOTE]
> This guide snapshots **/home** and **/etc** by default. Add paths you need, but be careful with `/` root-level backups unless you know what to exclude.

---

## 2) Install the backup script (hard-link snapshots + retention)
```bash
sudo tee /usr/local/sbin/rsync-backup.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

# ===== Config =====
# What to back up (space-separated)
SRC_LIST=(
  "/home"
  "/etc"
)

# Destination:
#  - Local:  DEST_BASE="/mnt/backup/$(hostname -s)"
#  - Remote: DEST_BASE="backup@backup.example:/srv/backups/$(hostname -s)"
DEST_BASE="/mnt/backup/$(hostname -s)"

# SSH (used only if DEST_BASE contains user@host:)
SSH_OPTS="-i /etc/backup/backup_ed25519 -o StrictHostKeyChecking=accept-new"

# Exclusions
EXCLUDES_FILE="/etc/backup/rsync-excludes.txt"

# Retention (snapshots to keep)
KEEP_DAILY=7
KEEP_WEEKLY=4
KEEP_MONTHLY=6

# Log file
LOG_FILE="/var/log/backup/rsync-backup.log"

# ===== Functions =====
ts() { date +"%Y-%m-%d %H:%M:%S"; }
info() { echo "[$(ts)] $*" | tee -a "$LOG_FILE"; }
err()  { echo "[$(ts)] ERROR: $*" | tee -a "$LOG_FILE" >&2; }

is_remote() {
  [[ "$DEST_BASE" =~ ^[^/:]+@[^/:]+: ]]
}

list_snapshots() {
  if is_remote; then
    ssh -o BatchMode=yes $SSH_OPTS "${DEST_BASE%%:*}" "ls -1t ${DEST_BASE#*:}/snapshots 2>/dev/null || true"
  else
    ls -1t "$DEST_BASE/snapshots" 2>/dev/null || true
  fi
}

remove_snapshot() {
  local snap="$1"
  if is_remote; then
    ssh -o BatchMode=yes $SSH_OPTS "${DEST_BASE%%:*}" "rm -rf -- ${DEST_BASE#*:}/snapshots/$snap"
  else
    rm -rf -- "$DEST_BASE/snapshots/$snap"
  fi
}

ensure_dirs() {
  if is_remote; then
    local host="${DEST_BASE%%:*}" path="${DEST_BASE#*:}"
    ssh -o BatchMode=yes $SSH_OPTS "$host" "mkdir -p '$path/snapshots'"
  else
    sudo install -d -m 0755 "$DEST_BASE/snapshots"
  fi
}

latest_snapshot_path() {
  local latest
  latest="$(list_snapshots | head -n1 || true)"
  [[ -n "$latest" ]] && echo "snapshots/$latest" || true
}

rsync_cmd() {
  local link_dest_arg=()
  local latest; latest="$(latest_snapshot_path || true)"
  if [[ -n "$latest" ]]; then
    link_dest_arg=(--link-dest="../$latest")
  fi

  # Build include/exclude args
  local ex_args=()
  [[ -f "$EXCLUDES_FILE" ]] && ex_args+=(--exclude-from="$EXCLUDES_FILE")

  local dest_snap="snapshots/$(date +'%Y-%m-%d_%H%M%S')"
  local dest_root
  if is_remote; then
    local host="${DEST_BASE%%:*}" path="${DEST_BASE#*:}"
    dest_root="$host:$path/$dest_snap"
  else
    dest_root="$DEST_BASE/$dest_snap"
  fi

  # Compose rsync
  local base=(rsync -aAXH --numeric-ids --delete --delete-excluded
              --info=stats2,progress2 --human-readable)
  if is_remote; then
    base+=(-e "ssh $SSH_OPTS")
  fi

  # Create an empty target dir first (helps with link-dest relative paths)
  if is_remote; then
    ssh -o BatchMode=yes $SSH_OPTS "${DEST_BASE%%:*}" "mkdir -p '${DEST_BASE#*:}/$dest_snap'"
  else
    mkdir -p "$DEST_BASE/$dest_snap"
  fi

  # Run one rsync call per SRC (allows mixing top-level paths)
  for SRC in "${SRC_LIST[@]}"; do
    "${base[@]}" "${ex_args[@]}" "${link_dest_arg[@]}" "$SRC"/ "$dest_root$SRC"/
  done

  # Write a marker with the run date and hostname
  if is_remote; then
    ssh -o BatchMode=yes $SSH_OPTS "${DEST_BASE%%:*}" \
      "printf '%s\n' \"host=$(hostname -f)\" \"date=$(date -Iseconds)\" > '${DEST_BASE#*:}/$dest_snap/._snapshot_meta'"
  else
    printf '%s\n' "host=$(hostname -f)" "date=$(date -Iseconds)" > "$DEST_BASE/$dest_snap/._snapshot_meta"
  fi

  echo "$dest_snap"
}

prune_old() {
  # Strategy: keep N most-recent daily, weekly (Sun), monthly (1st)
  mapfile -t snaps < <(list_snapshots)
  [[ ${#snaps[@]} -eq 0 ]] && return 0

  local -a daily=() weekly=() monthly=()
  for s in "${snaps[@]}"; do
    # s format: YYYY-MM-DD_HHMMSS
    local d="${s%%_*}"
    local y m day; IFS=- read -r y m day <<< "$d"
    # ISO weekday (1-7, Mon-Sun)
    local wd; wd="$(date -d "$d" +%u 2>/dev/null || date -j -f %F "$d" +%u)"
    # Day of month
    local dom; dom="$(date -d "$d" +%d 2>/dev/null || date -j -f %F "$d" +%d)"
    daily+=("$s")
    [[ "$wd" == "7" ]] && weekly+=("$s")
    [[ "$dom" == "01" ]] && monthly+=("$s")
  done

  prune_list=()
  keep_from_list() {
    local -n arr=$1
    local keep=$2
    local -a keep_arr=()
    for i in "${!arr[@]}"; do
      (( i < keep )) && keep_arr+=("${arr[$i]}")
    done
    printf '%s\n' "${keep_arr[@]}"
  }

  # Build a set of snapshots to keep
  mapfile -t keep_daily   < <(keep_from_list daily   "$KEEP_DAILY")
  mapfile -t keep_weekly  < <(keep_from_list weekly  "$KEEP_WEEKLY")
  mapfile -t keep_monthly < <(keep_from_list monthly "$KEEP_MONTHLY")
  keep_all="$(printf '%s\n' "${keep_daily[@]}" "${keep_weekly[@]}" "${keep_monthly[@]}" | sort -u)"

  # Delete anything not in keep_all
  while IFS= read -r s; do
    echo "$keep_all" | grep -qx "$s" || prune_list+=("$s")
  done < <(printf '%s\n' "${snaps[@]}")

  for s in "${prune_list[@]:-}"; do
    info "Pruning old snapshot: $s"
    remove_snapshot "$s" || err "Failed to remove $s"
  done
}

main() {
  info "=== rsync-backup start ==="
  ensure_dirs
  snap="$(rsync_cmd)"
  info "Created snapshot: $snap"
  prune_old
  info "=== rsync-backup done ==="
}

main "$@"
EOF

sudo chmod 0755 /usr/local/sbin/rsync-backup.sh
```

> [!TIP]
> Change `DEST_BASE` to a remote path like `backup@backup.example:/srv/backups/$(hostname -s)` to push over SSH. Make sure the **public key** from `/etc/backup/backup_ed25519.pub` is in the remote user’s `~/.ssh/authorized_keys`.

---

## 3) Run once manually, verify snapshot
```bash
sudo /usr/local/sbin/rsync-backup.sh

# List snapshots (local)
ls -1 /mnt/backup/$(hostname -s)/snapshots | head

# Or if remote:
# ssh -i /etc/backup/backup_ed25519 backup@backup.example 'ls -1 /srv/backups/$(hostname -s)/snapshots | head'
```

---

## 4) Add systemd service + timer (daily at 03:15)
```bash
# Service unit
sudo tee /etc/systemd/system/rsync-backup.service >/dev/null <<'EOF'
[Unit]
Description=Rsync incremental backup snapshot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/rsync-backup.sh
User=root
Group=root
Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=6
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/mnt/backup /var/log/backup /etc/backup
CapabilityBoundingSet=
AmbientCapabilities=
EOF

# Timer (daily at 03:15, with catch-up)
sudo tee /etc/systemd/system/rsync-backup.timer >/dev/null <<'EOF'
[Unit]
Description=Daily rsync snapshot timer

[Timer]
OnCalendar=*-*-* 03:15:00
Persistent=true
AccuracySec=1m

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now rsync-backup.timer

# Check next runs
systemctl list-timers | grep rsync-backup || true
```

> [!NOTE]
> Prefer **systemd timers** over cron for better visibility and catch-up. If you prefer cron, you can generate a line with the [/tools/cron](/tools/cron) builder.

---

## 5) Restore (browse snapshots like read-only full copies)
Each snapshot is a **complete view** of your files thanks to hard links.

```bash
# Example: restore a single file from a date
SNAP="/mnt/backup/$(hostname -s)/snapshots/2025-09-20_031500"
sudo cp -av "$SNAP/etc/yourapp/config.yml" /etc/yourapp/config.yml

# Or rsync a whole tree back:
sudo rsync -aAXH "$SNAP/home/youruser/" /home/youruser/
```

---

## Troubleshooting
- **Permission denied** on remote: ensure remote user owns target path and your key is authorized.
- **Disk fills quickly**: Add more paths to `rsync-excludes.txt`, or back up only critical dirs.
- **Sparse/huge files**: Consider `--sparse` if you have VM images; or exclude them.
- **SELinux/ACLs**: `-A` and `-X` preserve ACLs and extended attributes; remove if unneeded.

> [!TIP]
> Pair this with **Nginx + Let’s Encrypt** if you’re backing up web roots (see [/guides/nginx-reverse-proxy/](/guides/nginx-reverse-proxy/)). For service-managed apps, see [/guides/systemd-service/](/guides/systemd-service/).