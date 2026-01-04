# Create a Non-Root User on Linux (Safe Beginner Setup)
<!-- difficulty: Beginner -->
<!-- tags: users, sudo, security, ssh, linux-basics -->
<!-- summary: Learn how to safely create a non-root user on Linux, grant sudo access, and use it for daily work without locking yourself out. Beginner-friendly and copy/paste safe. -->

Using the **root** account for daily work is dangerous and unnecessary.  
This guide shows how to **create a non-root user**, give it **sudo access**, and switch safely — without breaking SSH access.

This is one of the **first and most important security steps** on any Linux system.

---

## Why you should not use root
The `root` user:
- Can delete **anything**
- Can break the system instantly
- Is a common attack target

Best practice:
👉 Log in as root **once**, then create a regular user.

---

## Step 1) Check who you are logged in as
```bash
whoami
```

If it returns `root`, continue.  
If not, make sure you have sudo access.

---

## Step 2) Create a new user
Replace `alice` with your desired username.

```bash
sudo adduser alice
```

You’ll be prompted to:
- Set a password
- Enter optional info (can press Enter to skip)

---

## Step 3) Grant sudo access
Add the user to the `sudo` group:
```bash
sudo usermod -aG sudo alice
```

Verify:
```bash
groups alice
```

You should see:
```text
alice : alice sudo
```

---

## Step 4) Switch to the new user
```bash
su - alice
```

Confirm:
```bash
whoami
```

---

## Step 5) Test sudo access (important)
```bash
sudo ls /root
```

If prompted for **alice’s password** and it works — you’re good.

> [!WARNING]
> Always test sudo before logging out of root.

---

## Step 6) Set up SSH access for the new user
On your **local machine**:
```bash
ssh-copy-id alice@server-ip
```

Test login:
```bash
ssh alice@server-ip
```

Do not proceed until this works.

---

## Step 7) (Optional) Disable root SSH login
Edit SSH config:
```bash
sudo nano /etc/ssh/sshd_config
```

Set:
```text
PermitRootLogin no
```

Restart SSH:
```bash
sudo systemctl restart ssh
```

> [!WARNING]
> Keep your current SSH session open while testing a new one.

---

## Step 8) Use the new user for daily work
From now on:
- Log in as `alice`
- Use `sudo` when needed
- Avoid logging in as root

---

## Common beginner mistakes
- Forgetting to add user to sudo group
- Logging out before testing sudo
- Disabling root SSH too early
- Using `chmod 777` instead of fixing ownership

---

## Quick checklist
- [ ] User created
- [ ] Sudo access works
- [ ] SSH login works
- [ ] Root login disabled (optional)

---

## Summary
- Root is for emergencies only
- Create a non-root user immediately
- Grant sudo carefully
- Test everything before logging out

---

> [!TIP]
> One rule to remember: **log in as user, escalate with sudo** — never the other way around.
