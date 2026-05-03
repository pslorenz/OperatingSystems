# Chapter 7: Disk and Filesystem

**You come in with:** ability to navigate the filesystem. You can `cd` and `ls` and read the basic shape of `/etc`, `/var`, and `/home`.
**You leave with:** the ability to answer "where did the disk space go," read `/etc/fstab` without panic, and know what to do when an fstab typo prevents the box from booting.

**Time:** 50 to 75 minutes including the exercises.

**Security+ alignment:** Domain 3.3 (resilience and recovery: backups, recovery sites, restoration). Domain 4.1 (hardening: disk encryption foundations). Domain 4.4 (monitoring: capacity monitoring as part of alerting). The fstab recovery walkthrough is the operational version of "you need a recovery plan," which Domain 3.3 tests at the conceptual level.

---

## Why this chapter matters

Disk problems are the most reliably recurring source of production outages. The disk fills up, a service that needed to write fails, the failure cascades. The boot disk has a typo in fstab, the box does not boot, the on-call is paged at 3 AM. A backup script writes to a path that turns out to be a different filesystem from what it expected, the backup quietly fails for six months until you need it.

These are not exotic scenarios. They happen routinely. The skill is not memorizing every tool; the skill is having a fast diagnosis pattern when something disk-shaped is wrong.

The chapter is shorter than chapter 6 because the topic is narrower. It is also one of the chapters where the reading-to-typing ratio shifts toward typing: most concepts only stick after you watch a `df` output or see what a typo in fstab does on a test box.

---

## Disks, partitions, and filesystems

Three layers, and confusion among them is the source of most beginner stumbles.

**A disk is a physical device.** Or, in a virtualized environment, a virtual block device that behaves like one. On Linux the disk shows up as `/dev/sda`, `/dev/sdb`, `/dev/nvme0n1`, or similar. The naming convention is historical and not always intuitive, but the device file is always under `/dev`.

**A partition is a slice of a disk.** A disk can have one partition or several. Each partition is independently formatted and mounted. Partitions show up as `/dev/sda1`, `/dev/sda2`, `/dev/nvme0n1p1`. The number is the partition; everything before the number is the disk.

**A filesystem is the format on top of a partition.** ext4, xfs, btrfs, fat32, ntfs are filesystems. *A filesystem is the on-disk layout that turns a raw partition into something with files and directories.* The same partition can be reformatted with a different filesystem (which destroys the contents).

The three layers are independent and you can have any combination. A 1 TB disk might have one partition with the entire space, formatted as ext4, mounted at `/`. Or it might have three partitions: a small EFI partition, a swap partition, and a large root partition. The combinations are the configuration choices an installer makes.

### Seeing the layout

```
lsblk
```

Output on a typical Ubuntu cloud VM:

```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0   40G  0 disk 
├─sda1    8:1    0 39.9G  0 part /
├─sda14   8:14   0    4M  0 part 
└─sda15   8:15   0  106M  0 part /boot/efi
```

Reading this:

- `sda` is the whole disk, 40 GB.
- It has three partitions.
- `sda1` is mounted at `/` and uses 39.9 GB.
- `sda14` and `sda15` are small partitions related to booting (the EFI partition, with the bootloader).

`lsblk` is the first command for "what does this box's storage look like." Run it on every new box. The five-second mental model it gives you is worth more than reading documentation.

### The filesystem in numbers

```
df -h
```

Output:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G   12G   28G  30% /
tmpfs           395M     0  395M   0% /dev/shm
tmpfs           158M  1.0M  157M   1% /run
/dev/sda15      105M  6.4M   99M   7% /boot/efi
```

`df` shows usage per mounted filesystem. `-h` is "human-readable" sizes. `-i` shows inodes instead of bytes (covered below).

Reading the output:

- The root filesystem is 30% full. Plenty of headroom.
- `tmpfs` entries are RAM-backed temporary filesystems, not real disk.
- `/boot/efi` is the EFI system partition, tiny by design.

When the disk is "full," `df -h` is the first command. The Use% column tells you which filesystem is the problem.

### Inodes: the other way to fill a disk

Every file on a Linux filesystem consumes one inode. *An inode is a data structure that holds the metadata for one file: permissions, ownership, timestamps, and pointers to where the file's data actually lives.* The number of inodes is fixed when the filesystem is created.

A filesystem can run out of inodes before it runs out of bytes. This happens when there are millions of small files: cache directories, mail spools, certain build artifacts. The symptom is "no space left on device" even though `df -h` shows plenty of free bytes.

```
df -i
```

Output:

```
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda1      2621440 134561 2486879    6% /
```

If `IUse%` is high (above 80%) while `Use%` is low, you have an inode problem. The fix is finding the directory with millions of small files and either cleaning it up or moving the workload to a filesystem with more inodes.

This is the kind of problem that catches admins who are good at general disk management but have never seen it before. Adding "check inodes when df says space is fine" to your diagnosis pattern saves an hour of confusion.

---

## Where did my disk go: the diagnosis pattern

The single most common operational question. The pattern is the same every time.

**Step 1.** Confirm with `df -h` that you actually have a disk problem and identify which filesystem is full.

```
df -h
```

If `/` is at 95%, the problem is on the root filesystem. If `/var/log` is on a separate partition and is at 95%, the problem is logs. The filesystem identifies the directory you investigate first.

**Step 2.** Use `du` to find the heaviest directories under the problem filesystem.

```
sudo du -sh /var/* 2>/dev/null | sort -h
```

Reading the command:

- `sudo` because reading some directories requires root.
- `du -sh` summarizes (one number per argument) in human-readable form.
- `/var/*` is every direct child of `/var`.
- `2>/dev/null` discards the permission errors for unreadable files.
- `sort -h` sorts numerically including the K/M/G/T suffixes.

The output is one line per directory under `/var`, sorted small to large. The biggest directory is at the bottom. Drill into it:

```
sudo du -sh /var/log/* 2>/dev/null | sort -h
```

Repeat one more level. Usually within three or four `du` commands you find the directory that is 30 GB instead of the 200 MB you expected.

**Step 3.** Investigate the directory.

```
sudo ls -lah /var/log/big-directory/ | tail -20
```

What is in there? When was it last modified? Often the answer is "logs that should have rotated but did not," or "a rogue cache that needs cleanup," or "a database that has grown beyond its expected size."

**Step 4.** Free the space, with care.

The crucial rule: **do not delete anything you cannot identify.** A 12 GB file in `/var/lib/postgresql` is the database; deleting it destroys data. A 12 GB file in `/var/cache` is almost always safe to delete; the worst case is the application reconstructs the cache.

Common safe targets:
- Old log files: `sudo journalctl --vacuum-size=500M` trims the journal to 500 MB.
- apt cache: `sudo apt clean` removes downloaded package files.
- old kernels: `sudo apt autoremove` removes kernel packages no longer needed.

Common unsafe targets without thought:
- Anything in `/var/lib/<application>` is the application's data.
- Anything in `/home`.
- `.deleted` files in `/proc` (which we cover in Chapter 10).

A junior admin who deletes a 12 GB postgres data file at 3 AM to "free space" is the start of a much longer story. Discipline first, freeing second.

### A useful one-liner for the heaviest directories

```
sudo du -h --max-depth=2 / 2>/dev/null | sort -h | tail -20
```

That walks two levels deep from `/`, reports each directory with a human-readable size, sorts, and shows the 20 heaviest. Run on a healthy box, the output is unremarkable. Run on a sick box, the answer is in the last few lines.

### The `/proc` filesystem trap

Sometimes a deleted file is still keeping disk space because a process has it open. `df` reports the space as used; `du` reports it as missing. The discrepancy is the smoking gun.

```
sudo lsof | grep deleted
```

That lists every open file that is marked as deleted. The space is not reclaimed until the process closes the file or exits. The fix is to restart the process holding the file. Common offenders: log writers that opened a log file, then someone deleted the log file with `rm` instead of truncating it, and the process is still writing to a now-anonymous inode.

The proper way to clear a log file that is being written to:

```
sudo truncate -s 0 /var/log/somefile.log
```

That zeroes the file in place, which the writing process handles correctly. `rm` followed by recreating the file does not work the same way because the process has a handle to the original inode, not the path.

---

## /etc/fstab: how mounts are configured

`/etc/fstab` is the file that tells the system what to mount where at boot.

```
sudo cat /etc/fstab
```

A typical entry:

```
UUID=abc123...  /  ext4  defaults  0  1
```

The six fields, in order:

| Field | Meaning |
|---|---|
| Source | What to mount. UUID is preferred over device name. |
| Target | Where to mount it. The mount point. |
| Filesystem type | `ext4`, `xfs`, `swap`, `nfs`, etc. |
| Options | Mount options, comma-separated. `defaults` is a sensible baseline. |
| Dump | Used by the legacy `dump` backup tool. Almost always `0`. |
| Pass | Order of fsck at boot. `1` for root, `2` for other, `0` for skip. |

### Why UUID instead of device name

`/dev/sda1` can change between boots. If you add a second disk, the kernel's enumeration order may shift, and what used to be `/dev/sda` is now `/dev/sdb`. fstab entries that referenced `/dev/sda1` would now mount the wrong disk.

UUIDs are unique per filesystem and stable. Use `blkid` to find them:

```
sudo blkid
```

Output:

```
/dev/sda1: UUID="abc123-..." TYPE="ext4" PARTUUID="..."
/dev/sda15: UUID="ABCD-1234" TYPE="vfat" PARTUUID="..."
```

The UUID for ext4 is a long hex string. For FAT/EFI it is shorter. Both work as identifiers in fstab.

### Mount options that matter

The `defaults` option is shorthand for `rw,suid,dev,exec,auto,nouser,async`. Most filesystems should use it. For specific cases:

- `nosuid` prevents setuid bits from taking effect on this filesystem. Use on filesystems where users can write files (like `/tmp` if it is on its own partition).
- `nodev` prevents device files from being interpreted. Use on filesystems where they have no business existing.
- `noexec` prevents binaries on this filesystem from being executed. Use on filesystems for data, not programs.
- `ro` mounts read-only. Use for archived data or recovery.
- `noatime` skips updating access times on every read, improving performance. The default on most modern installs.

The hardening pattern: `/tmp` and `/var/tmp` should ideally be on their own partition with `nosuid,nodev,noexec`. This is a CIS benchmark control. We come back to it in Chapter 12.

### Adding a new filesystem to fstab

The pattern: format the partition, get its UUID, decide where to mount it, decide on options, add the line.

```
# format (destroys existing data)
sudo mkfs.ext4 /dev/sdb1

# get UUID
sudo blkid /dev/sdb1

# create mount point
sudo mkdir -p /mnt/data

# add to fstab (use UUID from blkid output)
echo 'UUID=newuuid /mnt/data ext4 defaults 0 2' | sudo tee -a /etc/fstab

# mount everything in fstab now (does not require reboot)
sudo mount -a
```

The `mount -a` step is important. It validates that fstab is correct without rebooting. If it succeeds, the new filesystem is mounted and the next reboot will mount it the same way. If it fails, you fix fstab before rebooting.

---

## When fstab eats your boot

The reason `mount -a` matters: a typo in fstab can prevent the system from booting. The system tries to mount everything in fstab during startup. If a critical mount fails, boot stops.

Modern systemd handles this somewhat better than older init did, often dropping you into a recovery shell rather than a hard hang. But the recovery shell is a different environment with limited tooling, and you have to know what to do.

### The defensive habits

Every time you edit fstab:

1. Make a backup: `sudo cp /etc/fstab /etc/fstab.bak`.
2. Run `sudo mount -a` to test the new fstab without rebooting. If it succeeds, the entries are valid.
3. Run `sudo findmnt --verify` if you want a more thorough syntax check.

If `mount -a` fails, restore the backup and start over. Never reboot until `mount -a` succeeds.

### The recovery walkthrough

The scenario: you ignored the defensive habits, fstab has a typo, the box does not boot. Now what.

**On a cloud VM**, attach the disk to a running rescue instance, mount it, fix fstab, detach. The exact procedure depends on the cloud provider; AWS, GCP, and Azure all support this.

**On a physical box or self-hosted VM**, boot from an Ubuntu live ISO. Once the live system is up, mount your root partition manually:

```
sudo mkdir /mnt/recovery
sudo mount /dev/sda1 /mnt/recovery
sudo nano /mnt/recovery/etc/fstab     # fix the typo
sudo umount /mnt/recovery
sudo reboot
```

For more complex recovery (a chroot for things like reinstalling a bootloader), the pattern is to bind-mount the host's /dev, /proc, /sys into the recovery mount point and then `chroot` into it. That is past the scope of this chapter; the takeaway is "the live ISO can fix anything in fstab as long as you can boot the live ISO."

### The systemd boot failure pattern

When systemd encounters a fstab failure, it prints messages like "Failed to mount /var" and eventually drops you to either a recovery shell or an emergency shell, depending on configuration. From the emergency shell, the same `nano /etc/fstab` works (the root filesystem is mounted read-only by default; remount with `mount -o remount,rw /`).

Worth knowing: the exit code from `mount -a` after fixing the typo will tell you immediately if it is right. If it is, `systemctl reboot` from the recovery shell continues normal boot.

---

## Swap: what it is, when it matters

Swap is disk space the kernel uses when RAM is full. *A swap area is a partition or file the kernel can use to move data out of RAM, freeing memory at the cost of slower access.* On modern systems with plenty of RAM, swap is less important than it used to be. On smaller VMs, it still matters.

```
swapon --show
free -h
```

`swapon --show` lists active swap. `free -h` shows memory and swap usage together.

If a box has no swap and you want to add some without repartitioning, swap files are the easy answer:

```
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

That creates a 2 GB swap file, formats it, activates it, and adds it to fstab so it persists across reboots.

### swappiness

The kernel parameter `vm.swappiness` controls how aggressively the kernel uses swap. Default is 60 on most installs. For a server with plenty of RAM, lowering to 10 keeps the kernel from swapping unnecessarily.

```
cat /proc/sys/vm/swappiness          # current value
sudo sysctl vm.swappiness=10         # change for current session
```

To make it permanent, add `vm.swappiness=10` to `/etc/sysctl.conf` or to a file in `/etc/sysctl.d/`.

Whether to tune swappiness is a debate that the Linux community has had for two decades. The honest answer for a beginner: leave it alone unless you have a measured reason to change it. The defaults are reasonable.

---

## Filesystem types: ext4, xfs, and choosing

Two filesystems dominate Linux server use:

**ext4** is the long-time default on Ubuntu. Mature, well-understood, supports filesystems up to 1 EB. Defaults are sensible.

**xfs** is the default on Red Hat-family distributions and is increasingly used on Ubuntu for specific workloads. Better at very large filesystems and high-throughput workloads. Some operations (specifically shrinking) are not supported.

**btrfs** has features ext4 lacks (snapshots, compression, subvolumes) but its operational maturity for production workloads has been debated for years. Defensible to use; defensible to avoid.

The honest beginner advice: ext4 unless you have a specific reason. The reason is almost never "I want btrfs features"; it is almost always "the application I am running has documented best practices for xfs," in which case follow those practices.

### Checking the filesystem type

```
df -T
```

The `-T` flag adds a "Type" column showing what each mount is.

### Checking integrity

`fsck` (filesystem check) is the tool for checking and repairing filesystems. It runs automatically at boot for filesystems that need it. You generally do not run it by hand on a mounted filesystem (it can return wrong results); the right pattern is to run it from a recovery environment on an unmounted filesystem.

```
sudo fsck -n /dev/sda1
```

The `-n` flag means "do not actually fix anything, just report." Safe on mounted filesystems if you only want to see whether problems are reported.

For most working admins, fsck is a tool you read about more than you run. Modern filesystems on healthy hardware rarely need manual fsck. When they do, it is usually a sign of larger problems (failing disk, kernel bug) that fsck alone will not solve.

---

## Encryption foundations

Disk encryption is on the Security+ exam (Domain 4.1, hardening). The full LUKS configuration is past beginner scope, but knowing what is and is not encrypted on a typical box is important.

By default, an Ubuntu 24.04 Server install does not encrypt the disk. The installer offers full-disk encryption as an option; if you do not pick it, your disk is unencrypted.

To check whether a partition is encrypted:

```
sudo blkid
```

If the TYPE is `crypto_LUKS`, the partition is LUKS-encrypted. The decrypted view shows up as a separate device under `/dev/mapper/` once unlocked.

For most server workloads, you encrypt at the cloud-provider level (every major cloud has "encryption at rest" as a default option for storage) rather than configuring LUKS in the OS. The OS-level encryption matters mostly for laptops and on-premises servers with physical access risk.

The encrypted-by-default model on a Ubuntu desktop is LUKS over the entire disk, with a passphrase prompt at boot. The intermediate cohort goes deeper. Beginner takeaway: know whether your disk is encrypted, and know the default for your environment.

---

## Try this

**1. Tour your lab box's storage.**

Run:

```
lsblk
df -h
df -i
mount | column -t
```

For each mounted filesystem, identify: the source (device or UUID), the target (mount point), the filesystem type, and the size. Note any filesystem above 70% used.

**2. Find the heaviest directories.**

```
sudo du -h --max-depth=2 / 2>/dev/null | sort -h | tail -20
```

Read the output. The bottom line is the heaviest directory at depth 2. For each of the top three, drill in one more level (`du -h --max-depth=2 /<dir> 2>/dev/null | sort -h | tail`) until you find the actual cause.

This is a real diagnostic skill. Practice the loop.

**3. Add a swap file.**

If your lab box does not already have swap (check with `swapon --show`), create a 1 GB swap file using the procedure in this chapter. Confirm with `free -h` that the system sees it. Add it to fstab.

After editing fstab, run `sudo mount -a` to confirm the syntax is correct. Do not reboot until it returns clean.

**4. Run `findmnt --verify`.**

```
sudo findmnt --verify --verbose
```

This validates fstab in more detail than `mount -a`. Read the output. Anything not "success" is something to investigate.

**5. Read your fstab carefully.**

```
sudo cat /etc/fstab
```

For each non-comment line, identify all six fields and explain what the entry does. Pay attention to mount options. Note any line that uses `defaults` versus an explicit option list. Note any line that uses `/dev/sdaN` instead of UUID; if you find one, know that this is brittle and worth changing.

---

## Common stumbling blocks

> **`df -h` says space is available but I get "no space left on device."**
> Two usual causes. First, the inodes are exhausted; check `df -i`. Second, a deleted file is being held open by a process; check `sudo lsof | grep deleted` and restart whichever process is holding the file. Both are non-obvious until you have seen them.

> **`du -sh` and `df -h` disagree about how much space is used.**
> Same root cause as the previous point. `du` walks the filesystem and counts files that exist by path. `df` reads filesystem-level accounting. A file that is deleted but still open by a process counts in `df` but not `du`. The difference is the deleted-but-open files.

> **`mount -a` after editing fstab fails with "wrong fs type" or "bad option."**
> Re-read the fstab entry. The most common typos are: filesystem type wrong (using `ext3` instead of `ext4`, or typoing it entirely), mount options that the filesystem does not support, or a syntax error from a missing space or an extra comma. Compare with a working entry in the same fstab.

> **The new disk I added does not appear in `lsblk`.**
> The kernel may not have re-scanned the SCSI bus. On a cloud VM, this usually happens automatically when you attach the disk; if it did not, `sudo partprobe` or `echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan` forces a rescan. On a physical box, a reboot is the simplest fix.

> **I deleted a big log file and `df -h` did not change.**
> The process writing to the log still has the file open. The space is not reclaimed until the process closes the file or exits. Either restart the process, or use `truncate -s 0` next time instead of `rm`.

> **fstab edit broke boot, and the recovery shell is read-only.**
> By default the emergency shell mounts root read-only. Run `mount -o remount,rw /` to make it writable, then edit fstab. After fixing, `systemctl reboot` continues normal boot.

---

## What this gets you

After this chapter:

- You can diagnose "the disk is full" in under a minute on a system you have never seen.
- You know the difference between disks, partitions, and filesystems and can read `lsblk` output fluently.
- You can read every field in /etc/fstab and explain what each entry does.
- You can add a new filesystem or swap file to fstab safely, with the `mount -a` validation step.
- You know what to do if fstab breaks boot, conceptually if not in muscle memory.
- You know that inode exhaustion is a thing and how to check for it.
- You know the deleted-but-open file pattern and how to spot it.

The diagnosis pattern (`df -h` to find the filesystem, `du -sh` to find the directory, `ls` to see what is in there, then careful cleanup) is one of the highest-value patterns in this entire unit. You will use it constantly.

---

## What's next

Chapter 8 is Host networking tools. The chapter where `ip`, `ss`, `ping`, `dig`, and `curl` become muscle memory and you can answer "the network is broken from this box" with a four-step diagnosis.
