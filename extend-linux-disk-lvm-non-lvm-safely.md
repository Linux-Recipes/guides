# Extend a Linux Disk (LVM + Non-LVM Safely)
<!-- difficulty: Advanced -->
<!-- tags: disk, lvm, storage, resize, filesystem -->
<!-- summary: Safely extend a Linux disk on live systems. Covers LVM and non-LVM setups, cloud disks, partitions, and filesystem resizing without data loss. -->

Running out of disk space is stressful — **resizing disks doesn’t have to be**.  
This guide shows how to **safely extend a Linux disk**, whether you’re using **LVM** or **plain partitions**, with minimal risk and no downtime in most cases.

> Copy/paste friendly. Read the **Identify your setup** section first.

---

## Before you start (important)
- **Backups matter** — resizing is safe, but mistakes are permanent.
- Most operations below **do not require a reboot**.
- Commands assume **root or sudo** access.

---

## 1) Identify your disk setup (LVM or not)
First, see what you’re working with.

```bash
lsblk
```

### If you see:
- `vg-*`, `lv-*`, or `/dev/mapper/*` → **LVM**
- `/dev/sda1`, `/dev/vda1`, `/dev/nvme0n1p1` mounted directly → **Non-LVM**

Check filesystem type:
```bash
df -T
```

---

## PART A — LVM (Recommended & Easiest)

### Common LVM layout
```
Disk → Partition → Physical Volume → Volume Group → Logical Volume → Filesystem
```

---

## 2A) Extend the underlying disk (cloud or VM)
If you’re on a cloud/VPS:
- Increase disk size in the provider dashboard first
- Then rescan the disk

```bash
lsblk
sudo partprobe
```

If the disk still shows the old size:
```bash
echo 1 | sudo tee /sys/class/block/sda/device/rescan
```

(Change `sda` if needed.)

---

## 3A) Resize the LVM physical volume
```bash
sudo pvresize /dev/sda3
```

(Replace `/dev/sda3` with your actual LVM partition.)

Verify:
```bash
sudo pvs
```

---

## 4A) Extend the logical volume
To extend **all remaining free space**:
```bash
sudo lvextend -l +100%FREE /dev/vg0/root
```

Or extend by a fixed size (example: +20G):
```bash
sudo lvextend -L +20G /dev/vg0/root
```

---

## 5A) Resize the filesystem
### ext4
```bash
sudo resize2fs /dev/vg0/root
```

### xfs (must be mounted)
```bash
sudo xfs_growfs /
```

Verify:
```bash
df -h
```

✅ **Done — no reboot required**

---

## PART B — Non-LVM (Plain Partition)

This is slightly riskier but still safe if done carefully.

---

## 2B) Confirm free space exists on disk
```bash
lsblk
```

You must see unused space **after** the partition you’re resizing.

---

## 3B) Resize the partition (GPT or MBR)

### Interactive (recommended)
```bash
sudo growpart /dev/sda 1
```

- `/dev/sda` = disk
- `1` = partition number (`/dev/sda1`)

If `growpart` isn’t installed:
```bash
sudo apt install -y cloud-guest-utils
```

---

## 4B) Resize the filesystem

### ext4
```bash
sudo resize2fs /dev/sda1
```

### xfs
```bash
sudo xfs_growfs /
```

Verify:
```bash
df -h
```

---

## Common Cloud Examples

### AWS / GCP / Azure
```bash
lsblk
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
```

### DigitalOcean
```bash
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
```

---

## Troubleshooting

### “No space left on device” but disk looks bigger
- Filesystem not resized yet
- Run `resize2fs` or `xfs_growfs`

---

### `xfs_growfs: not a mounted XFS filesystem`
- Ensure the mount point is correct:
```bash
df -T
```

---

### LVM shows free space but filesystem didn’t grow
- Filesystem resize step missed
- LVM does *not* auto-resize filesystems

---

## Safety checklist (printable)
- [ ] Verified LVM vs non-LVM
- [ ] Confirmed free space exists
- [ ] Resized disk / partition
- [ ] Extended LV (if LVM)
- [ ] Resized filesystem
- [ ] Verified with `df -h`

---

## When *not* to resize live
Avoid live resizing if:
- You’re shrinking a filesystem
- The disk is heavily degraded
- You don’t have backups

---

## Summary
- **LVM**: safest, most flexible, zero-downtime
- **Non-LVM**: still safe if extending forward
- Filesystem resize is **always required**
- Backups turn stress into routine maintenance

---

> [!TIP]
> If you manage servers often, consider migrating to **LVM** during your next rebuild — future resizes become trivial.
