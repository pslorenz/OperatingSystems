# Chapter 1: Reading What Your Machine Is Connected To

**You come in with:** workshop-level familiarity with `ipconfig /all` and `ip addr` output. You can read the fields and roughly understand them. You've been told the words "subnet" and "gateway" and they're not totally abstract anymore.
**You leave with:** the ability to read every field of your machine's network configuration on Linux and Windows and explain what it does. The TCP/IP four-layer working model in your head. The OSI seven-layer model recognized as cert vocabulary, with the layer mappings the Security+ exam tests on drilled.

**Time:** 60 to 75 minutes.

**Security+ alignment:** Domain 3.1 (architecture concepts: OSI/TCP-IP layer mapping for understanding where security controls operate). Domain 4.5 (firewall rules: layer awareness for control selection). The OSI layer mapping for protocols and security controls is genuinely tested; this chapter drills it. The four-layer working model itself is foundation, not direct exam content.

---

## Why this chapter matters

When a working network admin sees `ipconfig /all` output, they don't see a wall of fields. They see a story: the machine has this identity, on this network, with this gateway, using these DNS servers, behind this MAC address. Every field has a purpose. The skill is reading that story fluently.

This chapter is the depth pass on what Block 1 of the workshop touched. By the end you'll be able to look at any machine's network configuration and explain it. That fluency is the foundation for everything else in this unit. Chapter 3 (the network diagnostic) reads these fields when troubleshooting. Chapter 5 (packet capture) shows the traffic these fields generate. Chapter 6 (firewalls) writes rules that filter on these fields.

The chapter also introduces the four-layer model that the rest of the unit thinks in. Working network engineers reason in TCP/IP four layers. The seven-layer OSI model is a translation layer for vendor documentation and certification questions; we cover that translation but don't think in it.

---

## Reading the machine's identity

Run this on your Windows VM:

```powershell
Get-NetIPConfiguration -All
```

Or in legacy CMD style:

```
ipconfig /all
```

Run this on your Linux VM:

```
ip addr
ip route
cat /etc/resolv.conf
```

You'll get output that looks intimidating until you know the fields. Walk through them with this chapter open.

### Fields you should be able to explain

**IP address (IPv4 Address, inet, etc.).** *A 32-bit number written as four decimal octets, identifying your machine at Layer 3 of the network stack.* When another machine wants to send your machine a packet, this is the destination address it uses. Common patterns:

- `192.168.x.x` is RFC 1918 private space, used inside organizations behind NAT.
- `10.x.x.x` is also RFC 1918 private space, often used in larger organizations.
- `172.16.x.x` through `172.31.x.x` is the third RFC 1918 range.
- `169.254.x.x` is APIPA (Automatic Private IP Addressing), assigned by Windows or Linux when DHCP fails. **Seeing this address means DHCP didn't work.** First-line diagnosis when networking is broken.

**Subnet mask (in dotted decimal, e.g., 255.255.255.0).** *The pattern that says which bits of the IP address are the network and which are the host.* `255.255.255.0` means "the first 24 bits are the network, the last 8 bits are the host." This is the same information CIDR notation expresses as `/24`. The mask is what your machine uses to decide whether to send a packet directly (destination is in my subnet) or to the gateway (destination is somewhere else).

**Default gateway.** *The router your machine sends packets to when the destination is not in your subnet.* If your subnet is 192.168.50.0/24 and you want to reach 8.8.8.8, your machine sends the packet to the gateway because 8.8.8.8 is not in your subnet. The gateway figures out where to send it next. **The gateway must be in your subnet** or your machine cannot reach it. A misconfigured gateway is one of the most common networking faults.

**MAC address (Physical Address, link/ether).** *A 48-bit hardware address identifying your network interface at Layer 2.* Written as six hex pairs separated by colons or hyphens (e.g., `aa:bb:cc:11:22:33`). This is permanent to the network card (mostly; you can change it in software for testing). When two machines on the same subnet talk, they use MAC addresses; the IP address is for routing across subnets.

**DNS servers.** *The DNS resolvers your machine asks when it needs to translate names to IP addresses.* Usually you have two listed (primary and secondary). Common patterns: your gateway acting as a DNS proxy, or public resolvers like `8.8.8.8` (Google), `1.1.1.1` (Cloudflare), `9.9.9.9` (Quad9). On an enterprise network these point at internal DNS servers that resolve internal names plus forward external queries.

**DHCP server.** *The server that gave your machine its addressing information.* If you're on DHCP, this field shows which server is responsible. Usually the same machine as the gateway in small networks. On an enterprise network, often a dedicated DHCP server or a Windows DC running DHCP.

**Lease obtained / lease expires.** *The time window during which your DHCP lease is valid.* DHCP leases are time-limited; your machine renews periodically. If you're on a network long enough, you'll see the renewal happen automatically. The lease times help diagnose "I can't get on the network" issues.

**Connection-specific DNS suffix.** *The default domain name appended to bare hostnames you query.* If your suffix is `corp.example.com` and you ping `webserver`, your machine actually queries `webserver.corp.example.com`. This is why "ping the server name" sometimes works without typing the full name.

**Interface MTU (Maximum Transmission Unit).** *The largest packet size the interface can send without fragmenting.* Usually 1500 bytes for Ethernet. Sometimes 1492 for PPPoE connections. Sometimes 9000 for jumbo frames on optimized networks. MTU mismatches cause weird intermittent problems where small packets work but large ones don't.

### A worked example

Here's typical Windows 11 output you might see:

```
Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : home.lan
   Description . . . . . . . . . . . : Intel(R) Ethernet Connection
   Physical Address. . . . . . . . . : 00-15-5D-12-34-56
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 192.168.50.142(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : Tuesday, October 14, 2025 8:32:14 AM
   Lease Expires . . . . . . . . . . : Wednesday, October 15, 2025 8:32:14 AM
   Default Gateway . . . . . . . . . : 192.168.50.1
   DHCP Server . . . . . . . . . . . : 192.168.50.1
   DNS Servers . . . . . . . . . . . : 192.168.50.1
                                       1.1.1.1
```

Reading this:

- **The machine is on a home/small office network.** The 192.168.50.0/24 subnet, the home.lan domain suffix, the same IP for gateway+DHCP+DNS all point at a consumer router or a small pfSense.
- **DHCP is working.** The IPv4 address is in the right range, the lease is valid for 24 hours, and we have the lease times.
- **DNS resolution will hit the local router first, then fall back to Cloudflare.** This is a reasonable home setup.
- **MAC address starts with 00-15-5D.** That's a Microsoft Hyper-V virtual MAC, telling us this is a VM (the workshop scenario).

That's the story-reading skill. Every field is a clue.

The Linux equivalent of the same machine:

```
$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:15:5d:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.142/24 brd 192.168.50.255 scope global dynamic eth0
       valid_lft 86400sec preferred_lft 86400sec
    inet6 fe80::215:5dff:fe12:3456/64 scope link
       valid_lft forever preferred_lft forever

$ ip route
default via 192.168.50.1 dev eth0 proto dhcp metric 100
192.168.50.0/24 dev eth0 proto kernel scope link src 192.168.50.142 metric 100

$ cat /etc/resolv.conf
nameserver 192.168.50.1
nameserver 1.1.1.1
search home.lan
```

Same story, different format. The same reading skill applies: link/ether is the MAC, inet is the IP/subnet, default via is the gateway, nameserver is DNS, search is the suffix.

Notice the IPv6 address (`fe80::...`) at scope link. That's a link-local IPv6 address every modern interface gets automatically. We'll cover IPv6 briefly in Chapter 2; for now, recognize that line and don't worry about it.

---

## The TCP/IP four-layer working model

Working network engineers think in four layers. The model is older than OSI, simpler than OSI, and matches how the actual internet protocols are organized. Stevens's book "TCP/IP Illustrated" (the network engineer's bible) uses this model.

### The layers

**Link layer (Layer 1 + 2).** *The physical and electrical reality of moving bits between machines on the same network segment.* Ethernet, WiFi, fiber. MAC addresses live here. Switches operate here. Frames are the unit of data.

**Internet layer (Layer 3).** *Logical addressing and routing across networks.* IP lives here. Routers operate here. Packets are the unit of data. The internet layer doesn't care about the physical medium below it; it just cares about getting a packet from one IP address to another, possibly across many physical hops.

**Transport layer (Layer 4).** *End-to-end communication between specific applications on specific hosts.* TCP and UDP live here. Port numbers are the addressing scheme that says "this connection is for this specific application on this host." Segments (TCP) or datagrams (UDP) are the units of data.

**Application layer (Layer 5+).** *Whatever the actual application protocol is.* HTTP for web, SMTP for mail, SSH for shell, DNS for name resolution. Messages are the unit of data, and they vary by protocol.

### Encapsulation: how data moves down the stack

When your browser fetches a web page, the data flows down through these layers:

1. **Application layer:** the browser builds an HTTP request: `GET / HTTP/1.1\r\nHost: example.com\r\n...`. This is just text.
2. **Transport layer:** TCP wraps the HTTP request in a TCP segment, adding a header with source port (some ephemeral number on your machine), destination port (443 for HTTPS), sequence numbers, and flags.
3. **Internet layer:** IP wraps the TCP segment in an IP packet, adding a header with source IP (your machine), destination IP (example.com's server), TTL, and a checksum.
4. **Link layer:** Ethernet wraps the IP packet in a frame, adding a header with source MAC (your interface) and destination MAC (your gateway, since example.com is not on your subnet).

The frame goes out on the wire. The gateway receives it, removes the link-layer header, looks at the IP packet, makes a routing decision, wraps it in a new link-layer header for the next hop, and sends it on. The IP packet survives end-to-end; the link-layer headers change at each hop.

At the destination, the layers unwrap in reverse. The application gets the original HTTP request.

This is the working model. It explains why MAC addresses change between routers but IP addresses don't. It explains why a firewall can filter on IP and port (Layer 3 + 4) without understanding the application data. It explains why a packet capture shows you Ethernet frames containing IP packets containing TCP segments containing HTTP requests, all nested.

### Where security controls operate

Different security controls operate at different layers, and where a control operates determines what it can see and act on:

- **VLANs** operate at Layer 2 (link). They separate traffic by tagging frames with a VLAN ID. They cannot inspect packet contents.
- **Packet filter firewalls** operate at Layer 3/4 (internet/transport). They filter on IP addresses, ports, and protocol type. They don't track connection state.
- **Stateful firewalls** operate at Layer 4 (transport). They track connection state and can distinguish between a new connection and a response to one already established.
- **Web Application Firewalls (WAFs)** operate at Layer 7 (application). They parse HTTP and can filter on URL paths, request bodies, headers.
- **Next-Generation Firewalls (NGFWs)** combine Layer 3-4 stateful filtering with Layer 7 inspection of specific protocols (TLS, HTTP).

This layer awareness is exactly what Security+ tests on. A question that asks "which firewall type would best filter SQL injection attacks against a web server" is asking which layer the attack lives at (Layer 7) and which control operates there (WAF). Knowing the layer tells you the answer.

---

## OSI as the cert vocabulary translation

The OSI seven-layer model exists, vendor documentation uses it, and the Security+ exam tests on it. But working engineers don't think in OSI; they think in TCP/IP four layers. The skill is the translation.

The seven OSI layers:

| Layer | Name | What it does | TCP/IP equivalent |
|---|---|---|---|
| 7 | Application | Protocol-specific application logic | Application |
| 6 | Presentation | Data formatting, encryption | Application |
| 5 | Session | Session establishment and teardown | Application |
| 4 | Transport | End-to-end communication | Transport |
| 3 | Network | Logical addressing and routing | Internet |
| 2 | Data Link | Frame delivery on the local segment | Link |
| 1 | Physical | The actual electrical/optical signal | Link |

Layers 5 and 6 (Session and Presentation) are largely vestigial in modern networking. Real protocols don't cleanly map to them; HTTP includes session-like behavior in its application logic, TLS encryption isn't really at Layer 6 in any clean way. The Security+ exam will sometimes ask about them anyway because they're in the model, but they don't reflect how actual systems work.

### What the exam actually tests

Sec+ questions involving OSI layers fall into a small set of patterns. Drilling these is the cert-aligned skill:

**Protocol layer mapping.** Protocols you should know the layer for:

| Protocol | Layer |
|---|---|
| Ethernet, MAC addresses, ARP | 2 (Data Link) |
| IP, ICMP | 3 (Network) |
| TCP, UDP | 4 (Transport) |
| HTTP, HTTPS, FTP, SMTP, DNS, SSH | 7 (Application) |
| TLS/SSL | Often called 6, sometimes 5, technically wraps 7 over 4 |

**Firewall and security control layer mapping.** Already covered above; for the exam:

| Control | Layer |
|---|---|
| VLAN | 2 |
| Packet filter | 3/4 |
| Stateful firewall | 4 |
| WAF | 7 |
| NGFW | 3-7 |
| VPN (IPSec) | 3 |
| VPN (SSL/TLS) | 4-7 |

**The exam phrasing.** Questions often embed the answer in a layer reference. "An attacker is exploiting a vulnerability in HTTP request parsing" tells you the answer involves Layer 7. "An attacker is sending malformed IP packets" tells you Layer 3. Read the layer in the question to find the answer.

Drill these mappings until they're automatic. The actual exam questions are usually scenario-based, so you read a scenario, identify the layer, and pick the matching control. The mapping is the muscle memory.

### Why not just teach OSI as the working model

Because it doesn't reflect how the protocols actually work. TCP/IP was designed; OSI was designed by committee. The TCP/IP four-layer model maps cleanly to actual protocol stacks; the OSI seven-layer model has Session and Presentation layers that real protocols don't use cleanly.

The honest framing: OSI is the seven-layer model that certifications and vendor docs speak. TCP/IP is the four-layer model that engineers think in. Both are useful. Today we think in TCP/IP and translate to OSI when documentation or exams require it.

---

## ARP: how Layer 2 and Layer 3 connect

When your machine wants to send a packet to another machine in your subnet, it needs the destination's MAC address (Layer 2) but it has only the destination's IP address (Layer 3). ARP (Address Resolution Protocol) is how it finds the MAC.

The flow:

1. Your machine wants to send to 192.168.50.142.
2. Your machine checks its ARP cache: do I already know the MAC for 192.168.50.142?
3. Cache miss. Your machine broadcasts an ARP request: "Who has 192.168.50.142? Tell 192.168.50.42."
4. Every machine on the subnet receives the broadcast. The one with that IP responds: "192.168.50.142 is at 00:15:5d:12:34:56."
5. Your machine caches the answer and sends the packet.

You can see your machine's ARP cache:

```
# Linux
ip neigh

# Windows PowerShell
Get-NetNeighbor

# Legacy CMD (still used on Windows)
arp -a
```

The output shows IP-to-MAC mappings your machine has learned. Entries time out after a few minutes; persistent traffic to a host keeps the entry warm.

ARP is also a security topic. ARP responses aren't authenticated, so an attacker on your subnet can send fake ARP responses claiming to be the gateway. Your machine caches the wrong MAC and sends gateway traffic to the attacker, who can read or modify it before forwarding. This is ARP spoofing, which underlies on-path (man-in-the-middle) attacks. We don't go deeper on this here; Chapter 5 (packet capture) and Chapter 7 (segmentation) come back to it.

---

## DHCP: how your machine gets its address

Most machines don't have static IP configuration; they get their address from DHCP (Dynamic Host Configuration Protocol).

The DHCP flow:

1. **Discover.** Your machine boots up, sees no IP configuration, broadcasts a DHCP Discover.
2. **Offer.** A DHCP server on the subnet responds with an Offer: here's an IP, mask, gateway, DNS, lease time.
3. **Request.** Your machine accepts the offer with a Request (back to broadcast, in case multiple servers offered).
4. **Acknowledge.** The server confirms with an Acknowledge.

This is the "DORA" sequence (Discover, Offer, Request, Acknowledge). All four messages happen before your machine has a working IP. Halfway through the lease, your machine starts trying to renew (skipping straight to the Request step with the existing server).

DHCP problems show up as APIPA addresses (169.254.x.x) when DORA fails. The diagnosis is: did the Discover broadcast reach a server, did the Offer come back, did the Request and Acknowledge complete. You can see these in a packet capture, which Chapter 5 covers.

DHCP is also why you have a DNS server, gateway, and other configuration items. The DHCP server pushes all of these to your machine in the Offer; your machine just uses what it's told. This is convenient but has a security implication: a rogue DHCP server on your subnet can give you bad gateway and DNS information, redirecting your traffic. Chapter 7 (segmentation) discusses defenses.

---

## A practical exercise

Run the full output of your machine's network configuration on both Linux and Windows. For each field, write a one-sentence explanation in your own words.

Then answer:

1. What is your machine's MAC address? Decode the first three bytes (the OUI, Organizationally Unique Identifier) at https://standards-oui.ieee.org/oui/oui.txt or with `Get-NetAdapter | Format-List MacAddress, NdisPhysicalMedium` plus a search. What manufacturer made your network card?

2. Look up 8.8.8.8 in your ARP cache. Is it there? Why or why not? (Hint: 8.8.8.8 is not in your subnet.)

3. Now ping your gateway. Look at your ARP cache. Is your gateway's MAC there? Why?

4. Force a DHCP renewal and watch the lease times change. On Linux: `sudo dhclient -r eth0 && sudo dhclient eth0`. On Windows: `ipconfig /release && ipconfig /renew`.

5. Ask an LLM: "Why does my ARP cache only show machines on my local subnet?" Verify the answer against this chapter and against `man ip-neighbour` (Linux) or Microsoft Learn (Windows).

The verification step is the discipline. You should be able to articulate the answer yourself after reading the source.

---

## Common stumbling blocks

> **My IP address is 169.254.x.x and I can't reach anything.**
> APIPA. DHCP failed. Walk DHCP diagnosis: is your interface connected, is there a DHCP server on the subnet, is the DHCP server responding? On Linux, check `journalctl -u dhclient` or `journalctl -u systemd-networkd` for errors. On Windows, run `ipconfig /release && ipconfig /renew` and watch for errors.

> **My subnet mask is 255.255.255.255.**
> Something is very wrong. A /32 mask means "only my own IP is in my subnet" which means you can't reach anything else without a gateway, and the gateway can't be in your subnet. Usually a misconfiguration; sometimes a placeholder when an interface is being set up.

> **I have two default gateways listed.**
> Usually a multi-NIC machine with both interfaces configured. Your machine picks one based on metric (lower wins). If both have the same metric, behavior is unpredictable. On a single-network setup, you almost never want two default gateways.

> **My DNS resolution sometimes works and sometimes doesn't.**
> Could be many things. The most common: your machine has multiple DNS servers configured and one is broken. Your machine queries the first; if it doesn't respond fast enough, it falls back to the second. Intermittent DNS problems often resolve to "remove the broken DNS server from the configuration."

> **My MAC address looks weird (`02:` something).**
> Locally administered MAC addresses have the second-from-last bit of the first byte set, which makes them start with `02`, `06`, `0a`, `0e`, or `12`, etc. Common in virtualized environments where the hypervisor generates MACs rather than using the hardware's burned-in address.

> **The OSI model has Session and Presentation layers but my real protocols don't.**
> Correct. They're abstractions that don't map cleanly. Don't try to force every protocol into a clean layer; some functions span multiple layers, some layers are functionally absent. The model is descriptive, not prescriptive.

> **My machine has both IPv4 and IPv6 addresses. Which one does it use?**
> Modern OSes prefer IPv6 when both are available (Happy Eyeballs algorithm). If IPv6 connectivity to the destination fails, the OS falls back to IPv4. For most internet traffic today, this means "uses IPv6 if the destination has it, IPv4 otherwise." We cover IPv6 briefly in Chapter 2.

---

## What this gets you

After this chapter:

- You can read every field of `ipconfig /all` and `ip addr` output and explain what it does.
- You understand the TCP/IP four-layer working model and can trace what happens at each layer when your browser fetches a web page.
- You have the OSI seven-layer model recognized as cert vocabulary and have drilled the layer mappings the Security+ exam tests.
- You know how ARP connects Layer 2 (MAC) and Layer 3 (IP).
- You know how DHCP gives your machine its addressing and what fails when you see APIPA.
- You have the verification habit applied to AI-assisted explanations.

These are the foundations. The rest of the unit builds on this fluency.

---

## What's next

Chapter 2 is subnetting and the IP plan. The waterfall as the working method, with binary fluency built up from the foundation this chapter laid. By the end of Chapter 2 you can design IP plans for small office scenarios and read CIDR notation without slowing down.
