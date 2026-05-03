# Chapter 12: CIS-aligned Hardening

**You come in with:** Chapter 3 (permissions and sudoers), Chapter 6 (logs and auditd), Chapter 8 (host networking and ufw), Chapter 10 (process inspection), Chapter 11 (artifacts you would not want left behind on your own systems).
**You leave with:** a hardened lab box. You took a default Ubuntu 24.04 install and made it materially more resistant to compromise, with a documented before-and-after diff. The diff is your portfolio piece.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Heavy alignment, this is the capstone chapter. Domain 4.1 (hardening techniques: encryption, configuration enforcement, host-based firewall, default password changes, removal of unnecessary software, applying security techniques to operating systems and servers). Domain 1.4 (cryptographic solutions: PKI applied to SSH keys). Domain 2.5 (mitigation techniques: segmentation, configuration enforcement, hardening, patching). Domain 4.5 (operating system security: SELinux/AppArmor at recognition depth). Domain 4.6 (access controls: least privilege applied across the host). Domain 5.4 (security awareness practices: documenting security configurations).

---

## Why the unit ends here

Twelve chapters in, you can operate a Linux box, audit a Linux box, and investigate a Linux box. The natural last question: how do you make a Linux box less likely to need investigation in the first place.

Hardening is the answer. Configuration changes that reduce the attack surface, that follow the principle of least privilege, that align with industry-recognized standards. The work is mostly small, individually unimpressive changes that combine into a meaningfully different security posture.

The CIS Benchmarks are the most widely referenced hardening standards. *CIS, the Center for Internet Security, publishes detailed configuration baselines for operating systems, applications, and cloud platforms.* The Ubuntu 24.04 LTS Benchmark has hundreds of items, organized into Profile Level 1 (essential, low operational impact) and Profile Level 2 (more aggressive, higher operational cost). A real production hardening exercise typically targets Level 1 across the board, with selected Level 2 items based on the system's role.

This chapter does not work through hundreds of items. It works through a curated subset of about 20, chosen to:

- Cover the major CIS sections (filesystem, network, logging, auth, mandatory access control, services).
- Reinforce skills from previous chapters.
- Produce a measurable change in posture.
- Fit in 60 to 90 minutes.

Your output is a before-and-after comparison: the configuration of your box before this chapter, the configuration after, and the diff between them. That diff is what you can show in an interview as evidence of what you learned.

---

## A note on benchmark numbers

The CIS items in this chapter reference the structure of the CIS Ubuntu 24.04 Benchmark v1.0.0 (or whichever version is current when you read this). The numbering uses the format `Section.Subsection.Item` (e.g., 5.2.5).

CIS occasionally renumbers items between releases. If the specific number on your benchmark differs slightly, the principle is what matters; look up the corresponding rule by name in your benchmark version. The benchmark itself is freely available at cisecurity.org with registration.

---

## Setting up the before-state baseline

Before you change anything, capture the starting state. Without a baseline, you cannot describe what you changed.

Create a working directory:

```
mkdir -p ~/hardening
cd ~/hardening
```

Capture the relevant config files and outputs:

```
sudo cp /etc/ssh/sshd_config sshd_config.before
sudo cp /etc/sysctl.conf sysctl.conf.before 2>/dev/null
sudo sysctl -a 2>/dev/null > sysctl.before
sudo ls -la /tmp /var/tmp > tmpmounts.before
sudo ufw status verbose > ufw.before 2>&1
sudo systemctl list-units --state=enabled --type=service > services.before
sudo cat /etc/login.defs > login.defs.before
sudo passwd -S root > root.passwd.before 2>&1
sudo journalctl --vacuum-files 1 2>&1 | head -1 > journal.before
mount > mount.before
```

You now have a snapshot. Every change in the rest of the chapter will be compared to this snapshot at the end.

---

## Section 1: filesystem and partition mount options (CIS 1.1)

Most Linux installations put `/tmp` and `/var/tmp` on the root filesystem with default mount options. CIS recommends moving them to dedicated partitions or tmpfs, with specific mount options that reduce attack surface.

The mount options that matter:

- `nosuid`: setuid bits on files in this filesystem are ignored. An attacker who plants a setuid binary in /tmp cannot use it for privilege escalation.
- `nodev`: device files in this filesystem are not honored. Reduces the attack surface of someone creating fake device files.
- `noexec`: binaries on this filesystem cannot be executed. Adds friction for "drop a binary in /tmp and run it" attacks.

The combination `nosuid,nodev,noexec` on /tmp is one of the cheapest, most effective hardening changes you can make.

### CIS 1.1.2 Configure /tmp

On modern Ubuntu, /tmp is often already a tmpfs (RAM-backed). Confirm:

```
mount | grep " /tmp "
```

If you see `tmpfs on /tmp type tmpfs (rw,nosuid,nodev,...)`, /tmp is already tmpfs and likely already has nosuid and nodev. If not, configure it.

The CIS-aligned config for /tmp uses systemd's tmp.mount unit:

```
sudo systemctl enable --now tmp.mount
sudo cp /usr/share/systemd/tmp.mount /etc/systemd/system/tmp.mount
sudo systemctl edit tmp.mount
```

In the override, add:

```
[Mount]
Options=mode=1777,strictatime,nosuid,nodev,noexec
```

Reload and apply:

```
sudo systemctl daemon-reload
sudo systemctl restart tmp.mount
mount | grep " /tmp "
```

You should see all three options in the mount line.

### CIS 1.1.4 Configure /var/tmp

Linking /var/tmp to /tmp is a common pattern. Otherwise, mount /var/tmp with the same options.

For most lab environments, the simplest correct answer is to bind-mount /var/tmp to /tmp:

```
echo "/tmp /var/tmp none bind,nosuid,nodev,noexec 0 0" | sudo tee -a /etc/fstab
sudo mount -a
mount | grep "/var/tmp "
```

After reboot, /var/tmp will be bound to /tmp with the same restrictive options.

---

## Section 2: SSH server hardening (CIS 5.2)

SSH is the most common remote access surface and the most common attack vector against it. CIS has many SSH items; this chapter covers the highest-impact ones.

Before changing anything, back up:

```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.before-hardening
```

Edit the config:

```
sudo vi /etc/ssh/sshd_config       # or nano
```

### CIS 5.2.5 Disable SSH root login

Find the line `#PermitRootLogin prohibit-password` (or similar) and change it to:

```
PermitRootLogin no
```

Direct root login over SSH is convenient and dangerous. Disabling forces use of named accounts plus sudo, which is auditable.

### CIS 5.2.4 Set strong protocol and configuration

Add or confirm:

```
Protocol 2
LogLevel VERBOSE
```

VERBOSE logging captures public key fingerprints used at login, which is invaluable for incident response.

### CIS 5.2.10 Disable empty passwords

Add or confirm:

```
PermitEmptyPasswords no
```

### CIS 5.2.11 Configure password authentication

If you have key-based auth working (which Chapter 0 set up), disable password auth entirely:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
```

**Critical**: confirm key-based auth works before saving and reloading sshd. If it does not, you will lock yourself out.

The safe pattern: in a separate terminal, confirm `ssh labbox` works without prompting for a password. If it does, the change is safe.

### CIS 5.2.13 Limit access via SSH (AllowUsers/AllowGroups)

Add a line restricting who can SSH:

```
AllowUsers student
```

Or, if you have created a sysadmin group:

```
AllowGroups sysadmin
```

This prevents accounts that should not have SSH access from getting it, even if they have a valid password or key.

### CIS 5.2.14 Configure idle timeout

Add:

```
ClientAliveInterval 300
ClientAliveCountMax 0
```

This drops idle SSH sessions after 5 minutes. Reduces the impact of an admin walking away from an open session.

### CIS 5.2.18 Limit MaxAuthTries

Add or confirm:

```
MaxAuthTries 4
```

After 4 failed authentication attempts in a session, sshd drops the connection. Slows down brute-force attempts.

### Apply and verify

After saving the file:

```
sudo sshd -t              # syntax check; must return cleanly
sudo systemctl reload ssh
```

`sshd -t` validates the config without applying it. If the syntax is bad, fix it before reloading. **Never reload sshd with a bad config; it will refuse to start and you will lose remote access.**

After reload, in a separate terminal, confirm SSH still works (it should be unchanged for you).

---

## Section 3: kernel and network parameters (CIS 3.2-3.3)

Sysctl parameters control kernel behavior. Many defaults are oriented toward compatibility rather than security; CIS recommends tighter values.

Create a drop-in sysctl file:

```
sudo tee /etc/sysctl.d/99-cis-hardening.conf > /dev/null <<'EOF'
# CIS 3.2.1 - Disable IP forwarding (host is not a router)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# CIS 3.2.2 - Do not send ICMP redirects (host is not a router)
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# CIS 3.3.1 - Do not accept source-routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# CIS 3.3.2 - Do not accept ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# CIS 3.3.3 - Do not accept secure ICMP redirects
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# CIS 3.3.4 - Log suspicious packets (martians)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# CIS 3.3.5 - Ignore broadcast ICMP requests
net.ipv4.icmp_echo_ignore_broadcasts = 1

# CIS 3.3.6 - Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# CIS 3.3.7 - Enable reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# CIS 3.3.8 - Enable TCP SYN cookies
net.ipv4.tcp_syncookies = 1

# CIS 1.5.3 - Enable address space randomization
kernel.randomize_va_space = 2
EOF
```

Apply:

```
sudo sysctl -p /etc/sysctl.d/99-cis-hardening.conf
```

Each line in the output confirms a parameter applied. If any line errors, that parameter does not exist on your kernel; remove the line and reapply.

Verify a few key values:

```
sysctl net.ipv4.ip_forward
sysctl net.ipv4.tcp_syncookies
sysctl kernel.randomize_va_space
```

---

## Section 4: host firewall (CIS 4.2)

You configured ufw briefly in Chapter 8. The CIS-aligned configuration is more deliberate.

### CIS 4.2.1 Ensure ufw is installed

```
sudo apt install ufw
```

### CIS 4.2.2 Default deny incoming, allow outgoing

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### CIS 4.2.3 Allow only required services

For a typical lab box: SSH, and HTTP/HTTPS if you are running nginx.

```
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### CIS 4.2.6 Disable IPv6 firewall if you do not use it

If you are not using IPv6 on this box, disable IPv6 firewall rules in `/etc/default/ufw`:

```
sudo sed -i 's/^IPV6=yes/IPV6=no/' /etc/default/ufw
```

If you are using IPv6 (most modern environments are), leave this alone.

### Enable

```
sudo ufw enable
sudo ufw status verbose
```

Confirm SSH still works in a separate terminal before doing anything else. If it does not, disable ufw immediately (`sudo ufw disable`) and reconfigure.

---

## Section 5: AppArmor (CIS 1.6)

AppArmor is Ubuntu's mandatory access control framework. *Mandatory access control is a security model where the system enforces access policies that even privileged users cannot override.* On Ubuntu, AppArmor is enabled by default and provides profiles for many common services.

### CIS 1.6.1 Verify AppArmor is enabled

```
sudo aa-status
```

Output should show enabled, with a count of loaded profiles. If aa-status reports "command not found" or AppArmor is not enabled, install:

```
sudo apt install apparmor apparmor-utils
sudo systemctl enable --now apparmor
```

### CIS 1.6.4 Ensure all profiles are in enforce mode

aa-status output groups profiles by mode: enforce (the policy is enforced), complain (violations are logged but allowed), and unconfined (no policy applied).

For maximum hardening, set all profiles to enforce. For most lab environments, accepting the defaults is fine: complain mode is itself a hardening layer because it logs.

If you want to enforce everything:

```
sudo aa-enforce /etc/apparmor.d/*
```

Be ready to debug applications that break under enforcement. The AppArmor logs are in the journal:

```
sudo journalctl -u apparmor -k --since "1 hour ago"
```

### CIS 1.6.2 Ensure the bootloader configuration includes AppArmor

```
sudo grep apparmor /etc/default/grub
```

Should include `apparmor=1 security=apparmor` in the GRUB_CMDLINE_LINUX line. If it does not, add it and run `sudo update-grub` (this requires a reboot to take effect).

---

## Section 6: account and password policies (CIS 5.4)

### CIS 5.4.1 Set password aging

Edit `/etc/login.defs` and ensure these values:

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   7
```

These apply to new accounts. For existing accounts, run:

```
sudo chage -M 90 -m 1 -W 7 <username>
```

Apply to each user account. The `--list` flag shows current settings.

### CIS 5.4.2 Set password complexity

This is in `/etc/security/pwquality.conf` (install `libpam-pwquality` if it is missing). Recommended minimums:

```
minlen = 14
minclass = 4
```

Combined: passwords must be at least 14 characters with at least one each of uppercase, lowercase, digit, and special character. (Note: password policy is a debate; this is the CIS recommendation, not a universal best practice.)

### CIS 5.4.4 Lock the root account

```
sudo passwd -l root
sudo passwd -S root
```

The status output should show `L` (locked). With root locked, the only way to gain root is via sudo from a privileged user. This combined with the SSH root-login disable from Section 2 means there is no way to log in directly as root, period.

This is one of the most impactful hardening changes you can make. It also requires that you have at least one user with sudo working before you apply it.

---

## Section 7: logging (CIS 4.1, 4.3)

### CIS 4.3.1 Ensure rsyslog or systemd-journald is installed

systemd-journald is installed by default on Ubuntu 24.04. Confirm:

```
sudo systemctl status systemd-journald
```

### CIS 4.3.2 Ensure journald is configured for persistence

This was covered in Chapter 6. Confirm:

```
ls /var/log/journal/ | wc -l
```

If non-empty, you have persistent journals.

If empty:

```
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

### CIS 4.3.3 Ensure journald is configured to compress and forward

Edit `/etc/systemd/journald.conf` and set:

```
Compress=yes
Storage=persistent
ForwardToSyslog=yes
```

Restart:

```
sudo systemctl restart systemd-journald
```

For a real production system, you would also configure log forwarding to a central SIEM. That is past the scope of this chapter; the intermediate cohort covers it.

---

## Section 8: install auditd (CIS 4.1.1)

You may have done this in Chapter 6. If not:

```
sudo apt install auditd audispd-plugins
sudo systemctl enable --now auditd
sudo aureport --start today
```

For real production hardening, you would also load a rule set (CIS publishes one, and several STIG-aligned alternatives exist). For this chapter, having auditd installed and running is sufficient.

---

## Capturing the after-state

You have made the changes. Now capture the after-state for the diff:

```
cd ~/hardening

sudo cp /etc/ssh/sshd_config sshd_config.after
sudo sysctl -a 2>/dev/null > sysctl.after
sudo ls -la /tmp /var/tmp > tmpmounts.after
sudo ufw status verbose > ufw.after 2>&1
sudo systemctl list-units --state=enabled --type=service > services.after
sudo cat /etc/login.defs > login.defs.after
sudo passwd -S root > root.passwd.after 2>&1
mount > mount.after
sudo aa-status 2>&1 > apparmor.after
```

Generate the diffs:

```
diff sshd_config.before sshd_config.after > sshd_config.diff
diff sysctl.before sysctl.after > sysctl.diff
diff mount.before mount.after > mount.diff
diff ufw.before ufw.after > ufw.diff
diff login.defs.before login.defs.after > login.defs.diff
diff root.passwd.before root.passwd.after > root.passwd.diff
```

Read each diff. Confirm each change matches what you intended. Anything in a diff you cannot explain is something to investigate; you may have changed something accidentally.

The collection of diff files is your portfolio piece. Together they show, concretely, what changed.

---

## A simple validation script

Write a script that confirms the hardening took. Save as `~/hardening/verify.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "== CIS verification (selected items) =="

check() {
    local label="$1"
    local cmd="$2"
    local expected="$3"
    local actual
    actual=$(eval "$cmd" 2>/dev/null)
    if [[ "$actual" == *"$expected"* ]]; then
        echo "PASS: $label"
    else
        echo "FAIL: $label (got: $actual, expected substring: $expected)"
    fi
}

check "SSH PermitRootLogin no"     "grep -E '^PermitRootLogin' /etc/ssh/sshd_config"  "no"
check "SSH PasswordAuthentication no"  "grep -E '^PasswordAuthentication' /etc/ssh/sshd_config"  "no"
check "SSH MaxAuthTries 4"         "grep -E '^MaxAuthTries' /etc/ssh/sshd_config"     "4"
check "ufw enabled"                "sudo ufw status | head -1"                          "active"
check "ip_forward = 0"             "sysctl -n net.ipv4.ip_forward"                       "0"
check "tcp_syncookies = 1"         "sysctl -n net.ipv4.tcp_syncookies"                   "1"
check "kernel.randomize_va_space = 2" "sysctl -n kernel.randomize_va_space"               "2"
check "/tmp mounted nosuid"        "mount | grep ' /tmp '"                              "nosuid"
check "Root account locked"        "sudo passwd -S root | awk '{print \$2}'"            "L"
check "AppArmor enabled"           "sudo aa-status | head -1"                            "loaded"
check "Persistent journal"         "ls /var/log/journal/ | head -1"                       "$(hostname)"

echo "== End of verification =="
```

Run it:

```
chmod +x ~/hardening/verify.sh
sudo ~/hardening/verify.sh
```

Each PASS line is one item validated. Each FAIL line tells you what is not yet configured as expected. This is the simplest version of "infrastructure-as-code with verification" applied to hardening.

---

## What you did not do

This chapter covered about 20 CIS items. The full CIS Ubuntu 24.04 Benchmark has hundreds. Items intentionally not covered, with brief explanations:

- **Disk encryption (CIS 1.7)**: requires choices at install time. If your lab box was not installed with full-disk encryption, retrofitting is invasive. Mentioned as a real concern; not a chapter exercise.
- **Specific service removal (CIS 2.x)**: removing CUPS (printing), NFS, RPC, etc. depends on whether the box uses them. The principle is "remove what you do not use"; the specifics depend on your lab box.
- **Detailed audit rules (CIS 4.1.3-4.1.16)**: the per-event rules for auditd. Heavy; covered in the intermediate cohort.
- **PAM configuration depth (CIS 5.4.5+)**: PAM rules for account lockout, password reuse, etc. Covered in the intermediate cohort.
- **Banner files (CIS 1.7.x)**: setting MOTD and warning banners. Mostly compliance theater; non-essential for actual security.

The full benchmark is worth reading. Working through it on a real production system is a multi-day project the first time; subsequent systems use automation (Ansible, Salt, configuration management).

---

## Try this

**1. Capture before, harden, capture after, generate diffs.**

This is the chapter's main exercise. Walk through every section, applying the changes. Capture before-state at the start (you should have done this already if you followed along). Capture after-state at the end. Generate the diffs.

The deliverable: a folder containing your before-state files, your after-state files, and the diffs. The diffs are your portfolio piece.

**2. Run the verification script.**

Use the script from the chapter. Run it after applying all the changes. Every line should PASS. If any FAIL, walk back to that section and identify what you missed.

**3. Add your own check.**

The verification script covers about 10 items. Pick two more CIS items from the chapter and add them as `check` lines in the script. Run again. The skill: you can extend the validation as you extend the hardening.

**4. Document any deviations.**

If you chose to skip an item (for example, because your environment requires it different), document why. The format: a markdown file in your hardening folder titled `deviations.md`, with one entry per skipped item:

```
## CIS 5.4.4 (Lock root account)
Skipped because: <reason>
Compensating control: <what you did instead, if anything>
```

Real-world hardening always involves deviations from the standard. The discipline of documenting them is what separates "I followed a checklist" from "I made informed security decisions."

**5. Run it on a clean second box.**

If your lab environment lets you spin up a second VM, do that. Apply the same hardening. The first time was learning; the second time is automation territory. Notice which steps you remember, which you have to look up, which you might want to script.

---

## Common stumbling blocks

> **I locked the root account and now I can't switch user.**
> Locking the root account prevents `su -` to root with a password. You can still run sudo. If you need an interactive root shell, use `sudo -i` instead of `su -`. The lock is intentional and is the right behavior.

> **I disabled PasswordAuthentication and now my second account cannot SSH in.**
> The second account does not have a public key in `authorized_keys`. With password auth disabled, only key-based auth works. Either copy a key for the second account first, or add the second user's name to AllowUsers and ensure their key is set up.

> **`sudo sshd -t` says the config is fine but `systemctl reload ssh` fails.**
> The reload may be blocked by AppArmor or by an issue with environment, not the config syntax. Check the journal: `journalctl -u ssh -n 30`.

> **My ufw rule allows port 80 but I cannot reach the web server externally.**
> Two layers may be in play. The host firewall (ufw) is one layer. On a cloud VM, the cloud's network firewall (security group, network security group) is another. Both must allow the traffic. ufw allowing port 80 only matters if the cloud firewall also does.

> **`aa-enforce` returned errors for some profiles.**
> Some installed profiles are not loadable on your kernel or are out of sync with the application they protect. The errors usually identify which profile and why. You can leave the broken profiles in complain mode or remove them; the rest are still enforced.

> **The verification script fails on `Root account locked` even though I locked it.**
> The output of `passwd -S root` varies slightly between distributions. The script looks for `L` as the second whitespace-separated field. If your output uses `LK` or another notation, adjust the check.

> **My sysctl change doesn't survive a reboot.**
> Changes via `sysctl -w` are runtime only. To persist, the value must be in a file under `/etc/sysctl.conf` or `/etc/sysctl.d/`. The chapter's instructions put them in `/etc/sysctl.d/99-cis-hardening.conf`, which is correct. If a value still does not persist, the value may be overridden by a setting in a higher-priority file; check `/etc/sysctl.d/` for conflicts.

---

## What this gets you

After this chapter:

- Your lab box is materially more resistant to compromise than it was at the start.
- You have produced a documented diff between the default and hardened states. That diff is portfolio-quality work.
- You have walked through about 20 CIS Benchmark items, enough to understand the structure and read the full benchmark on your own.
- You have a verification script that confirms the changes took. The pattern (config + verification) is what real production hardening looks like.
- You know where to go for the rest of the benchmark and which items are out of scope for this chapter.
- You can talk about hardening in the vocabulary the field uses (CIS, AppArmor, sysctl, defense in depth) rather than in vague terms.

The verification script is the part of this chapter that pays off the longest. Anyone can apply settings once. The discipline of writing tests for your security configuration is what separates one-off work from sustainable hardening practice. Working admins should have a verify.sh for every system they own.

---

## What's next

Nothing. This is the last chapter of the unit.

You came in 12 chapters ago with workshop-level Linux skills: SSH, basic navigation, running a service, scheduling a job. You leave able to operate, audit, investigate, and harden a Linux box. That is roughly the skill set of a junior sysadmin or junior security analyst.

Where to go from here:

**Immediate next step:** finish anything in the unit you skipped. The post-workshop chapters are designed to be re-readable. The diff you produced in this chapter is a real artifact; keep it.

**For the security path:** the intermediate cohort starts where this unit ended. Active Directory, then EDR (LimaCharlie), then detection engineering, then incident response, then identity in the cloud. That is the next 10 months.

**For the sysadmin path:** practice. Spin up real boxes. Break them. Fix them. Run a small service for yourself: a personal git server, a home lab, a static site. The boxes you administer in your spare time are the boxes you are best on. Working admins are working admins because they keep working.

**For Security+:** Chapter 9 of the broader course program (the cohort's month 9) is exam-prep. The foundations are now solid. The exam-prep block fills in the test-specific vocabulary and the small set of topics this unit did not cover (governance, risk frameworks, specific compliance details).

You are done with the unit. Take 10 minutes to read your hardening diff once more and notice what you built. Then go do something. The reading is over.
