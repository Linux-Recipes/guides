# Where Linux Logs Are Stored (Beginner-Friendly Guide)
<!-- difficulty: Beginner -->
<!-- tags: logs, troubleshooting, journald, syslog, linux-basics -->
<!-- summary: Learn where Linux logs are stored, how to read them, and which logs to check when something goes wrong. Covers /var/log, journalctl, and common services in a beginner-friendly way. -->

When something breaks on Linux, the answer is almost always in the **logs**.  
This guide explains **where Linux logs are stored**, what the common files mean, and how to read them safely as a beginner.

No prior logging knowledge required.

---

## The main log directory: `/var/log`
Most Linux logs live in:
```text
/var/log
```

List the files:
```bash
ls /var/log
```

You’ll see many files — don’t panic. You only need a few of them most of the time.

---

## Most important log files (quick map)

| Log file | What it’s for |
|--------|---------------|
| `/var/log/syslog` | General system messages |
| `/var/log/auth.log` | Logins, SSH, sudo |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dpkg.log` | Package installs/updates |
| `/var/log/apt/` | APT package manager logs |
| `/var/log/nginx/` | Nginx web server logs |
| `/var/log/apache2/` | Apache web server logs |

> [!NOTE]
> Some distros (RHEL/CentOS) use `/var/log/messages` instead of `syslog`.

---

## Viewing logs safely
Never edit log files — only **view** them.

### Use `less` (recommended)
```bash
sudo less /var/log/syslog
```

Navigation tips:
- `↑ ↓` scroll
- `/error` search
- `q` quit

---

### View the last lines with `tail`
```bash
sudo tail -n 50 /var/log/syslog
```

Follow live updates:
```bash
sudo tail -f /var/log/syslog
```

---

## Authentication & SSH logs
If login fails or sudo doesn’t work:
```bash
sudo less /var/log/auth.log
```

Common things logged:
- SSH logins
- Failed passwords
- `sudo` usage

---

## Service-specific logs
Many services have their own folders.

Example: Nginx
```bash
ls /var/log/nginx
```

Common files:
```text
access.log
error.log
```

View errors:
```bash
sudo less /var/log/nginx/error.log
```

---

## systemd logs (journalctl)
Modern Linux systems use **journald**.

### View system logs
```bash
journalctl
```

### Show recent errors only
```bash
journalctl -p 3 -xb
```

### Logs for one service
```bash
journalctl -u nginx
```

### Follow logs live
```bash
journalctl -u nginx -f
```

> [!TIP]
> `journalctl` is often more useful than digging through files.

---

## Kernel logs
Check hardware, drivers, and boot issues:
```bash
sudo dmesg | less
```

Or:
```bash
sudo less /var/log/kern.log
```

---

## Logs during boot
If the system fails to start properly:
```bash
journalctl -b
```

Previous boot:
```bash
journalctl -b -1
```

---

## Why some logs rotate
Logs don’t grow forever.

Rotation is handled by **logrotate**:
- Old logs get compressed (`.gz`)
- New logs start fresh

Example:
```text
syslog
syslog.1
syslog.2.gz
```

You can still read them:
```bash
zless /var/log/syslog.2.gz
```

---

## Common beginner mistakes
- Editing logs instead of viewing
- Looking in the wrong file
- Ignoring `journalctl`
- Forgetting `sudo`

---

## Quick troubleshooting guide

| Problem | Check |
|------|------|
| SSH login fails | `/var/log/auth.log` |
| Service won’t start | `journalctl -u service` |
| Website error | `/var/log/nginx/error.log` |
| System crash | `journalctl -p 3 -xb` |
| Boot issue | `journalctl -b` |

---

## Summary
- Most logs live in `/var/log`
- Use `less`, `tail`, and `journalctl`
- Logs explain *why* things fail
- Never edit log files

---

> [!TIP]
> When debugging: **check logs first, Google second**.
