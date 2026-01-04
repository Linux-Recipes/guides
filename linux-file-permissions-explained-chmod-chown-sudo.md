# Linux File Permissions Explained (chmod, chown, sudo)
<!-- difficulty: Beginner -->
<!-- tags: permissions, chmod, chown, sudo, security -->
<!-- summary: A beginner-friendly guide to Linux file permissions. Learn how read, write, and execute work, and how to use chmod, chown, and sudo safely. -->

Linux file permissions look confusing at first — but they follow a **simple, predictable model**.  
This guide explains **who can read, write, or run a file**, and shows how to safely use `chmod`, `chown`, and `sudo` without breaking your system.

Copy/paste friendly and beginner-safe.

---

## The permission model (the big idea)
Every file and directory has **three owners** and **three permissions**.

### Owners
- **user** — the file’s owner
- **group** — users in the file’s group
- **others** — everyone else

### Permissions
- **read (r)** — view file contents / list directory
- **write (w)** — modify file / create & delete files in a directory
- **execute (x)** — run a file / enter a directory

---

## Viewing permissions
```bash
ls -l
```

Example:
```text
-rwxr-xr-- 1 alice developers 4096 app.sh
```

Breakdown:
```
- rwx r-x r--
| |   |   |
| |   |   └─ others (read)
| |   └──── group (read + execute)
| └──────── user (read + write + execute)
└────────── file type (- = file, d = directory)
```

---

## Files vs directories (important difference)

| Permission | File | Directory |
|-----------|------|-----------|
| read (r) | View contents | List filenames |
| write (w) | Modify contents | Create/delete files |
| execute (x) | Run file | Enter directory |

> [!TIP]
> A directory without `x` cannot be entered, even if it’s readable.

---

## Using chmod (change permissions)

### Symbolic mode (recommended for beginners)
```bash
chmod u+x script.sh
chmod g-w config.txt
chmod o+r README.md
```

Meaning:
- `u` = user
- `g` = group
- `o` = others
- `+` add, `-` remove

---

### Numeric mode (quick but dangerous)
```bash
chmod 644 file.txt
chmod 755 script.sh
```

Number meaning:
- `4` = read
- `2` = write
- `1` = execute

| Mode | Meaning |
|-----|--------|
| 644 | rw-r--r-- |
| 755 | rwxr-xr-x |
| 600 | rw------- |

> [!WARNING]
> Avoid `chmod 777`. It removes protection and is almost never needed.

---

## Making a script executable
```bash
chmod +x deploy.sh
./deploy.sh
```

If you see:
```text
Permission denied
```
Check:
```bash
ls -l deploy.sh
```

---

## Using chown (change ownership)

### Change owner
```bash
sudo chown alice file.txt
```

### Change owner and group
```bash
sudo chown alice:developers project/
```

### Recursive (use carefully)
```bash
sudo chown -R alice:developers project/
```

> [!WARNING]
> Recursive `chown` on system directories can break your OS.

---

## What sudo actually does
`sudo` runs **one command** as another user (usually root).

```bash
sudo apt update
sudo systemctl restart nginx
```

It does **not**:
- Permanently switch users
- Change file ownership automatically

---

## Why sudo is safer than logging in as root
- Commands are **logged**
- Reduced chance of accidental damage
- Time-limited access

Check sudo access:
```bash
sudo -l
```

---

## Common permission errors (and fixes)

### “Permission denied”
- Missing execute bit on directory
- Wrong owner or group
```bash
ls -ld .
```

---

### “Operation not permitted”
- Requires root
```bash
sudo <command>
```

---

### Can’t write to a file
- Check ownership
```bash
ls -l file.txt
sudo chown $USER file.txt
```

---

## Safe defaults (cheat sheet)

| Use case | Permissions |
|--------|-------------|
| Config file | `600` |
| Script | `755` |
| Private dir | `700` |
| Web files | `644` (files), `755` (dirs) |

---

## When *not* to change permissions
Avoid changing permissions on:
- `/bin`, `/sbin`, `/lib`, `/usr`
- System-owned config files
- Anything you don’t fully understand

---

## Summary
- Permissions control **who can do what**
- `chmod` changes access
- `chown` changes ownership
- `sudo` grants temporary admin power
- Small changes can have big effects

---

> [!TIP]
> If permissions feel confusing, always start with `ls -l` — it explains almost everything.
