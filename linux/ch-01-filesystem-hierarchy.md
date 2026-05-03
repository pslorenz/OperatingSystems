# Chapter 1: The Filesystem Hierarchy

**You come in with:** workshop-level recognition of `/etc`, `/var/log`, and `/home`.
**You leave with:** a working mental map of where to look for what on any Linux box you encounter.

**Time:** 45 to 75 minutes including the exercises.

**Security+ alignment:** No direct exam content. The Filesystem Hierarchy Standard is not on the SY0-701 outline. Foundation skill that supports Chapter 6 (logs as data sources, Domain 4.9), Chapter 11 (artifacts and indicators of malicious activity, Domain 2.4), and Chapter 12 (hardening, Domain 4.1).

---

## Why this is chapter one

The workshop taught you how to navigate. This chapter teaches you where to navigate to. The difference matters.

A working Linux admin does not memorize the filesystem. A working Linux admin recognizes patterns. When something is broken, they have a default first directory they look in. When something needs configuring, they know where the config probably lives. When a process is running, they know where its files probably are. That recognition turns Linux from "endless search" into "I know roughly where to look."

The Linux filesystem follows a standard called the Filesystem Hierarchy Standard, or FHS. Ubuntu follows it. Most Linux distributions follow it. That is why the same skills move with you to every Linux box you ever touch.

Spend an hour with this chapter. You will refer back to it for years.

---

## The map

Every file on a Linux system lives somewhere under `/`. There is one root directory, no drive letters, no separate volumes that work like separate worlds. A USB drive plugged into the box does not get a `D:`. It gets mounted into the existing tree, somewhere like `/media/student/USB`. The single tree is the model.

Run this on your lab box:

```
ls /
```

You will see a list that looks similar across every Ubuntu install:

```
bin   dev  home  lib32  libx32      media  opt   root  sbin  srv  tmp  var
boot  etc  lib   lib64  lost+found  mnt    proc  run   snap  sys  usr
```

That is what we are walking through. Not all at once. The interesting ones first, then a tour of the rest.

---

## The directories you will use every day

### /etc: system configuration

Every system-wide configuration lives under `/etc`. SSH server settings. Network configuration. User accounts (the database, not the home directories). Cron schedules. Package repository lists. If something on the box is configured, the configuration is probably under `/etc`.

```
ls /etc | head -20
```

You will recognize `ssh` (the SSH config directory), `apt` (package management), `systemd` (the service system), `network` and `netplan` (network configuration), `passwd` (the user account file).

A few specific files and directories worth knowing:

- `/etc/passwd`: the user account database. Despite the name, it does not contain passwords. Every user, their UID, their GID, their home directory, their shell.
- `/etc/shadow`: the actual password hashes. Readable only by root. If you can read this file as a non-root user, that is a serious finding.
- `/etc/hostname`: the system's hostname.
- `/etc/hosts`: local hostname-to-IP mappings, checked before DNS.
- `/etc/sudoers` and `/etc/sudoers.d/`: sudo configuration. Editing the wrong way breaks sudo. Use `visudo`.
- `/etc/ssh/sshd_config`: SSH server configuration. The file you change to harden SSH.
- `/etc/cron.*` and `/etc/crontab`: scheduled jobs at the system level.
- `/etc/systemd/system/`: your custom systemd unit files.

The pattern: when you need to change the system's behavior, you change a file under `/etc`, then restart the service that reads it.

### /var: things that change

`/var` holds anything the system writes to during normal operation. Logs, mail spools, package caches, web server data, database files, runtime state. The contents grow and shrink. The configuration in `/etc` is mostly stable. The data in `/var` is constantly changing.

```
ls /var
```

The directories that matter most:

- `/var/log`: system and application logs. The first place you look when something is broken. The journal lives here on Ubuntu (`/var/log/journal/`) along with traditional log files like `/var/log/syslog` and `/var/log/auth.log`.
- `/var/cache`: data programs cache for performance. apt's package downloads land in `/var/cache/apt/archives`.
- `/var/lib`: application state. MySQL's databases live in `/var/lib/mysql`. Docker's data lives in `/var/lib/docker`. When you back up an application, you usually back up its `/var/lib` directory.
- `/var/spool`: work queued for processing. Print jobs, mail in transit, cron jobs about to run.
- `/var/tmp`: temporary files that should survive a reboot, unlike `/tmp` which does not.
- `/var/www`: the historical default location for web content. nginx and Apache often serve from here.

When you hit "the disk is full" on a Linux box, `/var` is the first place to look. Logs that grew unbounded, cached package downloads that nobody cleared, application data that filled up.

```
sudo du -sh /var/* 2>/dev/null | sort -h
```

That one-liner shows you what is eating `/var`, sorted from smallest to largest. It is the first command you run for "where did my disk go."

### /home: user files

Each regular user gets a directory under `/home`. Yours is `/home/student`. Your shell starts there. Your dotfiles live there. Anything you create that is not part of the system goes here.

The convention is one directory per user. There are exceptions (some sites use `/home/<group>/<user>` patterns for organization), but in this workshop and most environments, `/home/<username>` is the rule.

`/root` is separate. The root account's home directory is `/root`, not `/home/root`. This is intentional, because `/home` might be on a separate filesystem that is not mounted yet during early boot, and root needs to be able to log in regardless.

### /tmp: scratch space

Anyone can write to `/tmp`. The system clears it on every reboot. If you need a place to dump a file during testing, this is it. If you need a place to keep something for more than a session, this is not it.

`/tmp` has the sticky bit set, which means even though anyone can write to it, only the owner of a file (or root) can delete it. That is what stops user A from deleting user B's tempfiles. We cover the sticky bit in Chapter 3.

---

## The directories you will look at occasionally

### /usr: most installed programs

This is the largest directory on most Linux systems. It holds the vast majority of installed software. The structure mirrors the root in miniature:

- `/usr/bin`: most user-facing programs. `ls`, `grep`, `apt`, almost everything you type.
- `/usr/sbin`: system-administration programs. `useradd`, `nginx`, `sshd`.
- `/usr/lib`: shared libraries and architecture-independent data.
- `/usr/local`: software you (or someone administering this box) installed manually, outside the package manager. `/usr/local/bin` is a common destination for hand-installed tools.
- `/usr/share`: read-only data shared between programs. Documentation, locale files, icon themes.

When you run `ls`, the binary you actually run lives in `/usr/bin/ls`. Try this:

```
which ls
file /usr/bin/ls
```

`which` tells you the path. `file` tells you what kind of file it is.

### /bin and /sbin: historically distinct, now symlinks

On older Linux systems, `/bin` held essential programs needed before `/usr` was mounted, and `/usr/bin` held everything else. On modern Ubuntu, that distinction has been collapsed. `/bin` is now a symbolic link to `/usr/bin`, and `/sbin` to `/usr/sbin`. *A symbolic link, often shortened to symlink, is a special file that points to another file. When you read or run a symlink, the system follows it to the real target.*

```
ls -ld /bin /sbin
```

You will see the `l` at the start of the permissions and an arrow to the real location. This is called the "merged-/usr" change and most modern distributions have made it. The reason to know this: tutorials and books written before 2020 will say "the difference between `/bin` and `/usr/bin` is..." and you can mentally substitute "they are the same thing now."

### /opt: third-party software, manual install

When someone installs commercial or third-party software outside the package manager, by convention it goes in `/opt/<vendor>/<product>`. You will see things like `/opt/google/chrome` or `/opt/atlassian/`.

apt-installed software does not go here. snap-installed software has its own location (`/snap`). `/opt` is specifically for "we downloaded this and unpacked it ourselves."

### /boot: what the system needs to boot

The Linux kernel, the bootloader configuration, and the initial RAM disk. You rarely look here unless something has gone wrong with boot, or you are working on kernel updates. The contents are managed by the package manager. *The kernel is the core program of the operating system, the part that talks directly to the hardware. Everything else, including your shell and every program you run, runs on top of the kernel.*

```
ls /boot
```

You will see one or more files starting with `vmlinuz-` (the kernel) and `initrd.img-` (the initial RAM disk), plus a `grub` directory for the bootloader.

---

## The directories that are not really directories

Two of the entries under `/` are not actually files on disk. They are interfaces to the running kernel.

### /proc: the running system as files

`/proc` is a virtual filesystem. *A virtual filesystem looks like files and directories but is generated on the fly by the kernel rather than read from disk. Reading a "file" in /proc actually asks the kernel for the value at that moment.* The kernel generates the contents on demand. Every running process has a directory there, named by its PID:

```
ls /proc | head
```

You will see numbers (one per process), plus some named entries. Pick a number you saw, like the number for `1` (init), and look:

```
ls /proc/1
```

Each file in there describes something about that process. `cmdline` is the command that started it. `status` is its current state. `environ` is its environment variables. These are not files written to disk. The kernel computes them when you read them.

A few `/proc` files that describe the system as a whole, not a single process:

- `/proc/cpuinfo`: CPU information.
- `/proc/meminfo`: memory information.
- `/proc/uptime`: how long the system has been running.
- `/proc/version`: the kernel version.

```
cat /proc/cpuinfo | head -20
cat /proc/meminfo | head -10
cat /proc/uptime
```

Why this matters: tools like `ps`, `top`, and `free` are mostly just nicely-formatted readers of `/proc`. When those tools are not available or not telling you what you need, you can read `/proc` directly. Chapter 10 (Reading the system) goes deep on this.

### /sys: kernel configuration as files

Like `/proc`, but oriented toward devices and kernel parameters rather than processes. You will rarely read `/sys` directly as a beginner. Knowing it exists is enough for now. Chapter 10 covers it where it actually matters.

---

## The directories you might never touch

A short tour for completeness. You can recognize these and not need to use them.

- `/dev`: device files. Disks (`/dev/sda`), terminals (`/dev/pts/0`), pseudo-devices (`/dev/null`, `/dev/random`). Programs read and write devices through these files. You read about `/dev/null` in scripts as the place output goes to disappear.
- `/lib`, `/lib32`, `/lib64`, `/libx32`: shared library directories. On modern Ubuntu these are symlinks into `/usr/lib`. The package manager puts files here. You do not touch them by hand.
- `/media`: where removable devices get mounted. A USB stick plugged in becomes `/media/student/<label>`.
- `/mnt`: historical convention for temporary manual mounts. If you mount a remote filesystem during troubleshooting, `/mnt/something` is a polite place to put it.
- `/run`: runtime state for the current boot. Process ID files, socket files, lock files. Cleared on every reboot. You will see things like `/run/sshd.pid` (the PID of the SSH server).
- `/srv`: site-specific service data. Some distributions put web content or FTP roots here. Ubuntu rarely uses it. Recognize the name; do not expect to find anything there by default.
- `/snap`: where snap packages get mounted when installed. Each snap is mounted as a read-only filesystem. `ls /snap` shows you what snap packages are installed.
- `/lost+found`: a per-filesystem directory the filesystem itself uses during recovery from corruption. You hope to never have anything in it. If you do, something went wrong.

---

## Try this

Before moving on, work through these on your lab box. They build the recognition the rest of the unit depends on.

**1. Map the configs you care about.**

Without searching, write down what file you would edit to change each of these. Then check yourself with `ls` or `find`.

- The system's hostname.
- The list of users on the box.
- The SSH server's settings.
- The configuration for nginx.
- The list of package repositories apt uses.

If you got three or more right, you have the pattern. If not, walk through `/etc` directory by directory:

```
ls /etc
ls /etc/ssh
ls /etc/nginx
ls /etc/apt
```

**2. Find what's eating disk.**

Run this:

```
sudo du -sh /var/* 2>/dev/null | sort -h
```

The largest directory is usually `/var/log`, `/var/lib`, or `/var/cache`. On a fresh install, `/var/cache/apt` is often the largest because of downloaded package files. Confirm what is largest on your box.

**3. Walk a process through /proc.**

Find the PID of your shell:

```
echo $$
```

That number is your bash process. Look at it:

```
ls /proc/$$
cat /proc/$$/cmdline; echo
cat /proc/$$/status | head -10
```

The `; echo` after `cmdline` is because cmdline does not end with a newline, so without it the next prompt prints in a weird place.

You just looked at a running process by reading the kernel's view of it. That is one of the most powerful patterns in Linux. We use it heavily in Chapter 10 and Chapter 11.

**4. Where is your shell?**

Run:

```
which bash
file /usr/bin/bash
ls -l /bin/bash
```

The third command will show you that `/bin/bash` is a symlink (probably to `/usr/bin/bash`), confirming the merged-/usr layout we talked about.

---

## Common stumbling blocks

> **`ls /` shows different directories than the chapter lists.**
> Some directories only exist on certain installs. `/snap` is Ubuntu-specific. `/lost+found` only exists at the root of an actual filesystem mount. `/srv` is empty by default on Ubuntu and some installations skip it. The list in the chapter is what you usually see, not a guarantee of what is there.

> **`du -sh /var/*` runs forever.**
> `/var` can have a lot of small files (especially in `/var/cache/apt`), and `du` walks every one. Wait for it. If it really hangs, Ctrl-C cancels it. Add `2>/dev/null` (which the chapter shows) to suppress permission-denied warnings on directories you cannot read.

> **`/proc/$$/cmdline` looks empty or weird.**
> The contents of `cmdline` are separated by null characters, not spaces. `cat` displays them but the nulls are invisible, so adjacent arguments smash together. The `; echo` at the end of the chapter command is for this reason. If you want a clean view, run `tr '\0' ' ' < /proc/$$/cmdline; echo`.

> **I cannot find `/var/www`.**
> It only exists once you have installed a web server (which we did in the workshop) or some other package that uses it. On a fresh Ubuntu without nginx or Apache, `/var/www` is absent. The hierarchy is conventions plus what is actually installed.

---

## What this gets you

If you walked through everything in this chapter, you can now answer these questions without searching:

- Where do system configs live? `/etc`.
- Where do logs live? `/var/log`.
- Where does my stuff live? `/home/student`.
- Where do programs live? `/usr/bin` (mostly).
- Where does temporary stuff live? `/tmp`, cleared on reboot.
- Where can I see what is running on the system? `/proc` (mostly through tools like `ps`).
- Where did my disk space go? Probably `/var`, find out with `du`.

That recognition is what lets the rest of this unit go fast. Every time we say "edit the sshd config," you know the file is under `/etc/ssh`. Every time we say "check the journal," you know it lives under `/var/log/journal`. The names stop being arbitrary.

---

## What's next

Chapter 2 is Shell environment plus vi. The chapter where your prompt gets a personality, your aliases survive a logout, and you learn the editor that exists on every Linux box, including the ones nano did not get installed on.
