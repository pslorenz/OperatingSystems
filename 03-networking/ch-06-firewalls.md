# Chapter 6: Firewalls

**You come in with:** packet-level fluency from Ch05. You can read a TCP handshake. You know what a port is. You can identify the source and destination of any packet in a capture.
**You leave with:** hands-on competence with host firewalls (Windows Defender Firewall and Linux ufw/iptables), the ability to read network firewall rule sets (pfSense as the example), and a working understanding of firewall types (packet filter, stateful, NGFW, WAF) for both job and exam purposes.

**Time:** 90 to 120 minutes. The host firewall hands-on takes the most time; reading rules is faster.

**Security+ alignment:** Domain 3.2 (firewall types: WAF, UTM, NGFW, Layer 4/Layer 7). Domain 4.1 (hardening targets: host-based firewall). Domain 4.5 (firewall rules, access lists, ports/protocols, screened subnets). Domain 4.6 (access controls). Heavy alignment; this chapter is where many Sec+ exam-relevant concepts land in their practical form.

---

## Why this chapter shape

You'll touch firewalls constantly in any IT role. Junior practitioners specifically deal with host firewalls: the firewall on the workstation they administer, the firewall on the server they manage. They almost never configure network firewalls; that's senior engineer work. They do read network firewall rule sets to understand traffic flow during investigations.

The chapter shape reflects this. Host firewalls are hands-on: you'll write rules, test them, watch them block and allow. Network firewalls are reading-focused: pfSense as the visual anchor, real-looking rule sets to interpret. Both skills matter; they require different depth.

The Sec+ exam tests firewall types as a recognition skill (which firewall is appropriate for this scenario). The host firewall hands-on covers Domain 4.1 hardening directly. The reading skills support investigation work covered in M7 (SecOps) and M8 (IR).

---

## Firewall types: vocabulary that matters

Before getting hands-on, the conceptual frame. Working admins talk about firewalls in categories; the Sec+ exam tests the categories directly.

### Packet filter (stateless)

The earliest firewall type. Looks at each packet in isolation: source IP, destination IP, source port, destination port, protocol. Compares against a rule list. Allows or drops.

What it can't do: track connection state. A response packet looks like a new packet from the firewall's perspective. To allow responses, you need explicit rules in both directions.

Operates at: Layer 3 and 4.

Examples: simple ACLs on routers, basic iptables configurations without conntrack, AWS Security Groups (in some configurations).

Use today: rare as the only firewall. Sometimes used as a layer of defense in addition to stateful filtering.

### Stateful firewall

Tracks connection state. When a host inside the network initiates an outbound connection, the stateful firewall remembers it and automatically allows the responses. Inbound traffic that doesn't match an established connection is blocked.

This is the foundational improvement over packet filtering. It dramatically simplifies rule writing (you only write rules for connection initiation, not responses) and improves security (unsolicited inbound traffic is blocked by default).

Operates at: Layer 4.

Examples: pfSense in default config, modern iptables with conntrack, Cisco ASA, Windows Defender Firewall.

Use today: this is the baseline. "Firewall" without qualification usually means stateful packet filter.

### Web Application Firewall (WAF)

Operates at Layer 7 specifically for HTTP/HTTPS. Inspects request bodies, URLs, headers. Filters on application semantics: SQL injection patterns, XSS payloads, malformed requests.

Different problem from a stateful firewall: a WAF sits in front of a specific web app and protects it from web-specific attacks. It doesn't generally filter other protocols.

Operates at: Layer 7 (HTTP).

Examples: Cloudflare, AWS WAF, ModSecurity, F5 BIG-IP ASM, Imperva.

Use today: standard for any production web application exposed to the internet.

### Next-Generation Firewall (NGFW)

Combines stateful packet filtering with application-aware inspection. Identifies applications regardless of port (e.g., recognizes Skype traffic on port 80 and treats it differently than HTTP). Often includes integrated IPS, TLS inspection, user identity awareness.

The "next-generation" name is marketing; what makes it different is application identification and Layer 7 inspection on multiple protocols, not just HTTP.

Operates at: Layers 3 through 7.

Examples: Palo Alto Networks, Fortinet FortiGate, Cisco Firepower, Check Point.

Use today: standard at enterprise network perimeters.

### Unified Threat Management (UTM)

A firewall that bundles multiple security functions: stateful filtering, IPS, antivirus, web filtering, VPN, sometimes more. Aimed at small-to-medium businesses that don't want to manage separate appliances.

The lines between UTM and NGFW are blurry. Generally: NGFW is the term for enterprise products with deep application awareness; UTM is the term for SMB products with a broader feature bundle but possibly less depth in any specific area.

Examples: Sophos UTM, Fortinet FortiGate (which is sometimes called UTM, sometimes NGFW), pfSense with package add-ons.

Use today: small business standard.

### How the exam tests these

Sec+ scenarios usually embed the answer in the question:

- "Filter SQL injection attacks against a web application" → WAF.
- "Filter Skype traffic regardless of which port it uses" → NGFW.
- "Default protection on a Windows workstation" → host-based firewall.
- "Block traffic at the network boundary" → stateful (or NGFW for enterprise).

The pattern: read the threat model, identify the layer, pick the firewall type that operates at that layer.

---

## Host firewalls: Windows Defender Firewall

Windows Defender Firewall (WDF) is the host-based firewall built into Windows. It's stateful by default. It's enabled by default. It has three profiles (Domain, Private, Public) that apply different rule sets based on the network type.

### Profiles

When Windows joins a network, it asks (or auto-detects) whether the network is:

- **Domain:** part of an Active Directory domain. Used at work.
- **Private:** trusted (home, small office). More relaxed defaults.
- **Public:** untrusted (coffee shop, airport). More restrictive defaults.

Each profile has its own rules and default behavior. The default rules differ:

- **Domain:** balanced; permits things needed for AD function.
- **Private:** permits file/printer sharing and network discovery.
- **Public:** blocks file/printer sharing and network discovery; only allows essential traffic.

Check the current profile and state:

```powershell
Get-NetFirewallProfile | Format-Table Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

You should see all three profiles listed. Enabled should be True for all of them. DefaultInboundAction is usually Block (drop unsolicited inbound). DefaultOutboundAction is usually Allow (let outbound through).

### Inspecting rules

```powershell
# Get all firewall rules (warning: there are hundreds)
Get-NetFirewallRule | Measure-Object  # see the count

# Get only enabled rules
Get-NetFirewallRule -Enabled True

# Filter to a specific direction
Get-NetFirewallRule -Direction Inbound -Enabled True

# Find a specific rule by name
Get-NetFirewallRule -DisplayName "*remote desktop*"
```

Rules have many properties; the important ones:

- **DisplayName:** human-readable name.
- **Direction:** Inbound or Outbound.
- **Action:** Allow, Block.
- **Enabled:** True/False. Many built-in rules are present but disabled by default.
- **Profile:** Domain, Private, Public, or Any.
- **Group:** which feature/category the rule belongs to. Useful for filtering.

To see the full details of a rule including ports and addresses:

```powershell
$rule = Get-NetFirewallRule -DisplayName "Remote Desktop - User Mode (TCP-In)"
$rule | Get-NetFirewallPortFilter
$rule | Get-NetFirewallAddressFilter
$rule | Get-NetFirewallApplicationFilter
```

The associated filter cmdlets give you the specific match conditions: which ports, which addresses, which applications.

### Writing a rule

Scenario: you've installed a custom service that listens on TCP port 8080. You want to allow inbound connections from the local subnet only.

```powershell
New-NetFirewallRule `
    -DisplayName "Custom Service 8080" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 8080 `
    -RemoteAddress 192.168.50.0/24 `
    -Profile Private,Domain `
    -Enabled True
```

Verify it took:

```powershell
Get-NetFirewallRule -DisplayName "Custom Service 8080"
```

Test that it works: from another machine on 192.168.50.0/24, connect to port 8080 of this machine. Should succeed.

Test that it doesn't allow more than intended: from a machine outside 192.168.50.0/24, connect. Should fail.

### Removing a rule

```powershell
Remove-NetFirewallRule -DisplayName "Custom Service 8080"
```

### Logging blocked traffic

By default, Windows Defender Firewall doesn't log dropped packets. To enable:

```powershell
Set-NetFirewallProfile -Profile Public,Private,Domain `
    -LogBlocked True `
    -LogMaxSizeKilobytes 16384
```

The log file is at `%SystemRoot%\System32\LogFiles\Firewall\pfirewall.log`. Format is fixed-width with a header explaining columns. Useful when troubleshooting "why isn't this connection working."

### The GUI version

`wf.msc` opens the Windows Defender Firewall with Advanced Security GUI. Same rules visible, just clickable. For learning what rules exist and what they do, the GUI is more discoverable than PowerShell. For automation and scripting, PowerShell wins.

---

## Host firewalls: Linux ufw

ufw (Uncomplicated Firewall) is a frontend for iptables/nftables, designed to make basic rule management easy. Common on Ubuntu and Debian.

### Basic ufw operations

```bash
# Check status
sudo ufw status verbose

# Enable ufw (if not already)
sudo ufw enable

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow a service by port
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 80/tcp        # HTTP
sudo ufw allow 443/tcp       # HTTPS

# Allow from specific source
sudo ufw allow from 192.168.50.0/24 to any port 8080

# Allow specific service by name (shortcut for known services)
sudo ufw allow ssh
sudo ufw allow http

# Deny something
sudo ufw deny 25/tcp         # block SMTP

# Delete a rule
sudo ufw delete allow 80/tcp

# Show numbered rules (useful for delete by number)
sudo ufw status numbered
sudo ufw delete 3            # delete rule 3
```

### A typical ufw setup

For a basic web server:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp        # SSH (you'll lock yourself out without this)
sudo ufw allow 80/tcp        # HTTP
sudo ufw allow 443/tcp       # HTTPS
sudo ufw enable
```

The default-deny incoming policy is the right starting place. Then you explicitly allow what you need.

The first rule (allowing SSH) is critical. If you enable ufw without allowing SSH, you immediately disconnect yourself if you're connected via SSH. Always allow SSH first.

### Logging

```bash
sudo ufw logging on        # default level, useful info
sudo ufw logging medium    # more detail
sudo ufw logging high      # high detail (warning: high volume)
```

Logs go to `/var/log/ufw.log` and (depending on distribution) to `/var/log/syslog`.

---

## Host firewalls: Linux iptables (when ufw isn't enough)

ufw is built on top of iptables (or nftables on newer distributions). For complex rules or understanding what ufw is doing, you sometimes drop to iptables directly.

The basics of iptables structure:

- **Chains** are rule lists. The main ones for filtering: INPUT (traffic to your machine), OUTPUT (traffic from your machine), FORWARD (traffic through your machine if it's a router).
- **Tables** are groups of chains for different purposes. The default `filter` table is what you usually care about.
- **Rules** match traffic and specify an action (target).

```bash
# Show the current ruleset (filter table is implicit)
sudo iptables -L -v -n

# Show ufw's translation to iptables
sudo iptables -L -v -n        # See what ufw built

# Add a rule directly (rarely needed if using ufw)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

iptables has a steep learning curve and many subtle behaviors. The takeaway for this chapter: ufw is the right tool for most jobs; iptables is what's underneath; if ufw doesn't do what you need, dropping to iptables is the next step.

Modern distributions are migrating to nftables, which is the successor to iptables. Same conceptual model, different syntax. ufw works on both transparently.

---

## A practical hands-on: harden a Linux host

Take your Linux VM and harden the host firewall step-by-step.

```bash
# Check current state
sudo ufw status

# Set default-deny incoming, default-allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH from your subnet only (not the world)
sudo ufw allow from 192.168.50.0/24 to any port 22 proto tcp

# Enable ufw
sudo ufw enable

# Verify
sudo ufw status verbose
```

Test that SSH still works (connect from a machine in 192.168.50.0/24).

Test that other services are blocked. From another machine:

```bash
nc -vz <linux-vm-ip> 80      # Should fail
nc -vz <linux-vm-ip> 22      # Should succeed (from inside subnet)
```

Now log everything that's denied and check the log:

```bash
sudo ufw logging medium
# Make a few connection attempts that should fail
# From another machine: nc -vz <linux-vm-ip> 80
# Then check the log on the Linux VM:
sudo grep UFW /var/log/syslog | tail -20
```

The log should show the denied attempts with source, destination, port, and the reason for the action.

This is the core host-firewall hardening workflow. Default-deny, allow what's needed, log what's blocked. The same pattern applies to Windows.

---

## A practical hands-on: harden a Windows host

Same logic, different syntax. From elevated PowerShell on the Windows VM:

```powershell
# Check current state
Get-NetFirewallProfile | Format-Table Name, Enabled, DefaultInboundAction, DefaultOutboundAction

# Confirm all profiles are enabled with default-deny inbound
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultInboundAction Block
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Allow

# Enable logging of blocked packets
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -LogBlocked True -LogMaxSizeKilobytes 16384

# Allow RDP from your subnet only (if you use RDP)
New-NetFirewallRule -DisplayName "RDP from local subnet" `
    -Direction Inbound -Action Allow `
    -Protocol TCP -LocalPort 3389 `
    -RemoteAddress 192.168.50.0/24 `
    -Profile Domain,Private -Enabled True

# Disable the default RDP rule that would allow from anywhere
Get-NetFirewallRule -DisplayName "Remote Desktop - User Mode (TCP-In)" |
    Disable-NetFirewallRule

# Verify
Get-NetFirewallProfile | Format-Table Name, Enabled, DefaultInboundAction
Get-NetFirewallRule -DisplayName "RDP from local subnet"
```

Test from another machine that RDP works from inside the subnet, fails from outside.

Check the log:

```powershell
Get-Content "$env:SystemRoot\System32\LogFiles\Firewall\pfirewall.log" -Tail 20
```

---

## Reading network firewalls: pfSense rule sets

You won't configure pfSense from scratch in your first few jobs. You will read existing pfSense rule sets to answer questions like "is the dev environment reachable from the office network" or "why is this user's traffic being blocked." This section gives you the reading skill.

### What a pfSense rule looks like

In the pfSense web UI (Firewall > Rules > LAN), each row is a rule. The columns:

- **Action:** Pass (allow), Block (silently drop), Reject (drop and notify sender).
- **Protocol:** TCP, UDP, ICMP, or any.
- **Source:** the originating address. Could be a single host, a subnet, an alias (named group), `any`, or the special "LAN net" / "WAN net" / "this firewall."
- **Source port:** usually `*` (any). Source ports are typically ephemeral.
- **Destination:** the target address.
- **Destination port:** the port being accessed. Often the meaningful filter.
- **Description:** a free-text label. Good rule sets have meaningful descriptions.

### Reading a typical LAN ruleset

A small business pfSense LAN rule set might look like:

```
1. Pass | TCP/UDP | LAN net | * | This firewall | 53 | DNS to this firewall
2. Pass | TCP    | LAN net | * | any        | 80,443 | Web traffic
3. Pass | TCP    | LAN net | * | any        | 25,587,465 | Email submission
4. Pass | TCP    | LAN net | * | any        | 22 | SSH outbound
5. Block | any   | LAN net | * | RFC1918    | * | Block other RFC1918 traffic
6. Pass | any    | LAN net | * | any        | * | Default LAN allow
```

Reading this:

- Rule 1: LAN clients can use this firewall as their DNS server.
- Rule 2: LAN clients can browse the web.
- Rule 3: LAN clients can submit email through standard ports.
- Rule 4: LAN clients can SSH outbound.
- Rule 5: LAN clients cannot reach other RFC 1918 networks (block lateral movement to other internal nets).
- Rule 6: any other LAN traffic is allowed.

The order matters because pfSense processes rules top-to-bottom and stops at the first match. Rule 5 blocks RFC1918 destinations; rule 6 (default allow) catches everything else. Without rule 5, the default allow would let LAN clients reach other internal networks freely.

### Reading a WAN ruleset

WAN rule sets are typically more restrictive because the WAN side is untrusted:

```
1. Block | any  | RFC1918 sources | * | * | * | Block private addresses from internet (anti-spoofing)
2. Block | any  | bogon list      | * | * | * | Block bogon (unallocated) addresses
3. Pass  | TCP  | any             | * | this firewall | 443 | VPN endpoint
4. Default behavior: block everything else
```

Reading this:

- Rules 1-2: drop traffic from spoofed sources (RFC 1918 ranges, bogon lists).
- Rule 3: allow inbound HTTPS to the firewall itself, presumably for the VPN portal.
- Default: everything else is blocked silently.

### Reading aliases

Real pfSense configurations use aliases to group addresses:

```
Alias: web-servers = 10.10.20.10, 10.10.20.11, 10.10.20.12
Alias: admin-workstations = 192.168.50.50, 192.168.50.51

Rule: Pass | TCP | admin-workstations | * | web-servers | 22 | Admins can SSH to web servers
Rule: Pass | TCP | LAN net | * | web-servers | 80,443 | Anyone on LAN can hit web
```

Aliases make rules readable. When you see an alias, look it up (Firewall > Aliases) to see what it expands to.

### What to look for when reading rules adversarially

When investigating "is this traffic allowed?":

1. Identify the source and destination of the traffic in question.
2. Walk the rule set top-to-bottom looking for the first rule that matches.
3. The action of that first match is what happens.

When investigating "is this rule set well-designed?":

- Are there explicit deny rules where appropriate, or just default-allow at the end?
- Are aliases used to make rules readable?
- Is logging enabled on the rules that matter?
- Are there rules so old nobody knows what they're for? (Common in real environments; these are landmines.)

For learning purposes, get any pfSense rule set you can find (your lab, a friend's homelab, public examples online) and practice walking through them. Predict what would happen for various traffic flows; verify by checking the rule processing logic.

### Logs in pfSense

Status > System Logs > Firewall shows what pfSense has been doing. Each log entry includes:

- Timestamp.
- Interface where the traffic was seen.
- Action (Pass/Block).
- Source IP and port.
- Destination IP and port.
- Protocol.

Enabling logging on specific rules (the gear icon in the rule editor > "Log packets matching this rule") writes both passes and blocks to the log. By default, only blocked traffic is logged.

---

## Connecting it back to packet capture

The chapter started with packet capture in your toolkit. Firewalls and packet capture work together:

- A capture upstream of the firewall shows what's being sent.
- A capture downstream shows what got through.
- The difference is what the firewall blocked.

This is how you debug firewall problems. If you suspect a firewall is dropping traffic but the firewall log doesn't show it, capture upstream and downstream simultaneously. The packet count difference tells you what's being silently dropped.

---

## Common stumbling blocks

> **I enabled ufw and now I can't SSH to the server.**
> Default-deny applied to the SSH port. You needed to allow SSH before enabling. Recovery: console access (or out-of-band) to add the rule, or `sudo ufw disable` if you can get console.

> **My Windows firewall rule isn't taking effect.**
> Several common causes: rule profile doesn't match the current network profile (rule is for Domain, current is Public). Rule is disabled. Rule is for Outbound when you needed Inbound. Another rule earlier in the list is blocking before yours allows. Use `Get-NetFirewallProfile` to see current profile, and `Get-NetFirewallRule` to verify rule state.

> **iptables doesn't seem to do anything.**
> Modern Linux distributions use nftables under the hood; iptables is a compatibility layer. Make sure you're looking at the right tool: `sudo nft list ruleset` shows the actual current rules.

> **pfSense rule looks right but traffic is still blocked.**
> Three things to check: rule is enabled, rule is on the right interface, rules above it aren't matching first. The rule list in pfSense has a left-side icon that's clear when the rule is enabled and matches; clicking through tests the assumptions.

> **Reading a complex ruleset makes my eyes glaze over.**
> Real production rule sets can be hundreds of rules. Three tactics: filter to the relevant interface (most rules don't apply to most traffic), look up aliases to understand the names, focus on rules that match the specific traffic you're investigating rather than reading every rule.

> **My firewall log shows traffic I didn't generate.**
> Modern OSes generate background traffic constantly. Network discovery, software updates, telemetry. You'll see a steady stream of denials in any restrictive firewall log. Most of it is noise; the actually interesting events are isolated patterns.

> **A WAF is blocking legitimate traffic.**
> WAFs are inherently noisy because web traffic is messy. False positives are common. Production WAFs need tuning per application. The fix usually involves whitelisting specific patterns or paths that the WAF is incorrectly flagging.

---

## What this gets you

After this chapter:

- You understand the major firewall types (packet filter, stateful, WAF, NGFW, UTM) and which operates at which layer.
- You can configure Windows Defender Firewall via PowerShell: rules, profiles, logging.
- You can configure Linux ufw: default policies, allow/deny by port and source, logging.
- You know enough iptables to understand what ufw is doing under the hood.
- You can read pfSense rule sets and predict what they do for given traffic.
- You can connect packet captures to firewall behavior for debugging.
- You know that host firewalls are the practical control juniors use, while network firewalls are mostly read-only for the audience.

The host firewall hands-on skills are immediately useful in your first IT job. The reading skills set you up for investigation work later in the program.

---

## What's next

Chapter 7 is segmentation and zones. The chapter that explains why firewalls matter at the architectural level: networks aren't flat, traffic has trust boundaries, and segmentation is how you build them. We also cover the wireless content there: SSID separation, guest network isolation, BYOD as segmentation problems.
