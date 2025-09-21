# Fail2ban Hardening: Block SSH & Web Brute-Force (UFW/NFT, Recidive)

Automatically ban attackers using **Fail2ban**. We’ll protect **SSH** (and optionally **Nginx**) with **rate-limited bans**, whitelist your IP, and enable the **recidive** jail for repeat offenders. Safe to copy/paste.

## Prerequisites
- A sudo-capable user
- Debian/Ubuntu (other distros fine with path tweaks)
- Logs available via **systemd-journal** (default) or files
- Optional: **UFW** already configured (see [/guides/ufw-hardening/](/guides/ufw-hardening/))

---

## 1) Install Fail2ban
```bash
sudo apt update
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo systemctl status --no-pager fail2ban
```

> [!NOTE]
> The default install already protects SSH on many distros, but we’ll create **/etc/fail2ban/jail.local** with sane, documented settings you can tweak later.

---

## 2) Create `/etc/fail2ban/jail.local` (SSH, logging, sane defaults)
The config below:
- Uses the **systemd journal** as the log backend (no fragile file paths)
- Bans get **longer** for repeat offenses (`bantime.increment = true`)
- Uses the **UFW** action if present; else falls back to **nftables** (adjust if you use iptables)

```bash
sudo tee /etc/fail2ban/jail.local >/dev/null <<'EOF'
[DEFAULT]
# Use systemd-journal for parsing (robust across services)
backend = systemd

# Base ban policy
findtime = 10m
maxretry = 5
bantime  = 30m
bantime.increment = true
bantime.factor = 1.5
bantime.maxtime = 12h

# Write our own log for recidive & audits
logtarget = /var/log/fail2ban.log

# Whitelist (replace with your office/home IP/CIDR)
ignoreip = 127.0.0.1/8 ::1

# Firewall action:
# If UFW is enabled, use it; otherwise nftables is a good modern default.
# Uncomment ONE "action" line below as appropriate.

#action = ufw
action = nftables-multiport

# ---- JAILS ----

[sshd]
enabled = true
port = ssh
filter = sshd
# Ban quick guessers more aggressively
maxretry = 4
findtime = 10m
bantime = 1h

# Optional: extend ban if they hit many different usernames quickly
# (requires filter shipped by fail2ban)
# mode = aggressive

# Example nginx auth jail (optional; enable if you use HTTP auth)
[nginx-http-auth]
enabled = false
port    = http,https
filter  = nginx-http-auth
logpath = journal
backend = systemd

# Example: hunt for bad/known bot paths (ships with fail2ban)
[nginx-botsearch]
enabled = false
port    = http,https
filter  = nginx-botsearch
logpath = journal
backend = systemd

# Catch repeat offenders across all jails (reads fail2ban.log)
[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
bantime  = 24h
findtime = 12h
maxretry = 5
EOF
```

> [!TIP]
> If you’re using **UFW**, set `action = ufw` in `[DEFAULT]`. For raw iptables, use `iptables-multiport` (legacy) or `nftables-multiport` on nft systems.

---

## 3) Whitelist your IP(s)
```bash
# Add your current public IP (replace 198.51.100.25)
sudo sed -i 's/^ignoreip = .*/ignoreip = 127.0.0.1\/8 ::1 198.51.100.25/' /etc/fail2ban/jail.local
```

> [!WARNING]
> Always whitelist a known-good IP **before** tightening rules on a remote machine, so you don’t lock yourself out.

---

## 4) Reload and verify jails
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo tail -n 50 /var/log/fail2ban.log
```

Expected: `sshd` should appear enabled with your chosen `bantime/findtime/maxretry`.

---

## 5) Test the SSH jail (safe)
From another host/IP (not your whitelist), attempt a few bad logins to trigger a ban:

```bash
# Intentionally fail a few times (replace host)
ssh -p 22 baduser@your.server.example
# … repeat 4-5 times …
```

Then check:
```bash
sudo fail2ban-client status sshd
sudo fail2ban-client get sshd banip
sudo journalctl -u ssh -n 100 --no-pager
```

To **unban** for testing:
```bash
# Replace 203.0.113.77 with the test IP
sudo fail2ban-client set sshd unbanip 203.0.113.77
```

---

## 6) Enable Nginx jails (optional)
If you use HTTP auth or want to block common bot probes, toggle these to `enabled = true` in `/etc/fail2ban/jail.local`, then:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status nginx-http-auth
sudo fail2ban-client status nginx-botsearch
```

> [!NOTE]
> Make sure your **Nginx logs** go to the journal (`backend = systemd`) or set `logpath = /var/log/nginx/access.log` accordingly.

---

## 7) Useful commands (daily ops)
```bash
# Overview
sudo fail2ban-client status

# Inspect a jail
sudo fail2ban-client status sshd

# See banned IPs
sudo fail2ban-client get sshd banip

# Ban/unban manually
sudo fail2ban-client set sshd banip   203.0.113.77
sudo fail2ban-client set sshd unbanip 203.0.113.77

# Test a filter against sample log lines
sudo fail2ban-regex /var/log/fail2ban.log /etc/fail2ban/filter.d/sshd.conf
```

---

## 8) Troubleshooting
- **No bans?** Check backend/logs:
  ```bash
  sudo tail -n 100 /var/log/fail2ban.log
  sudo journalctl -u fail2ban --no-pager -n 100
  ```
  Ensure `backend = systemd` (journal) or correct `logpath` if using files.
- **UFW vs nftables**: Pick *one* `action` in `[DEFAULT]`.
- **Containers**: If sshd/nginx run in Docker, expose logs to the host or use a jail pointed at container logs.
- **Aggressive SSH**: Add `mode = aggressive` under `[sshd]` to catch username spam.

---

## Quick baseline (copy/paste)
```bash
# Install + enable
sudo apt update && sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# Baseline jail.local (SSH + recidive, nftables action)
sudo tee /etc/fail2ban/jail.local >/dev/null <<'EOF'
[DEFAULT]
backend = systemd
findtime = 10m
maxretry = 5
bantime  = 30m
bantime.increment = true
bantime.factor = 1.5
bantime.maxtime = 12h
logtarget = /var/log/fail2ban.log
ignoreip = 127.0.0.1/8 ::1
action   = nftables-multiport

[sshd]
enabled  = true
maxretry = 4
findtime = 10m
bantime  = 1h

[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
bantime  = 24h
findtime = 12h
maxretry = 5
EOF

sudo systemctl restart fail2ban
sudo fail2ban-client status
```

> [!TIP]
> Pair Fail2ban with our **UFW hardening** and **SSH hardening** guides for a solid baseline. For scheduled audits/alerts, see the **systemd timer** helper: [/guides/systemd-service/](/guides/systemd-service/).