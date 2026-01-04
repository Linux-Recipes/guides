# Test if a Port Is Open on Linux
<!-- difficulty: Beginner -->
<!-- tags: networking, ports, firewall, ufw, troubleshooting -->
<!-- summary: Learn simple, reliable ways to test if a port is open on Linux. Covers local checks, remote tests, TCP vs UDP, and common mistakes. Beginner-friendly and copy/paste safe. -->

When something won’t connect, the first question is usually:  
**“Is the port actually open?”**

This guide shows **easy, reliable ways to test if a port is open on Linux**, both **locally** and **from another machine**, without advanced networking knowledge.

Beginner-friendly and safe.

---

## What “open” really means
A port is usable only if **all three** are true:
1. A service is **listening** on the port
2. The **firewall allows** the port
3. (For remote access) The **network path** allows it

We’ll test each layer step by step.

---

## Step 1) Check if a service is listening (local)
On the server itself:

```bash
ss -tulpn
```

Look for your port:
```text
LISTEN 0 128 0.0.0.0:3000
```

If you see it → a service is listening.

Filter by port:
```bash
ss -tulpn | grep :3000
```

---

## Step 2) Test locally with curl (TCP services)
Good for HTTP-based services.

```bash
curl http://localhost:3000
```

If you get a response (even an error page), the port is open locally.

---

## Step 3) Test locally with nc (netcat)
Works for most TCP services.

```bash
nc -vz localhost 3000
```

Success:
```text
Connection to localhost 3000 port [tcp/*] succeeded!
```

---

## Step 4) Check the firewall (UFW)
```bash
sudo ufw status
```

Look for your port:
```text
3000/tcp   ALLOW   Anywhere
```

If missing:
```bash
sudo ufw allow 3000/tcp
```

---

## Step 5) Test from another machine (most important)
From your **local computer** or another server:

```bash
nc -vz server-ip 3000
```

Or with curl:
```bash
curl http://server-ip:3000
```

If this fails but local tests work:
- Firewall or network is blocking it

---

## Step 6) Test UDP ports (different behavior)
UDP does not confirm connections the same way.

```bash
nc -vzu server-ip 51820
```

Success may look silent — this is normal.

For UDP services (VPNs, DNS), also check logs:
```bash
journalctl -u your-service-name
```

---

## Step 7) Common port numbers (quick reference)

| Service | Port |
|------|------|
| SSH | 22 |
| HTTP | 80 |
| HTTPS | 443 |
| PostgreSQL | 5432 |
| MySQL | 3306 |
| Redis | 6379 |
| WireGuard | 51820 (UDP) |

---

## Common problems (and fixes)

### “Connection refused”
- No service listening
- Service crashed

Check:
```bash
ss -tulpn
systemctl status your-service
```

---

### “Connection timed out”
- Firewall blocking
- Cloud firewall blocking
- Wrong IP

Check:
```bash
sudo ufw status
```

---

### Works locally but not remotely
- Cloud provider firewall not open
- NAT or router blocking traffic

> [!NOTE]
> Many VPS providers have **two firewalls**: UFW *and* a cloud firewall.

---

## Quick troubleshooting checklist
- [ ] Service running
- [ ] Port listening
- [ ] Firewall allows port
- [ ] Correct IP used
- [ ] Tested from another machine

---

## Tools used (beginner-friendly)
- `ss` — check listening ports
- `curl` — test HTTP services
- `nc` (netcat) — generic port testing
- `ufw` — firewall status

---

## Summary
- Always test locally first
- Then test remotely
- TCP and UDP behave differently
- Firewalls are the most common blocker

---

> [!TIP]
> If in doubt: **listen → firewall → remote test** — in that order.
