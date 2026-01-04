# Open a Port on Linux (UFW Beginner Guide)
<!-- difficulty: Beginner -->
<!-- tags: firewall, ufw, networking, security, ports -->
<!-- summary: A beginner-friendly guide to opening ports on Linux using UFW. Learn how to allow SSH, HTTP/HTTPS, custom ports, and verify firewall rules safely without locking yourself out. -->

Opening a port on Linux is a common task — and a common source of mistakes.  
This guide shows how to **safely open ports using UFW (Uncomplicated Firewall)**, with clear examples and checks so you don’t lock yourself out.

Beginner-friendly and copy/paste safe.

---

## Before you start
- This guide assumes **Ubuntu/Debian**
- You must have **sudo** access
- If you’re connected via SSH, **do not close your session**

Check if UFW is installed:
```bash
sudo ufw status
```

If not installed:
```bash
sudo apt install -y ufw
```

---

## What is UFW?
UFW is a simple firewall frontend that manages `iptables`/`nftables` for you.

It:
- Blocks unwanted traffic
- Allows only the ports you specify
- Is enabled by default on many servers

---

## Step 1) Always allow SSH first (critical)
If you’re connected over SSH, allow it **before enabling UFW**.

```bash
sudo ufw allow OpenSSH
```

Equivalent to:
```bash
sudo ufw allow 22/tcp
```

Verify:
```bash
sudo ufw status verbose
```

> [!WARNING]
> Forgetting this step is the #1 way people lock themselves out.

---

## Step 2) Enable the firewall
```bash
sudo ufw enable
```

Confirm when prompted.

Check status:
```bash
sudo ufw status
```

---

## Step 3) Open common ports

### HTTP (websites)
```bash
sudo ufw allow 80/tcp
```

### HTTPS (secure websites)
```bash
sudo ufw allow 443/tcp
```

### Both at once
```bash
sudo ufw allow 'Nginx Full'
```

---

## Step 4) Open a custom port

### Example: open port 3000
```bash
sudo ufw allow 3000/tcp
```

### Example: UDP port (VPNs, games)
```bash
sudo ufw allow 51820/udp
```

---

## Step 5) Restrict access (recommended)
Limit a port to one IP:
```bash
sudo ufw allow from 203.0.113.10 to any port 22
```

Limit a subnet:
```bash
sudo ufw allow from 192.168.1.0/24 to any port 5432
```

---

## Step 6) Check open ports
```bash
sudo ufw status numbered
```

Example output:
```text
[ 1] 22/tcp    ALLOW IN    Anywhere
[ 2] 80/tcp    ALLOW IN    Anywhere
```

---

## Step 7) Remove a rule

Using rule number:
```bash
sudo ufw delete 2
```

Or by rule:
```bash
sudo ufw delete allow 3000/tcp
```

---

## Step 8) Verify from the server
Check listening services:
```bash
ss -tulpn
```

Test locally:
```bash
curl http://localhost:3000
```

---

## Step 9) Test from outside
From another machine:
```bash
nc -vz server-ip 3000
```

Or use an online port checker.

---

## Common beginner mistakes
- Enabling UFW before allowing SSH
- Opening ports for services that aren’t running
- Forgetting to open UDP when required
- Confusing cloud firewalls with UFW

> [!NOTE]
> Cloud providers often have **their own firewalls** — both must allow the port.

---

## Safe defaults (cheat sheet)

| Service | Port |
|------|------|
| SSH | 22/tcp |
| HTTP | 80/tcp |
| HTTPS | 443/tcp |
| PostgreSQL | 5432/tcp |
| MySQL | 3306/tcp |
| WireGuard | 51820/udp |

---

## When *not* to open a port
Avoid opening:
- Database ports to the public internet
- Admin interfaces without auth
- Anything you don’t recognize

---

## Summary
- Always allow SSH first
- Use UFW to manage ports safely
- Open only what you need
- Verify rules and test connectivity

---

> [!TIP]
> For servers, **less open ports = more security**.
