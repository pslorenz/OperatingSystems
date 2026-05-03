# Chapter 10: Reading the System

**You come in with:** workshop-level `ps`, `top`. You can list processes and see what is using CPU.
**You leave with:** the ability to answer "what is this process actually doing, what is it touching, what is it talking to" without guessing, plus a working understanding of `/proc` as a tool you query rather than a curiosity you read about.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 2.4 (indicators of malicious activity: process injection, processes with unusual parents, processes with unexpected file handles). Domain 4.4 (alerting and monitoring concepts and tools: file integrity monitoring foundations, process monitoring). Domain 4.8 (incident response: analysis activities). The "process is running but the binary is deleted" pattern in this chapter is the practical version of detecting in-memory persistence, which Domain 2.4 tests at the conceptual level.

---

## Why this chapter matters

This chapter is where the reading-the-system skill genuinely starts to look like security work. Until now you have been learning to operate a Linux box. Now you are learning to interrogate one.

The skills here are the prerequisite for Chapter 11 (Linux artifacts). When you investigate a process during incident response, every question you have ("what is it running, where did it come from, what is it touching, what is it talking to") has an answer in `/proc` or in one of the tools that read `/proc`. Knowing this turns "I do not know what is going on" into "I can find out in 30 seconds."

For sysadmin work, the same skills apply to mundane debugging. "Why is this process using all the disk I/O" or "why is this service taking 30 seconds to start" both reduce to questions about what the process is doing under the hood.

---

## ps in depth

The workshop showed you `ps aux | grep <name>`. There is more.

### The flag conventions

ps has two competing flag styles, BSD and SysV, and bash accepts both. They produce slightly different output.

**BSD style (no dash):**

```
ps aux           # every process, every user, with extra detail
ps axjf          # tree view of all processes
```

**SysV style (with dash):**

```
ps -ef           # every process, full format
ps -eLf          # every process, including threads
```

`ps aux` and `ps -ef` show roughly the same thing in different formats. Most working admins memorize one and use it constantly. I prefer `ps aux` because the columns are more readable; some prefer `ps -ef` because the full command line is at the end without truncation.

### Useful columns

The default `ps aux` output has columns: USER, PID, %CPU, %MEM, VSZ, RSS, TTY, STAT, START, TIME, COMMAND.

Reading them:

- **USER**: the user the process runs as.
- **PID**: process ID.
- **%CPU**: instantaneous CPU usage. A snapshot, not a sustained measure.
- **%MEM**: percent of system RAM used.
- **VSZ**: virtual memory size in KB. Includes memory the process has reserved but not actually used.
- **RSS**: resident set size in KB. Memory actually in RAM right now. **RSS is the number that matters for "how much memory is this using."**
- **TTY**: the terminal that started the process, or `?` for none (system services).
- **STAT**: process state.
- **START**: when the process started.
- **TIME**: cumulative CPU time the process has used.
- **COMMAND**: the command line.

The `STAT` column is a one-or-two-letter code with surprising depth. The first character is the state:

| Code | State |
|---|---|
| R | Running or runnable |
| S | Sleeping (waiting for an event) |
| D | Uninterruptible sleep, usually waiting on disk I/O |
| Z | Zombie (process exited but parent has not collected exit status) |
| T | Stopped (suspended) |

Modifiers can follow:

| Modifier | Meaning |
|---|---|
| < | High priority |
| N | Low priority (niced) |
| s | Session leader |
| + | In the foreground process group |
| l | Multi-threaded |

`Ss+` is "sleeping, session leader, foreground process group." Common for shells. `Z` alone is a zombie, which usually means a parent process is buggy.

### Custom output

You can ask ps for exactly the columns you want:

```
ps -eo pid,user,cmd,stat,start
ps -eo pid,user,cmd --sort=-pcpu | head
ps -eo pid,user,rss,cmd --sort=-rss | head
```

`--sort=-pcpu` sorts descending by CPU percentage. `--sort=-rss` sorts descending by memory. Both are useful for "what is the heaviest thing on this box right now."

### Trees

Sometimes you need to see the parent-child relationships:

```
ps auxf
ps -ejH
pstree -p
```

`pstree -p` is often the most readable. It shows the process hierarchy with PIDs. Very useful for "what spawned this process" investigations.

---

## pgrep and pkill: shortcuts when you know the name

When you know the name of a process and you want its PID:

```
pgrep nginx
pgrep -l nginx
pgrep -f /usr/local/bin/myscript
```

`-l` shows the name alongside the PID. `-f` matches the full command line, not just the process name (essential for matching scripts).

When you want to kill processes by name:

```
pkill nginx              # send SIGTERM to all nginx processes
pkill -9 nginx           # SIGKILL (last resort)
pkill -f myscript        # match the full command line
```

Use `pkill` with care. A typo in the name can match more processes than you intended. Always run `pgrep` first to see what would match, then `pkill` to actually kill them.

The signal model in brief: SIGTERM (15, the default) asks a process to clean up and exit. Most well-written processes respond to SIGTERM by saving state and exiting gracefully. SIGKILL (9) cannot be caught or ignored; the kernel kills the process immediately, no cleanup. Use SIGKILL only when SIGTERM has been ignored and you need the process gone now.

---

## top and htop

`top` is the live process view. The workshop covered the basics. A few keys worth knowing inside top:

| Key | What it does |
|---|---|
| q | Quit |
| h | Help |
| 1 | Toggle per-CPU view |
| M | Sort by memory |
| P | Sort by CPU |
| T | Sort by time |
| k | Kill a process (prompts for PID) |
| r | Renice a process |
| c | Toggle full command line |
| u | Filter by username |

The filter-by-user (`u`) is genuinely useful. When investigating a specific user's activity, `top` with the user filter on shows only their processes.

### htop

`htop` is the friendlier version of top:

```
sudo apt install htop
htop
```

htop has the same role but with a more readable interface, mouse support, and more visible filters. For most working admins, htop is the daily-use tool and top is the fallback for boxes where htop is not installed.

A few htop-specific keys:

| Key | What it does |
|---|---|
| F2 | Setup (configure columns) |
| F4 | Filter (by name, persistent) |
| F5 | Tree view |
| F6 | Sort by column |
| F9 | Send signal |
| F10 | Quit |

The tree view in htop is one of the cleanest ways to see process relationships visually. Press F5 once you are running.

---

## /proc as a tool

The Filesystem Hierarchy chapter introduced /proc as a virtual filesystem. This chapter uses it.

### Per-process directories

Every running process has a directory under /proc named for its PID. Find your shell's PID:

```
echo $$
```

That returns a number. Look at the directory:

```
ls /proc/$$
```

Many entries. The useful ones:

| File | Contents |
|---|---|
| cmdline | The full command line the process was started with. Null-separated. |
| comm | Just the command name, max 15 chars. |
| environ | Environment variables. Null-separated. |
| status | Process status in human-readable form. |
| stat | Process status in machine-readable form (one line). |
| exe | Symlink to the executable on disk. |
| cwd | Symlink to the current working directory. |
| root | Symlink to the process's root directory (almost always /). |
| fd/ | Directory of file descriptors the process has open. |
| maps | Memory map of the process. |
| net/ | Network state visible to this process. |
| ns/ | Namespace handles for the process (containerization). |

### Common queries via /proc

**What is this process running?**

```
cat /proc/<PID>/cmdline | tr '\0' ' '; echo
```

The `tr` and `echo` part is because cmdline uses null separators, so direct cat smashes arguments together.

**What is the executable on disk?**

```
sudo ls -l /proc/<PID>/exe
```

The symlink resolves to the actual binary path, even if the binary has been deleted.

**Where is the process running from?**

```
sudo ls -l /proc/<PID>/cwd
```

The current working directory, as the process sees it.

**What files does it have open?**

```
sudo ls -l /proc/<PID>/fd
```

One symlink per open file descriptor. Network sockets show as `socket:[N]`. Files on disk show as the actual path. The /proc/<PID>/fd view is the lowest-level "what is this process touching" answer.

### The deleted-binary pattern

Run this on your lab box:

```
sudo find /proc/[0-9]*/exe -lname '*deleted*' 2>/dev/null
```

That finds every running process whose binary on disk has been deleted. On a healthy box, the result is usually empty or only includes processes that were running during a package upgrade (the binary was replaced; the running process still references the old, now-unlinked, copy).

On a compromised box, this query is gold. A common malware persistence pattern: drop a binary, run it, delete the binary. The process keeps running from memory; the binary is gone, so file-based scans miss it; the only artifact is the `(deleted)` indicator under /proc.

This is one of the most reliable single-command security checks you can run. We come back to it in Chapter 11.

---

## lsof: the higher-level view

`lsof` (list open files) is the human-friendly version of /proc/PID/fd queries. Files, network connections, anything a process has open.

### lsof basics

```
sudo lsof -p <PID>
```

Lists every open file for one process. Columns: COMMAND, PID, USER, FD (file descriptor number), TYPE, DEVICE, SIZE/OFF, NODE, NAME.

Most useful TYPE values:

- **REG**: regular file on disk
- **DIR**: directory
- **CHR**: character device
- **IPv4** or **IPv6**: network socket
- **unix**: unix domain socket (inter-process communication)

### lsof for network connections

```
sudo lsof -i
sudo lsof -i :80
sudo lsof -i tcp
sudo lsof -i @1.2.3.4
```

`-i` alone lists all internet connections. With a `:port`, filters to that port. With `tcp` or `udp`, filters to that protocol. With `@host`, filters to that remote host.

This is the answer to "what is this box talking to right now." Combined with `ss` from Chapter 8, you have multiple views of the same data.

### lsof for finding what is using a file

```
sudo lsof /var/log/auth.log
sudo lsof +D /var/log
```

The first finds every process with that file open. The second walks the whole directory tree and finds processes with any file under it open. This answers "why can't I unmount this filesystem" (the answer is "process X is using a file in it").

### lsof for the deleted-but-open pattern

```
sudo lsof | grep deleted
```

Same finding as the /proc query above, in friendlier output. Anything that shows up is a deleted file still being held open. For log files, this is the symptom of "I deleted the log to free space but disk usage did not change." For binaries, it is a possible compromise indicator.

---

## strace: watching syscalls

`strace` shows the system calls a process makes. Every read, write, open, close, network operation. *A system call is the boundary between userspace code and the kernel; programs make syscalls when they need the kernel to do something privileged or hardware-related.*

### Two warnings before using strace

First, **strace makes the target process much slower**. It traps every syscall through the kernel's audit infrastructure. A program that runs in 1 ms might run in 100 ms under strace. Do not strace production processes during peak load without thinking about it.

Second, **strace's output is voluminous**. Even a simple `ls` produces hundreds of syscall lines. You filter by what you are looking for or you drown.

### Basic usage

```
strace ls /tmp
```

That runs `ls /tmp` and shows every syscall. The output is one line per syscall, in the format `syscall_name(args) = return_value`.

### Filtering

```
strace -e trace=open,openat ls /tmp
strace -e trace=network curl -s http://example.com
strace -e trace=file ls /tmp
```

`-e trace=` filters the syscall categories. `file` is a useful shortcut for "anything filesystem-related."

### Attaching to a running process

```
sudo strace -p <PID>
```

Attach to a running process. Press Ctrl-C to detach. The process keeps running.

This is the answer to "what is this process actually doing right now." The syscall stream tells you whether it is waiting on a file (poll, read), waiting on the network (recvfrom, sendto), waiting on a lock (futex), or actually working.

### A practical pattern

A service is hung. Find what it is waiting on:

```
sudo strace -p <PID> -t
```

`-t` adds timestamps. After 10 seconds of output, press Ctrl-C. Read the last few lines. If they are all the same syscall, you know what the process is stuck on. If they are `read(N, ...` from a network socket, the process is waiting for the network. If they are `flock(...)`, the process is waiting for a lock. The blocking syscall is the answer.

For most working admins, strace is a power tool you reach for occasionally. Recognition-level fluency (knowing it exists, knowing how to attach, knowing the output is filterable) is the right depth.

---

## Identifying programs: which, type, file

When you type a command, the shell may run several different things behind the scenes. A bash builtin, an alias, a function, or an actual binary. To see which:

```
type ls
type cd
type python3
```

`type` is a bash builtin that tells you exactly what would run. For external programs, it shows the path. For aliases, it shows the alias definition. For functions, it shows the function body.

```
which ls
```

`which` only finds external programs. It searches PATH and prints the first match. It does not know about aliases or functions, which makes it less informative than `type` in many cases.

For deeper inspection of a binary:

```
file /usr/bin/ls
file /etc/passwd
file /tmp/somefile.tar.gz
```

`file` reads the start of a file and tells you what it is. It does not trust filenames; it inspects content. A file named `report.pdf` that is actually a Windows executable will be identified as such. This is the first tool to reach for when triaging unknown files.

```
ldd /usr/bin/ls
```

`ldd` (list dynamic dependencies) shows the shared libraries a binary needs. Useful for "why does this binary not run on this system" (the answer is usually "a required library is missing").

---

## /sys: kernel configuration as files

A brief mention. `/sys` is another virtual filesystem like /proc, but oriented toward devices and kernel parameters rather than processes.

```
ls /sys
ls /sys/class/net
cat /sys/class/net/eth0/operstate
```

For most beginner work, you do not query /sys directly; tools like `ip`, `lsblk`, and `sysctl` do it for you. Recognition is enough. The intermediate cohort goes deeper.

One useful pattern: temporary kernel parameter changes via /proc/sys/ or /sys/. For example, to disable IP forwarding on a host that does not need it:

```
sudo sysctl -w net.ipv4.ip_forward=0
```

The `sysctl` command reads from and writes to /proc/sys/. We see more of this in Chapter 12 (Hardening).

---

## A practical investigation

Walking through a question end-to-end. The question: a process named `runner` is using 80% of the CPU on the box. What is it doing.

**Step 1.** Find the PID and basic info.

```
ps aux | grep runner
```

You see something like `webrunner 4321 80.0 1.2 ...` with the command line.

**Step 2.** What binary is it?

```
sudo ls -l /proc/4321/exe
```

Output like `-> /usr/local/bin/runner`. Confirms the binary path.

**Step 3.** Is the binary still on disk?

```
sudo ls -l /usr/local/bin/runner
```

If it exists, look at it more carefully:

```
sudo file /usr/local/bin/runner
sudo dpkg -S /usr/local/bin/runner
```

If `dpkg -S` returns "not found," the binary was not installed by the package manager. Suspicious; investigate further.

**Step 4.** What files is it touching?

```
sudo lsof -p 4321 | head -30
```

Read the output. If it has `/dev/urandom` open and is reading constantly, it might be doing crypto work. If it has a network socket open to an external IP, that is a question.

**Step 5.** What is it actually doing right now?

```
sudo strace -p 4321 -t -c
```

`-c` summarizes syscalls instead of streaming them. Run for 10 seconds, Ctrl-C, read the summary. The dominant syscall tells you the dominant work.

**Step 6.** What are its parent and children?

```
ps -ef | grep -E "^[^ ]+\s+(4321|.*\s+4321\s)"
pstree -p 4321
```

The parent is who started it. Children are anything it forked.

This sequence is the spine of process investigation. It works for "this is using too much CPU," for "what is this strange process I have never seen before," and for "is this process doing what I think it is doing." The same six steps, every time.

---

## Try this

**1. Take inventory of your shell process.**

Find the PID of your current shell:

```
echo $$
```

Now use /proc to answer:

- What command line is it running? (`/proc/$$/cmdline`)
- What is the binary path? (`/proc/$$/exe`)
- What is the working directory? (`/proc/$$/cwd`)
- What files does it have open? (`ls -l /proc/$$/fd`)

You should see fd 0 (stdin), 1 (stdout), 2 (stderr) at minimum, plus the connection to the terminal.

**2. Find the deleted-binary pattern.**

```
sudo find /proc/[0-9]*/exe -lname '*deleted*' 2>/dev/null
```

On a healthy box this is usually empty or shows a few processes that were running during a recent apt upgrade. Run apt upgrade if your box has updates pending, then run this immediately and you may see services that need a restart to pick up the new binaries.

**3. Watch a process work.**

In one terminal:

```
sudo strace -p $(pidof sshd | tr ' ' '\n' | head -1) -t
```

In another terminal, SSH to your lab box (creating a new connection). Watch the strace output. You see sshd's actual syscalls handling the new connection: accepting the socket, reading from it, calling crypto routines, forking off a worker.

Press Ctrl-C in the strace terminal when you have seen enough. The point is to confirm strace works and to see what real syscall traffic looks like.

**4. Find what is talking to the network.**

```
sudo lsof -i -n -P
```

`-n` does not resolve names. `-P` does not resolve port numbers to service names. Read the output. For each entry, identify what process is making each connection. Anything you cannot identify is a question.

**5. Practice the investigation pattern.**

Pick any process on the box. The cron daemon (`pgrep cron`) is a good target: it is well-known but you may not know exactly what it is doing. Walk through the six steps from the practical investigation section. Answer:

- What is the binary?
- Is it package-managed (`dpkg -S`)?
- What files does it have open?
- What is its parent? Its children?
- Is its dominant syscall what you would expect?

The point is to have done the investigation pattern at least once on a known-good process so you know what normal looks like.

---

## Common stumbling blocks

> **`ps aux | grep <name>` matches my grep command itself.**
> A real bug: `grep nginx` matches the literal string "nginx" in the ps output, including the line for grep nginx itself. The classic fix is `ps aux | grep [n]ginx`, which uses a character class that does not match itself. Or use `pgrep`, which has its own filtering and avoids the issue entirely.

> **`/proc/<PID>/exe` shows "permission denied" even with sudo.**
> Usually means the process is gone. /proc entries are alive only while the process is running. By the time you ran ls, the process exited. Confirm with `ps -p <PID>`; if the process is not there, /proc/<PID> is also not there.

> **`strace` shows nothing or shows "ptrace: Operation not permitted."**
> Several Linux security frameworks (yama, AppArmor, Docker) restrict who can ptrace whom. Check `cat /proc/sys/kernel/yama/ptrace_scope`. If it is 1, only direct parents can ptrace by default. Use sudo. If you are inside a container, ptrace may be entirely disabled by the container's security settings.

> **`lsof` shows `(unknown)` for many file paths.**
> The process has files open that lsof cannot resolve, often because they are anonymous (memory-mapped, pipe, etc.). Sometimes also because lsof needs root to read certain process info. Re-run with sudo.

> **htop or top shows different memory numbers than free.**
> Different tools count memory differently. RES (resident set size) shows physical memory in use. VIRT (virtual size) includes shared libraries and memory-mapped files counted across multiple processes. SHR is shared portions. Adding up RES across all processes overcounts because shared memory is counted once per process. The kernel's view (free, /proc/meminfo) is the authoritative single number; per-process tools always have approximation.

> **The PID I am investigating disappears between commands.**
> Short-lived processes are gone before you can investigate. Either capture the data fast (`while true; do ...; sleep 0.1; done` patterns), or use auditd (Chapter 6) to log process executions, or use `pgrep -f <pattern>` which scans more reliably than ps in a tight loop.

---

## What this gets you

After this chapter:

- You can read ps output fluently and explain every column.
- You can use /proc to answer "what is this process running, where, with what files, talking to what."
- You can find processes whose binaries have been deleted, which is one of the most reliable single security checks on Linux.
- You can use lsof for the higher-level "what files and sockets does this process have open" question.
- You can use strace at survival depth to watch what a process is actually doing.
- You can investigate a process end-to-end with a six-step pattern that works regardless of the symptom.
- You know about /sys exists and what it is for.

The deleted-binary pattern is the single most important specific finding from this chapter. Run it on every box you investigate. The investigation pattern is the most important general skill. Use it on every "what is this process doing" question.

---

## What's next

Chapter 11 is Linux artifacts and what attackers leave behind. The chapter that pulls together everything from the previous chapters and applies it adversarially. By the end, you can do basic IR triage on a Linux host you did not build.
