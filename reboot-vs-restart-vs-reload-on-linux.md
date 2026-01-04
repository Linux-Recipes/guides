# Reboot vs Restart vs Reload on Linux (What’s the Difference?)
<!-- difficulty: Beginner -->
<!-- tags: reboot, restart, reload, systemctl, linux-basics -->
<!-- summary: Understand the difference between reboot, restart, and reload on Linux. Learn when to use each safely, with real examples using systemctl and common services. -->

Linux offers several ways to apply changes — but choosing the **wrong one** can cause unnecessary downtime.  
This guide explains the difference between **reboot**, **restart**, and **reload**, and shows **when to use each safely**.

Beginner-friendly, production-safe.

---

## The short answer
- **Reboot** → restart the entire system
- **Restart** → stop and start a service
- **Reload** → apply config changes without stopping the service

---

## Reboot (restart the whole system)
A reboot shuts down **everything**, then starts Linux again from scratch.

### When to reboot
- Kernel updates
- Low-level system changes
- System instability
- You’re not sure what state the system is in

### How to reboot
```bash
sudo reboot
```

Or schedule one:
```bash
sudo shutdown -r +5
```

> [!WARNING]
> A reboot disconnects all users and stops all services.

---

## Restart (stop and start a service)
Restarting affects **one service**, not the whole system.

### When to restart
- After changing service configs
- When a service is stuck or misbehaving
- When reload is unsupported

### Example
```bash
sudo systemctl restart nginx
```

What happens:
1. Service stops
2. Service starts again
3. Active connections may be dropped

---

## Reload (apply changes without stopping)
Reload tells a service to **re-read its config** without stopping.

### When to reload
- Configuration changes only
- Zero-downtime desired
- Service explicitly supports reload

### Example
```bash
sudo systemctl reload nginx
```

Or:
```bash
sudo systemctl reload-or-restart nginx
```

> [!TIP]
> `reload-or-restart` reloads if possible, otherwise restarts — safest option.

---

## Not all services support reload
Check with:
```bash
systemctl show nginx -p CanReload
```

If `yes`, reload is supported.

---

## Common examples

### Web servers
- **Nginx** → reload preferred
- **Apache** → reload preferred

```bash
sudo systemctl reload nginx
```

---

### SSH
```bash
sudo systemctl restart ssh
```

Reload is not always supported; restart is safe.

---

### Databases
- Often require restart
- Reload may only apply partial settings

Always check documentation.

---

## Why reload is safer
- No dropped connections
- Faster
- Less disruption

But:
- Not all changes apply
- Not all services support it

---

## How to choose (decision table)

| Situation | Action |
|--------|--------|
| Kernel updated | Reboot |
| Changed app config | Reload (if supported) |
| Service crashed | Restart |
| Unsure | Restart |
| System unstable | Reboot |

---

## Checking service status
```bash
systemctl status nginx
```

View logs:
```bash
journalctl -u nginx -n 50
```

---

## Common beginner mistakes
- Rebooting for simple config changes
- Restarting services that support reload
- Reloading services that don’t support it
- Restarting SSH without a backup session

> [!WARNING]
> Always keep one SSH session open when restarting SSH.

---

## Summary
- **Reboot** affects the whole system
- **Restart** affects one service
- **Reload** applies config changes with minimal impact
- Prefer reload when supported
- Restart when in doubt

---

> [!TIP]
> In production, **reload first**, **restart second**, **reboot last**.
