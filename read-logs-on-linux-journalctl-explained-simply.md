# Read Logs on Linux (journalctl Explained Simply)
<!-- difficulty: Beginner -->
<!-- tags: logs, journalctl, systemd, troubleshooting -->
<!-- summary: A simple, beginner-friendly guide to reading Linux logs with journalctl. Learn the most useful commands to find errors, follow logs live, and debug services without overwhelm. -->

Modern Linux systems use **systemd**, which stores logs in a central journal.  
That journal is read using one command: **`journalctl`**.

This guide explains **journalctl in plain English**, with the few commands you actually need as a beginner.

---

## What is journalctl?
`journalctl` reads logs collected by **systemd-journald**.

It replaces (and complements):
- `/var/log/syslog`
- `/var/log/messages`

Think of it as a **searchable timeline of everything your system does**.

---

## View all logs (start here)
```bash
journalctl
```

This shows logs from:
- System boot
- Services
- Kernel
- Authentication

Use:
- `↑ ↓` to scroll
- `q` to quit

---

## Show recent logs only
Show the most recent entries:
```bash
journalctl -n 50
```

Follow logs live (like `tail -f`):
```bash
journalctl -f
```

---

## Show errors only (very useful)
```bash
journalctl -p 3 -xb
```

What this means:
- `-p 3` → errors only
- `-x` → extra explanations
- `-b` → current boot

This is often the **best first command when something breaks**.

---

## Logs for a specific service
Replace `nginx` with any service name.

```bash
journalctl -u nginx
```

Last 50 lines:
```bash
journalctl -u nginx -n 50
```

Follow live:
```bash
journalctl -u nginx -f
```

---

## Logs from the current boot
```bash
journalctl -b
```

Previous boot:
```bash
journalctl -b -1
```

Helpful after crashes or reboots.

---

## Search logs by keyword
```bash
journalctl | grep error
```

Or:
```bash
journalctl | grep failed
```

---

## Show logs with timestamps
```bash
journalctl --since "10 minutes ago"
```

Examples:
```bash
journalctl --since today
journalctl --since yesterday
journalctl --since "2024-01-01"
```

---

## Check who logged in or used sudo
```bash
journalctl _COMM=sshd
```

Or:
```bash
journalctl | grep sudo
```

---

## View logs as a normal user
By default, full logs require sudo.

To allow non-root access:
```bash
sudo usermod -aG systemd-journal youruser
```

Log out and back in.

---

## Why journalctl is powerful
- Centralized logs
- Structured metadata
- Time-based filtering
- Service-aware

You don’t need to memorize everything — just a few commands.

---

## Common beginner mistakes
- Forgetting `sudo`
- Reading old log files instead
- Not filtering by service
- Panic-scrolling instead of searching

---

## Quick cheat sheet

| Goal | Command |
|----|----|
| All logs | `journalctl` |
| Errors only | `journalctl -p 3 -xb` |
| Service logs | `journalctl -u service` |
| Follow live | `journalctl -f` |
| Last boot | `journalctl -b` |

---

## When to use journalctl vs /var/log
- **Use journalctl** → systemd services, crashes, boot issues
- **Use /var/log** → app-specific logs (nginx, apache)

They work together.

---

## Summary
- `journalctl` reads system logs
- Filter by service, time, or severity
- Errors first, details second
- Logs explain *why* things fail

---

> [!TIP]
> When debugging: run `journalctl -p 3 -xb` before doing anything else.
