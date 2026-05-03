# Chapter 10: Network Hardening Capstone

**You come in with:** Ch06 (host firewalls), Ch07 (segmentation thinking), Ch08 (VPN protocols), Ch09 (AI as copilot). You can configure host firewalls, you understand why segmentation matters, and you have a verification habit.
**You leave with:** a hardened host firewall on both Linux and Windows, with a documented before-and-after diff. The diff is your portfolio piece, parallel in role to the M1 Ch12 (Linux hardening) and M2 Ch12 (Windows hardening) capstones.

**Time:** 60 to 90 minutes.

**Security+ alignment:** Heavy. Domain 4.1 (hardening techniques: host-based firewall, removal of unnecessary software, secure baselines). Domain 2.5 (mitigation: hardening, configuration enforcement). Domain 3.2 (security zones: applying segmentation principles to host-level firewalls). Domain 5.4 (documenting security configurations).

---

## Why this chapter is the capstone

Twelve weeks into the program, you can operate Linux, operate Windows, and (after this month) reason about networks. The capstone of each unit asks the same question in different shapes: can you take a default-configured system and apply a coherent set of changes that materially improve its security posture, then prove it.

This chapter applies that pattern to network-facing security on hosts. The artifact is a verification script and a documented diff. The skill is the discipline: configuration plus verification, repeatable, with evidence.

Why host firewalls rather than network firewalls: per the unit's framing, juniors don't deploy network firewalls. They configure host firewalls. The capstone targets where the audience will actually do work.

Why CIS as the framing: Center for Internet Security publishes detailed configuration baselines for major platforms. Working from CIS gives you opinionated, defensible configuration choices. You're not making it up; you're applying an industry-recognized baseline.

---

## The shape of the capstone

You'll do three things:

1. **Capture the before-state** of host firewall configuration on your Linux VM and Windows VM.
2. **Apply CIS-aligned hardening** through a documented sequence of changes.
3. **Capture the after-state and write a verification script** that confirms the changes took.

The deliverable is a folder containing before-state files, after-state files, the verification script, and a short writeup explaining what you changed and why.

This is portfolio work. Future-you in an interview can show the verification script and the before/after diff and explain the reasoning. That's a concrete artifact of "I can do hardening work" rather than a vague claim.

---

## Setting up

Create a working directory on each VM:

**On Linux:**

```bash
mkdir -p ~/hardening
cd ~/hardening
```

**On Windows (elevated PowerShell):**

```powershell
mkdir C:\hardening -Force
cd C:\hardening
```

The before-state captures, the configuration changes, and the after-state captures all happen in this directory.

---

## Step 1: Capture the Linux before-state

```bash
cd ~/hardening

# ufw current state
sudo ufw status verbose > ufw.before.txt

# iptables ruleset (what ufw is doing under the hood)
sudo iptables -L -v -n > iptables.before.txt

# Listening ports (what's actually exposed)
sudo ss -tlnp > listening.before.txt

# DNS resolver configuration
cat /etc/resolv.conf > resolv.before.txt

# Active sysctl values relevant to network hardening
sudo sysctl -a 2>/dev/null | grep -E '^(net\.ipv4\.|net\.ipv6\.|kernel\.)' > sysctl.before.txt

# Installed packages (to identify what's there)
dpkg -l | head -50 > packages-head.before.txt

# Services running
systemctl list-units --type=service --state=running > services.before.txt

# SSH configuration
sudo cat /etc/ssh/sshd_config | grep -v '^#' | grep -v '^$' > sshd.before.txt
```

You now have a snapshot of the relevant network-facing state. Every change in the rest of the chapter will be compared to this snapshot.

---

## Step 2: Capture the Windows before-state

```powershell
cd C:\hardening

# Firewall profile state
Get-NetFirewallProfile | Format-List > firewall-profiles.before.txt

# All firewall rules (focus on enabled ones)
Get-NetFirewallRule -Enabled True | Select-Object DisplayName, Direction, Action, Profile, Group |
    Format-Table -AutoSize > firewall-rules.before.txt

# Listening connections
Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess |
    Format-Table -AutoSize > listening.before.txt

# Defender state
Get-MpComputerStatus | Format-List > defender.before.txt

# Network adapters and their IP config
Get-NetIPConfiguration -All | Format-List > netip.before.txt

# Services running
Get-Service | Where-Object Status -eq Running | Select-Object Name, DisplayName, Status |
    Format-Table -AutoSize > services.before.txt

# Network shares (file/printer sharing surfaces)
Get-SmbShare > shares.before.txt
```

Same idea, different platform. The state is captured before any changes.

---

## Step 3: Apply Linux hardening

The CIS Benchmark for Ubuntu has hundreds of items. We're selecting a curated subset focused on network-facing hardening.

### Section 1: Configure ufw with default-deny inbound

The single most impactful host firewall change is default-deny inbound with explicit allows for needed services.

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow only what's needed. For a basic workstation: nothing inbound.
# For a server: allow specific services. Example for SSH-only:
sudo ufw allow from 192.168.50.0/24 to any port 22 proto tcp comment 'SSH from local subnet'

# Enable logging
sudo ufw logging medium

# Enable ufw if not already
sudo ufw enable

# Verify
sudo ufw status verbose
```

Test that legitimate access still works (SSH from your subnet) and that other services are blocked (try connecting on a different port; should fail).

### Section 2: Harden sshd

If SSH is in use:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply these changes:

- `PermitRootLogin no` (don't allow root to log in directly)
- `PasswordAuthentication no` (require key-based auth)
- `PermitEmptyPasswords no` (defense in depth)
- `ClientAliveInterval 300` (kill idle sessions after 5 minutes)
- `ClientAliveCountMax 0` (disconnect immediately when interval expires)
- `MaxAuthTries 3` (reduce brute-force window)
- `Protocol 2` (no SSH protocol 1; mostly default but worth being explicit)

After editing:

```bash
sudo sshd -t  # check for syntax errors
sudo systemctl restart sshd
```

**CRITICAL:** before restarting sshd, verify your authorized_keys is set up if you're disabling password auth. If you disable password auth without keys configured, you lock yourself out.

### Section 3: Apply network sysctl hardening

Add these to a new file `/etc/sysctl.d/99-hardening.conf`:

```
# Disable IP forwarding (we're not a router)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Don't accept ICMP redirects (mitigates routing manipulation attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Don't send ICMP redirects (we're not a router)
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Don't accept source-routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Enable reverse path filtering (anti-spoofing)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Log martian packets (packets with impossible source addresses)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Don't reply to broadcast pings (mitigates Smurf-style amplification)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable SYN cookies (mitigates SYN flood attacks)
net.ipv4.tcp_syncookies = 1
```

Apply with:

```bash
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
```

These settings are safe defaults for most workstations and servers. Some environments need to override specific items (a router needs `ip_forward = 1`, for example), but for a standard host the values above are appropriate.

### Section 4: Disable unnecessary services

```bash
# List services that are running and decide what's needed
systemctl list-units --type=service --state=running

# Common candidates for disabling on a workstation that's not running them:
# (run only the ones that match your situation)
sudo systemctl disable --now avahi-daemon  # mDNS, often unnecessary
sudo systemctl disable --now cups          # printing, if not needed
sudo systemctl disable --now bluetooth     # if no bluetooth needed
```

Be careful with this. Disabling a service that something depends on can break things. The principle is "what can be turned off without affecting your work" rather than "turn everything off."

### Section 5: Configure resolved DNS hardening

If using systemd-resolved (default on modern Ubuntu):

```bash
sudo nano /etc/systemd/resolved.conf
```

Set:

```
DNSSEC=allow-downgrade
DNSOverTLS=opportunistic
DNS=1.1.1.1 1.0.0.1 9.9.9.9
FallbackDNS=8.8.8.8 8.8.4.4
```

This enables DNSSEC validation (allow-downgrade means use it if available) and DNS-over-TLS opportunistically. The DNS servers are public privacy-respecting resolvers.

Restart resolved:

```bash
sudo systemctl restart systemd-resolved
```

Verify:

```bash
resolvectl status
```

You should see DNSSEC and DoT settings reflected.

---

## Step 4: Apply Windows hardening

Same pattern, Windows-shaped.

### Section 1: Confirm Defender Firewall is fully enabled

```powershell
# All profiles enabled, default-deny inbound, default-allow outbound
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -DefaultInboundAction Block -DefaultOutboundAction Allow

# Enable logging of dropped packets
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -LogBlocked True -LogMaxSizeKilobytes 16384

# Disable network discovery rules in Public profile (high-risk environments)
Get-NetFirewallRule -Group "@FirewallAPI.dll,-32752" |
    Where-Object Profile -match "Public" |
    Disable-NetFirewallRule
```

### Section 2: Disable SMBv1

SMBv1 is the deprecated, insecure version of Server Message Block. Vector for WannaCry and many other attacks. Should be off everywhere.

```powershell
# Check current state
Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol

# Disable
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart

# Also disable at the SMB server level
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

### Section 3: Tighten SMB security

For modern SMB:

```powershell
# Require SMB signing (prevents man-in-the-middle on file sharing)
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force
Set-SmbClientConfiguration -RequireSecuritySignature $true -Force

# Disable insecure guest fallback
Set-SmbClientConfiguration -EnableInsecureGuestLogons $false -Force
```

### Section 4: Restrict LLMNR and NBT-NS

LLMNR (Link-Local Multicast Name Resolution) and NBT-NS (NetBIOS over TCP/IP Name Service) are name resolution protocols that can be abused for credential theft. Disable them.

```powershell
# Disable LLMNR via registry
$path = 'HKLM:\Software\Policies\Microsoft\Windows NT\DNSClient'
if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
Set-ItemProperty -Path $path -Name 'EnableMulticast' -Value 0 -Type DWord

# Disable NBT-NS on each interface
$interfaces = Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces' |
    Where-Object PSChildName -ne 'Tcpip_GUID'
foreach ($iface in $interfaces) {
    Set-ItemProperty -Path $iface.PSPath -Name 'NetbiosOptions' -Value 2 -Type DWord
}
```

The interface enumeration filters out the placeholder and applies to actual interfaces.

### Section 5: Disable IPv6 if not in use

This one is contextual. If your environment uses IPv6, leave it alone. If it doesn't, disabling reduces attack surface:

```powershell
# Check whether IPv6 is in use
Get-NetIPAddress -AddressFamily IPv6 | Format-Table InterfaceAlias, IPAddress, AddressState

# To disable on a specific interface (don't run blindly):
# Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6
```

**Do not run this without understanding your environment.** If applications depend on IPv6 (some do), disabling breaks them.

### Section 6: Block outbound RDP and SMB to public networks

Default Windows allows outbound to anywhere. For workstations, you probably don't want RDP or SMB outbound to the internet.

```powershell
# Block outbound SMB to non-private IPs
New-NetFirewallRule -DisplayName "Block outbound SMB to public" `
    -Direction Outbound -Action Block -Protocol TCP -RemotePort 445,139 `
    -RemoteAddress Internet -Profile Public -Enabled True

# Block outbound RDP to non-private IPs
New-NetFirewallRule -DisplayName "Block outbound RDP to public" `
    -Direction Outbound -Action Block -Protocol TCP -RemotePort 3389 `
    -RemoteAddress Internet -Profile Public -Enabled True
```

The `Internet` keyword in PowerShell firewall rules means "not RFC1918 ranges." This is exactly the targeting we want: SMB and RDP to the local network are normal; SMB and RDP to the public internet are almost always wrong.

---

## Step 5: Capture after-state

Same commands as the before-state, with `.after.txt` instead of `.before.txt`. Reuse the commands from Steps 1 and 2.

**Linux:**

```bash
cd ~/hardening
sudo ufw status verbose > ufw.after.txt
sudo iptables -L -v -n > iptables.after.txt
sudo ss -tlnp > listening.after.txt
cat /etc/resolv.conf > resolv.after.txt
sudo sysctl -a 2>/dev/null | grep -E '^(net\.ipv4\.|net\.ipv6\.|kernel\.)' > sysctl.after.txt
systemctl list-units --type=service --state=running > services.after.txt
sudo cat /etc/ssh/sshd_config | grep -v '^#' | grep -v '^$' > sshd.after.txt
```

**Windows:**

```powershell
cd C:\hardening
Get-NetFirewallProfile | Format-List > firewall-profiles.after.txt
Get-NetFirewallRule -Enabled True | Select-Object DisplayName, Direction, Action, Profile, Group |
    Format-Table -AutoSize > firewall-rules.after.txt
Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess |
    Format-Table -AutoSize > listening.after.txt
Get-MpComputerStatus | Format-List > defender.after.txt
Get-NetIPConfiguration -All | Format-List > netip.after.txt
Get-Service | Where-Object Status -eq Running | Select-Object Name, DisplayName, Status |
    Format-Table -AutoSize > services.after.txt
Get-SmbShare > shares.after.txt
```

### Generate diffs

**Linux:**

```bash
diff ufw.before.txt ufw.after.txt > ufw.diff.txt
diff sysctl.before.txt sysctl.after.txt > sysctl.diff.txt
diff sshd.before.txt sshd.after.txt > sshd.diff.txt
diff services.before.txt services.after.txt > services.diff.txt
```

**Windows:**

```powershell
function Diff-File {
    param($Before, $After, $Output)
    Compare-Object (Get-Content $Before) (Get-Content $After) | Out-File $Output
}

Diff-File firewall-profiles.before.txt firewall-profiles.after.txt firewall-profiles.diff.txt
Diff-File firewall-rules.before.txt firewall-rules.after.txt firewall-rules.diff.txt
Diff-File defender.before.txt defender.after.txt defender.diff.txt
Diff-File services.before.txt services.after.txt services.diff.txt
```

Read each diff file. Confirm each change matches what you intended. The collection of diff files is your portfolio piece.

---

## Step 6: Write the verification script

The verification script confirms the hardening took. Run after the configuration; expects all checks to pass.

### Linux verification script

Save as `~/hardening/verify.sh`:

```bash
#!/usr/bin/env bash

PASS=0
FAIL=0

pass() {
    echo "PASS: $1"
    PASS=$((PASS + 1))
}

fail() {
    echo "FAIL: $1"
    FAIL=$((FAIL + 1))
}

echo "== Linux hardening verification =="

# ufw enabled
if sudo ufw status | grep -q "Status: active"; then
    pass "ufw is active"
else
    fail "ufw is not active"
fi

# Default deny inbound
if sudo ufw status verbose | grep -q "Default: deny (incoming)"; then
    pass "ufw default deny incoming"
else
    fail "ufw default not deny incoming"
fi

# Specific sysctl values
check_sysctl() {
    local key=$1
    local expected=$2
    local actual=$(sysctl -n "$key" 2>/dev/null)
    if [ "$actual" = "$expected" ]; then
        pass "$key = $expected"
    else
        fail "$key (got: $actual, expected: $expected)"
    fi
}

check_sysctl "net.ipv4.tcp_syncookies" "1"
check_sysctl "net.ipv4.icmp_echo_ignore_broadcasts" "1"
check_sysctl "net.ipv4.conf.all.accept_redirects" "0"
check_sysctl "net.ipv4.conf.all.accept_source_route" "0"
check_sysctl "net.ipv4.conf.all.rp_filter" "1"
check_sysctl "net.ipv4.conf.all.log_martians" "1"
check_sysctl "net.ipv4.ip_forward" "0"

# SSH config
if grep -q "^PermitRootLogin no" /etc/ssh/sshd_config; then
    pass "SSH PermitRootLogin no"
else
    fail "SSH PermitRootLogin not no"
fi

if grep -q "^PasswordAuthentication no" /etc/ssh/sshd_config; then
    pass "SSH PasswordAuthentication no"
else
    fail "SSH PasswordAuthentication not no"
fi

echo ""
echo "Total: $((PASS + FAIL)). Pass: $PASS. Fail: $FAIL."
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

Make it executable and run:

```bash
chmod +x ~/hardening/verify.sh
sudo ~/hardening/verify.sh
```

### Windows verification script

Save as `C:\hardening\verify.ps1`:

```powershell
#Requires -Version 5.1
#Requires -RunAsAdministrator

$Pass = 0
$Fail = 0

function Check {
    param($Label, $Test, $Expected)
    if ($Test -eq $Expected) {
        Write-Host "PASS: $Label" -ForegroundColor Green
        $script:Pass++
    } else {
        Write-Host "FAIL: $Label (got: $Test, expected: $Expected)" -ForegroundColor Red
        $script:Fail++
    }
}

Write-Host "== Windows hardening verification ==" -ForegroundColor Cyan

# Firewall profiles
Get-NetFirewallProfile | ForEach-Object {
    Check "Firewall $($_.Name) enabled" $_.Enabled $true
    Check "Firewall $($_.Name) default inbound" $_.DefaultInboundAction "Block"
}

# SMBv1 disabled
$smb1 = (Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol).State
Check "SMBv1 disabled (feature state)" $smb1 "Disabled"
Check "SMBv1 server disabled" (Get-SmbServerConfiguration).EnableSMB1Protocol $false

# SMB signing required
Check "SMB server requires signing" `
    (Get-SmbServerConfiguration).RequireSecuritySignature $true
Check "SMB client requires signing" `
    (Get-SmbClientConfiguration).RequireSecuritySignature $true

# LLMNR disabled
$llmnr = (Get-ItemProperty 'HKLM:\Software\Policies\Microsoft\Windows NT\DNSClient' `
    -ErrorAction SilentlyContinue).EnableMulticast
Check "LLMNR disabled" $llmnr 0

# Defender
Check "Defender realtime monitoring on" `
    (Get-MpComputerStatus).RealTimeProtectionEnabled $true

Write-Host ""
Write-Host "Total: $($Pass + $Fail). Pass: $Pass. Fail: $Fail." -ForegroundColor Cyan
if ($Fail -eq 0) { exit 0 } else { exit 1 }
```

Run from elevated PowerShell:

```powershell
C:\hardening\verify.ps1
```

Both scripts produce a clear PASS/FAIL output. Every PASS is one verified change; every FAIL is something you missed or that didn't apply.

---

## Step 7: Document deviations

Real-world hardening always involves deviations from the standard baseline. Document them.

Create `deviations.md` in your hardening directory. Format:

```
## CIS 18.9.85.1 - Disable IPv6
Status: NOT APPLIED
Reason: Production application uses IPv6 for internal service discovery
Compensating control: IPv6 firewall rules applied to restrict to expected sources

## CIS 1.1.1 - Restrict at-job to root
Status: NOT APPLIED
Reason: Lab VM doesn't have at-job installed
Compensating control: N/A; will apply if installed
```

Each entry covers what you didn't do, why, and what makes the residual risk acceptable. This is real-world thinking; perfect compliance is rare in practice.

The discipline of documenting deviations is what separates "I followed a checklist" from "I made informed security decisions."

---

## A practical exercise

The exercise is the chapter itself.

1. Capture before-states on both VMs.
2. Walk through the configuration steps.
3. Capture after-states.
4. Generate diffs.
5. Run the verification scripts.
6. Document any deviations.
7. Run the M2 Ch11 (Windows Artifacts) triage queries against the hardened Windows. Confirm hardening didn't introduce anything that looks like compromise.
8. Run M1 Ch11 (Linux Artifacts) checks against the hardened Linux. Same goal: hardening shouldn't introduce IR-relevant artifacts.

Submit (or just keep) the folder structure:

```
~/hardening/
├── ufw.before.txt
├── ufw.after.txt
├── ufw.diff.txt
├── sysctl.before.txt
├── sysctl.after.txt
├── sysctl.diff.txt
├── ...
├── verify.sh
└── deviations.md

C:\hardening\
├── firewall-profiles.before.txt
├── firewall-profiles.after.txt
├── firewall-profiles.diff.txt
├── ...
├── verify.ps1
└── deviations.md
```

The folder is the portfolio piece. In an interview, you can say "I hardened a default install of Linux and Windows against the CIS Benchmark, with a verification script and documented deviations" and you have an actual artifact to show.

---

## Common stumbling blocks

> **I locked myself out of SSH.**
> Likely caused by enabling ufw or changing sshd_config without verifying access first. Recovery: console access (CourseStack lets you connect via the web console without SSH). Once in, fix the config, restart sshd, restore access. The lesson: always have console access before changing remote-access config.

> **My verification script reports FAIL on something I configured.**
> Most common cause: the configuration didn't apply because of a typo or missing dependency. Read the actual current state (`sudo ufw status`, `Get-NetFirewallProfile`, etc.) and compare to what the script expects. Sometimes the script's expected value is wrong; sometimes the configuration didn't take.

> **A change broke something legitimate.**
> Expected. Hardening always involves trade-offs. Identify what broke, decide whether the broken thing is more important than the hardening, and either roll back the specific change or document the deviation.

> **I can't verify a change because the tool doesn't exist on my system.**
> Some CIS items reference tools that aren't always installed. The right answer is either install the tool (if it should be there) or document that the item doesn't apply (if it shouldn't be).

> **The diff is huge because I changed many things.**
> That's the point. The diff is the evidence. Don't try to make the diff smaller by rolling back changes; document the changes that justify the diff.

> **My verification script passes but I'm not sure I actually configured everything.**
> The script tests what it tests. If you want to test more, add Check lines for the additional items. The script is a starting point, not a complete audit.

---

## What this gets you

After this chapter:

- You have a hardened Linux host and a hardened Windows host with verifiable evidence.
- You have a verification script for each platform that confirms the hardening took.
- You have a documented before/after diff suitable for portfolio use.
- You've practiced the discipline: capture state, change state, capture state, verify, document deviations.
- You've worked CIS-aligned hardening end-to-end on the network-facing surfaces.
- You have a forward path: when you encounter a different system or different scope, you apply the same pattern.

The discipline is the durable piece. Specific configuration items change as standards evolve and as threats shift. The "capture state, change state, verify, document" pattern is what stays useful for a career.

---

## What's next

Nothing. This is the last chapter of the unit.

You came in twelve weeks ago able to drive a Linux box and a Windows box. You leave able to reason about networks, configure host firewalls with confidence, read network firewall rule sets, design segmentation schemes, and use AI tools as a copilot with a verification habit. That's a meaningful skill expansion.

Where to go from here:

- **Immediately:** keep your hardening folder. The verification script is the start of a personal hardening toolkit you'll grow over years.
- **For the program:** M4 starts with Windows Server and Active Directory. The skills you built in this unit (segmentation thinking, host firewall fluency, network diagnosis) all extend into AD environments.
- **For your own learning:** practice on real captures (Wireshark sample captures available on wireshark.org), real CTF-style network puzzles, and your own homelab. The networking skills compound with practice.
- **For Sec+:** the cert-aligned content from this unit (firewall types, segmentation patterns, DMARC/DKIM/SPF, VPN protocols, OSI/TCP-IP layer mappings) plus the M1 and M2 cert-aligned content covers a meaningful fraction of the exam. The exam-specific drilling happens in M9.

Your hardening folder is the portfolio piece. Take 10 minutes to look at the diffs once more and notice what you built. Then go practice.

The reading is over. The work continues.
