# Chapter 8: VPNs and Remote Access

**You come in with:** segmentation thinking from Ch07. You can read network diagrams and identify trust zones. You understand why segmentation matters.
**You leave with:** working knowledge of VPN protocols. WireGuard as the modern answer for new deployments. IPsec at "don't suck" depth, including the failure modes that come up in production. An understanding of why SSL VPN keeps appearing in major vendor RCE advisories and why most organizations are migrating away from it.

**Time:** 75 to 90 minutes.

**Security+ alignment:** Domain 3.2 (VPN, tunneling, IPsec, TLS, SD-WAN, SASE). Domain 4.1 (cryptographic protocols). The protocol-level VPN content is direct exam material; the practical "how to read VPN logs" content is job material.

---

## What a VPN actually is

A VPN (Virtual Private Network) is a tunneled, encrypted connection between two points. Inside the tunnel, traffic is encrypted; on the wire between, attackers can see only encrypted bytes. The endpoints decrypt and act as if the traffic came from a directly-connected network.

The two main use cases:

**Remote access VPN.** A single user connects to a corporate network from somewhere else. The user's machine becomes part of the corporate network for the duration of the VPN session. Examples: an employee working from home connecting to internal resources.

**Site-to-site VPN.** Two networks are connected as if they were one. Used between branch offices, between an office and a cloud environment, between two organizations sharing infrastructure. Both networks act as if they have a direct link, even though traffic actually traverses the internet.

In both cases, the VPN provides confidentiality (encryption), integrity (authenticated encryption prevents tampering), and authentication (only authorized endpoints can establish the tunnel).

What VPNs don't provide automatically:

- **Authorization:** the VPN gets you onto the network, but doesn't decide what you're allowed to do once there. That's what firewall rules and identity systems are for.
- **Endpoint security:** the device you're connecting from is still as secure or insecure as it was before. A compromised laptop on a VPN gives the attacker direct access to internal resources.
- **Visibility:** VPN traffic is encrypted, so monitoring inside the tunnel requires endpoint-side logging or TLS-inspection at the VPN gateway.

Working admins miss these distinctions sometimes. "We have a VPN" is not the same as "our remote access is secure." VPN is a piece of the puzzle.

---

## WireGuard: the modern answer

WireGuard is a modern VPN protocol designed for simplicity, speed, and security. It runs in the Linux kernel (as well as on most other platforms), uses modern cryptography by default, and has dramatically simpler configuration than previous protocols.

### Why WireGuard

The key advantages over older VPNs:

- **Small codebase.** WireGuard's reference implementation is around 4,000 lines. OpenVPN is around 100,000. Smaller code means fewer bugs and easier audit.
- **Modern cryptography only.** Curve25519, ChaCha20-Poly1305, BLAKE2s, SipHash24, HKDF. No legacy options to misconfigure.
- **Stateless on the wire.** No connection state to track between peers; the protocol is designed so that lost packets and timeout recovery are simple.
- **Roaming.** Endpoints can change IP addresses (cellular handoff, switching networks) without breaking the tunnel.
- **Performance.** In the kernel, on modern hardware, WireGuard saturates 10 Gbps links easily. Very low CPU overhead per byte.

### How a WireGuard config looks

WireGuard uses public-key authentication. Each peer has a key pair; configuration tells each peer about the public keys of the peers it should accept.

A minimal server config (`/etc/wireguard/wg0.conf`):

```
[Interface]
PrivateKey = SERVER_PRIVATE_KEY_BASE64
Address = 10.100.0.1/24
ListenPort = 51820

[Peer]
PublicKey = CLIENT_PUBLIC_KEY_BASE64
AllowedIPs = 10.100.0.2/32
```

A matching client config:

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY_BASE64
Address = 10.100.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY_BASE64
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Reading these:

- The server has a private key and listens on UDP port 51820.
- The server is configured to accept the specific client (identified by its public key) at IP 10.100.0.2.
- The client has a private key and knows the server's public key.
- AllowedIPs on the client (`0.0.0.0/0`) means "send all traffic through the tunnel" (a full-tunnel VPN). Setting it to specific subnets would be a split-tunnel (only that traffic goes through).
- PersistentKeepalive sends a keepalive every 25 seconds to keep NAT mappings alive.

That's it. No certificate authority, no IKE phases, no negotiation parameters to misconfigure. Generate keys, exchange public keys, run.

### Generating keys

```bash
# Generate a private key
wg genkey > server_private.key

# Derive the public key
wg pubkey < server_private.key > server_public.key

# Or in one go
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

Same on every peer. The private key stays on the device that owns it; only public keys get exchanged.

### Setting up a WireGuard tunnel

On the server:

```bash
# Install
sudo apt install wireguard

# Generate keys
cd /etc/wireguard
sudo umask 077  # ensures keys aren't world-readable
sudo wg genkey | sudo tee server_private.key | sudo wg pubkey | sudo tee server_public.key

# Create config (above)
sudo nano /etc/wireguard/wg0.conf

# Bring up the interface
sudo wg-quick up wg0

# Verify
sudo wg show

# Make it persistent across reboots
sudo systemctl enable wg-quick@wg0
```

On the client:

```bash
# Same install steps
sudo apt install wireguard

# Generate keys (same as server, save the public key to share)

# Create the client config

# Bring up
sudo wg-quick up wg0

# Verify
sudo wg show
```

The first time both sides are up, you should be able to ping across the tunnel:

```bash
# From the client
ping 10.100.0.1

# Should respond from the server.
```

### WireGuard in practice

WireGuard is the right answer for new deployments where you control both endpoints. It's especially good for:

- Remote access for technical users (developers, sysadmins).
- Site-to-site connections between cloud environments.
- Mesh networks of small devices.

It's less good for:

- Environments that need certificate-based authentication (WireGuard uses static public keys, not certificates).
- Environments where the VPN must integrate with enterprise identity (no built-in LDAP/AD/SAML).
- Environments running legacy VPN concentrators where adding WireGuard requires new infrastructure.

For the gaps, several products wrap WireGuard with enterprise features: Tailscale, Netbird, Twingate. These provide identity integration, key rotation, and management UIs while using WireGuard as the underlying tunnel protocol.

---

## IPsec: the protocol that won't die

IPsec is a suite of protocols for encrypting and authenticating IP traffic. It's older, more complex, and more entrenched than WireGuard. Most enterprise site-to-site VPNs run IPsec because that's what the firewall vendors implement.

You won't deploy IPsec in your first job, but you will inherit IPsec configurations and you will need to read logs when they break. This section is about survival: enough IPsec to not be confused when it breaks.

### The pieces

IPsec has multiple components that work together:

- **Authentication Header (AH):** authenticates packets but doesn't encrypt. Rarely used alone; mostly historical.
- **Encapsulating Security Payload (ESP):** encrypts and authenticates. The component that does the actual work.
- **Internet Key Exchange (IKE):** negotiates the keys and parameters used by ESP. Two versions: IKEv1 (older, more complex) and IKEv2 (newer, recommended).

When you set up IPsec, you configure IKE first to establish the secure channel, and IKE then negotiates the ESP parameters. This negotiation is what causes most IPsec headaches.

### Tunnel mode vs transport mode

**Tunnel mode:** the original packet is encapsulated inside a new IP packet. Used for site-to-site VPNs and most remote access. The new outer header has the VPN endpoints; the inner header has the original source and destination.

**Transport mode:** only the payload is encrypted; the original IP header remains. Used for host-to-host encryption rather than network-to-network. Less common.

For most working scenarios, tunnel mode is what you'll see.

### Phase 1 and Phase 2

IKE establishes the IPsec tunnel in two phases:

**Phase 1:** establishes the IKE Security Association (SA). This is the secure channel for negotiating the actual data SA. Phase 1 negotiates encryption algorithm, hash algorithm, Diffie-Hellman group, authentication method, and lifetime. Output: a secure channel between the peers that they use for further negotiation.

**Phase 2:** establishes the IPsec SA. Negotiates which traffic is protected (the "interesting traffic" or "selectors"), what encryption is used for ESP, and the data SA lifetime. Output: keys for the actual data tunnel.

Both phases must succeed for the tunnel to come up. They renegotiate periodically (lifetime is configurable).

### Common failure modes

Reading IPsec logs is half the job. The common failures and their signatures:

**Phase 1 mismatch.** The two peers don't agree on Phase 1 parameters. Common causes: different DH groups, different encryption algorithms, different lifetimes. Log entries: "no proposal chosen," "no matching SA," "informational exchange." Fix: get both sides to agree on parameters.

**Phase 2 mismatch.** Phase 1 succeeded but Phase 2 fails. Often a selector mismatch (one side thinks the tunnel covers 10.10.0.0/16, the other thinks 10.10.0.0/24). Log entries: "no proposal chosen" in Phase 2, "selectors do not match." Fix: align the traffic selectors.

**Pre-shared key mismatch.** If using PSK authentication and the keys don't match, Phase 1 fails. Log entries: "authentication failed," "decryption error." Fix: verify the PSK on both sides.

**NAT-T issues.** When IPsec traverses NAT, it needs NAT Traversal (NAT-T) which encapsulates IPsec in UDP/4500 instead of native IP protocol 50 (which NAT can't handle). If one side has NAT-T disabled or the firewall blocks UDP/4500, the tunnel fails over NAT. Log entries: "no NAT-T support," "received packet from unexpected source." Fix: enable NAT-T on both sides; allow UDP/4500.

**DPD timeouts.** Dead Peer Detection sends periodic keepalives. If they're not received, the tunnel is torn down. If your tunnel keeps dying, check DPD timer settings. Log entries: "DPD timeout," "peer not responding." Fix: adjust DPD timers, or check for actual connectivity loss.

**Phase 2 lifetime mismatch.** The two sides have different lifetime values. The tunnel comes up, runs for a while, then one side tries to rekey while the other isn't ready. Sometimes the tunnel renegotiates fine; sometimes it stays broken. Log entries: "tunnel rekey failed," "old SA torn down before new SA established." Fix: align lifetimes; the lower value wins so it's safer to set both sides to the same lower number.

### Reading an IPsec log

Example log from a strongSwan-based system trying to bring up a tunnel:

```
charon: 09[IKE] received unsupported DH_GROUP MODP_2048; expected MODP_1536
charon: 09[IKE] no proposal chosen
charon: 09[IKE] received NO_PROPOSAL_CHOSEN error notify
```

Reading this: the local side proposed DH group MODP_1536, the remote side proposed (or expected) MODP_2048. They don't match; no proposal chosen; tunnel fails. Fix: align DH groups in the IKE configuration.

Another:

```
charon: 13[IKE] establishing CHILD_SA tun{1}
charon: 13[IKE] generating CREATE_CHILD_SA request 5
charon: 13[NET] sending packet: from 192.0.2.10[4500] to 192.0.2.20[4500]
charon: 13[IKE] received SELECTORS_DO_NOT_MATCH error
```

Reading this: Phase 1 succeeded (we're on a CREATE_CHILD_SA which is Phase 2), but selectors don't match. Local thinks the tunnel covers different subnets than remote. Fix: align the traffic selectors.

The vocabulary of IPsec logs is unfriendly. Working through real logs with the protocol concepts in mind is how you get fluent.

### When IPsec is the right answer

For new deployments where you control both ends, WireGuard is usually better. IPsec is the right answer when:

- You have to interoperate with existing IPsec gear (most enterprise firewalls).
- You need certificate-based authentication tied to a CA.
- You need standards compliance for regulatory reasons (some industries require IPsec).
- You're running site-to-site connections where the firewall vendor's support matters.

In practice, you'll inherit IPsec deployments more often than you'll choose them. Knowing how to read the logs and fix the common failures is the working skill.

---

## SSL VPN: why it keeps showing up in CVE advisories

SSL VPN (also called TLS VPN) is a category of VPNs that run over TLS, typically port 443. The appeal: TLS traffic is allowed through almost any firewall, so users on restrictive networks (hotel WiFi, public WiFi, restrictive corporate guest networks) can still connect.

The major SSL VPN products: Cisco AnyConnect, Pulse Secure (now Ivanti Connect Secure), Palo Alto GlobalProtect, Fortinet FortiClient SSL VPN, SonicWall NetExtender, Citrix Gateway, OpenVPN.

### The problem

Every major SSL VPN vendor has had multiple critical RCE (Remote Code Execution) vulnerabilities in recent years. A non-exhaustive list:

- **Pulse Secure / Ivanti Connect Secure:** CVE-2019-11510 (path traversal, exploited massively), CVE-2023-46805 + CVE-2024-21887 (chained authentication bypass + RCE, exploited in the wild), CVE-2024-22024 (XXE).
- **Fortinet FortiOS SSL VPN:** CVE-2018-13379 (path traversal), CVE-2022-42475 (RCE), CVE-2023-27997 (heap overflow in SSL VPN), CVE-2024-21762 (out-of-bounds write).
- **SonicWall:** CVE-2021-20016 (SQL injection), CVE-2024-40766 (improper access control with active exploitation).
- **Citrix:** CVE-2019-19781 (path traversal, exploited massively as "Shitrix"), CVE-2023-3519 (RCE), CVE-2023-4966 (Citrix Bleed, session hijacking, exploited in major breaches).
- **Cisco:** CVE-2020-3452, CVE-2023-20269, CVE-2024-20337.

These aren't isolated bugs in otherwise-secure products. The pattern is clear: SSL VPN appliances ship with custom HTTP parsing, custom auth, custom session handling, all written in C and exposed directly to the internet. The attack surface is enormous, the code quality is mixed, and attackers have learned that SSL VPN compromises give immediate internal network access.

### What you should know for jobs

If your environment uses SSL VPN:

- It's a high-priority patching target. Vendor advisories should be applied within hours, not weeks.
- It should be subject to threat intelligence monitoring (alerts when there's active exploitation of your specific product).
- Authentication should be MFA-enforced. Many of the exploited CVEs were combined with stolen credentials or weak auth.
- The VPN appliance should be on its own segment with limited access, treated as exposed-to-the-internet infrastructure.
- Plans to migrate away from SSL VPN are appropriate.

What's replacing SSL VPN:

- **Zero Trust Network Access (ZTNA).** Per-application access rather than full network access. Cloudflare Access, Tailscale, Twingate, Zscaler ZPA. The user authenticates per application; the gateway proxies the connection without exposing the internal network.
- **WireGuard-based remote access.** The simpler protocol with smaller attack surface. Often wrapped in a ZTNA-style product for enterprise features.
- **SASE (Secure Access Service Edge).** A broader category combining ZTNA with SD-WAN, web filtering, and DLP. Vendor offerings: Cloudflare, Zscaler, Cato.

For Sec+ purposes: know that SSL VPN exists, that it's a category with significant security history, and that the modern alternatives are ZTNA, SASE, and direct WireGuard. The exam may not phrase it as bluntly as this chapter does, but the trajectory of the industry is captured.

---

## Split tunnel vs full tunnel

A configuration choice that comes up in every VPN deployment.

**Full tunnel:** all traffic from the client goes through the VPN, including internet-bound traffic. The internet sees the corporate network's IP, not the user's local network.

**Split tunnel:** only traffic destined for specific subnets goes through the VPN. Internet traffic goes directly from the user's local connection.

### Trade-offs

**Full tunnel:**

- Pros: all traffic is inspectable by corporate security tools (DLP, web filtering, threat detection). User's location is hidden from the internet.
- Cons: more bandwidth on the corporate network. Latency for internet sites that aren't near the corporate network. Local services (printers, smart home devices) become unreachable while connected.

**Split tunnel:**

- Pros: less corporate bandwidth. Better latency to internet services. Local resources still reachable.
- Cons: corporate security tools don't see internet traffic from the user's machine. Compromised endpoint has internet access without inspection.

For Sec+ purposes: split tunnel is sometimes flagged as a security risk on the exam because it bypasses corporate inspection. The trade-off is real. Most modern deployments use a middle ground: split tunnel for known-safe destinations (Microsoft 365, Google services), full tunnel for everything else.

---

## A practical exercise

Pick one:

**Option A: Set up a WireGuard tunnel between your two lab VMs.**

1. Generate keys on each VM.
2. Configure WireGuard on each side (one as server, one as client).
3. Bring up the tunnel.
4. Verify connectivity (ping across the tunnel using the VPN IPs).
5. Capture traffic on the physical interface during a ping over the tunnel. Verify the traffic is encrypted (you should see UDP 51820, not the inner ICMP).

**Option B: Read an IPsec configuration and explain what it does.**

If you have access to a real strongSwan, Openswan, or similar configuration (your homelab, a public example), walk through one config file. For each parameter, explain what it controls. Identify what would happen if the remote side had a different value for each.

**Option C: Read recent CVE advisories for one SSL VPN vendor.**

Pick one vendor (Ivanti, Fortinet, SonicWall, Citrix). Look at advisories from the past 12 months. For each critical or high-severity CVE:

1. What was the vulnerability (RCE, auth bypass, info disclosure)?
2. What component was affected?
3. Was it exploited in the wild?
4. What was the patch timeline?

Submit a short writeup. The point is to internalize the operational reality: SSL VPN advisories are frequent, severe, and often actively exploited.

---

## Common stumbling blocks

> **My WireGuard tunnel is up but I can't ping across it.**
> Three common causes. (1) Routing isn't configured: your machine doesn't know to send traffic for the tunnel IP through the WireGuard interface. Check `ip route`. (2) Firewall is blocking: ufw or iptables drops the inner traffic. (3) AllowedIPs mismatch: the server's AllowedIPs for that client doesn't include the source IP. Check `wg show` and look at the AllowedIPs.

> **My IPsec tunnel comes up but no traffic flows.**
> Almost always selectors. The tunnel is established (Phase 1 OK, Phase 2 OK) but traffic doesn't match the selectors. Either the source/destination doesn't match what the tunnel expects, or routing isn't sending traffic into the tunnel. Check the IPsec policy and the kernel routing.

> **NAT-T isn't working.**
> Make sure UDP 500 (IKE) and UDP 4500 (NAT-T) are open in any firewall between the peers. Some networks block UDP 4500 specifically; if you can't get NAT-T working, the alternative is reaching the IPsec endpoint without NAT (uncommon for client VPN, possible for site-to-site).

> **WireGuard works on my laptop at home but not at the coffee shop.**
> The coffee shop network might be blocking UDP 51820, or doing aggressive NAT that breaks the persistent connection. Try a different listening port on your WireGuard server (some networks block well-known VPN ports). Make sure PersistentKeepalive is enabled (it keeps NAT mappings alive).

> **The VPN is connecting but I'm not getting an internal IP.**
> Different from "tunnel is up but no traffic flows." If the VPN client doesn't get an IP, the VPN protocol's address assignment isn't working. WireGuard doesn't assign IPs (you configure them statically in the config). IPsec and SSL VPN both have IP assignment mechanisms that can fail.

> **My SSL VPN client says the certificate is invalid.**
> Could be a real problem (cert expired, cert hostname doesn't match) or a man-in-the-middle attack. Check the certificate details and verify against what you expect. If it's a corporate VPN, the cert should be signed by the corporate CA; if it's a public VPN, by a public CA.

> **DPD keeps killing my tunnel.**
> Dead Peer Detection is too aggressive for your network conditions. Increase the DPD timer or interval. Sometimes a sleeping laptop or marginal cellular connection misses keepalives; the fix is more tolerance, not strict DPD.

---

## What this gets you

After this chapter:

- You can set up a WireGuard tunnel from scratch.
- You understand IPsec well enough to read configurations and diagnose common failures (Phase 1 mismatches, NAT-T, lifetime issues, DPD).
- You know why SSL VPN keeps appearing in critical CVE advisories and what's replacing it.
- You can articulate the trade-offs between split tunnel and full tunnel.
- You recognize the modern remote access categories (ZTNA, SASE) and what problems they solve.
- You have a foundation for reading vendor VPN documentation and figuring out which knobs matter.

The "don't suck at IPsec" treatment is the one that matters most for working life. You'll inherit IPsec configurations. You'll need to read the logs. The knowledge of Phase 1, Phase 2, NAT-T, DPD, and selector mismatches is the working vocabulary that makes those tasks possible.

---

## What's next

Chapter 9 is the AI-assisted practitioner. The audience-defining chapter for this unit, parallel in role to the artifact chapters in M1 and M2. We've been touching AI throughout the unit (verification habit, guarded believability); Chapter 9 is where we anchor it.
