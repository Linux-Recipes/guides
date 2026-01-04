# First Login to a Linux Server (What to Do After SSH)
<!-- difficulty: Beginner -->
<!-- tags: ssh, server, security, linux-basics, vps -->
<!-- summary: A safe, beginner-friendly checklist of what to do after your first SSH login to a Linux server. Covers updates, users, SSH hardening, firewall basics, and sanity checks. -->

Logging into a Linux server for the first time can feel intimidating.  
This guide walks you through **exactly what to do after your first SSH login**, in a safe order, without locking yourself out.

Designed for VPS, cloud servers, and home lab machines.

---

## Before you start
- You should already be connected via SSH:
```bash
ssh user@server-ip
```
- Commands assume **Ubuntu/Debian**, but apply broadly
- Use `sudo` when prompted

> [!WARNING]
> Do **not** close your SSH session until you’ve confirmed everything works.

---

## 1) Check who and where you are
```bash
whoami
hostname
pwd
```

This confirms:
- Your user
- The server name
- Your current directory

---

## 2) Update the system (always first)
```bash
sudo apt update
sudo apt upgrade -y
```

This fixes known bugs and security issues immediately.

> [!TIP]
> If you see a kernel upgrade, a reboot may be required later.

---

## 3) Set the correct timezone
```bash
timedatectl
```

If incorrect:
```bash
sudo timedatectl set-timezone America/Toronto
```

List options:
```bash
timedatectl list-timezones
```

---

## 4) Create a non-root user (if logged in as root)
If you logged in as `root`, create a regular user now.

```bash
adduser alice
usermod -aG sudo alice
```

Switch to the new user:
```bash
su - alice
```

> [!WARNING]
> Daily work should **never** be done as root.

---

## 5) Set up SSH keys (recommended)
On **your local machine**:
```bash
ssh-keygen
```

Copy the key to the server:
```bash
ssh-copy-id alice@server-ip
```

Test a new login **before proceeding**.

---

## 6) Lock down SSH (safe basics)
Edit the SSH config:
```bash
sudo nano /etc/ssh/sshd_config
```

Recommended settings:
```text
PermitRootLogin no
PasswordAuthentication no
```

Restart SSH:
```bash
sudo systemctl restart ssh
```

> [!WARNING]
> Test a second SSH session before closing the first one.

---

## 7) Enable a firewall (UFW)
Allow SSH first:
```bash
sudo ufw allow OpenSSH
```

Enable firewall:
```bash
sudo ufw enable
```

Check status:
```bash
sudo ufw status
```

---

## 8) Verify disk space and memory
```bash
df -h
free -h
```

This helps you spot:
- Tiny disks
- Memory constraints early

---

## 9) Check logs for errors
```bash
sudo journalctl -p 3 -xb
```

Look for obvious failures or warnings.

---

## 10) Reboot (if required)
If updates installed a new kernel:
```bash
sudo reboot
```

Reconnect via SSH after reboot.

---

## Quick security checklist
- [ ] System updated
- [ ] Non-root user created
- [ ] SSH keys working
- [ ] Root login disabled
- [ ] Firewall enabled
- [ ] SSH access confirmed after changes

---

## Common mistakes to avoid
- Disabling SSH before allowing it in the firewall
- Closing your only SSH session too early
- Working as root long-term
- Skipping updates

---

## What to do next
After first login, consider:
- **Secure SSH on Linux (Keys, UFW, Fail2ban)**
- **Automatic Security Updates**
- **Create a systemd Service**
- **Set up backups**

---

## Summary
- Update first
- Create a regular user
- Secure SSH carefully
- Enable a firewall
- Verify access before logging out

---

> [!TIP]
> If something goes wrong, most VPS providers offer a **console or recovery mode** — don’t panic.
