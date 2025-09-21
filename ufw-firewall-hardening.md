# UFW Firewall Hardening (Default-Deny, Rate-Limit SSH, HTTP/HTTPS)

Lock down your server with **UFW** (Uncomplicated Firewall). We’ll set **default-deny**, allow **SSH with rate-limit**, open **HTTP/HTTPS**, and add a few handy checks. Safe to copy/paste.

## Prerequisites
- A sudo-capable user
- Debian/Ubuntu (UFW is native). On RHEL-family, prefer `firewalld`, or install UFW from EPEL.
- An existing SSH session (so you don’t lock yourself out)

---

## 1) Install (if needed) and show current status
```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y ufw

sudo ufw status verbose
```

> [!NOTE]
> If UFW is inactive, that’s fine—we’ll configure rules first, then enable.

---

## 2) Set sane defaults (deny incoming, allow outgoing)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

---

## 3) Allow SSH (with rate-limit) and enable UFW
Allow SSH first, **then** enable UFW to avoid losing access.

```bash
# Allow SSH on the default port (22) and apply a brute-force rate limit
sudo ufw allow ssh
sudo ufw limit ssh comment 'Rate-limit SSH to deter brute-force'

# If you use a non-standard SSH port, do this instead (example: 2222)
# sudo ufw allow 2222/tcp
# sudo ufw limit 2222/tcp comment 'Rate-limit custom SSH port'

# Now enable UFW
sudo ufw enable

# Re-check
sudo ufw status numbered
```

> [!TIP]
> You can generate variations (custom port, protocol) with our **Command Builder**: [/tools/builder](/tools/builder)

---

## 4) Open web traffic (HTTP/HTTPS)
If you run Nginx or Apache, UFW ships with profiles. Use them:

```bash
# See available profiles
sudo ufw app list

# Nginx examples
sudo ufw allow "Nginx Full"      # opens 80/tcp and 443/tcp
# sudo ufw allow "Nginx HTTP"    # only 80/tcp
# sudo ufw allow "Nginx HTTPS"   # only 443/tcp

# Or generic rules if you prefer:
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp
```

> [!NOTE]
> If you followed our Let’s Encrypt guide, `Nginx Full` is what you want.

---

## 5) Common extras (pick what you need)
```bash
# Dev servers / Node / proxies
sudo ufw allow 3000/tcp
sudo ufw allow 8080/tcp

# Databases (prefer binding to 127.0.0.1 instead of exposing)
# sudo ufw allow from 203.0.113.0/24 to any port 5432 proto tcp comment 'PostgreSQL VPC only'
# sudo ufw allow from 10.0.0.0/8 to any port 3306 proto tcp comment 'MySQL internal'
```

> [!WARNING]
> Exposing databases to the internet is risky. Use private networks or SSH tunnels where possible.

---

## 6) Enable logging (light) and view events
```bash
# Levels: off, low, medium, high, full
sudo ufw logging on

# Log locations: /var/log/ufw.log (rsyslog), and kernel drops via journal
sudo tail -f /var/log/ufw.log
# Or:
journalctl -f -u rsyslog.service 2>/dev/null || true
```

> [!TIP]
> Want a scheduled log snapshot or alert? Use a **systemd timer** with our guide: [/guides/systemd-service/](/guides/systemd-service/)

---

## 7) Ensure IPv6 is covered
```bash
grep -E '^IPV6=' /etc/ufw/ufw.conf || true
sudo sed -i 's/^IPV6=.*/IPV6=yes/' /etc/ufw/ufw.conf
sudo ufw reload
```

> [!NOTE]
> With `IPV6=yes`, UFW manages both IPv4 and IPv6 rules.

---

## 8) Restrict by source (allowlist) — optional
```bash
# Only allow SSH from your office IP (example)
sudo ufw delete allow ssh
sudo ufw delete limit ssh
sudo ufw allow from 198.51.100.25 to any port 22 proto tcp comment 'Office SSH'
sudo ufw limit from 198.51.100.25 to any port 22 proto tcp

# Open HTTPS to the world, but a private admin panel only to your CIDR:
sudo ufw allow 443/tcp
sudo ufw allow from 203.0.113.0/24 to any port 8443 proto tcp comment 'Admin panel'
```

---

## 9) Verify effective rules
```bash
sudo ufw status numbered
sudo iptables -S | head -n 30     # ufw backend iptables view (legacy)
sudo nft list ruleset | less      # on nftables-backed systems
```

---

## Troubleshooting & maintenance

### I locked myself out 😬
- If you still have a connected SSH session, **re-open** your port:
  ```bash
  sudo ufw allow 22/tcp && sudo ufw reload
  ```
- If not, use host console/VNC from your provider to disable UFW:
  ```bash
  sudo ufw disable
  ```

### Delete or edit a specific rule
```bash
sudo ufw status numbered
# Example: delete rule #4
sudo ufw delete 4
```

### Reset to factory and re-apply
```bash
sudo ufw reset
# Re-add needed rules (SSH first!), then:
sudo ufw enable
```

### Make a repeatable policy (scriptable)
Use our **Cron Builder** to check and reconcile rules on a schedule: [/tools/cron](/tools/cron)

---

## Quick policy template (copy/paste)
```bash
# Defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH (adjust port as needed)
sudo ufw allow 22/tcp
sudo ufw limit 22/tcp comment 'Rate-limit SSH'

# Web
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Optional extras
# sudo ufw allow 8080/tcp
# sudo ufw allow from 10.0.0.0/8 to any port 5432 proto tcp

# IPv6 on and logging
sudo sed -i 's/^IPV6=.*/IPV6=yes/' /etc/ufw/ufw.conf
sudo ufw logging on

# Apply
sudo ufw enable
sudo ufw status verbose
```

> [!TIP]
> Pair this with **Fail2ban** for SSH & web auth hardening, and keep your system updated. See our SSH and Let’s Encrypt guides for a secure baseline.