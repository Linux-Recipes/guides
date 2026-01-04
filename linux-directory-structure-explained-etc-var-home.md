# Linux Directory Structure Explained (/etc, /var, /home)
<!-- difficulty: Beginner -->
<!-- tags: filesystem, directories, linux-basics, etc, var, home -->
<!-- summary: A beginner-friendly explanation of the Linux directory structure. Learn what /etc, /var, /home and other top-level directories are for, and which ones you should (and should not) touch. -->

If you’re new to Linux, the directory layout can feel strange at first.  
There are **no C: drives**, and everything lives under a single root: `/`.

This guide explains the **most important Linux directories**, what they’re used for, and what’s safe to explore or edit.

Beginner-friendly and read-only safe.

---

## The root directory (`/`)
`/` is the top of the filesystem tree.  
Every file, folder, disk, and device lives **somewhere under `/`**.

```text
/
├── bin
├── boot
├── dev
├── etc
├── home
├── lib
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── tmp
├── usr
└── var
```

---

## /home — user files (most important for beginners)
`/home` contains personal directories for each user.

Example:
```text
/home/alice
/home/bob
```

What you’ll find here:
- Documents, downloads, projects
- App settings (hidden “dotfiles” like `.bashrc`)
- SSH keys (`~/.ssh/`)

Safe to:
- Create, edit, delete files
- Experiment and learn

> [!TIP]
> `~` is shorthand for your home directory.

---

## /etc — system configuration
`/etc` contains **configuration files**, not programs.

Examples:
- `/etc/ssh/sshd_config`
- `/etc/nginx/nginx.conf`
- `/etc/passwd`

Rules:
- Mostly **text files**
- Changes affect the whole system
- Usually requires `sudo`

> [!WARNING]
> Editing files in `/etc` incorrectly can break services or prevent login.

---

## /var — changing (variable) data
`/var` stores data that **changes while the system is running**.

Common subdirectories:
- `/var/log` — system & application logs
- `/var/lib` — application state data
- `/var/www` — web server files (common)

If your disk fills up, **/var is often the reason**.

```bash
sudo du -sh /var/*
```

---

## /usr — installed software
`/usr` contains most installed programs and libraries.

Think of it as:
- `/usr/bin` — user commands
- `/usr/lib` — libraries
- `/usr/share` — shared data (docs, icons)

> [!NOTE]
> Despite the name, `/usr` is **not user data**.

---

## /bin and /sbin — essential commands
- `/bin` — basic commands (`ls`, `cp`, `mv`)
- `/sbin` — system/admin commands (`fsck`, `ip`, `reboot`)

On modern systems, these may link into `/usr/bin`.

---

## /boot — bootloader files
Contains files needed to start Linux:
- Kernel images
- Bootloader config

Do **not** edit unless you know what you’re doing.

---

## /dev — devices
`/dev` represents hardware as files:
- Disks (`/dev/sda`, `/dev/nvme0n1`)
- USB devices
- Terminals

These files are **created dynamically** by the kernel.

---

## /tmp — temporary files
- Used by programs for short-lived data
- Often cleared on reboot

Safe to:
- Ignore
- Use for testing

---

## /proc and /sys — kernel interfaces
These aren’t real files on disk.

- `/proc` — process and system info
- `/sys` — kernel and device settings

Used mainly by:
- System tools
- Advanced debugging

---

## /root — root user’s home
This is the home directory for the `root` user.

Not the same as `/`.

Regular users normally don’t access this directory.

---

## /media and /mnt — mounted drives
- `/media` — auto-mounted USB drives
- `/mnt` — temporary/manual mounts

Example:
```bash
ls /media
```

---

## What beginners should and should not touch

### Safe places
- `/home`
- `/tmp`
- `/media`

### Be careful
- `/etc`
- `/var`
- `/usr`

### Avoid
- `/boot`
- `/proc`
- `/sys`
- `/dev`

---

## How to explore safely
Use **read-only commands**:

```bash
ls
ls -l
tree -L 2 /
```

---

## Summary
- Linux uses **one unified directory tree**
- `/home` is where your files live
- `/etc` controls configuration
- `/var` grows over time
- Many directories are system-critical

---

> [!TIP]
> When in doubt, **don’t edit** — explore with `ls` and `cat` first.
