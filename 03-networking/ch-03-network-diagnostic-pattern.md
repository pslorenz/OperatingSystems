# Chapter 3: When the Network Breaks

**You come in with:** the ability to read your machine's network configuration (Ch01) and the ability to subnet (Ch02). You can identify the gateway, DNS server, and IP plan. You haven't yet built a habit for what to do when something stops working.
**You leave with:** a four-step diagnostic pattern you can apply to any network problem. The pattern walks the TCP/IP layers from bottom to top: link, network, transport, application. By the end you know what tools answer questions at each layer and how to identify which layer is the problem.

**Time:** 75 minutes. The pattern itself is short; the time is in seeing it applied to several common scenarios.

**Security+ alignment:** Domain 4.4 (alerting and monitoring concepts: troubleshooting). Domain 4.8 (incident response: detection and analysis). Domain 4.9 (data sources to support an investigation: network logs, packet captures). The diagnostic pattern itself isn't directly tested, but the tools and the layered thinking are. Knowing where in the layer stack a problem lives is the same skill the exam tests when asking which control to apply.

---

## Why this chapter matters

"The network is broken" is the most common ticket type for sysadmins and the second most common for security teams (after malware). What that ticket actually means varies wildly: "I can't reach the web," "this specific app stopped working," "everything is slow," "DNS is weird." The complaint is the same; the cause is anywhere from "cable unplugged" to "BGP hijack."

Without a diagnostic pattern, troubleshooting is guessing. You try whatever pops into your head, and if you're lucky you stumble into the answer. With a pattern, troubleshooting is systematic. You walk the layers, eliminate possibilities, and find the problem.

The pattern in this chapter is what working network admins use. It's not the only valid pattern, but it's a good one and it scales from "user can't reach the printer" to "we have a regional outage." The discipline of working bottom-up, layer by layer, is what separates "stares at it for an hour" from "figures it out in five minutes."

This chapter also pulls together routing, DNS, and basic services into a job-shaped chapter. We don't have separate chapters for each because juniors don't troubleshoot routing in isolation; they troubleshoot "the network is broken" and the answer might be routing or DNS or a firewall rule. The pattern is what unifies them.

---

## The four-step pattern

When the network is broken, walk these four steps in order:

1. **Link/physical:** Is the interface up and connected to the right network?
2. **Network:** Can I reach the gateway and beyond?
3. **Transport:** Is the destination port reachable?
4. **Application:** Is the service responding correctly?

You stop at the first step that fails. Whatever fails first is your problem; layers above it can't possibly work if the layer below is broken.

Each step has specific commands that answer the question. The rest of this chapter walks each step.

---

## Step 1: Link and physical

The first question: is your machine actually connected to a working network?

### What you're checking

- Is the interface up (not disabled, not in a "no carrier" state)?
- Does the interface have an IP address (not APIPA)?
- Can you reach the gateway?

If any of these fail, you're not getting anywhere. Stop here, fix the link layer, then move on.

### Tools and commands

**Linux:**

```
ip link show
ip addr show
ping -c 4 <gateway>
```

`ip link show` shows interface state. Look for "UP" and "LOWER_UP" in the flags. "DOWN" means software has the interface disabled. "NO-CARRIER" means there's no physical link (cable unplugged, WiFi not associated).

`ip addr show` shows IP configuration. Confirm you have an address in the right range. APIPA (`169.254.x.x`) means DHCP failed.

`ping -c 4 <gateway>` tests reachability to the gateway. Replace `<gateway>` with whatever your default route points to. If this works, your link layer is fine.

**Windows:**

```powershell
Get-NetAdapter | Format-Table Name, Status, LinkSpeed
Get-NetIPAddress -AddressFamily IPv4
Test-NetConnection <gateway>
```

`Get-NetAdapter` shows interface state. Status should be "Up." If "Disabled" the interface is software-disabled. If "Disconnected" no physical link.

`Get-NetIPAddress` shows IP configuration.

`Test-NetConnection` is PowerShell's all-in-one connectivity tool. With just an address, it does an ICMP ping. If this responds, the link layer is fine.

### Common failures and their fixes

**Interface DOWN.** On Linux: `sudo ip link set <interface> up`. On Windows: enable the adapter in Network Connections or `Enable-NetAdapter -Name "Ethernet"`.

**No carrier.** Physical problem. Check the cable. Check the switch port. For WiFi, check the association.

**APIPA (169.254.x.x).** DHCP failed. Force a renewal: `sudo dhclient -r && sudo dhclient` on Linux, `ipconfig /release && ipconfig /renew` on Windows. If renewal fails, look for "no DHCP server response" errors. Could be the DHCP server is down, could be a switch port misconfigured, could be a VLAN issue.

**Wrong subnet.** Your IP looks fine but your subnet mask is wrong. Result: your machine thinks the gateway is unreachable because it's not "in your subnet." Verify the mask.

**Can't ping gateway.** You have an IP, the mask looks right, but the gateway doesn't respond. Two possibilities: the gateway is down (rare), or there's a Layer 2 problem between you and the gateway (cable, switch port, ACL on a managed switch).

### What you've eliminated

If Step 1 succeeds (you have an IP, you can ping the gateway), you've eliminated link-layer problems. The cable works, the switch works, DHCP gave you an address, the gateway is alive and responsive. Move to Step 2.

---

## Step 2: Network

The next question: can you reach beyond the gateway?

### What you're checking

- Can you reach a known address outside your subnet?
- If using DNS-resolved destinations, can DNS resolve the name?

Most "I can't reach the internet" problems live here. The gateway is up but something past it is broken: DNS isn't working, the gateway can't route to the destination, there's a firewall in the way.

### Tools and commands

**Linux:**

```
ping -c 4 8.8.8.8
ping -c 4 google.com
dig google.com
traceroute 8.8.8.8
```

The pattern: ping a known IP that should always be reachable (8.8.8.8 is Google's public DNS, designed to respond to ICMP). If that works, network-layer reachability is fine. Then ping a name. If the name fails but the IP works, DNS is your problem.

`dig` queries DNS directly. If `dig google.com` returns an answer, DNS is working. If it times out or returns SERVFAIL, DNS is broken.

`traceroute` shows the path packets take to a destination. Useful when reachability is partial: traceroute reveals where the path stops working.

**Windows:**

```powershell
Test-NetConnection 8.8.8.8
Test-NetConnection google.com
Resolve-DnsName google.com
Test-NetConnection 8.8.8.8 -TraceRoute
```

`Test-NetConnection -TraceRoute` is PowerShell's traceroute equivalent. The legacy `tracert` command also works.

### Common failures and their fixes

**Can ping IP, can't ping name.** DNS problem. Check `/etc/resolv.conf` (Linux) or DNS servers in network config (Windows). Try a different DNS server: `dig @1.1.1.1 google.com` or `Resolve-DnsName google.com -Server 1.1.1.1`. If that works, your configured DNS server is the problem.

**Can't ping IP either.** Routing problem or firewall problem. Run traceroute. Where does it stop?

- Stops at your gateway: gateway routing problem, or your gateway can't reach upstream.
- Stops at an intermediate hop: routing problem at that hop, or firewall blocking.
- Stops at the destination: destination is down, or destination's firewall blocks ICMP.

**ICMP blocked but TCP works.** Some networks block ICMP for security. Test with a TCP connection: `Test-NetConnection google.com -Port 443`. If TCP works to a known port, the network is reachable; ICMP is just blocked.

**DNS works, ping works, but specific destinations fail.** Could be a firewall rule, route asymmetry, or the destination genuinely being down. Move to Step 3 to test the specific port.

### A worked example

User says: "I can't reach our internal wiki at wiki.corp.example.com."

Step 1: confirm link. `ip addr` shows valid address. `ping gateway` works. Link is fine.

Step 2: confirm network.

```
$ ping -c 2 8.8.8.8
PING 8.8.8.8: 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=53 time=12.4ms

$ dig wiki.corp.example.com
; <<>> DiG 9.18.1 <<>> wiki.corp.example.com
;; QUESTION SECTION:
;wiki.corp.example.com. IN A
;; ANSWER SECTION:
wiki.corp.example.com. 300 IN A 10.10.20.50

$ ping -c 2 10.10.20.50
PING 10.10.20.50: 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
```

DNS resolves correctly. We get back 10.10.20.50. But we can't ping 10.10.20.50 even though we can ping 8.8.8.8.

That's interesting. We can reach the internet but not an internal address. Possible causes: the wiki server is down, ICMP is blocked at the wiki server, there's a route problem to the internal subnet.

Run traceroute to 10.10.20.50:

```
$ traceroute 10.10.20.50
1 192.168.50.1 (192.168.50.1) 0.5ms
2 * * *
3 * * *
```

Stops at the gateway. The gateway can't (or won't) route to 10.10.20.50. Now we have a specific hypothesis: the gateway is missing a route to the 10.10.20.0/24 network, or there's a firewall rule blocking it.

This is what the pattern gets you: from a vague complaint to a specific hypothesis in three commands.

### What you've eliminated

If Step 2 succeeds (you can reach addresses outside your subnet, DNS resolves), you've eliminated routing and DNS problems. Move to Step 3.

---

## Step 3: Transport

The next question: is the destination's specific port reachable?

### What you're checking

- Can you make a TCP connection to the destination on the expected port?
- If UDP, can you send to the destination on the expected port and get a reasonable response?

Many failures live here. The destination is reachable (you can ping it) but the specific service isn't. Maybe the service is down. Maybe there's a firewall rule blocking that port. Maybe the service is bound to a different interface.

### Tools and commands

**Linux:**

```
nc -vz <destination> <port>
nmap -p <port> <destination>
ss -tlnp
```

`nc -vz <host> <port>` does a TCP connect and reports whether it succeeded. `-v` is verbose, `-z` is "zero data" (just the connection, no data).

`nmap -p <port> <destination>` does the same thing with more detail. Reports open, closed, or filtered.

`ss -tlnp` (on the destination, if you can get there) shows which ports are listening locally. Useful for confirming the service is bound to the right interface and port.

**Windows:**

```powershell
Test-NetConnection <destination> -Port <port>
Get-NetTCPConnection -State Listen
```

`Test-NetConnection -Port` does a TCP connect test. Returns `TcpTestSucceeded: True` if the port is open.

`Get-NetTCPConnection -State Listen` shows local listening ports.

### Common failures and their fixes

**Connection refused.** The destination port is reachable (TCP RST came back), but no service is listening. Either the service is down, or the service is on a different port. SSH a different way to confirm; check the service status on the destination.

**Connection times out.** Something is blocking the packet (firewall, routing) or the destination is genuinely unreachable on that port. Time out is the silent failure.

**Some ports work, others don't.** Selective firewall rule. Could be on the destination, on the network in between, or on an intermediate firewall. Look at firewall logs if you can get to them.

**Service is listening on the destination but Test-NetConnection fails from another machine.** Three common causes: host firewall blocking the port (check Windows Defender Firewall, ufw, iptables), service bound to localhost-only (`127.0.0.1` instead of all interfaces), or a network firewall in between.

### Practical port reference

Ports you should know without looking up:

| Port | Service |
|---|---|
| 22 | SSH |
| 23 | Telnet (deprecated, insecure) |
| 25 | SMTP |
| 53 | DNS (both UDP and TCP) |
| 80 | HTTP |
| 110 | POP3 |
| 143 | IMAP |
| 161 | SNMP |
| 389 | LDAP |
| 443 | HTTPS |
| 445 | SMB (Windows file sharing) |
| 587 | SMTP submission |
| 636 | LDAPS (LDAP over TLS) |
| 993 | IMAPS (IMAP over TLS) |
| 995 | POP3S |
| 3389 | RDP |
| 5900 | VNC |

These show up constantly in firewall rules, packet captures, and Sec+ exam questions. Memorize. The cheat sheet from M2 has the full list with notes.

### What you've eliminated

If Step 3 succeeds (you can connect to the specific port), the network and the service are both up. Move to Step 4 for application-level investigation.

---

## Step 4: Application

The final question: is the application doing what you expect?

### What you're checking

- Does the application respond correctly?
- Are you authenticating successfully?
- Is the response format correct?

Step 4 is where things get protocol-specific. The diagnostic depends on what application protocol is involved.

### Tools and commands

**HTTP/HTTPS:**

```
# Linux
curl -v https://example.com
curl -I https://example.com   # Headers only

# Windows
Invoke-WebRequest https://example.com -UseBasicParsing
(Invoke-WebRequest https://example.com -UseBasicParsing).StatusCode
```

`curl -v` shows the full request and response, including TLS handshake details. Useful for diagnosing certificate problems, slow responses, redirect issues.

`curl -I` requests just the headers. Faster than fetching the whole page when you just want to confirm the service is responding.

**SSH:**

```
ssh -v <user>@<host>
```

The `-v` flag (or `-vv` or `-vvv` for more verbosity) shows the SSH negotiation. Useful when authentication fails: shows which auth methods are tried, why each fails.

**DNS:**

```
dig +trace <name>
nslookup <name> <server>
```

`dig +trace` walks the recursive resolution from the root. Useful when DNS is misbehaving and you want to know which step is wrong.

**SMTP, IMAP, etc.:**

```
# Test SMTP with telnet (yes, it still works for diagnosing)
telnet smtp.example.com 25
# Or with curl
curl -v telnet://smtp.example.com:25
```

Many text-based protocols can be tested manually with telnet or curl. You speak the protocol by hand, see exactly what the server says back. Useful for diagnosing handshake failures.

### Common failures and their fixes

**HTTP 500 errors.** Server side bug. Not your problem; report it.

**HTTP 503 errors.** Server overloaded or in maintenance. Wait, retry, escalate.

**HTTPS certificate errors.** Cert expired, cert hostname doesn't match, cert isn't trusted. Look at the cert details: `openssl s_client -connect example.com:443 -showcerts`. The output shows the certificate chain.

**SSH "Permission denied (publickey)."** Your key isn't accepted. Verify the public key is in the server's `~/.ssh/authorized_keys`. Check `~/.ssh/authorized_keys` permissions (should be 600 or 644, with the directory 700).

**SSH "Connection refused."** SSH service isn't running on the destination. Step 3 should have caught this, but sometimes the port is filtered (silent), not refused (RST). Either way: confirm SSHd is running on the destination.

**DNS NXDOMAIN.** The name doesn't exist. Could be a typo, could be DNS misconfiguration, could be domain expired. Verify by querying authoritative servers directly.

---

## A complete worked diagnostic

Scenario: a user reports "I can't access the company file server."

### Walk the pattern

**Step 1: Link.**

```
$ ip addr show
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.50.42/24 brd 192.168.50.255 scope global dynamic eth0

$ ping -c 2 192.168.50.1
64 bytes from 192.168.50.1: icmp_seq=0 ttl=64 time=0.5 ms
```

Link is fine. IP is valid, gateway responds.

**Step 2: Network.**

```
$ ping -c 2 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=0 ttl=53 time=12.1 ms

$ dig fileserver.corp.example.com
;; ANSWER SECTION:
fileserver.corp.example.com. 300 IN A 10.10.20.50

$ ping -c 2 10.10.20.50
64 bytes from 10.10.20.50: icmp_seq=0 ttl=63 time=2.1 ms
```

Network is fine. Internet reachable. DNS resolves the name. The file server IP is reachable.

**Step 3: Transport.**

The user said file server, which usually means SMB. Test port 445:

```
$ nc -vz 10.10.20.50 445
nc: connect to 10.10.20.50 port 445 (tcp) failed: Connection timed out
```

Connection timed out. We can ping the server but can't reach port 445. Either SMB isn't running on the server, the server's host firewall is blocking 445, or there's a network firewall in between.

**Hypothesis:** something between us and the server blocks port 445. Could be the destination's firewall, could be a network firewall.

To narrow down, try a different port that should work (SSH if available):

```
$ nc -vz 10.10.20.50 22
Connection to 10.10.20.50 port 22 [tcp/ssh] succeeded!
```

Port 22 works to the same destination. So we have route, we have network reachability. Specifically port 445 is blocked.

Most likely cause given the symptom (some ports to the same host work, some don't): a firewall rule, either on the destination or on a network firewall. If you can SSH into the file server, check its firewall:

```
# On the file server, via SSH
$ sudo iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https
```

Found it. The file server's iptables INPUT chain has explicit ACCEPT rules for SSH, HTTP, and HTTPS, but no rule for SMB (445). Default policy is DROP. So SMB traffic gets dropped.

The fix depends on whether SMB was supposed to be open. If the host should be a file server, add the rule. If the host shouldn't be a file server, the user is reaching for the wrong machine.

### Why the pattern matters

Without the pattern, you'd have started by asking "what's wrong" and trying things. With the pattern, you went from "I can't access the file server" to "the file server has iptables blocking port 445" in five commands. The user gets an answer; if you're escalating, you escalate with specifics.

---

## Tools to keep in your head

The diagnostic pattern uses a small set of tools repeatedly. Make sure these are second nature:

| Tool | Purpose | Linux | Windows |
|---|---|---|---|
| Show interface state | Layer 1/2 | `ip link show` | `Get-NetAdapter` |
| Show IP config | Layer 3 | `ip addr show` | `Get-NetIPAddress` or `ipconfig /all` |
| Test reachability | Layer 3 | `ping <host>` | `Test-NetConnection <host>` |
| Test specific port | Layer 4 | `nc -vz <host> <port>` | `Test-NetConnection <host> -Port <port>` |
| Trace network path | Layer 3 | `traceroute <host>` | `Test-NetConnection <host> -TraceRoute` or `tracert` |
| Query DNS | Layer 7 (DNS-specific) | `dig <name>` | `Resolve-DnsName <name>` |
| Show ARP cache | Layer 2 | `ip neigh` | `Get-NetNeighbor` or `arp -a` |
| Show route table | Layer 3 | `ip route` | `Get-NetRoute` |
| Show local listening | Layer 4 | `ss -tlnp` | `Get-NetTCPConnection -State Listen` |

The cheat sheet from the M2 closeout has these in side-by-side form. Keep it open while you're learning the pattern.

---

## A practical exercise

Each scenario below describes a network problem. Walk the four-step pattern. Identify what step the problem is at and what command would diagnose it.

**Scenario 1.** User says: "I can't reach our intranet."

**Scenario 2.** Server admin says: "The new web server I deployed isn't reachable, but I can ping it."

**Scenario 3.** User says: "Our connection has been dropping randomly for two days."

**Scenario 4.** Application owner says: "The HTTPS API is returning errors. The app team says the API is up."

**Scenario 5.** Support says: "All users in the third-floor office can't reach the printer. Other floors are fine."

Try to walk the pattern for each before checking the answers below.

---

## Answers to the practical exercise

**Scenario 1.** Walk all four steps. Step 1: confirm link. Step 2: ping the intranet IP, query DNS. If DNS fails, that's the problem. If DNS works but the IP is unreachable, traceroute to find where the path breaks. If the IP is reachable, move to Step 3 and test port 80/443.

**Scenario 2.** Step 1 already passed implicitly (server is on the network). Step 2 passed (ping works). Skip to Step 3: `nc -vz <server> 80` or `Test-NetConnection <server> -Port 80`. If that fails, check whether the web service is running on the server (`systemctl status nginx` on Linux, `Get-Service` on Windows) and whether the host firewall allows the port.

**Scenario 3.** Intermittent problems are harder. The pattern still applies but you'll need to capture state during a failure. Run `ping -c 100 <gateway>` continuously and watch for drops. Run `ping -c 100 8.8.8.8` simultaneously. Check whether drops correlate. Run a packet capture (Chapter 5) during a failure window to see what actually happens.

**Scenario 4.** Step 1-3 are likely fine if the API is generally working. Skip to Step 4: `curl -v https://api.example.com/endpoint`. Read the response. Could be auth (401), could be server error (5xx), could be cert problem (TLS handshake failure). The verbose output tells you which.

**Scenario 5.** "All users on third floor" tells you the problem is local to that floor. Likely Step 1 or Step 2 issue specific to that subnet. Walk the pattern from a third-floor machine: can it reach its gateway, can it reach 8.8.8.8, can it ping the printer's IP, can it connect to the printer's port. The answer is usually: their VLAN/subnet has a configuration issue, or there's a switch problem, or the printer's port has a host firewall problem.

---

## Common stumbling blocks

> **I'm skipping to Step 4 because the user told me it's an application problem.**
> Often the user is wrong. They see "the app is broken" but the cause is at Layer 1. Walk the pattern from the bottom every time. Skipping steps is how you spend three hours on a Layer 7 hypothesis when the problem was a cable.

> **My ping fails but everything else works.**
> ICMP is sometimes blocked at firewalls, especially at the network edge. Failed ping doesn't always mean the host is down. Use TCP connect tests (`nc`, `Test-NetConnection -Port`) instead.

> **traceroute shows asterisks but I can reach the destination.**
> Some hosts in the path don't respond to traceroute (ICMP TTL exceeded). The path works; you just can't see all the hops. Modern traceroute can use TCP or UDP probes (`-T` or `-U` on Linux, `Test-NetConnection -TraceRoute` uses TCP) which sometimes reveal hops that block ICMP.

> **My DNS works for some names but not others.**
> Could be a specific authoritative server is down. Could be a split-DNS configuration where internal names resolve through one server and external through another. Run `dig` against multiple servers (`dig @1.1.1.1 name.example.com`, `dig @<internal-server> name.example.com`) to see which fails.

> **The diagnostic pattern doesn't fit my situation.**
> The pattern is for "I can't reach X" problems. Other problem types (slow performance, packet loss, intermittent failures) need different approaches. The pattern still helps narrow the layer; once you know the layer, the right tool varies.

> **I solved the problem but I don't know how to write up what I did.**
> "User reported X. Walked diagnostic pattern: step 1 OK (output), step 2 OK (output), step 3 failed (output). Hypothesis: firewall on destination blocking port. Verified by checking iptables on destination. Resolution: added accept rule for port. Verified user can now reach service." That's a complete ticket resolution.

---

## What this gets you

After this chapter:

- You have a four-step diagnostic pattern that applies to any "the network is broken" ticket.
- You know which command answers which question at each layer.
- You can walk the pattern for common scenarios and identify the problem layer.
- You can write up your work in a way that makes sense to whoever is reading the ticket after you.

The discipline of working bottom-up, layer by layer, is what working network admins do. With this pattern internalized, troubleshooting becomes systematic instead of stressful.

---

## What's next

Chapter 4 is DNS as a service. Half of the "network is broken" tickets resolve to DNS, so we go deeper than this chapter touched. Record types in detail, the resolver chain, DMARC/DKIM/SPF for email security, and DNS attacks. By the end of Chapter 4 you understand DNS at the depth needed to answer "is DNS the problem here" with confidence.
