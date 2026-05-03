# Chapter 2: Subnetting and the IP Plan

**You come in with:** workshop-level binary fluency for a single octet. You've used the waterfall once in pairs to divide a /24. You have the rough idea but you don't yet trust yourself to do this without help.
**You leave with:** the ability to subnet any IPv4 network in your head, design IP plans for small business scenarios, and read CIDR notation as fluently as you read decimal. Plus a working understanding of IPv6 addressing format so you're not surprised by `fe80::` addresses on every interface.

**Time:** 90 minutes. Subnetting is mechanical once you internalize it; the time is in the practice.

**Security+ alignment:** Domain 3.2 (security zones, network infrastructure: physical isolation, logical segmentation). The IP planning skill itself isn't directly tested, but it's prerequisite to thinking about segmentation in Chapter 7. CIDR notation appears throughout the exam without explanation; fluency here pays off across the cert.

---

## Why this chapter matters

You can pass Sec+ without designing an IP plan. You cannot work as a network or systems admin without it. Every time someone says "we need a network for the new building" or "the dev environment needs its own subnet" or "let's segment the printers," what they mean concretely is "tell me the CIDR." This chapter is where that capability becomes durable.

The chapter also drills the subnetting math until it's automatic. Working admins do this in their heads constantly: "what's the broadcast for this /27?" "how many usable hosts in a /29?" "is .130 in the same subnet as .126 if we're /25?" These are not exam questions; they're job questions. The exam tests the same fluency in different framing.

We also cover IPv6 briefly. Not because you'll need to design IPv6 plans (you won't, in your first job) but because IPv6 addresses appear on every modern interface and you should recognize what you're seeing.

---

## Binary, properly this time

The workshop introduced binary at the depth needed to use the waterfall. This chapter goes one step deeper: the why and the patterns that make subnetting fast.

### The bit positions matter

An octet has eight bit positions. Each represents a power of 2:

```
Position:    8    7    6    5    4    3    2    1
Bit value: 128   64   32   16    8    4    2    1
```

To convert decimal to binary: subtract powers of 2 starting from 128. Mark each one you used.

Convert 195:
- 195 - 128 = 67. Mark position 8.
- 67 - 64 = 3. Mark position 7.
- 3 - 2 = 1. Mark position 2.
- 1 - 1 = 0. Mark position 1.
- Result: `11000011`.

Convert 240:
- 240 - 128 = 112. Mark position 8.
- 112 - 64 = 48. Mark position 7.
- 48 - 32 = 16. Mark position 6.
- 16 - 16 = 0. Mark position 5.
- Result: `11110000`.

To convert binary to decimal: add the bit values where you have a 1.

Convert `10101010`:
- 128 + 32 + 8 + 2 = 170.

The pattern that speeds this up: a string of 1s followed by 0s sums to a specific value depending on how many 1s. Specifically:

| 1s | Sum | Value (decimal) |
|---|---|---|
| 1 | 128 | 128 |
| 2 | 128+64 | 192 |
| 3 | 128+64+32 | 224 |
| 4 | 128+64+32+16 | 240 |
| 5 | 128+64+32+16+8 | 248 |
| 6 | 128+64+32+16+8+4 | 252 |
| 7 | 128+64+32+16+8+4+2 | 254 |
| 8 | All ones | 255 |

This table is the "subnet mask values" table. Memorize it. When you see `255.255.255.224` you know the last octet is `11100000` (3 ones), which means 3 network bits in the last octet, which means the prefix is /27.

### CIDR to mask in your head

CIDR prefix to subnet mask is a quick conversion once you know the table:

- /24 = 255.255.255.0 (24 bits = 3 full octets of 1s + 0 in the last)
- /25 = 255.255.255.128 (24 + 1 bit in the last octet = 128)
- /26 = 255.255.255.192 (24 + 2 bits = 192)
- /27 = 255.255.255.224 (24 + 3 bits = 224)
- /28 = 255.255.255.240 (24 + 4 bits = 240)
- /29 = 255.255.255.248 (24 + 5 bits = 248)
- /30 = 255.255.255.252 (24 + 6 bits = 252)
- /16 = 255.255.0.0 (2 full octets)
- /20 = 255.255.240.0 (16 + 4 bits in third octet = 240)
- /23 = 255.255.254.0 (16 + 7 bits in third octet = 254)

The pattern: figure out which octet the prefix lands in, then look up the partial-octet value in the table above.

---

## Block sizes: the pattern that matters

The block size of a subnet is how many addresses it contains. For each prefix length:

| Prefix | Block size | Usable hosts |
|---|---|---|
| /24 | 256 | 254 |
| /25 | 128 | 126 |
| /26 | 64 | 62 |
| /27 | 32 | 30 |
| /28 | 16 | 14 |
| /29 | 8 | 6 |
| /30 | 4 | 2 |
| /31 | 2 | 0 (special use) |
| /32 | 1 | 1 (special use) |

Block size is `2^(host bits)`. Usable hosts is block size minus 2 (subtracting the network address and broadcast address). Memorize the block sizes because they're what the waterfall uses to step through subnets.

Block sizes also tell you what subnet a given IP belongs to. If you have 192.168.50.130/27, the block size is 32. The subnets within 192.168.50.0/24 are at .0, .32, .64, .96, .128, .160, .192, .224. So .130 is in the .128 subnet (192.168.50.128/27), which has range .128 to .159 (network .128, broadcast .159, usable .129-.158).

This is the mental math working admins do constantly. Practice on contrived examples until you can do it without thinking. The waterfall worksheets you got in the workshop are good practice; the exercises at the end of this chapter are more.

---

## The subnetting waterfall, properly

The waterfall is a structured process for dividing a network into multiple equal-size subnets. The workshop walked through one division; this section gives you the full method so you can do it for any starting network and any subnet size.

### The method

Given: a starting network (in CIDR) and a target subnet size (also in CIDR, smaller).

1. **Find the block size of the target subnets.** That's `2^(32 - prefix)` for that prefix.
2. **Find the network address of the starting network.** That's the IP with all host bits cleared.
3. **The first subnet starts at the starting network address.** Network = start, broadcast = start + (block size - 1), usable = start+1 to start+(block size - 2).
4. **Each subsequent subnet starts at (previous network + block size).** Continue until you've covered the original starting range.

### A worked example

Divide 192.168.50.0/24 into /27s.

Block size of a /27 is 32. The /24 has 256 addresses. So we get 256/32 = 8 subnets.

| Subnet | Network | First host | Last host | Broadcast |
|---|---|---|---|---|
| 1 | 192.168.50.0 | .1 | .30 | .31 |
| 2 | 192.168.50.32 | .33 | .62 | .63 |
| 3 | 192.168.50.64 | .65 | .94 | .95 |
| 4 | 192.168.50.96 | .97 | .126 | .127 |
| 5 | 192.168.50.128 | .129 | .158 | .159 |
| 6 | 192.168.50.160 | .161 | .190 | .191 |
| 7 | 192.168.50.192 | .193 | .222 | .223 |
| 8 | 192.168.50.224 | .225 | .254 | .255 |

The pattern: each network is at a multiple of 32, the broadcast is one less than the next network, the first host is one more than the network, the last host is one less than the broadcast.

### A trickier example: when the starting prefix isn't a /24

Divide 172.16.32.0/20 into /24s.

The /20 starts at 172.16.32.0. To find the /20's range: a /20 has block size 4096 (2^12, since 32-20=12 host bits). The /20 covers from 172.16.32.0 to 172.16.47.255 (32 + 16 = 48, so the broadcast is 172.16.47.255).

Wait, that math needs unpacking. Let me show the work.

A /20 has 12 host bits. The block size is 2^12 = 4096 addresses. In four-octet form, that's 16 third-octet values × 256 fourth-octet values = 4096. So the /20 starting at 172.16.32.0 covers the third octet from 32 through 47 (16 values), with the full fourth octet for each.

A /24 has 256 addresses (just the fourth octet). So we can fit 4096 / 256 = 16 /24s in the /20.

| /24 | Range |
|---|---|
| 1 | 172.16.32.0/24 (172.16.32.0 - 172.16.32.255) |
| 2 | 172.16.33.0/24 |
| 3 | 172.16.34.0/24 |
| ... | |
| 16 | 172.16.47.0/24 |

Each /24 is a different value of the third octet, fourth octet covers the full 0-255. The waterfall worksheet for Class B-style subnets walks through this kind of division, where the block size affects the third octet rather than the fourth.

### The tricky bits

The boundaries are where mistakes happen. Three to internalize:

**The broadcast is one less than the next network.** Not the same as it. If two subnets are 192.168.50.0/27 and 192.168.50.32/27, the first's broadcast is .31, not .32.

**The block size is a power of 2.** Always. There's no /27.5. Subnets are always powers of 2 in size, which is why you can divide a /24 cleanly into 2 /25s, 4 /26s, 8 /27s, etc., but not into 3 of anything.

**Subnets must be aligned.** A /27 has to start at a multiple of 32. You can't have a /27 starting at 192.168.50.16. The mask itself enforces this; you can compute the network address by ANDing the IP with the mask, and the result will always be aligned.

---

## Designing an IP plan

The waterfall divides a network mechanically. Designing an IP plan is about choosing the right subnets for the requirements.

### The requirements

Real IP plans answer questions like:

- How many hosts in each subnet, with growth headroom?
- Which subnets need to talk to each other? Which don't?
- What's the addressing scheme that makes operational life easy?

You don't always start with full requirements. Sometimes the requirements are vague and you're making engineering decisions. That's normal; document your decisions so future-you can read them.

### A worked design

Scenario: a small office is moving into a new building. Requirements:

- About 30 workstations now, expected to grow to 50 over three years.
- Two servers (file server, print server). Probably won't change.
- A guest WiFi network for visitors. Maybe 20 simultaneous devices.
- A management network for switch and access point configuration. Maybe 10 devices.
- Available address space: 10.10.0.0/16.

Design:

The 10.10.0.0/16 is big (65,536 addresses). We're using a tiny fraction of it. That's fine; private space is cheap, and leaving room is good.

I'll use /24 subnets for workstations, servers, and guest. /28 for management (it's small).

| Network | CIDR | Use | Hosts available |
|---|---|---|---|
| 10.10.10.0/24 | /24 | Workstations | 254 |
| 10.10.20.0/24 | /24 | Servers | 254 |
| 10.10.30.0/24 | /24 | Guest WiFi | 254 |
| 10.10.40.0/28 | /28 | Management | 14 |

Why these choices:

- **/24 for workstations:** 254 hosts is way more than needed. But /24 is the easiest size to work with operationally (subnet math is mental, every address is just the last octet), and we have plenty of room. A /25 would give 126 hosts which is also enough; the simplicity argument usually wins for small offices.
- **/24 for servers:** same logic. Maybe overkill but operationally simple.
- **/24 for guest:** ditto. Plus we want isolation; using a separate /24 makes the firewall rules trivially clear ("source = 10.10.30.0/24 is guest, deny everything except internet").
- **/28 for management:** management traffic is small. A /28 has 14 usable hosts which is enough, and the small size signals "this is restricted."
- **The .10/.20/.30/.40 spacing:** intentional. Leaves room between subnets if requirements grow. We're not packed; we have lots of unused space between, which is fine.
- **Why not start at .0?** Personal preference. Starting subnets at .10 leaves .0 for future expansion or special purposes. Other shops start at .0 happily; either is fine.

The design gets written down as a document the IT team can reference. Future-you, or future-other-admin, needs to be able to answer "which subnet is the printers on" by reading the document.

### What I'd document for that design

A short document with:

- The subnet table above.
- Gateway addresses (typically .1 of each subnet by convention).
- DNS servers used by each subnet.
- DHCP scope ranges (e.g., "workstations: 10.10.10.10 - 10.10.10.200, leaving .1 for gateway, .2-.9 for static, .201-.254 for static future use").
- VLAN IDs if VLANs are used (which they will be; Chapter 7 covers that).
- Notes on why specific decisions were made.

This document is the contract between current-you and future-you. It's also what you hand to a junior admin when they ask "how is our network structured."

---

## CIDR you should be able to read

A working network admin reads CIDR fluently. Some patterns to internalize:

- **/24 = 256 addresses, 254 usable.** The most common LAN size. If someone says "give us a /24," they know what they're getting.
- **/30 = 4 addresses, 2 usable.** Used for point-to-point links between routers. Two addresses for the two routers, plus network and broadcast.
- **/29 = 8 addresses, 6 usable.** Small management networks, small DMZs.
- **/28 = 16 addresses, 14 usable.** Medium management networks.
- **/16 = 65,536 addresses.** A whole "Class B" sized network. Often allocated to organizations and subnetted internally.
- **/8 = 16 million addresses.** A whole "Class A" sized network. Three of these are RFC 1918 (10.0.0.0/8); others are public allocations.

**RFC 1918 ranges (private space).** You should know these cold:

- 10.0.0.0/8 (10.0.0.0 - 10.255.255.255)
- 172.16.0.0/12 (172.16.0.0 - 172.31.255.255)
- 192.168.0.0/16 (192.168.0.0 - 192.168.255.255)

These are the addresses you can use freely inside an organization without needing public registration. They're not routable on the internet; NAT translates them to public addresses for outbound traffic.

**Special-purpose ranges to recognize:**

- 127.0.0.0/8: loopback. 127.0.0.1 is "this machine."
- 169.254.0.0/16: APIPA. Automatic addressing when DHCP fails.
- 224.0.0.0/4: multicast. Special routing.
- 0.0.0.0/0: "everything." Used in routing as the default route.

---

## IPv6, briefly

IPv6 is the long-term future of IP addressing. The internet is migrating slowly. In your first job you'll probably work mostly in IPv4, with IPv6 enabled but not actively configured. This chapter covers enough IPv6 that you recognize what you're seeing.

### The format

IPv6 addresses are 128 bits, written as eight groups of four hex digits separated by colons:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

That's a mouthful. The compression rules:

- **Leading zeros in each group can be omitted.** `0db8` becomes `db8`. So the above can be written `2001:db8:85a3:0:0:8a2e:370:7334`.
- **One run of consecutive zero groups can be replaced with `::`.** Only one such run per address. So the above becomes `2001:db8:85a3::8a2e:370:7334`.

### Address types you'll see

**Link-local:** `fe80::` followed by interface-specific bits. Every modern interface auto-configures one of these. Used for communication on the same link, never routed. You see these in `ip addr` output on Linux and `ipconfig` on Windows.

**Unique local (ULA):** `fc00::/7`, similar to RFC 1918 in IPv4. Private space. Common ranges start with `fd00::`.

**Global unicast:** `2000::/3`, public IPv6 addresses. What you'd get from your ISP if you have IPv6 service.

**Loopback:** `::1`, equivalent to 127.0.0.1.

**Unspecified:** `::`, equivalent to 0.0.0.0. "No address" or "all addresses" depending on context.

### CIDR in IPv6

Same idea as IPv4 but bigger numbers. A /64 is the standard subnet size for an end network. A /48 is common for an organization. A /128 is a single address. The math is the same; the prefix tells you how many bits are network.

### Why IPv6 isn't taking over faster

NAT works. RFC 1918 plus NAT means most organizations have effectively unlimited internal addresses while presenting just a few public ones. The pressure to migrate to IPv6 is real but slow because IPv4-plus-NAT meets most needs.

That said, IPv6 is enabled by default in modern OSes, IoT devices increasingly use it, and cellular networks heavily depend on it. You won't escape IPv6 forever. Recognize what you're seeing today; deeper IPv6 work is for when you encounter it.

---

## A practical exercise

Practice problems. Do these without a calculator. Show your work mentally.

**Subnetting:**

1. Convert 200 to binary in your head.
2. Convert `11010110` to decimal.
3. Given 192.168.10.0/24, divide into 4 equal subnets. List each subnet's network, broadcast, first host, last host.
4. Given 172.20.0.0/22, divide into /24s. How many do you get? List the first three.
5. Given 10.0.16.0/20, what's the network's range (first and last addresses)?
6. Is 192.168.50.130 in the same /27 as 192.168.50.128? Same /26?

**IP plan design:**

7. A small school needs networks for: students (200 devices), teachers (50 devices), administration (15 devices), and a separate IoT network for cameras and access control (40 devices). Available space: 192.168.0.0/16. Design the subnets. Document gateway, DHCP scope (if applicable), and any other notes.

**CIDR fluency:**

8. /22 has how many usable hosts?
9. /29 has how many usable hosts?
10. What's the subnet mask for /23 in dotted decimal?

**IPv6 reading:**

11. Compress this address: `fe80:0000:0000:0000:abcd:1234:5678:9abc`.
12. Expand this address: `2001:db8::1`.
13. What's the address type for `fc00::123`?

The answers are at the end of the chapter. Try them all before checking.

---

## Common stumbling blocks

> **The block size confuses me.**
> Block size is "how big is this subnet." For /27, block size is 32 (2^(32-27) = 2^5 = 32). Each subnet of that size occupies 32 consecutive addresses. The next subnet of the same size starts 32 addresses later. That's the whole pattern.

> **I keep forgetting to subtract 2 for usable hosts.**
> Network address (all host bits 0) and broadcast address (all host bits 1) aren't usable for hosts. Every subnet has both. /24 has 256 addresses, 254 usable. /27 has 32 addresses, 30 usable. The exception is /31 which is special-cased for point-to-point links and uses both addresses.

> **My subnet starts on an odd number.**
> Then it's not a valid subnet. Subnets always start at multiples of their block size. 192.168.50.16/27 is invalid because 16 isn't a multiple of 32. The valid /27s in 192.168.50.0/24 are at .0, .32, .64, .96, .128, .160, .192, .224.

> **CIDR keeps confusing me at the boundary.**
> The "boundary" being the octet boundary. /20, /21, /22, /23 are tricky because they affect the third octet rather than the fourth. The trick: figure out which octet the prefix is in (first three are octets 1, 2, 3, 4 corresponding to /1-/8, /9-/16, /17-/24, /25-/32). Then the partial-octet value uses the table from the binary section.

> **I don't know whether to use /16 or /24 for our office.**
> If you have to ask, use /24. It's the comfortable size for a single LAN. /16 is what a whole organization gets allocated and then subnets internally. For a single network segment, /24 is right unless you have a specific reason for something different.

> **My IPv6 address has a percent sign in it.**
> That's a zone ID, which Windows adds to link-local addresses to specify which interface they belong to. `fe80::1%eth0` means "the address fe80::1 reachable on interface eth0." Necessary because every interface has its own link-local address space; the OS needs to know which interface to use.

---

## What this gets you

After this chapter:

- You can subnet any IPv4 network in your head.
- You can read CIDR fluently and convert to mask in dotted decimal.
- You can design IP plans for small business scenarios with documented decisions.
- You know the RFC 1918 private ranges and special-purpose ranges.
- You have working IPv6 reading skills (you can compress, expand, and identify address types).

The practical IP plan exercise (problem 7 above) is the kind of work juniors do during their first year on the job. If yours feels close to professional quality, you're ready for it.

---

## What's next

Chapter 3 is the network diagnostic pattern. When something is broken, where do you start? The answer is the four-step pattern that walks the layers from the bottom up. By the end of Chapter 3 you have a repeatable diagnostic for "the network isn't working" tickets.

---

## Answers to practice problems

1. 200 = `11001000` (128 + 64 + 8).
2. `11010110` = 214 (128 + 64 + 16 + 4 + 2).
3. /26 each. Networks at .0, .64, .128, .192. Broadcasts at .63, .127, .191, .255. First hosts at .1, .65, .129, .193. Last hosts at .62, .126, .190, .254.
4. /22 has block size 1024 = 4 /24s. The /22 at 172.20.0.0 covers 172.20.0.0 - 172.20.3.255. The four /24s: 172.20.0.0/24, 172.20.1.0/24, 172.20.2.0/24, 172.20.3.0/24.
5. /20 has block size 4096. Starting at 10.0.16.0, the range is 10.0.16.0 - 10.0.31.255.
6. /27 boundary: .128 to .159. So .130 is in 192.168.50.128/27. Yes, same /27. /26 boundary: .128 to .191. Yes, same /26.
7. Multiple valid answers. One reasonable design:
   - Students: 192.168.10.0/24 (254 hosts, room for growth)
   - Teachers: 192.168.20.0/25 (126 hosts, plenty for 50)
   - Administration: 192.168.30.0/27 (30 hosts, fits 15 with room)
   - IoT cameras/access: 192.168.40.0/26 (62 hosts, fits 40)
   - Document gateways at .1 of each subnet, document DHCP scopes, note isolation requirements (IoT shouldn't reach Administration, etc.).
8. /22: 1024 - 2 = 1022 usable hosts.
9. /29: 8 - 2 = 6 usable hosts.
10. /23: 255.255.254.0 (16 + 7 bits in third octet = 254).
11. `fe80::abcd:1234:5678:9abc`.
12. `2001:0db8:0000:0000:0000:0000:0000:0001`.
13. `fc00::/7` is unique local (private). Equivalent to RFC 1918 in IPv4.
