<!-- level: Beginner -->
<!-- tags: system, performance, memory -->

# Add a Swap File on Linux (Safe Defaults)

Create a secure swap file with sane defaults, enable it **now** and on **boot**, and tune swappiness. Safe to copy/paste.

## Prerequisites
- A sudo-capable user
- Free disk space (e.g., 2–4 GiB)
- A Linux distro where you control `/etc/fstab`

---

## 1) Check current swap
```bash
swapon --show
free -h
