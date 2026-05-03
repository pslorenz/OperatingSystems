# Chapter 7: Segmentation and Zones

**You come in with:** firewall fluency from Ch06. You can write host firewall rules. You can read pfSense rule sets. You understand stateful filtering at Layer 4.
**You leave with:** the architectural thinking behind why networks aren't flat. VLANs at recognition depth. Trust zones and the screened subnet pattern. Wireless network design as a segmentation problem. The skill to look at a small business network diagram and explain what should be segmented from what, and why.

**Time:** 75 to 90 minutes.

**Security+ alignment:** Heavy. Domain 3.2 (architecture: physical isolation, air-gapped, logical segmentation, security zones, screened subnet, jump server, proxy server). Domain 4.1 (hardening targets: switches, wireless devices). Domain 4.5 (network access control). Domain 2.5 (mitigation techniques: segmentation). Wireless content lands here: SSID separation, BYOD considerations, wireless attacks as motivation for segmentation choices.

---

## Why this chapter exists

Firewalls do nothing if your network is flat. A flat network is one where every device can reach every other device with no controls in between. Once an attacker compromises any one device, they can move laterally to anything else. Modern security depends on segmentation: making the network not-flat so the firewall has something to filter between.

This chapter is about the architectural thinking. You won't deploy VLANs in your first job; that's senior network engineer work. But you'll absolutely read network diagrams, understand why segments exist, and reason about whether traffic between two zones should be allowed. That reading-and-reasoning skill is what juniors need.

The chapter also covers wireless. The original M3 trajectory had wireless as a separate concern; on reflection, wireless is a segmentation problem. Guest WiFi exists because guests should be on a separate trust zone. BYOD exists because employee phones shouldn't be on the same network as production servers. Wireless attacks (evil twin, deauth, KRACK) motivate segmentation choices. So we cover wireless here, where it fits.

---

## The flat network problem

Imagine a small business with one network. Everything is on 192.168.1.0/24:

- Workstations
- Servers (file server, mail server, internal web app)
- Printers
- Wireless access points
- The CEO's laptop
- The temp worker's laptop
- A few IoT devices (security camera, smart thermostat)

Every device can reach every other device. The firewall is at the edge between this network and the internet, doing its stateful filtering against external threats. Internal traffic is unfiltered.

Now an attacker compromises the temp worker's laptop. From there, they can:

- Scan the network and discover everything else.
- Try default credentials on the IoT devices (often successful).
- Exploit the printer (printer firmware is full of vulnerabilities).
- Move laterally to the file server, mail server, or any workstation.
- Pivot to the CEO's laptop using whatever credentials they captured.

The firewall at the edge does nothing for any of this. It's filtering external traffic; internal traffic flows freely.

This is why flat networks are bad. The compromise of any single device exposes everything.

---

## Segmentation: making the network not flat

Segmentation breaks the network into zones with controlled communication between them. Each zone is a different subnet, possibly a different VLAN, with firewall rules controlling what crosses the boundary.

The same small business with reasonable segmentation:

- **Workstations:** 192.168.10.0/24, can reach servers on specific ports.
- **Servers:** 192.168.20.0/24, can be reached by workstations, can reach the internet for updates.
- **Printers:** 192.168.30.0/24, can be reached by workstations on print ports, cannot reach the internet (printers shouldn't be making outbound connections).
- **Wireless guest:** 192.168.40.0/24, can reach only the internet, isolated from everything else.
- **IoT:** 192.168.50.0/24, can reach a specific cloud endpoint for the device's service, isolated from everything else.

Now an attacker compromising a temp worker's laptop is contained to the workstation zone. They can reach the servers on the specific ports the workstations are allowed to use, but they can't sweep the IoT devices, can't pivot through the printer, can't reach the wireless guest network.

The compromise still happens. But it's contained. Containment is the point of segmentation.

This is what working network architects design. Junior practitioners read these designs, understand why each zone exists, and execute the operational pieces (configuring host firewalls, monitoring traffic).

---

## VLANs: how segmentation happens at Layer 2

VLAN stands for Virtual LAN. *A VLAN is a logical Layer 2 broadcast domain that's independent of physical wiring.* Multiple VLANs can share a single switch; a switch port is configured to carry one VLAN (or multiple, if trunked).

### The basic idea

Without VLANs: every port on a switch is in the same broadcast domain. All connected devices see each other at Layer 2.

With VLANs: each port is configured to belong to a specific VLAN. Devices in different VLANs cannot reach each other directly at Layer 2; traffic between VLANs must go through a router (or a Layer 3 switch).

This means:

- Workstations on VLAN 10 cannot ARP for servers on VLAN 20. They have to send traffic to their gateway, which routes between VLANs.
- The gateway can apply firewall rules to inter-VLAN traffic.
- Compromised devices can't easily move laterally because Layer 2 boundaries enforce the segmentation.

### VLAN tagging (802.1Q)

When traffic traverses a trunk port (a switch-to-switch or switch-to-router link carrying multiple VLANs), each frame is tagged with a VLAN ID using the 802.1Q standard. The receiving switch reads the tag and forwards the frame to the correct VLAN.

This is mostly invisible to end devices. Your workstation doesn't know it's on VLAN 10; the switch port handles the tagging.

### VLAN hopping

The classic VLAN attack. The attacker sends frames with two 802.1Q tags stacked. The first switch strips the outer tag and forwards the frame; the second switch sees the inner tag and forwards to a different VLAN than the attacker is supposed to access.

This works on misconfigured networks where:

- The native VLAN of the trunk port is the same as a user VLAN.
- The switch doesn't properly handle double-tagged frames.

Modern switches, properly configured, prevent this. The fix: native VLAN should never match a user VLAN, and switches should drop double-tagged frames at access ports.

For Sec+ purposes: know that VLAN hopping is the attack, and that misconfiguration enables it. The exam tests this.

### Where VLANs sit in the layer model

VLANs are Layer 2. They separate broadcast domains and require Layer 3 routing for inter-VLAN traffic. They do not encrypt; if you're on the same VLAN as someone else, you can read their traffic with a packet capture.

This means VLANs are about traffic flow control, not confidentiality. A VLAN protects against casual lateral movement and broadcast leakage; it doesn't protect against an attacker who has access to a switch's management interface or to a port on the VLAN itself.

---

## Trust zones: the conceptual frame

VLANs are mechanism. Trust zones are intent. A trust zone is a group of resources at the same trust level, with controls between zones that match the trust difference.

### Common trust zone patterns

**Internet:** untrusted. Things from the internet are presumed hostile. Firewall blocks everything by default.

**DMZ (Demilitarized Zone) / screened subnet:** semi-trusted, internet-facing. Hosts servers that need to be reachable from the internet (web servers, mail relays). Isolated from the internal network so that compromise of a DMZ host doesn't immediately expose internal resources.

**Internal:** trusted, your organization's standard zone. Workstations, servers, internal services.

**Restricted:** more trusted than internal. Sensitive systems that even most internal users shouldn't reach (financial systems, HR systems, source code repositories). Access requires specific authorization.

**Critical / high-value:** maximum trust. Systems whose compromise would be catastrophic. Sometimes air-gapped (physically disconnected from any network); sometimes accessed through a jump server or PAW (Privileged Access Workstation).

### Screened subnet (DMZ)

The screened subnet pattern is so common it gets its own name in the Sec+ exam. The architecture:

```
Internet → [external firewall] → DMZ → [internal firewall] → Internal network
```

Two firewalls (or one firewall with two interfaces). Hosts in the DMZ are reachable from the internet on specific ports; the internal network is reachable only from the DMZ on specific ports; the internet cannot reach the internal network directly.

If an attacker compromises a DMZ host (a web server, say), they get a foothold but not internal access. They have to break the internal firewall to move further.

The exam uses both terms (DMZ and screened subnet) interchangeably. They mean the same thing.

### Jump servers

A jump server (or jump box, or bastion host) is a hardened server that sits between zones. To access a sensitive zone, you SSH or RDP into the jump server first, then from the jump server into the target.

The point: the jump server is the only gateway, so it's the only thing the firewall has to allow into the sensitive zone. Logging on the jump server captures all access. Hardening the jump server provides a security control point.

Typical pattern:

```
Admin workstation → [firewall] → Jump server → [firewall] → Critical system
```

Without a jump server, every admin workstation needs firewall rules allowing access to every critical system. With a jump server, you only allow admin workstations to reach the jump server, and only the jump server to reach critical systems.

### Air-gapped systems

The extreme end of segmentation. The system is physically disconnected from any network. To get data in or out, you use removable media or manual entry.

Used for:

- Industrial control systems controlling physical processes.
- Classified systems in government and defense.
- Recovery infrastructure that must remain pristine.
- Some financial transaction systems.

Air gaps aren't perfect (Stuxnet famously crossed an air gap via USB), but they're the strongest segmentation available. The trade-off is operational pain: every interaction with the system is manual.

---

## Reading network diagrams

Working network architects draw diagrams showing the trust zones and firewall placements. Junior practitioners read these diagrams.

### A typical small business diagram

```
                              [Internet]
                                  |
                              [Edge firewall (pfSense)]
                                  |
                +-----------------+-----------------+
                |                                   |
            [DMZ VLAN 100]                  [Internal trunk]
            10.10.100.0/24                          |
                |                       +-----------+-----------+
                |                       |                       |
        [Public web server]      [Workstation VLAN 10]   [Server VLAN 20]
                                 192.168.10.0/24         192.168.20.0/24
                                       |                       |
                                       +----[Internal core]----+
                                                    |
                                              [WiFi Guest VLAN 40]
                                              192.168.40.0/24
```

What you read from this:

- The pfSense is the edge between the internet and everything else.
- DMZ is on its own VLAN (100), separate from internal traffic.
- Workstations and servers are on different VLANs, separated by the firewall.
- Guest WiFi is its own VLAN, separate from internal.
- Inter-VLAN traffic must traverse the firewall.

What you'd ask in review:

- What firewall rules control DMZ-to-internal traffic? (Should be very restrictive.)
- What controls guest WiFi from reaching internal? (Should be blocked.)
- What does the internal-to-internet rule look like? (Probably permissive.)
- Is there a path from workstations to the DMZ? (Sometimes yes for management access.)

### Real-world messiness

Production diagrams are messier. They have:

- More VLANs (engineering, finance, HR, lab, etc.).
- More firewalls (perimeter, internal, between sensitive zones).
- More legacy systems on weird VLANs because someone added them years ago.
- Documented and undocumented exceptions.

Reading messy diagrams is a job skill. Practice on whatever diagrams you can find: your homelab if you've built one, public examples online, internal documentation if your job exposes you to it.

---

## Wireless as a segmentation problem

Wireless access points expose your network to anyone in range. Segmentation matters more on wireless than on wired because:

- The physical layer is shared with everyone in radio range.
- Authentication is the only access control.
- Compromised wireless devices have direct access to whatever VLAN they're on.

### SSID design

An SSID (Service Set Identifier) is the name of a wireless network. Modern enterprise wireless typically has multiple SSIDs serving different purposes:

- **Corporate SSID:** for employees on company-managed devices. Often uses 802.1X authentication tying back to AD.
- **Guest SSID:** for visitors. Web-portal authentication or open with a captive portal. Isolated from internal.
- **BYOD SSID:** for employee personal devices. Some access (internet, maybe email) but not full internal access.
- **IoT SSID:** for cameras, badge readers, environmental sensors. Heavily restricted.

Each SSID can map to a different VLAN. The access point tags traffic with the appropriate VLAN ID before sending to the switched network. The firewall sees traffic per-VLAN and enforces the trust boundaries.

This is what lets you have wireless without making the network flat. The SSID determines the VLAN; the VLAN determines what the device can reach.

### Guest network isolation

Guest WiFi should be isolated from internal. Specifically:

- Guests cannot reach internal subnets.
- Guests can reach the internet (that's the whole point of guest WiFi).
- Guest devices cannot reach each other on the guest network (client isolation).

The last point is sometimes overlooked. Without client isolation, guests on the same SSID can see each other's traffic, attempt to attack each other, and run services that affect each other. Modern wireless controllers support client isolation as a configuration option.

### BYOD: the messy middle

BYOD (Bring Your Own Device) means employees use personal devices for work. This creates a segmentation challenge: the device isn't trusted (it's personal, not managed), but it needs more access than a guest.

Common approaches:

**Limited access:** BYOD devices get internet plus a defined set of internal services (maybe email, maybe a published web app). Everything else is blocked.

**Conditional access:** the device must meet certain conditions (recent OS patches, enabled lock screen, anti-malware running) before getting access. This is what Microsoft Intune and similar MDM systems implement.

**Network access control (NAC):** the device authenticates and the network grants access based on identity and posture. Often uses 802.1X with a RADIUS server.

NAC and 802.1X are deeper material that lives in M6 (cloud and identity). For now: recognize that BYOD is a segmentation problem and the answer involves either limiting access, conditional access, or NAC.

### Wireless attacks as motivation

Several wireless attacks make segmentation choices concrete:

**Evil twin.** Attacker stands up a rogue access point with the same SSID as the legitimate corporate WiFi. Users connect to the attacker's AP, which routes traffic through the attacker (capturing credentials, injecting payloads) before forwarding to the real network. Defense: WPA3 with mutual authentication makes this harder; user training to verify they connect to the right network helps; corporate WiFi using 802.1X with certificates that the attacker can't replicate prevents it.

**Deauthentication attack.** Attacker sends spoofed deauth frames to legitimate clients, kicking them off the WiFi. The client reconnects, which gives the attacker an opportunity to capture the WPA handshake or to redirect to an evil twin. WPA3 includes management frame protection that defeats this.

**KRACK (Key Reinstallation Attack).** Vulnerability in WPA2 announced in 2017. Allowed attackers to force key reuse during the four-way handshake, which under some conditions allowed traffic decryption. Patched in modern OSes; the exam still mentions it because it's an example of WPA2 weakness that drove WPA3 adoption.

**WPS attacks.** WiFi Protected Setup was supposed to make joining a network easier; turned out to have weaknesses that let attackers brute-force the PSK. Disable WPS on any AP that has it.

These attacks happen at Layer 2 over the radio. The defenses are mostly protocol-level (WPA3 for crypto, management frame protection, certificate-based auth). Once an attacker is on the wireless network, segmentation determines what they can reach.

### WPA3 briefly

WPA3 is the current Wi-Fi security standard, replacing WPA2. The improvements:

- **Simultaneous Authentication of Equals (SAE):** replaces WPA2's PSK exchange with a stronger protocol resistant to offline attacks.
- **Forward secrecy:** even if the password is later captured, past traffic can't be decrypted.
- **Management frame protection:** required, defeating deauth and similar attacks.
- **Enhanced Open:** for public WiFi, opportunistic encryption even without authentication.

Adoption is increasing but mixed. Most enterprise deployments now support WPA3 alongside WPA2 for compatibility. Pure-WPA3 deployments are still uncommon because of legacy device support.

For Sec+: know what WPA3 is and that it replaces WPA2. The exam tests this.

---

## A practical exercise

Design a segmentation scheme for this scenario:

> A small business has 25 employees. They have:
>
> - 25 workstations (all employees)
> - 3 servers (file server, internal web app, accounting database)
> - 2 printers
> - A wireless network used by employees on their work laptops
> - A guest wireless network for visitors
> - 4 employees who BYOD (use their phones for email)
> - A few IoT devices (security cameras and a thermostat)
>
> Available address space: 10.50.0.0/16. They have a switch that supports VLANs and a pfSense at the edge.

For your design:

1. List the trust zones and which devices belong to each.
2. Assign each zone a VLAN number and a subnet.
3. Specify the firewall rules between zones (in plain English, not pfSense syntax).
4. Identify which zone is the most sensitive and why.
5. Identify any zones that should be completely isolated from each other.

There are multiple valid answers. The point is the reasoning. After you've designed it, ask an LLM to design the same scheme and compare. Notice what the LLM got right, what it missed, and what choices it made differently.

---

## Common stumbling blocks

> **VLAN seems the same as subnet.**
> They're related but distinct. A VLAN is a Layer 2 broadcast domain; a subnet is a Layer 3 address range. In practice they usually map 1:1: each VLAN has one subnet, and vice versa. But conceptually they're different. A VLAN can have no IP traffic at all; a subnet exists at Layer 3 regardless of VLAN.

> **Why do I need both VLANs and firewall rules?**
> VLANs separate broadcast domains and force inter-VLAN traffic to traverse a router. Firewall rules then control what's allowed across that traversal. VLANs alone don't filter; a router with no firewall rules between VLANs allows everything by default. You need both: VLANs to enforce the separation, firewall rules to control what's allowed across.

> **My DMZ has direct access to the internal database server.**
> Then it's not really a DMZ. The whole point is the DMZ is isolated. If a DMZ server needs to reach an internal database, it should go through specific narrow firewall rules (this DMZ host, to this internal host, on this port, that's it), or through a jump server or proxy. A DMZ with broad internal access defeats the purpose.

> **Guest WiFi can reach my network shares.**
> Common misconfiguration. Either guest VLAN doesn't actually exist (guests are on the same VLAN as employees) or firewall rules don't block guest-to-internal. Verify by connecting to guest WiFi and trying to reach an internal resource; if you can, the segmentation isn't working.

> **The diagram I'm reading doesn't show all the connections I'd expect.**
> Diagrams are abstractions. They show logical structure, not every cable. Inter-VLAN routing happens in the firewall or core switch, which the diagram represents as a single block. Detailed wiring diagrams exist for the people who pull cables; logical diagrams are for the people who reason about traffic.

> **Wireless feels separate from the wired network.**
> It's not, in any architectural sense. WiFi is just another way to attach a device to a VLAN. The access point bridges between WiFi (Layer 2 over radio) and Ethernet (Layer 2 over wire). From the network's perspective, a device on WiFi VLAN 10 is functionally on the same network as a wired device on VLAN 10.

> **NAC seems like a lot.**
> It is. Real NAC deployments are months of work. Small businesses often skip NAC entirely and rely on simpler controls (separate SSIDs, port-based VLAN assignment, MAC filtering for IoT). NAC pays off at scale where you have hundreds of devices and dynamic posture matters.

---

## What this gets you

After this chapter:

- You understand why flat networks are a security problem.
- You can read a network diagram and identify the trust zones.
- You know what VLANs are and how they separate broadcast domains.
- You recognize the screened subnet (DMZ) pattern, jump server pattern, and air-gapped system pattern.
- You understand wireless network design as a segmentation problem.
- You recognize the common wireless attacks (evil twin, deauth, KRACK) and what defends against them.
- You know what WPA3 provides over WPA2.
- You can sketch a segmentation scheme for a small business.

The architectural thinking from this chapter is what separates "I configure firewall rules" from "I understand why this network is shaped this way." Junior practitioners benefit enormously from this thinking even when they're not designing networks themselves.

---

## What's next

Chapter 8 is VPNs and remote access. The protocols and patterns for connecting remote workers and remote sites. WireGuard as the modern primary, IPsec at survival depth, and an honest treatment of why SSL VPN keeps showing up in vendor RCE advisories.
