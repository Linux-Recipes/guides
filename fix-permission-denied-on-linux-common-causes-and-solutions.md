# Fix “Permission Denied” on Linux (Common Causes & Solutions)
<!-- difficulty: Beginner -->
<!-- tags: permissions, chmod, chown, sudo, troubleshooting -->
<!-- summary: Learn why Linux shows “Permission denied” and how to fix it safely. Covers file permissions, ownership, execute bits, directories, sudo, and common real-world scenarios. -->

“Permission denied” is one of the **most common Linux errors** — and one of the most confusing for beginners.

This guide explains **why it happens**, how to **identify the cause**, and how to **fix it safely** without breaking system security.

---

## What “Permission denied” means
Linux is telling you:
> You are not allowed to do this **as your current user**.

Permissions are enforced using:
- **Ownership** (user & group)
- **Permission bits** (read, write, execute)
- **Privilege level** (sudo vs normal user)

---

## Step 1) Check the exact error
Example:
```bash
bash: ./script.sh: Permission denied
```

Or:
```bash
rm: cannot remove 'file.txt': Permission denied
```

Different messages hint at different causes.

---

## Step 2) Check file permissions
Run:
```bash
ls -l filename
```

Example output:
```text
-rw-r--r-- 1 root root 1200 file.txt
```

Breakdown:
- `-rw-r--r--` → permissions
- `root root` → owner and group

If you are **not the owner**, you may not have access.

---

## Step 3) Fix execute permission (very common)
If you try to run a script:
```bash
./script.sh
```

And get permission denied, check:
```bash
ls -l script.sh
```

If no `x` (execute) permission exists:
```bash
chmod +x script.sh
```

Try again:
```bash
./script.sh
```

---

## Step 4) Fix ownership (common with copied files)
If files belong to root or another user:
```bash
sudo chown youruser:youruser filename
```

For directories:
```bash
sudo chown -R youruser:youruser directory/
```

> [!WARNING]
> Be careful with `-R` (recursive). Double-check paths.

---

## Step 5) Fix directory permissions
Directories require **execute permission** to access.

Example:
```bash
ls directory/
```

Error:
```text
ls: cannot open directory 'directory': Permission denied
```

Fix:
```bash
chmod +x directory
```

---

## Step 6) Use sudo (when appropriate)
Some actions require admin rights:
```bash
sudo apt update
sudo nano /etc/hosts
```

If it works with sudo, permissions are correct — you just lacked privileges.

> [!NOTE]
> Don’t use sudo for daily file operations in your home directory.

---

## Step 7) Check parent directories
Even if a file looks fine, parent directories may block access:
```bash
namei -l /path/to/file
```

This shows permissions for every directory in the path.

---

## Step 8) SELinux/AppArmor (advanced cases)
On some systems, security modules may block access even with correct permissions.

Check AppArmor:
```bash
sudo dmesg | grep DENIED
```

This is less common on Ubuntu/Debian but common on servers.

---

## Common permission-denied scenarios

### Running a script
Fix:
```bash
chmod +x script.sh
```

---

### Editing system files
Fix:
```bash
sudo nano /etc/nginx/nginx.conf
```

---

### Accessing web files
Fix ownership:
```bash
sudo chown -R www-data:www-data /var/www/html
```

---

### Copying files from root
Fix:
```bash
sudo chown youruser:youruser file
```

---

## What NOT to do
❌ `chmod 777` everything  
❌ Run everything with sudo  
❌ Change ownership of `/etc`, `/usr`, `/bin`

These weaken system security.

---

## Quick troubleshooting checklist
- [ ] Am I the file owner?
- [ ] Does the file have execute permission?
- [ ] Do parent directories allow access?
- [ ] Do I need sudo?
- [ ] Is a security module blocking access?

---

## Summary
- “Permission denied” is normal and fixable
- Check permissions first
- Fix ownership or execute bits carefully
- Use sudo only when required

---

> [!TIP]
> If unsure: **check permissions → check owner → then sudo**.
