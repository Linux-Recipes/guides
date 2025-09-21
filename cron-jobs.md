---
level: Beginner   # Beginner | Intermediate | Advanced
tags: ssh, security   # optional, comma-separated
---

# Cron Jobs on Linux (Safe Scheduling, Logging, No Overlaps)

Set up reliable **cron** jobs with proper **logging**, **lock files** (no overlapping runs), and sane **environment** defaults. Safe to copy/paste. Use the [Cron Builder](/tools/cron) to generate schedules quickly.

## Prerequisites
- A sudo-capable user (root only when necessary)
- A script or command you want to run on a schedule
- Basic terminal access

---

## 1) Create a wrapper script for your job
Keep your crontab clean and put the real work in a script. This also makes local testing easy.

```bash
sudo install -d -m 0755 /opt/myjobs
sudo tee /opt/myjobs/backup.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

# Example job: rsync a directory (adjust to your needs)
SRC="/srv/data/"
DST="/srv/backups/data-$(date +%F)"
LOG="/var/log/myjobs/backup.log"

mkdir -p "$DST"
rsync -a --delete "$SRC" "$DST"

echo "$(date -Is) OK: backup completed to $DST" >> "$LOG"
EOF

sudo chmod 0755 /opt/myjobs/backup.sh
sudo install -d -m 0755 /var/log/myjobs
```

> [!TIP]
> Keep job logic in version-controlled scripts; your crontab stays small and readable.

---

## 2) Prevent overlapping runs with `flock`
If a job might still be running when the next one starts, add a **lock**. This avoids corruption, duplicate work, or resource spikes.

```bash
# Run the script guarded by a non-blocking lock:
flock -n /var/lock/backup.lock -c "/opt/myjobs/backup.sh"
# If the lock exists, cron run will skip (non-zero exit). That's OK.
```

> [!NOTE]
> `flock` is provided by the util-linux package on most distros.

---

## 3) Set up a robust crontab (per user)
Open your crontab and add a sensible environment header so jobs behave like your shell.

```bash
crontab -e
```

Paste this header at the top (tune paths/timezone):

```bash
# --- sane cron environment ---
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=""                 # empty: disable mail; set to your email to receive output
CRON_TZ=UTC               # run jobs in UTC; change/remove if you prefer localtime
```

Now add your job lines. Use **absolute paths** and redirect output so logs go to files you control:

```bash
# Every day at 03:30 UTC (build with /tools/cron)
30 3 * * * flock -n /var/lock/backup.lock -c "/opt/myjobs/backup.sh" >>/var/log/myjobs/backup.log 2>&1
```

> [!TIP]
> Generate schedules with the [Cron Builder](/tools/cron). Copy the five-field expression directly into your crontab.

---

## 4) Test your job now (don’t wait for the schedule)
Run it manually to validate permissions, paths, and logs:

```bash
/opt/myjobs/backup.sh
tail -n 50 /var/log/myjobs/backup.log
```

If you are testing the **exact cron line** behavior (including `flock` and redirection):

```bash
bash -lc 'flock -n /var/lock/backup.lock -c "/opt/myjobs/backup.sh" >>/var/log/myjobs/backup.log 2>&1'
```

---

## 5) Useful cron examples
- **Every 5 minutes**: `*/5 * * * *`
- **Hourly at minute 7**: `7 * * * *`
- **Daily at 02:00**: `0 2 * * *`
- **Mon–Fri at 18:30**: `30 18 * * 1-5`
- **1st of month at 01:15**: `15 1 1 * *`

> [!WARNING]
> Avoid `* * * * *` polling unless the task is trivial and idempotent—use backoff or a queue.

---

## 6) Logging patterns you can copy
Append and rotate with `logrotate` (recommended):

```bash
# Append both stdout and stderr to a dated log (rotate daily via logrotate)
flock -n /var/lock/report.lock -c "/opt/myjobs/report.sh" >>/var/log/myjobs/report.log 2>&1
```

Minimal `logrotate` snippet (save as `/etc/logrotate.d/myjobs`):

```bash
/var/log/myjobs/*.log {
  daily
  rotate 14
  compress
  missingok
  notifempty
  copytruncate
}
```

---

## 7) Common gotchas (and fixes)
- **Relative paths** fail under cron. Use absolute paths everywhere (`/opt/...`, `/usr/bin/...`).
- **Different PATH** under cron. Add a PATH line at the top of your crontab or call binaries with full paths (`/usr/bin/rsync`).
- **Locale-dependent scripts**: set `LC_ALL=C` (or desired locale) in the crontab header.
- **Jobs that need network**: schedule a bit after boot or use a **systemd service/timer** that `Wants=network-online.target`. See: [Create a systemd Service](/guides/systemd-service/).

---

## 8) Bonus: convert a cron job to a systemd timer
For long-running or network-dependent tasks, timers are often more reliable than cron and integrate with `journalctl`.

- Guide: [Create a systemd Service (Auto-Restart, Logs, Security)](/guides/systemd-service/)
- Then add a timer in that guide (e.g., `OnCalendar=daily`).

---

## Remove a job cleanly
List, edit, or remove entries:

```bash
crontab -l
crontab -e
crontab -r   # removes the entire crontab for the current user (careful)
```

> [!DANGER]
> `crontab -r` deletes *all* entries for that user. Prefer `crontab -e` and comment out the lines you no longer need.