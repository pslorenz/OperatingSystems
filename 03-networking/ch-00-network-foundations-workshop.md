# Ch00: Networking Foundations Workshop

**Audience:** You finished M1 (Linux Foundations) and M2 (Windows Workstation). You can navigate either system in its native shell, manage local users and services, read event logs, and investigate processes. What you have not done yet is think carefully about what happens when those systems talk to each other.

**You come in with:** working command-line fluency on both Linux and Windows. The ability to read `ipconfig` and `ip addr` output without panicking, even if you cannot fully explain every field yet.

**You leave with:** the ability to read your machine's network configuration and explain every field. The ability to subnet by hand using the waterfall method. The ability to use a packet capture tool to see what's actually on the wire. The ability to look at a pfSense firewall rule set and explain what it does.

**Time:** 5 hours, three blocks of about 75 minutes plus opening and wrap.

**Security+ alignment:** This chapter has minimal direct exam content because it's an overview. The post-workshop chapters carry the cert weight: Ch04 (DNS attacks, DMARC/DKIM/SPF), Ch06 (firewall types, host-based firewalls), Ch07 (segmentation, screened subnets), Ch08 (VPN tunneling, IPSec). Per the program's stance: we teach the work, and surface where it ties back to the exam. We don't teach to the exam.

---

## Why this month exists

Two months in, you can drive a Linux box and a Windows box from the keyboard. That's a real capability. But every working system you'll touch in your career is connected to other systems. The "operating environment" isn't just one host; it's a host plus everything it talks to.

This month is where networking stops being abstract. By the end of the workshop you can answer "what is my machine connected to and how do I know" in concrete, hands-on terms. By the end of the post-workshop unit you can do the same for a small business network, including reading firewall rules, recognizing common attacks, and understanding how segmentation contains compromise.

Networking is also where AI tools start showing up in working practice. The unit introduces AI as a copilot, with explicit attention to where LLMs help and where they confidently mislead. We start that thread today and anchor it in Ch09.

---

## What we are doing today

Three blocks. Each block has teaching content followed by hands-on lab work. Each block builds on the previous.

**Block 1: IP addressing and subnetting (~75 minutes).** Why does your machine have an IP address. What is a subnet. How do you do the math that turns a network requirement ("we need to support 50 hosts on each of three departments") into addressing decisions. The subnetting waterfall as the working method, with binary fluency built up first so the waterfall makes sense rather than feeling magical.

**Block 2: Routing, DNS, and reading the wire (~75 minutes).** How packets actually move between networks. Why DNS is the answer to half the network problems you'll encounter. Wireshark as the tool that lets you see what's really happening, with a pre-captured trace as the directed exercise.

**Block 3: pfSense as your edge (~75 minutes).** A real router/firewall, configured by you. Initial setup, interfaces, your first firewall rule. You won't deploy pfSense in production any time soon, but reading and reasoning about firewall configurations is a skill juniors absolutely need.

We open with a 15-minute orientation. We close with a 30-minute wrap. Two 15-minute breaks between blocks.

---

## Your lab box

Today's labs run on a CourseStack-hosted environment with three components:

- **Linux VM** (the same Ubuntu Server you used in M1).
- **Windows 11 VM** (the same as M2).
- **pfSense VM** (new today; this is what Block 3 is built around).

The three VMs sit on a small virtual network so they can talk to each other. The pfSense VM acts as the gateway between them and the internet. You'll be able to SSH into the Linux VM, RDP into the Windows VM, and access the pfSense web UI from either.

You connect the same way you have all along: through CourseStack's lab interface in your browser. Your instructor will provide the specific lab URL.

---

## A note on AI tools

You'll see AI tools come up in working IT practice, including networking. Reasonable things to use them for: explaining unfamiliar output you're seeing, drafting a script you then read and own, suggesting commands you verify against documentation. Unreasonable things to use them for: anything you'd be embarrassed to defend in a meeting, anything involving secrets, anything where you can't verify the answer.

The working principle for this month is **guarded believability**. AI will give you confident answers. Some of those answers will be right. Some will be confidently wrong. The discipline that makes AI useful is verification. Every meaningful answer gets cross-checked against a real source before you act on it.

Humans get things wrong too, of course. The difference is most humans know when they're guessing. AI usually doesn't. Which means the verification habit isn't because AI is bad. It's because actions in IT have consequences, and verification is what keeps you from carrying confident wrongness into production.

We'll see this play out concretely in Block 2 when we use AI to explain a packet capture, and again in Block 3 when we use it to explain a firewall rule. The chapter that anchors the AI thread is Ch09.

---

## Block 1: IP addressing and subnetting

### Block 1 learning objectives

By the end of this block, you can:

- Read your machine's IP configuration on Linux and Windows and explain every field.
- Convert between binary and decimal for a single octet without a calculator.
- Use CIDR notation correctly. Read /24 as "the first 24 bits are the network."
- Use the subnetting waterfall to divide a network into subnets.
- Design a basic IP plan for a small office.

### What's actually happening when your machine is "on the network"

Open a terminal on the Linux VM and run:

```
ip addr
ip route
```

Open PowerShell on the Windows VM and run:

```
Get-NetIPAddress
Get-NetRoute
```

Both machines show you something like this: an IPv4 address, a subnet mask (or the equivalent CIDR prefix), a default gateway. That's the three-part answer to "what is my machine doing on the network."

The IP address is your machine's identity at Layer 3. The subnet mask defines which other machines you can reach directly without involving a router. The default gateway is the router you send traffic to when the destination is somewhere your machine can't reach directly.

Worth pausing on the four-layer TCP/IP model that frames the rest of the unit:

- **Link layer:** Ethernet, WiFi, the physical and electrical realities of getting bits from one machine to another in the same room.
- **Internet layer:** IP. Logical addressing. Routing. How a packet finds its way across the world to a specific machine.
- **Transport layer:** TCP and UDP. Reliable byte streams (TCP) or fire-and-forget messages (UDP). Port numbers as the addressing scheme inside a single host.
- **Application layer:** HTTP, DNS, SMTP, SSH. The protocols actual applications speak.

This is the model working network engineers think with. You'll see the seven-layer OSI model in vendor documentation and in the Sec+ exam; we'll cover the translation in Ch01. For now, four layers are enough to navigate the next few hours.

### Binary, decimal, and the math you actually need

An IPv4 address is 32 bits. The dotted notation (192.168.1.10) is shorthand for four 8-bit groups (octets), each represented in decimal. A subnet mask works the same way: 255.255.255.0 is shorthand for "the first 24 bits are 1, the last 8 bits are 0," which is what /24 in CIDR notation means.

To do networking math, you need to be able to translate between binary and decimal for a single octet. The pattern:

```
Bit position:  128  64  32  16   8   4   2   1
Bit value:       1   1   0   0   0   0   0   0  = 192
Bit value:       1   1   1   1   1   1   1   0  = 254
Bit value:       1   0   0   0   0   0   0   0  = 128
Bit value:       0   1   0   1   0   1   0   0  = 84
```

Each bit position represents a power of 2 (128, 64, 32, 16, 8, 4, 2, 1). To convert binary to decimal, add the values of the bits set to 1. To convert decimal to binary, subtract powers of 2 starting from 128 and mark each bit you used.

Practice (in your head, no calculator):

- 192 in binary
- 168 in binary
- 224 in binary
- 11000000 in decimal
- 11111110 in decimal

The answers: 11000000, 10101000, 11100000, 192, 254. If those felt slow, that's fine; speed comes with reps. If they felt impossible, slow down and walk the bit-by-bit subtraction. We're going to use this fluency in five minutes.

Why this matters: a subnet mask in binary is a string of contiguous 1s followed by contiguous 0s. The 1s mark "network bits" and the 0s mark "host bits." When you see /26, that's 26 1s followed by 6 0s, which is 11111111.11111111.11111111.11000000, which is 255.255.255.192. Without binary fluency, /26 is a mystery. With it, /26 tells you the subnet has 64 addresses (2^6), of which 62 are usable hosts (subtracting the network address and broadcast).

### CIDR notation and what it tells you

Classless Inter-Domain Routing notation (CIDR) writes a network as `address/prefix`. Examples:

- **192.168.1.0/24**: 24 network bits. 8 host bits. 256 addresses (2^8). 254 usable hosts. Common LAN size.
- **10.0.0.0/16**: 16 network bits. 16 host bits. 65,536 addresses. Far too big for any one LAN; usually subnetted further.
- **172.16.0.0/12**: 12 network bits. 20 host bits. Million-plus addresses. The whole RFC 1918 172.16.0.0 range.
- **192.168.100.0/26**: 26 network bits. 6 host bits. 64 addresses. 62 usable hosts.

The math you need to do quickly:

- Number of addresses = 2^(host bits)
- Number of usable hosts = 2^(host bits) - 2 (subtracting network address and broadcast)
- Network address = the lowest address in the range (all host bits 0)
- Broadcast address = the highest address in the range (all host bits 1)

For a /26 network at 192.168.100.0, the host bits are the last 6 bits of the last octet. The network is 192.168.100.0, the broadcast is 192.168.100.63 (because 6 bits of 1s in the last octet is 63), the usable hosts are .1 through .62.

### The subnetting waterfall

The waterfall is a structured way to divide a larger network into multiple smaller subnets. It's called a waterfall because each division flows down to the next, and the math becomes mechanical once you internalize the pattern.

Example: divide 192.168.100.0/24 into four /26 networks.

Each /26 has 64 addresses. Four /26s use 4 × 64 = 256 addresses, which is exactly the size of the /24 we started with.

| Subnet | Network | First host | Last host | Broadcast |
|---|---|---|---|---|
| 1 | 192.168.100.0 | 192.168.100.1 | 192.168.100.62 | 192.168.100.63 |
| 2 | 192.168.100.64 | 192.168.100.65 | 192.168.100.126 | 192.168.100.127 |
| 3 | 192.168.100.128 | 192.168.100.129 | 192.168.100.190 | 192.168.100.191 |
| 4 | 192.168.100.192 | 192.168.100.193 | 192.168.100.254 | 192.168.100.255 |

The pattern: each subnet starts at a multiple of 64 (the block size for /26). The first host is one more than the network address. The broadcast is one less than the next subnet's network address. The last host is one less than the broadcast.

The waterfall worksheet your instructor hands out walks through this pattern progressively, with you filling in network/first-host/last-host/broadcast for each subnet in sequence. You work it in pairs (Person A and Person B alternating rows) because explaining a step to someone else is one of the fastest ways to internalize it.

We'll do the Class C waterfall (/24 starting point) together, then you'll work through the Class B waterfall (/20 starting point) in pairs as the lab.

### Block 1 lab

In pairs, complete the subnetting waterfall worksheet your instructor provides. Class C (/24) first as the warm-up; Class B (/20) as the main exercise.

After the waterfall, each pair designs an IP plan for this scenario:

> A small office needs three networks: one for workstations (50 hosts expected), one for servers (10 hosts), and one for guest WiFi (up to 30 simultaneous devices). They have the 192.168.50.0/24 range to work with. Design the three subnets. For each, specify the CIDR, the network address, the gateway address (use the first usable host as a convention), and the broadcast.

Submit your designs for instructor review. There are multiple valid answers; we'll discuss the tradeoffs in the debrief.

### Block 1 debrief

The most common stumbling block is forgetting that the network address and broadcast address are not usable for hosts. A /26 has 64 addresses but only 62 usable hosts. This trips up almost everyone the first time.

The second most common stumbling block is the off-by-one when computing the broadcast. The broadcast is one less than the next subnet's network address, not the same as it. /26 starting at 192.168.100.64 has broadcast 192.168.100.127, not 192.168.100.128.

The third is forgetting that /24 is the most common LAN size and 256 addresses divides cleanly: into two /25s, four /26s, eight /27s, sixteen /28s, and so on. The block sizes (128, 64, 32, 16, 8, 4) are powers of 2 and they're worth memorizing.

---

*Break: 15 minutes.*

---

## Block 2: Routing, DNS, and reading the wire

### Block 2 learning objectives

By the end of this block, you can:

- Read a host's route table and explain what each entry does.
- Use `dig` and `Resolve-DnsName` to query DNS records.
- Capture network traffic with Wireshark and read the result.
- Identify a TCP three-way handshake in a packet capture.
- Recognize a TLS handshake at the protocol level.

### How packets actually move

You learned in Block 1 that your machine has an IP address, a subnet mask, and a default gateway. When your machine wants to send a packet, it uses these three to make a routing decision:

1. **Destination is in my subnet.** Send the packet directly to the destination machine via Layer 2 (Ethernet/WiFi). The subnet mask tells me what counts as "in my subnet."
2. **Destination is not in my subnet.** Send the packet to my default gateway. The gateway is responsible for figuring out where to send it next.

That's the whole logic, on a host. Routers have more complex logic (multiple routes, longest-prefix matching, dynamic routing protocols) but a host just has "is it local, or is it the gateway's problem."

Your machine's route table is the data structure that records this decision-making. On Linux:

```
ip route
```

You'll see lines like:

```
default via 192.168.50.1 dev eth0
192.168.50.0/24 dev eth0 proto kernel scope link src 192.168.50.42
```

The first line is the default route: anything not matched by a more specific route goes through 192.168.50.1 (the gateway). The second line is the subnet route: anything in 192.168.50.0/24 goes directly out the eth0 interface.

On Windows:

```powershell
Get-NetRoute -AddressFamily IPv4 | Format-Table -AutoSize
```

You'll see equivalent entries. The DestinationPrefix `0.0.0.0/0` is the default route. The DestinationPrefix matching your local subnet (e.g., `192.168.50.0/24`) is the link-local route.

### When the network breaks: the diagnostic mindset

When something is wrong with the network, the diagnostic pattern walks the layers from bottom to top:

1. **Link/physical:** Is the interface up? Can I ping my own gateway?
2. **Network:** Can I route to a known external address (8.8.8.8)?
3. **Transport:** Can I reach a specific port on the destination?
4. **Application:** Is the service responding correctly?

We won't drill this today. Ch03 walks the pattern in detail. For now, the framing matters: when someone says "the network is broken," you start at the bottom and work up.

### DNS as a service

DNS resolves human-readable names into IP addresses. Without DNS, the internet is a list of IPs you'd have to memorize.

The resolver chain when you type `google.com` into a browser:

1. Your browser asks the OS to resolve `google.com`.
2. The OS checks its local cache. Hit? Return.
3. Miss. The OS sends a query to its configured DNS server (often the router, sometimes 8.8.8.8 or 1.1.1.1).
4. The DNS server checks its cache. Hit? Return.
5. Miss. The DNS server walks the recursive resolution: ask a root server, get pointed to a TLD server, get pointed to the authoritative server for google.com, get the answer.
6. The answer comes back through the chain. Caches at each level remember it for the TTL.

You can do steps 3-5 manually. On Linux:

```
dig google.com
dig google.com MX
dig +trace google.com
```

On Windows:

```powershell
Resolve-DnsName google.com
Resolve-DnsName google.com -Type MX
Resolve-DnsName google.com -Server 8.8.8.8
```

The record types worth knowing today:

- **A:** name to IPv4 address.
- **AAAA:** name to IPv6 address.
- **MX:** mail exchanger (where to send mail for this domain).
- **TXT:** arbitrary text (often used for SPF, DKIM, DMARC; we cover these in Ch04).
- **NS:** which name servers are authoritative for this domain.
- **CNAME:** an alias pointing to another name.

DNS gets its own chapter (Ch04) because it's deep enough to deserve one and because half the "the network is broken" tickets resolve to "DNS isn't doing what it should." For today, the basics: dig and Resolve-DnsName are the tools, and you should be comfortable querying common record types.

### Wireshark and the art of seeing what's really there

A packet capture is a log of every network frame that hit a particular interface. Wireshark is the GUI tool that lets you read packet captures, with protocol parsers that decode raw bytes into "this is a TCP SYN" or "this is a TLS Client Hello."

We're going to look at a pre-captured trace your instructor provides. The trace contains:

- A full TCP three-way handshake (SYN, SYN-ACK, ACK).
- A TLS handshake (Client Hello, Server Hello, Certificate, etc.).
- An HTTP request and response.
- A few normal data packets.

Open the trace in Wireshark. The display shows three panes: packet list at the top (one row per packet), protocol details in the middle (the parsed structure of the selected packet), raw bytes at the bottom (the original on-the-wire representation).

The capture filter vs display filter distinction is the one beginners trip on:

- **Capture filter:** restricts which packets get captured in the first place. Set before capture starts. Uses BPF syntax (`tcp port 443`, `host 8.8.8.8`).
- **Display filter:** restricts which captured packets are shown. Set after capture, applied to the existing capture file. Uses Wireshark's display filter syntax (`tcp.port == 443`, `ip.addr == 8.8.8.8`).

You almost always want display filters. Capture filters are for when you're capturing a lot of traffic and need to keep the file size manageable. For a short capture, capture everything and filter the display.

Apply the display filter `tcp.flags.syn == 1 and tcp.flags.ack == 0` to find every connection initiation in the trace. Then `tcp.flags.syn == 1 and tcp.flags.ack == 1` for the responses. Then click through one specific connection in sequence to see the full three-way handshake.

Apply `tls.handshake.type == 1` to find the TLS Client Hello. Click into the packet and look at the protocol details pane. You'll see the SNI (Server Name Indication) field, which is the hostname the client is trying to reach. You'll see the cipher suites the client offers. None of this is encrypted yet; the encryption starts after the handshake completes.

This is the "see what's actually on the wire" moment. A lot of intuition about networking comes from doing this kind of reading repeatedly. Mastery is not the goal today. Functional literacy is.

### A guarded-believability moment

Ask an LLM to explain what `tcp.flags.syn == 1 and tcp.flags.ack == 0` filters for. The answer you get back will probably be reasonable.

Now ask the LLM to explain what the `Window` field in a TCP header means. The answer might be reasonable. Or it might confuse the receive window (how much data the receiver is willing to accept) with the congestion window (how much the sender thinks the network can handle). Both are real concepts; conflating them is a real mistake.

How do you know which it is? You verify against a real source. The TCP RFC (RFC 9293, the current one) has the answer. Wireshark's own documentation has the answer. The Stevens book has the answer. Cross-check.

This is what guarded believability looks like in practice. Not "don't use AI." Use it; verify what it tells you.

### Block 2 lab

Open the pre-captured trace your instructor provides in Wireshark. Answer:

1. How many distinct connections are in the trace? List the source and destination IP/port pairs.
2. For one specific HTTPS connection, identify the SNI (the hostname the client was trying to reach).
3. For the same connection, identify the cipher suite that was negotiated.
4. Find the HTTP request in the trace (it's in cleartext; not all traffic is encrypted). What URL was requested? What was the response code?
5. Run a brief live capture on your Linux VM while you make a connection to a public website (`curl https://github.com` works). Identify the same things in your own trace: SNI, cipher suite, response.

Submit your answers for review.

### Block 2 debrief

The most common stumbling block is the SNI vs Host header distinction. SNI is in the TLS Client Hello (Layer 5/6, before encryption starts). Host is in the HTTP request (Layer 7, inside the encrypted tunnel for HTTPS). For HTTP traffic you can read both; for HTTPS you can only read SNI without decrypting.

The second is forgetting that the capture only shows traffic that traversed the capture interface. If your VM has a packet filter doing NAT, you might see the NATed addresses rather than the original ones. We'll see this play out in Block 3.

The third is filter syntax confusion. Wireshark's display filters are different from BPF capture filters, and both differ from tcpdump syntax. The cheat sheet your instructor provides has the common patterns; lean on it.

---

*Break: 15 minutes.*

---

## Block 3: pfSense as your edge

### Block 3 learning objectives

By the end of this block, you can:

- Log into the pfSense web UI and identify the major sections.
- Read a pfSense firewall rule and explain what it does.
- Write a simple firewall rule to allow or block specific traffic.
- Read pfSense logs to confirm a rule is working.

### What pfSense actually is

pfSense is a free, open-source firewall and router based on FreeBSD. It runs the same packet filter (`pf`) that BSD systems use, with a web UI on top for configuration. It's deployed in small businesses, homelabs, and some larger environments as a credible alternative to commercial appliances.

You're not going to deploy pfSense in a job tomorrow. Junior practitioners almost never configure network firewalls; that's senior network engineer work. What juniors do is read existing firewall configurations to understand traffic flow, ask questions about why rules exist, and respond to incidents that involve firewall logs.

The skills we're building today are reading-focused with light hands-on. You'll see what a real firewall configuration looks like, write a rule or two, see them take effect, and read the logs. The configuration syntax matters less than the conceptual fluency.

### Logging in and orienting

The pfSense VM is already configured with a basic LAN/WAN split. The LAN interface (where the Linux and Windows VMs sit) is 192.168.50.0/24. The WAN interface is the upstream connection to the internet.

Open the pfSense web UI from your Windows VM browser at the LAN IP your instructor provides (typically 192.168.50.1). Default credentials are `admin` / `pfsense`. Change the password on first login if prompted.

The major sections you'll use:

- **Status > Dashboard:** the landing page. Shows interface status, system load, recent log entries.
- **Interfaces:** which network interfaces exist and their addressing.
- **Firewall > Rules:** the firewall rule sets, organized by interface.
- **Firewall > NAT:** Network Address Translation rules.
- **Services > DHCP Server:** DHCP server configuration per interface.
- **Status > System Logs > Firewall:** what the firewall has been blocking and allowing.

Spend two minutes clicking around. Don't change anything yet. Notice that the rule sets are per-interface (rules on the LAN interface apply to traffic entering pfSense from the LAN; rules on the WAN interface apply to traffic entering from the WAN).

### Reading a firewall rule

Navigate to Firewall > Rules > LAN. You'll see the default ruleset, which typically includes:

- An "Anti-Lockout Rule" allowing access to the pfSense web UI from the LAN. (Critical: without this, you could lock yourself out by mistake.)
- A "Default allow LAN to any rule" allowing all outbound traffic from the LAN. This is permissive; in production environments you'd narrow it down.

Each rule has these columns:

- **Action:** Pass (allow), Block (silently drop), or Reject (drop and inform sender).
- **Protocol:** TCP, UDP, ICMP, or any.
- **Source:** what address(es) the rule applies to as the originator.
- **Destination:** what address(es) the rule applies to as the target.
- **Port:** the destination port (for TCP/UDP).
- **Description:** a free-text label.

Click on a rule to see its full configuration. Notice the "logging" checkbox; rules that match logged traffic write to the firewall log. Notice the "advanced features" expansion; pfSense has many more options than the basic columns suggest.

### A firewall rule walkthrough together

Let's walk through writing a rule together. Scenario: you want to block the Linux VM from reaching the internet, but allow it to reach the Windows VM.

Step 1: Identify the addresses.
- Linux VM IP: whatever DHCP gave it (find in `ip addr` from the Linux VM).
- Windows VM IP: same way (`ipconfig` from the Windows VM).

Step 2: Build the rule. Firewall > Rules > LAN > Add (the up arrow adds at the top, which matters for rule order; pfSense processes rules top-down and stops at the first match).

- Action: Block
- Interface: LAN
- Address Family: IPv4
- Protocol: any
- Source: Single host or alias, [Linux VM IP]
- Destination: any
- Description: "Block Linux VM from internet"

Step 3: But that would also block the Linux VM from reaching the Windows VM, which we don't want. Add a more specific rule above it:

- Action: Pass
- Source: Single host, [Linux VM IP]
- Destination: Single host, [Windows VM IP]
- Description: "Allow Linux VM to Windows VM"

Step 4: Apply the rules. pfSense doesn't activate changes until you click Apply Changes (top of the page after saving).

Step 5: Test. From the Linux VM, `ping <windows-ip>` should succeed. `ping 8.8.8.8` should fail. From the Windows VM, `ping <linux-ip>` should still succeed (because we only blocked outbound from Linux, not inbound to Linux).

Step 6: Read the logs. Status > System Logs > Firewall. You should see the blocked attempts to 8.8.8.8 with action "block."

### Block 3 lab

Working independently:

1. Read the existing firewall rules on the LAN interface. For each one, write one sentence explaining what it does.
2. Write a rule that blocks the Windows VM from reaching the Linux VM on port 22 (SSH) but allows it to reach the Linux VM on port 80 (HTTP). Test by attempting both connections from the Windows VM.
3. Read the firewall log. Identify the entries from your test in step 2.
4. Use AI to ask "What does the rule I wrote in step 2 actually do?" Compare the AI's answer to the rule's actual behavior. Note any differences.

Submit your work for review.

### Block 3 debrief

The most common stumbling block is rule order. pfSense processes rules top-to-bottom and stops at the first match. If you write a more permissive rule above a more restrictive one, the restrictive rule never fires. Always think about which rule will match first.

The second is forgetting that pfSense rules apply to traffic entering the interface, not leaving it. A rule on the LAN interface applies to traffic from the LAN going out; a rule on the WAN interface applies to traffic from the internet coming in. This trips up people coming from other firewall vendors who use direction-based rule sets.

The third is the difference between Block (silent drop) and Reject (drop and tell the sender). Block is the right default for the WAN-facing rules (you don't want to advertise that you're there); Reject is sometimes the right choice for internal rules (so the sender knows quickly rather than waiting for a timeout).

---

## Wrap

### What you can now do

- Read your machine's IP configuration on Linux and Windows.
- Convert between binary and decimal for a single octet.
- Use the subnetting waterfall to design IP plans.
- Read a host route table.
- Use dig and Resolve-DnsName for basic DNS queries.
- Open a packet capture in Wireshark and identify a TCP handshake, a TLS handshake, and an HTTP exchange.
- Log into pfSense, read existing rules, write a basic rule, and read the firewall log.
- Use AI tools as a copilot with a verification habit.

That's a meaningful skill expansion in five hours. The post-workshop unit goes deeper on each of these.

### What's deferred to the post-workshop unit

Ten chapters, roughly 11 hours of guided content:

- **Ch01:** TCP/IP four-layer model, OSI translation for cert vocabulary, encapsulation in detail.
- **Ch02:** Subnetting deeper, CIDR mastery, IPv6 introduction.
- **Ch03:** When the network breaks: the diagnostic pattern in detail, walking all four layers.
- **Ch04:** DNS as a service, including DMARC/DKIM/SPF and DNS attacks.
- **Ch05:** Packet capture: deeper Wireshark, TLS handshake analysis, retransmission patterns.
- **Ch06:** Host firewalls (Defender Firewall, ufw) hands-on, network firewall rule reading, firewall types.
- **Ch07:** Segmentation and zones, including wireless network design and the screened subnet pattern.
- **Ch08:** VPNs and remote access. WireGuard primary, IPSec at survival depth, why we're moving off SSL VPN.
- **Ch09:** The AI-assisted practitioner. Patterns for using LLMs as a copilot in IT work.
- **Ch10:** Network hardening capstone. CIS-aligned host firewall hardening with a verification script.

### What's next in the program

- **M4:** Windows Server and Active Directory introduction.
- **M5:** Secure administration foundations.
- **M6:** Cloud and identity (Azure flavor). Where RADIUS, 802.1X, and EAP variants live.
- **M7:** SecOps foundations.
- **M8:** Incident response foundations.
- **M9:** Security+ exam prep.
- **M10:** Capstone and certification sit.

You're a third of the way through the program. The skills compound from here.

---

## Common stumbling blocks across the workshop

> **Binary math feels slow.**
> Reps fix this. The fluency that lets you convert 192 to 11000000 in a second comes from doing 50 conversions, not from understanding the algorithm better. Practice on subnetting problems even after the workshop ends.

> **The waterfall feels like magic.**
> If you can do the binary translation, the waterfall is mechanical: identify the block size (2^host bits), increment by that block size for each subnet. The mystery dissolves once binary clicks.

> **Wireshark has too many fields.**
> True. Most of them you'll never use. Focus on the columns shown by default (No, Time, Source, Destination, Protocol, Length, Info) and add others as you need them. The protocol details pane is where the depth lives; you only dig in when you have a specific question.

> **pfSense rule order is confusing.**
> Top-to-bottom, first match wins. When in doubt, click each rule in order and ask "would this match the traffic I'm thinking about." The first one that matches is the one that fires.

> **The AI gave me a confident answer that turned out to be wrong.**
> Welcome to the discipline. The verification habit is what keeps confident wrongness out of production. If you don't have a way to verify, you don't act on the answer.

---

## What's next

For the next session, read Ch01 (Reading what your machine is connected to). It picks up the IP configuration material from Block 1 and goes deeper, including the TCP/IP four-layer model in detail and the OSI translation for cert vocabulary. By the end of Ch01 you'll be able to fully explain every field of `ipconfig /all` and `ip addr` output.

If you want to get ahead, the post-workshop unit is designed to be read in any order after Ch01. Most students do them sequentially. Some skip ahead to Ch04 (DNS) because it's high-value standalone content. Either works.

Skill, not talent. The people who get good at networking are the people who keep practicing on real systems.
