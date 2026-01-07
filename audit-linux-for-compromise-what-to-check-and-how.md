# Audit Linux for Compromise (What to Check & How)
<!-- difficulty: Advanced -->
<!-- tags: security, incident-response, auditing, linux-security -->
<!-- summary: A practical guide to auditing a Linux system for signs of compromise. Covers user accounts, SSH access, running processes, persistence mechanisms, logs, and what to do if you find something suspicious. -->

If you suspect a Linux system may be compromised, **do not panic** — but **do act carefully**.  
This guide walks through a **practical, step-by-step audit** to help you identify common signs of compromise and decide what to do next.

This is **defensive auditing**, not hacking.

---

## Before you begin (important)
- Assume **attackers may still have access**
- Do **not** install random tools yet
- Avoid rebooting unless necessary
- If this is production, consider isolating the server first

> [!WARNING]
> If sensitive data is involved (PII, credentials, payments), treat this as a **security incident** and follow your organization’s response policy.

---

## Step 1) Check logged-in users
See who is logged in right now:
```bash
who
w
```

List recent logins:
```bash
last -a
```

Red flags:
- Logins at odd hours
- Unknown IPs
- Users you don’t recognize

---

## Step 2) Audit user accounts
List all users:
```bash
cut -d: -f1 /etc/passwd
```

Look for:
- Unknown users
- Users with shells who shouldn’t have them

Check sudo access:
```bash
getent group sudo
```

---

## Step 3) Check SSH access
Inspect authorized keys:
```bash
sudo find /home -name authorized_keys -exec ls -l {} \;
```

View keys:
```bash
sudo less /home/*/.ssh/authorized_keys
```

Check root keys:
```bash
sudo less /root/.ssh/authorized_keys
```

Red flags:
- Keys you didn’t add
- Keys without comments
- Recently modified files

---

## Step 4) Review authentication logs
```bash
sudo less /var/log/auth.log
```

Or with journalctl:
```bash
journalctl -u ssh -n 200
```

Look for:
- Repeated failed logins
- Successful logins after many failures
- Logins from unexpected IPs

---

## Step 5) Check running processes
List processes:
```bash
ps auxf
```

Look for:
- Unknown processes
- Processes running as root unexpectedly
- Strange names mimicking system processes

Sort by start time:
```bash
ps -eo pid,lstart,cmd | less
```

---

## Step 6) Check network activity
Active connections:
```bash
ss -tulpn
```

Outbound connections:
```bash
ss -antp
```

Red flags:
- Unexpected listening ports
- Connections to unknown external IPs

---

## Step 7) Check persistence mechanisms
Attackers often add persistence.

### systemd services
```bash
systemctl list-unit-files --type=service
```

Check suspicious services:
```bash
systemctl status suspicious.service
```

---

### Cron jobs
System-wide:
```bash
ls /etc/cron*
```

User crons:
```bash
sudo crontab -l
sudo crontab -u username -l
```

---

### Startup scripts
```bash
ls -la /etc/init.d
ls -la /etc/rc.local
```

---

## Step 8) Check recently modified files
```bash
sudo find / -xdev -mtime -2 -type f 2>/dev/null | less
```

Look for:
- Files modified recently
- Changes in system directories

---

## Step 9) Check package integrity (Debian/Ubuntu)
```bash
sudo debsums -s
```

Missing or modified files are suspicious.

---

## Step 10) Look for common malware signs
Check for:
- Crypto miners (high CPU usage)
- Hidden directories (e.g., `.cache/.something`)
- Suspicious binaries in `/tmp`, `/var/tmp`

```bash
ls -la /tmp
ls -la /var/tmp
```

---

## Step 11) Check kernel messages
```bash
dmesg | less
```

Look for:
- Module load errors
- Security warnings

---

## What to do if you find something
- Disconnect the server from the network
- Preserve logs and evidence
- Rotate all credentials
- Rebuild from a known-good backup if possible

> [!WARNING]
> **Do not trust a compromised system again.** Reinstall if integrity matters.

---

## What NOT to rely on
- Antivirus alone
- “I don’t see anything wrong”
- Restarting and hoping it’s gone

---

## After cleanup (hardening checklist)
- Change all passwords & SSH keys
- Enable automatic updates
- Use Fail2ban
- Disable root SSH
- Review firewall rules
- Audit users regularly

---

## Summary
- Audit users, SSH, processes, network, persistence
- Logs are your best evidence
- When in doubt, rebuild
- Security is about **reducing trust, not restoring it**

---

> [!TIP]
> If compromise is confirmed, **reinstalling is safer than trying to clean**.
