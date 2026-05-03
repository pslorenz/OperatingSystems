# Chapter 11: Linux Artifacts and What Attackers Leave Behind

**You come in with:** Chapter 6 (logs and auditd), Chapter 9 (reading scripts adversarially), Chapter 10 (process inspection). You know how to read a system; you have not yet read one adversarially end-to-end.
**You leave with:** the ability to do basic IR triage on a Linux host you did not build, with concrete artifact-level checks tied to real attacker techniques.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** This chapter is the densest single mapping to Security+. Domain 2.4 (indicators of malicious activity: account takeover, persistence, defense evasion, command and control). Domain 4.4 (alerting and monitoring concepts and tools: file integrity monitoring). Domain 4.8 (incident response: detection, analysis, root cause analysis, threat hunting). Domain 4.9 (data sources to support an investigation: log data, file metadata, command history, host artifacts). MITRE ATT&CK technique IDs are cited throughout this chapter; ATT&CK alignment is implicitly tested via Security+ Domain 2 and explicitly tested in many follow-on certifications (CySA+, GCFE, GCIH).

---

## Why this is the audience-defining chapter

Up to this point, the unit has been "Linux for sysadmins and security pros" with a security flavor. This is the chapter that makes the security focus explicit. If you remove this chapter, the unit reads as a generalist intro. With this chapter, the unit reads as foundational training for someone heading into security operations, incident response, or threat hunting.

The framing matters. Most introductory Linux courses teach you to operate a Linux box. This chapter teaches you to investigate one. Same commands as the previous chapters, used differently, with a different mental model: assume nothing, verify what you can, treat the system as a witness.

The pattern this chapter teaches:

1. Establish what normal looks like for the kind of system you are looking at.
2. Look for specific artifacts known to be left by specific attacker techniques.
3. Decide which findings are real and which are noise.
4. Document what you found, with reproducible commands, in a way an investigator coming after you can follow.

That pattern, applied repeatedly, is most of what junior incident response analysts do.

---

## A note on lab setup

This chapter assumes you have a "compromised" lab box prepared by your instructor. The box will have planted artifacts representing common attacker behaviors. The exercises walk you through finding and characterizing them.

If your instructor has not set up the compromised box yet, you can still work through the chapter conceptually. The artifact patterns and commands work on any Linux box; on a clean system, the queries will return empty results, which is informative on its own (it shows you what "no findings" looks like).

The placeholder artifacts described in this chapter are the ones the instructor will plant. The list is at the end of the chapter; the exercises will tell you what to look for in general terms, and you will find the specific instances on your lab box.

---

## The mental model: assume nothing

When you walk up to a Linux box you suspect is compromised, the first thing to fight is the impulse to trust the system's own tools. Why:

- `ls`, `ps`, `find`, and other system tools are binaries. If an attacker replaced them with versions that hide their own activity, the system will lie to you. (This is rare in modern attacks but the model still applies.)
- Logs can be edited or rotated to remove evidence.
- Bash history can be cleared.
- File timestamps can be manipulated.
- The package manager database can be tampered with.

The defense: cross-check. When two independent sources agree, you have higher confidence. When they disagree, that disagreement is itself information.

The practical version: do not assume because `ls` says a directory has 5 files that it has 5 files. Run multiple checks. If `ls` and `find` and `du` give different counts, something is interesting.

For most real-world investigations, the attacker is not sophisticated enough to backdoor system binaries. But the mental model still serves you: assume nothing, verify what you can, document what you find.

---

## Establishing baseline: what normal looks like

The first task on any new system is figuring out what is supposed to be there. Without a baseline, every finding looks suspicious.

### Users

```
sudo cat /etc/passwd
sudo cat /etc/shadow | head      # password hashes; root only
```

For each user, ask:

- Should this user exist?
- Is the UID reasonable? (UIDs below 1000 are usually system accounts; UID 0 is root.)
- Is the home directory expected?
- Is the shell expected? (System accounts should usually have `/usr/sbin/nologin` or `/bin/false`.)
- Does the user have a password set? (Empty or `*` in shadow means no password authentication.)

Reference for what a default Ubuntu system has:

```
awk -F: '$3 < 1000' /etc/passwd      # system accounts
awk -F: '$3 >= 1000' /etc/passwd     # regular users
```

A regular Ubuntu install has a `student` user (or whatever you set during install), `nobody`, and a list of system accounts (root, daemon, bin, sys, sync, mail, www-data, sshd, etc.). Anything outside this set warrants a question.

### Listening services

```
sudo ss -tlnp
```

For each listening port, ask: do I expect this service to be running? On the lab box, ssh and possibly nginx are expected. Anything else (a shell on a high port, a service binding to all interfaces unexpectedly) is a question.

### Active SSH keys

```
sudo find / -name authorized_keys 2>/dev/null
```

Every authorized_keys file is a potential entry point. For each one found:

```
sudo cat <path>
```

Read every key. Each key represents someone (or something) that can SSH in as the owner of that file. Unexpected keys are a finding. Keys with comments like "test" or "old laptop" that nobody knows about are findings worth investigating.

### Sudoers

```
sudo cat /etc/sudoers
sudo ls /etc/sudoers.d/
sudo cat /etc/sudoers.d/*
```

Anyone with sudo access has root. Audit what is granted, by whom, with what scope. NOPASSWD entries are particularly worth auditing.

### Cron and timers

```
sudo ls -la /etc/cron.*
sudo ls -la /var/spool/cron/crontabs/ 2>/dev/null
sudo systemctl list-timers --all
sudo find / -name "*.timer" -path "*/systemd/*" 2>/dev/null
```

Every scheduled job is a recurring opportunity. Each one should have an owner you can identify and a purpose you understand.

### Setuid binaries

```
sudo find / -perm -4000 -type f 2>/dev/null
```

You did this in Chapter 3. Compare against the expected list for Ubuntu 24.04. Anything unfamiliar is a finding.

These six checks (users, listeners, SSH keys, sudoers, scheduled jobs, setuid) constitute the first 30 seconds of any triage. Run them on every box you investigate. Document the output. The patterns repeat.

---

## Persistence: how attackers stay

Once an attacker compromises a box, their first priority is persistence. They want to survive a reboot, a session timeout, a password change. There are dozens of persistence techniques on Linux. This chapter covers the most common ones, with the artifact and the ATT&CK reference for each.

### authorized_keys (T1098.004)

Adding an SSH key to a user's `authorized_keys` is the most common Linux persistence mechanism. It survives reboots, password changes, and most defensive reactions short of finding and removing the key.

The artifact: an SSH key in `authorized_keys` that should not be there.

The triage:

```
sudo find / -name authorized_keys 2>/dev/null -exec ls -l {} \;
```

For each file:

```
sudo cat /home/<user>/.ssh/authorized_keys
sudo cat /root/.ssh/authorized_keys
```

For each key, decide: is this expected? Was it added by the user, an attacker, or automation? The comment field at the end of the key (the part after the key data) often hints at provenance. Keys with no comment, or with generic comments like "user@host", are harder to attribute.

A particularly sneaky variant: `authorized_keys` files in unusual locations. The OpenSSH server consults sshd_config to decide where to look. If the config has been changed to also check `/var/cache/ssh/authorized_keys` or something similar, an attacker can put their key in a location that is not in any user's home directory.

```
sudo grep -i AuthorizedKeysFile /etc/ssh/sshd_config
```

The default is `.ssh/authorized_keys`. Anything else warrants a look.

### Cron jobs (T1053.003)

Cron is a classic persistence mechanism. The artifact is a cron entry that calls back to attacker infrastructure.

Locations to check:

```
sudo cat /etc/crontab
sudo ls -la /etc/cron.d/
sudo cat /etc/cron.d/*
sudo ls -la /etc/cron.{hourly,daily,weekly,monthly}/
sudo cat /var/spool/cron/crontabs/*
```

For each cron entry, read what it runs. Anything that downloads from a URL, executes from `/tmp`, or runs from a non-package-managed location is a finding.

A specific pattern to look for: cron entries that run scripts in `/tmp/.something` (hidden directories) or in user home directories. These are common attacker hiding spots.

### systemd timers (T1053.003)

The modern equivalent of cron. Same persistence purpose.

```
sudo systemctl list-timers --all
sudo find /etc/systemd/system /usr/lib/systemd/system -name "*.timer" 2>/dev/null
sudo find /home -path "*/systemd/user/*.timer" 2>/dev/null
```

For each timer, read what service it triggers and read that service's ExecStart. The same red flags apply: downloading from URLs, running from /tmp, running scripts you cannot identify.

User-level timers in `~/.config/systemd/user/` are easy to miss. Check every user's home directory.

### .bashrc, .profile, and similar shell profiles (T1546.004)

Shell startup files run automatically when a user logs in. An attacker who can write to these can run code every time the user logs in.

```
sudo find /home /root -maxdepth 2 -name ".bashrc" -o -name ".bash_profile" -o -name ".profile" -o -name ".bash_login" -o -name ".zshrc" 2>/dev/null
```

For each file, scan for unexpected content. Look for:

- `curl ... | bash` or `wget ... | bash` patterns
- references to scripts in /tmp or hidden directories
- definitions of aliases that override common commands (especially `ls`, `ps`, `who`, `last`)
- code that runs only at certain times or after certain conditions

A particularly dangerous pattern: aliases like `alias ls='ls --someharmless-flag; /tmp/.malware'`. The malware runs every time the user types `ls`. Read shell profile files carefully.

### /etc/profile.d/ (T1546.004)

System-wide shell profile scripts. Every interactive shell sources every file in this directory.

```
sudo ls -la /etc/profile.d/
sudo cat /etc/profile.d/*.sh
```

Most files here are package-installed (you can confirm with `dpkg -S /etc/profile.d/<file>`). Files not owned by any package are worth investigating.

### Modified system binaries (T1554)

Replacing a system binary with a backdoored version is the classic rootkit technique. Less common today than 20 years ago but still happens.

```
sudo dpkg -V 2>/dev/null
```

`dpkg -V` verifies that installed package files have not been modified since installation. Output is empty for a clean system. Any output indicates a file that has changed.

Caveats: configuration files in /etc are expected to change (they show up with a `c` in the second column, which dpkg recognizes). Library updates and other legitimate changes can also produce output. The point is not "any output is malicious" but "anything unexpected here warrants investigation."

### Service unit modifications (T1543.002)

Adding a systemd service is the modern rootkit technique. It survives reboots, runs at boot, and can be configured to restart if killed.

```
sudo find /etc/systemd/system /usr/lib/systemd/system -name "*.service" -newer /etc/passwd
```

That finds service units modified more recently than `/etc/passwd`, which is a rough baseline for "since system install." Recent service units that are not from a package are worth investigating.

For each service unit found, read:

```
sudo systemctl cat <service>
sudo dpkg -S /etc/systemd/system/<service>
```

If `dpkg -S` says "no path found," the service is not from a package. Read its ExecStart and decide whether it makes sense.

---

## Defense evasion: what attackers hide

After persistence, attackers try to hide. Each technique leaves a different artifact.

### Cleared bash history (T1070.003)

Already discussed in Chapter 2 and Chapter 6. Worth recapping with the adversarial framing.

The artifact: an empty or near-empty `~/.bash_history` for a user who has clearly done substantial work.

```
sudo ls -la /home/*/.bash_history /root/.bash_history
sudo wc -l /home/*/.bash_history /root/.bash_history
```

For each user, ask: does the size of the history file match the activity I expect from this user? A user who has been on the system for months but has 5 lines of history is suspicious.

Variant patterns to look for:

- HISTFILE pointing to /dev/null in a shell profile
- HISTSIZE=0 in a shell profile
- An empty file with very recent mtime (someone cleared it; the file remains)

```
sudo grep -l HISTFILE /home/*/.bashrc /home/*/.profile /root/.bashrc /root/.profile 2>/dev/null
```

### Modified or cleared logs (T1070.002)

Logs can be edited or rotated to remove evidence.

```
sudo ls -la /var/log/auth.log* /var/log/syslog*
sudo ls -la /var/log/audit/ 2>/dev/null
```

Compare modification times. Auth.log should be growing over time, with the current file (auth.log) being the most recently modified. Files that are unexpectedly small or recently truncated are suspicious.

Cross-reference with journalctl: the journal is harder to selectively edit than text logs. If auth.log shows nothing for the last 3 days but the journal has authentication events, the auth.log was tampered with.

### Files in unusual locations (T1564)

Hidden files (starting with `.`) and files in temporary directories are common hiding spots.

```
sudo find /tmp /var/tmp /dev/shm -type f -mtime -7 2>/dev/null
sudo find /tmp -name ".*" 2>/dev/null
sudo find /home -name ".*" -type d 2>/dev/null | grep -v -E "\.cache|\.config|\.local|\.ssh|\.gnupg"
```

The first finds recent files in temp directories. The second finds hidden files in /tmp specifically. The third finds hidden directories in user home directories that are not the standard ones (cache, config, local, ssh, gnupg).

`/dev/shm` is particularly worth scanning. It is a tmpfs (RAM-backed filesystem) that is world-writable and survives normal scans because it is "just temp space." Attackers love it.

### World-writable files in unexpected places (T1574)

Files that anyone can write are potential hijack targets.

```
sudo find / -perm -o+w -type f 2>/dev/null | grep -v -E "^/proc|^/sys|^/dev/shm|^/tmp|^/var/tmp"
```

The grep filters out locations where world-writable is normal. What remains is unexpected. World-writable files in /etc, /usr/bin, /var/lib, or anywhere else system-level are findings.

### Recently modified files (T1070)

When investigating an incident with an approximate timeframe:

```
sudo find / -type f -newer /tmp/marker -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
```

(Where `/tmp/marker` is a file with the timestamp of "before the suspected incident.")

For an unbounded "what changed recently":

```
sudo find / -type f -mtime -1 -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
```

That finds files modified in the last day. Filter further to focus on suspicious paths (/etc, /usr, /var/lib).

---

## Command and control: what attackers talk to

Persistence and evasion put the attacker in. C2 is how the attacker uses access. The artifacts are the network connections and the configuration that enables them.

### Active connections (T1071)

```
sudo ss -tunap state established
sudo lsof -i -n -P
```

For every active connection, ask: do I expect this process to be talking to that destination? A web server talking out to an external IP on port 443 is suspicious. A backup script connecting to its known backup target is fine.

Cross-reference with DNS. A connection to a raw IP (no hostname) is more suspicious than a connection to a known service. A connection to a high-port-number listener on the box (1024-65535) is more suspicious than one to a known service port.

### Outbound proxy/SSH tunnels (T1572)

SSH local and remote port forwarding can create reverse tunnels. The artifact is sshd or ssh processes with unusual command lines.

```
sudo ps -ef | grep -E "ssh.*-R |ssh.*-L "
```

Reverse SSH tunnels (where this box reaches out to attacker infrastructure and the attacker tunnels back through that connection) are the canonical pattern. The `-R` flag is the giveaway.

### DNS to suspicious destinations (T1071.004)

If you have access to DNS query logs, look at what hostnames the box is resolving. DGA (Domain Generation Algorithm) domains and known-bad domains are signals. Without DNS logs, you can correlate ARP table and connection state with `dig` to figure out what is being talked to.

For a simple check:

```
sudo grep -E "(curl|wget|nc).*http" /home/*/.bash_history /root/.bash_history 2>/dev/null
```

That finds commands in user histories that fetched from URLs. Each instance is worth understanding.

---

## A complete triage walkthrough

Putting it all together. The scenario: you have been told that "something is wrong" with the lab box. Walk the triage from the start.

**Phase 1: situational awareness (5 minutes).**

```
hostname; date; uptime
who -a
sudo ss -tlnp
sudo ps -ef --sort=-pcpu | head -20
df -h
free -h
```

Know who you are looking at, what is running, what is listening, what is heavy.

**Phase 2: persistence checks (10 minutes).**

```
sudo cat /etc/passwd
awk -F: '$3 == 0' /etc/passwd                 # any UID 0 besides root?
awk -F: '$3 >= 1000' /etc/passwd               # regular users
sudo find / -name authorized_keys 2>/dev/null -exec cat {} \;
sudo grep -E "^[^#]" /etc/sudoers /etc/sudoers.d/* 2>/dev/null
sudo systemctl list-timers --all
sudo crontab -l 2>/dev/null
sudo ls /var/spool/cron/crontabs/ 2>/dev/null
sudo ls -la /etc/cron.{hourly,daily,weekly,monthly}/
```

For each output, decide: expected or not.

**Phase 3: evasion checks (10 minutes).**

```
sudo wc -l /home/*/.bash_history /root/.bash_history 2>/dev/null
sudo find /tmp /var/tmp /dev/shm -type f 2>/dev/null
sudo find / -perm -4000 -type f 2>/dev/null
sudo find /proc/[0-9]*/exe -lname '*deleted*' 2>/dev/null
```

The deleted-binary check is the high-value one. Everything else is a sweep for the obvious patterns.

**Phase 4: connection checks (5 minutes).**

```
sudo ss -tunap state established
sudo lsof -i -n -P 2>/dev/null
sudo ps -ef | grep -E "ssh.*-R |ssh.*-L " | grep -v grep
```

What is talking to the network. Anything you cannot identify is a question.

**Phase 5: timeline (10 minutes).**

```
sudo find /etc -newer /etc/hostname -type f 2>/dev/null | head -50
sudo find /usr/bin /usr/sbin -newer /etc/hostname -type f 2>/dev/null
sudo dpkg -V 2>/dev/null
sudo journalctl --since "24 hours ago" -p warning
sudo grep -E "Failed|Accepted" /var/log/auth.log | tail -100
```

What changed recently? When did it happen? What does the auth log say?

That is roughly 40 minutes of triage, applied as a checklist. It is not exhaustive. It is the first pass that sets up the deeper investigation.

---

## Try this

These exercises assume the instructor has prepared a compromised lab box with planted artifacts. If you are working through the chapter without that box, do the exercises on a clean box and confirm the queries return empty (which is what a clean baseline should look like).

**1. Run the full triage walkthrough on the prepared lab box.**

Follow the five phases from the walkthrough above. Document your findings as you go: write each command you ran, what it returned, and your interpretation. Aim for a written triage report at the end.

**2. Find the planted persistence.**

The instructor has planted at least one persistence mechanism. Find it. The likely categories are: a backdoor account, an unexpected SSH key, a malicious cron entry, a malicious systemd timer, or a modified shell profile.

For each finding, document:
- The exact location (file path, line number if applicable)
- The relevant artifact content
- The MITRE ATT&CK technique it represents

**3. Find the planted evasion.**

The instructor has planted at least one evasion technique. Find it. Likely categories: cleared bash history, modified log file, file in /dev/shm, hidden directory in a user home, world-writable file in an unexpected location.

Document as before, with ATT&CK references.

**4. Find the deleted-binary process, if planted.**

If the instructor has planted a deleted-binary scenario, run:

```
sudo find /proc/[0-9]*/exe -lname '*deleted*' 2>/dev/null
```

For any matches, identify the process, what it appears to be doing, and what binary it ran from.

**5. Write the report.**

Take your findings from exercises 1-4 and write a one-page IR report. Format:

- **Summary**: one paragraph stating what you found and your assessment.
- **Findings**: numbered list, one per artifact, with the location, evidence, and ATT&CK reference.
- **Recommendations**: what you would do to remediate, prioritized by what to do first.

The report is the deliverable. Working IR analysts produce these constantly, and the discipline of writing one is itself the skill.

---

## Common stumbling blocks

> **I cannot find anything suspicious on the lab box.**
> Either the artifacts are not yet planted (instructor has not configured the box), or you are not running the queries with sudo where needed. Most of the queries in this chapter need root to read the relevant files. Confirm you are using sudo on each.

> **`dpkg -V` returns hundreds of lines on a normal-looking system.**
> Some output is normal: `c` in the second column means a config file in /etc that is expected to differ from the package version. The findings worth investigating are files where neither column starts with `c` and the file is in a system path. Filter out config-file noise: `dpkg -V 2>/dev/null | grep -v '^..5...c '`.

> **The "find authorized_keys" search returns my own key.**
> Yes, that is correct: your own SSH key is in `authorized_keys` (you put it there). The exercise is about identifying every entry, including yours, and confirming each one is expected. Your own legitimate key is one of the entries you confirm; an attacker's key would be among the entries you cannot confirm.

> **I found a finding but I cannot tell if it is real or planted by the instructor or a normal Ubuntu artifact.**
> When in doubt, run the same query on a known-clean Ubuntu 24.04 reference image. If the artifact appears on the clean image too, it is normal Ubuntu. If it only appears on your lab box, it is either a planted artifact or a real anomaly. The reference comparison is a real-world technique called "baselining" and it is exactly what professional investigators do.

> **`find / ...` queries take forever.**
> Several of the queries in this chapter walk the whole filesystem. On large systems they can take minutes. Add `-xdev` to limit to one filesystem, or add `-not -path "/proc/*" -not -path "/sys/*"` to skip the virtual ones.

> **The journalctl query returns nothing for events I know happened.**
> Either the journal is volatile and was cleared on the most recent reboot (check `/var/log/journal/` exists and has content; from Chapter 6), or the time filter is wrong (the lab box may be in a different timezone than you), or the events were redirected away from the journal by an attacker.

---

## What this gets you

After this chapter:

- You can do basic IR triage on a Linux host you did not build.
- You can run the five-phase triage walkthrough as a repeatable checklist.
- You can identify the most common Linux persistence techniques and find their artifacts.
- You can identify the most common Linux evasion techniques and find their artifacts.
- You can produce an IR report that an investigator coming after you can follow.
- You know enough MITRE ATT&CK to map findings to techniques and discuss them in the vocabulary the field uses.

This chapter is the bridge from "I can administer Linux" to "I can investigate Linux." Both skills are valuable on their own. The combination is what makes someone effective in a security operations or IR role.

The intermediate cohort builds significantly on this chapter, with deeper artifact analysis, /proc forensics, memory forensics introduction, and the LimaCharlie-mediated detection and hunting workflow. This chapter is where that path begins.

---

## Placeholder: artifacts the instructor will plant

The compromised lab box will have a subset of the following artifacts planted. The exercises above will tell you which categories to look for; this list is the recommended set for the instructor to build against. Each artifact maps to a category and a MITRE ATT&CK technique.

**1. Backdoor account.** A user named something innocuous (e.g., `support`, `mysql_admin`, `webops_ops`) with UID 0 in `/etc/passwd`. Despite the legitimate-looking name, the UID 0 makes it root-equivalent. ATT&CK T1136.001.

**2. Unexpected SSH key.** A new key in `/root/.ssh/authorized_keys` with a comment that does not match the box's expected administrators. ATT&CK T1098.004.

**3. Malicious cron entry.** A drop-in file in `/etc/cron.d/` named to look like a system task (e.g., `apt-cleanup`, `man-db-update`) that actually runs `curl <URL> | bash`. ATT&CK T1053.003.

**4. Malicious systemd service.** A service unit in `/etc/systemd/system/` named to blend in (e.g., `system-update.service`, `network-monitor.service`) that runs an attacker-controlled binary. Enabled to start at boot. ATT&CK T1543.002.

**5. Modified shell profile.** An entry in `/etc/profile.d/system-init.sh` (or similar) that runs every time any user logs in. Could be `curl ... | bash` or could spawn a reverse shell. ATT&CK T1546.004.

**6. Cleared bash history with the suspicious-emptiness pattern.** Root's `.bash_history` is empty or has 1-2 lines, despite root having clearly performed substantial work. Combined with HISTFILE redirected to /dev/null in `/root/.bashrc`. ATT&CK T1070.003.

**7. File in /dev/shm.** A binary or script in `/dev/shm/.cache` (hidden, in shared memory). Possibly running, possibly a staged payload. ATT&CK T1564.003 or T1059.

**8. Setuid binary in an unexpected location.** A copy of `/bin/bash` (or a custom binary) with setuid root in `/tmp/.system/`, `/home/<user>/.local/bin/`, or similar. ATT&CK T1548.001.

**9. World-writable file in /etc.** A specific file in /etc that should not be world-writable (e.g., `/etc/cron.daily/cleanup`) but has mode 777. The file content is benign-looking but the writability lets anyone replace it. ATT&CK T1574.

**10. Deleted-binary process.** A running process whose binary on disk has been deleted. The process is doing something low-grade (sleeping in a loop, periodically writing to a log) but its binary is gone. ATT&CK T1070.004 in combination with T1059.

The instructor should plant 4-6 of the 10 for the chapter exercises, mixed across persistence, evasion, and C2 categories. The exercises ask students to find at least one persistence and one evasion artifact; the rest are stretch goals or material for instructor-led debrief.

The instructor should also have a debrief session after students complete the exercises, walking through every planted artifact (whether students found it or not) with the ATT&CK reference and the reasoning. Students learn as much from the artifacts they missed as from the ones they found.

---

## What's next

Chapter 12 is CIS-aligned hardening. The capstone of the unit. By the end you will have hardened your lab box against a curated subset of the CIS Ubuntu 24.04 Benchmark, with a documented before-and-after diff. That diff is your portfolio piece.
