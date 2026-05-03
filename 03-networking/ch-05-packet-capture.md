# Chapter 5: Packet Capture

**You come in with:** the four-layer model from Ch01, the diagnostic pattern from Ch03, and the DNS depth from Ch04. You've heard "look at a packet capture" but haven't done it for real.
**You leave with:** the ability to capture network traffic on Linux, Windows, and the wire-level fluency to read what you captured. You can identify a TCP handshake, recognize a TLS handshake, follow a stream, and use display filters to narrow down to the traffic you care about. Wireshark mastery is a career skill; this chapter is functional literacy.

**Time:** 90 minutes. The concepts are quick; the time is in actually capturing and reading traffic.

**Security+ alignment:** Domain 4.4 (alerting and monitoring tools: NetFlow, packet captures). Domain 4.9 (data sources for investigation: packet captures). Domain 2.4 (indicators of malicious activity: on-path attacks). Real working knowledge of packet capture is genuinely tested through scenario questions about what you'd find in a capture; this chapter teaches that skill.

---

## Why packet capture is foundational

Everything in the previous chapters described what was supposed to happen on the network. Packet capture lets you see what is actually happening. The gap between "supposed to" and "actually" is where most network and security work lives.

When DNS works, ping returns, but the application breaks: packet capture tells you whether the request even left your machine. When the firewall says it's allowing traffic but the destination doesn't see it: packet capture tells you what's getting dropped along the way. When a security alert fires and you need to know what was actually exchanged: packet capture is the evidence.

The chapter is also the gateway into deeper investigation work. SOC analysts read packet captures constantly. Network engineers read them when troubleshooting. Forensic investigators read them to reconstruct what an attacker did. Functional literacy here opens those paths.

This is introduction, not mastery. Wireshark could fill a 40-hour course on its own; the entire packet analysis discipline could be a career. We get you to "I can capture and read traffic well enough to answer common questions." The deeper skills come from doing this hundreds of times in working contexts.

---

## What's actually getting captured

A packet capture is a copy of every frame that crossed a network interface during the capture window. The capture file format is `.pcap` (or `.pcapng`, the modern variant), and it's a sequence of timestamped frames with metadata about how they were captured.

A frame includes the link-layer header (Ethernet, WiFi), the network-layer header (usually IP), the transport-layer header (usually TCP or UDP), and the application data. When you read a capture, tools parse these headers and present them in a readable form.

What you see in a capture depends on where you captured:

- **On your own host:** you see traffic to and from your machine, plus broadcasts and multicasts on your local segment.
- **On a switch span port:** you see traffic to and from any host on the configured ports. Common in enterprise environments.
- **On a tap (network tap device):** you see all traffic passing through the tap, no exceptions. Used in security operations.
- **On a router/firewall:** you see traffic crossing that device, often including NAT translation.

The capture position determines what's visible. A capture on your own host won't show traffic between two other machines on the same subnet (you're not on the path). Understanding the capture's vantage point is part of reading it.

---

## Capturing on Linux

Linux has two main tools: `tcpdump` for command-line captures, and Wireshark for the full GUI experience.

### tcpdump basics

```
# Capture everything on eth0
sudo tcpdump -i eth0

# Capture and write to a file (for later analysis)
sudo tcpdump -i eth0 -w capture.pcap

# Read a saved capture
sudo tcpdump -r capture.pcap

# Limit output to specific traffic
sudo tcpdump -i eth0 'tcp port 443'
sudo tcpdump -i eth0 'host 8.8.8.8'
sudo tcpdump -i eth0 'src 192.168.50.42 and dst port 80'

# Show packet contents in addition to headers
sudo tcpdump -i eth0 -A 'tcp port 80'    # ASCII output
sudo tcpdump -i eth0 -X 'tcp port 80'    # hex + ASCII

# Limit how many packets to capture
sudo tcpdump -i eth0 -c 10 'tcp port 443'
```

The filter expressions (`tcp port 443`, etc.) are BPF (Berkeley Packet Filter) syntax. BPF filters happen at capture time: only matching packets are written to disk. This is a capture filter.

`tcpdump` output looks like this:

```
14:23:45.123456 IP 192.168.50.42.43210 > 8.8.8.8.443: Flags [S], seq 1234567890, win 64240, options [mss 1460,sackOK,TS val 12345 ecr 0,nop,wscale 7], length 0
14:23:45.135678 IP 8.8.8.8.443 > 192.168.50.42.43210: Flags [S.], seq 9876543210, ack 1234567891, win 65535, options [mss 1460,sackOK,TS val 67890 ecr 12345,nop,wscale 7], length 0
14:23:45.135890 IP 192.168.50.42.43210 > 8.8.8.8.443: Flags [.], ack 1, win 502, options [nop,nop,TS val 12346 ecr 67890], length 0
```

That's a TCP three-way handshake. Reading it:

- Line 1: from 192.168.50.42 (port 43210) to 8.8.8.8 (port 443). Flags `[S]` means SYN. Sequence number 1234567890.
- Line 2: from 8.8.8.8 to 192.168.50.42. Flags `[S.]` means SYN+ACK. Server's sequence number 9876543210, acknowledging the client's seq+1.
- Line 3: client back to server. Flags `[.]` means just ACK. Handshake complete.

This is what every TCP connection starts with. Recognizing the SYN, SYN-ACK, ACK pattern is fundamental.

### When tcpdump is enough

tcpdump is great for:

- Quick checks: "is traffic actually leaving my machine?"
- Targeted captures with a specific filter ("show me DNS traffic to this server").
- Captures on remote machines via SSH where Wireshark isn't available.
- Capturing to a file for later analysis in Wireshark.

tcpdump becomes painful for:

- Detailed protocol analysis (parsing TLS, HTTP, etc.).
- Following a long conversation across many packets.
- Visual analysis of timing or sequence patterns.

For those, you move to Wireshark.

---

## Wireshark

Wireshark is the GUI packet analyzer. It reads the same `.pcap` files tcpdump writes, plus it can capture directly from interfaces. The display is what makes Wireshark valuable: protocol parsers decode the bytes into readable fields.

### Three panes

The Wireshark window has three main panes:

**Packet list (top):** one row per packet. Columns: number, time, source, destination, protocol, length, info. Clicking a packet selects it and populates the other panes.

**Packet details (middle):** the protocol breakdown of the selected packet. Layered: Ethernet header, IP header, TCP header, application data. Click any field to expand it and see what's inside.

**Packet bytes (bottom):** the raw bytes of the packet, with the field you have selected highlighted. Useful when you need to verify what's actually in the packet vs how Wireshark is interpreting it.

You spend most of your time in the top two panes. The bytes pane is for confirmation when something is weird.

### Capture filters vs display filters

The distinction beginners trip on:

**Capture filter:** restricts what gets captured. Set before capture starts. Uses BPF syntax (`tcp port 443`, `host 8.8.8.8`). Once you stop the capture, the filter cannot be changed; you only have the packets that matched.

**Display filter:** restricts what's shown from packets you already have. Set after capture. Uses Wireshark's display filter syntax (`tcp.port == 443`, `ip.addr == 8.8.8.8`). You can change the display filter freely; the underlying capture file isn't modified.

You almost always want display filters. The standard workflow:

1. Start capture without a filter (or with a very loose one).
2. Stop the capture.
3. Apply display filters to drill into specific traffic.

Capture filters are for when you're capturing a lot of traffic and need to keep the file size manageable. For most diagnostic work, capture everything for a short window and filter the display.

### The display filter syntax that matters

Common display filters worth memorizing:

```
# By IP
ip.addr == 8.8.8.8                          # any direction
ip.src == 192.168.50.42                     # source only
ip.dst == 8.8.8.8                           # destination only

# By port
tcp.port == 443                             # TCP, any direction
tcp.dstport == 443                          # destination only
udp.port == 53                              # UDP

# By protocol
http
dns
tls

# Combine with logical operators
ip.src == 192.168.50.42 and tcp.dstport == 443
http or dns

# Specific patterns
tcp.flags.syn == 1 and tcp.flags.ack == 0  # connection initiations
tcp.flags.reset == 1                        # connection resets (often interesting)
http.request.method == "POST"              # HTTP POST requests
dns.qry.name contains "example"             # DNS queries for names containing "example"
tls.handshake.type == 1                     # TLS Client Hello
```

Practice these on a real capture until they feel natural. The cheat sheet for display filters is built into Wireshark itself: View > Filter Expression Buttons > New shows you available fields per protocol.

### Following a stream

Right-click any TCP or UDP packet > Follow > TCP Stream (or UDP Stream). Wireshark opens a window showing the entire conversation reconstructed as a sequence of bytes, with client traffic in one color and server traffic in another.

For HTTP, this gives you the request and response in plain text. For most other protocols, you see the raw application data, which may or may not be readable.

This is the workflow for "what was actually exchanged in this conversation":

1. Find any packet in the conversation.
2. Right-click > Follow > TCP Stream.
3. Read.

For HTTPS (TLS), the stream is encrypted, so you see encrypted gibberish. Decrypting requires either having the server's private key (rare) or having logged the TLS keys at the time of capture (possible with browsers configured to log).

---

## Reading TCP

Most of network traffic is TCP. Recognizing TCP patterns is the most reusable skill in this chapter.

### The three-way handshake

Every TCP connection starts with three packets:

1. **SYN** from client to server. Client picks an initial sequence number.
2. **SYN-ACK** from server to client. Server picks its own sequence number, acknowledges client's seq+1.
3. **ACK** from client to server. Acknowledges server's seq+1.

After these three packets, the connection is established and either side can send data. Display filter to find handshake initiations:

```
tcp.flags.syn == 1 and tcp.flags.ack == 0
```

That's just SYN packets (no ACK yet). Each match is a connection attempt. The matching SYN-ACK and final ACK are usually the next two packets.

If you see SYN but no SYN-ACK response, the destination didn't respond. Could be filtered, down, or unreachable. If you see SYN and SYN-ACK but no final ACK, the client gave up. Could be a firewall rule, could be a half-open scan.

### The four-way teardown (FIN exchange)

Connections close gracefully with FIN packets:

1. Client sends FIN.
2. Server ACKs the FIN.
3. Server sends its own FIN.
4. Client ACKs.

In practice, steps 2 and 3 often happen in one packet (FIN+ACK), making it look like a three-way teardown.

Display filter:

```
tcp.flags.fin == 1
```

### RST: the abrupt close

RST (Reset) packets close a connection immediately, no handshake. Common reasons:

- Receiving end didn't have a connection state for the packet (e.g., port closed, sent SYN but never completed).
- Application explicitly closed the socket without proper teardown.
- Firewall rejecting the connection (some firewalls send RST instead of silently dropping).
- Stack timeout decided the connection was dead.

Display filter:

```
tcp.flags.reset == 1
```

RSTs in a capture are often interesting. They indicate something went wrong or someone wanted the connection gone fast.

### Retransmissions

When a sender doesn't get an ACK in time, it retransmits the data. Wireshark detects and marks these:

```
tcp.analysis.retransmission
tcp.analysis.fast_retransmission
tcp.analysis.duplicate_ack
```

Frequent retransmissions indicate network problems: packet loss, congestion, or a slow path. Occasional retransmissions are normal.

The "Expert Information" view in Wireshark (Analyze > Expert Information) summarizes these issues, sorted by severity. Useful when you want a quick overview of what's wrong.

### Sequence numbers and acknowledgments

Each TCP segment has a sequence number (where in the byte stream this data sits) and an acknowledgment number (next expected byte from the other side). Wireshark shows both in absolute and relative form. The relative form starts at 1 for each connection, which is more readable.

The deep details of sequence number analysis are intermediate work. For now, recognize that sequence numbers exist and Wireshark can detect when they're suspicious (out of order, gaps, duplicates).

---

## Reading TLS

TLS is everywhere now. Most "interesting" traffic is encrypted. You can't see the application data without decryption keys, but you can see the handshake and learn things from it.

### The TLS handshake

Modern TLS 1.2 and 1.3 handshakes:

1. **Client Hello.** Client tells server: TLS version, cipher suites it supports, server name (SNI), random bytes.
2. **Server Hello.** Server tells client: chosen TLS version, chosen cipher, certificate, random bytes.
3. **Key exchange + change cipher.** Both sides establish session keys.
4. **Encrypted application data flows.**

What you can read from the handshake (it's not encrypted yet):

- **SNI (Server Name Indication):** the hostname the client is trying to reach. Visible in plaintext in the Client Hello.
- **Cipher suites offered:** what the client claims to support.
- **Cipher chosen:** what the server picked.
- **Server certificate:** the public certificate for the server. Includes hostnames covered, issuer, validity dates.
- **TLS version negotiated:** 1.0 (deprecated), 1.1 (deprecated), 1.2 (still common), 1.3 (modern).

Display filter for TLS Client Hello:

```
tls.handshake.type == 1
```

For Server Hello:

```
tls.handshake.type == 2
```

### What you can't see

Inside the encrypted application data, you can't see:

- HTTP requests and responses.
- Authentication credentials.
- Application-specific protocols (SSH, IMAP-over-TLS, etc.).

This is by design. TLS is supposed to make this stuff invisible to anyone who isn't an endpoint.

The only ways to see encrypted content:

1. **Have the server's private key.** Rare; usually only the server admin has it.
2. **Have the session keys logged at capture time.** Browsers can be configured to log session keys to a file (`SSLKEYLOGFILE` environment variable), and Wireshark can use that file to decrypt traffic captured during the same session.
3. **Be the man-in-the-middle.** Some enterprise environments install custom CA certificates on managed devices and use a TLS-inspecting proxy. The proxy decrypts traffic, inspects it, re-encrypts to the destination. The endpoint trusts the proxy's certs because the corporate CA is installed.

For a typical capture from your own machine, expect TLS traffic to be opaque. The handshake metadata is still useful.

### TLS as a security indicator

Several patterns in TLS handshakes are interesting from a security perspective:

- **TLS 1.0 or 1.1:** deprecated since 2020. Seeing these in a capture indicates legacy systems that should be upgraded.
- **Weak cipher suites:** RC4, DES, MD5-based MACs are all weak or broken. Modern handshakes negotiate AES-GCM or ChaCha20-Poly1305 with SHA-256 or SHA-384.
- **Self-signed certificates:** the certificate isn't trusted by a public CA. Sometimes legitimate (internal services), often suspicious (man-in-the-middle attacks, malware C2).
- **Mismatched SNI and server cert:** client says "I want to connect to A," server presents cert for B. Could be legitimate (CDN serving multiple sites from one IP) or could be DNS hijacking.

These patterns become recognizable with practice. For now, know they exist and that the handshake metadata supports security analysis.

---

## Capturing on Windows: Pktmon

Modern Windows ships with Pktmon, a built-in packet capture tool. No installation required.

```powershell
# List interfaces
pktmon list

# Start a capture (requires admin)
pktmon start --capture

# Stop and save
pktmon stop

# Convert to .pcapng (which Wireshark reads)
pktmon etl2pcap PktMon.etl

# Now you have PktMon.pcapng to load into Wireshark
```

Pktmon is the right tool for "I'm on a Windows server, I need a quick capture, and I don't want to install anything." For deeper analysis, transfer the pcapng to a workstation with Wireshark.

For real workflows, also worth knowing:

- **Microsoft Network Monitor (NetMon)** is the older tool. Some shops still have it.
- **Microsoft Message Analyzer** was the modern replacement, but Microsoft retired it.
- **Wireshark on Windows** works fine; install from wireshark.org.

Most working Windows admins use Wireshark when they need a GUI. Pktmon for quick captures, especially on servers.

---

## A guarded-believability moment

When you're learning Wireshark, AI tools can help explain unfamiliar protocol fields. The danger is they can confidently invent fields that don't exist or describe deprecated behavior.

Try this experiment:

1. Pick a TCP option from a real capture (e.g., "TCP Window Scale," "Selective ACK Permitted," "Maximum Segment Size").
2. Ask an LLM to explain what it means.
3. Verify the answer against RFC 9293 (the current TCP specification) or against Wireshark's own field documentation.

For well-known fields, the LLM is usually right. For obscure or recently-changed fields, it's sometimes wrong in ways that sound plausible. The verification habit catches these.

The same applies for TLS fields. Cipher suite names are confusingly similar (`TLS_AES_128_GCM_SHA256` vs `TLS_AES_128_CBC_SHA256`). LLMs sometimes describe one when you asked about the other. Verify against the IANA TLS Cipher Suites Registry or the IETF specs.

---

## A practical exercise

Run captures and answer questions.

**Exercise 1: Capture your own browser traffic.**

On your machine (Windows or Linux), start a packet capture on the active interface. Then load `https://example.com` in your browser. Stop the capture.

In Wireshark:

1. Apply the display filter `dns` to find the DNS queries. What names did your browser look up? (Hint: you'll see at least the literal `example.com`, but you might see others depending on your browser.)
2. Apply `tcp.flags.syn == 1 and tcp.flags.ack == 0` to find connection initiations. How many distinct connections did the browser open?
3. Apply `tls.handshake.type == 1` to find Client Hello packets. For one of them, find the SNI in the packet details. Does it match what you typed in the browser?
4. Apply `tls.handshake.type == 2` to find the corresponding Server Hello. What TLS version was negotiated? What cipher was chosen?
5. Apply `tcp.flags.fin == 1` to find connection closures. Compare the count to the connection initiations. Do they match?

**Exercise 2: Capture a DNS lookup.**

Run a capture filtered to DNS only:

```
sudo tcpdump -i any -w dns.pcap 'udp port 53'
```

Or in Wireshark, capture without filter on the interface and apply display filter `dns` after.

Now run:

```
dig github.com
```

Stop the capture. In Wireshark:

1. Find the query packet. What's in it (query name, query type, transaction ID)?
2. Find the response packet. Match it to the query by transaction ID. What did the response contain?
3. How many round-trips did the resolution take? (Note: if your resolver had the answer cached, it's just one query+response. If it had to do recursion, you wouldn't see it from your local capture; you'd need to capture on the resolver itself.)

**Exercise 3: Find a TCP RST in the wild.**

Most browsing traffic doesn't generate RSTs. To force one, do a port scan to a host that doesn't expose the port:

```
nc -vz 8.8.8.8 81    # port 81 isn't open
```

Capture during this. In Wireshark, filter for `tcp.flags.reset == 1`. You should see an RST from 8.8.8.8 telling you the port is closed.

Alternatively, browse to an HTTPS site and watch for RSTs at connection close (some servers RST instead of FIN for performance reasons).

---

## Common stumbling blocks

> **My capture doesn't show traffic between two other machines.**
> You're not on their path. Switches forward traffic only to the destination port; you don't see traffic that's not addressed to you. Solutions: capture on a span port, use a tap, capture on the destination machine itself.

> **`sudo tcpdump` shows nothing.**
> Wrong interface? Check `ip link show` for available interfaces. Common mistakes: capturing on the loopback when traffic is on Ethernet, capturing on Ethernet when you're using WiFi.

> **Wireshark shows me weird traffic I didn't initiate.**
> Modern OSes and applications generate background traffic constantly: NTP, DNS for various purposes, telemetry, software update checks, broadcast/multicast for service discovery (mDNS, Bonjour, LLMNR). The "I didn't ask for this" traffic is normal.

> **Display filter syntax errors.**
> Wireshark's filter syntax has its quirks. Common gotchas: `==` not `=`, `and`/`or` not `&&`/`||` (though both work), `contains` for substring (not the protocol's name). The filter bar turns red on syntax errors and green on valid filters.

> **I captured TLS traffic but Wireshark shows it as encrypted gibberish.**
> Expected. Without the keys, TLS payload is unreadable. Wireshark still shows the handshake, which is unencrypted.

> **My capture file is huge.**
> Long captures or busy interfaces produce large files. Solutions: use a capture filter to narrow what's recorded, capture for shorter windows, use ring buffer mode in Wireshark (Capture > Options > Output) to keep only the most recent N MB.

> **`tcpdump` complains about permissions.**
> tcpdump needs raw socket access, which requires root. Use `sudo`. Some Linux distributions allow specific users to run tcpdump via setcap; check your distribution's documentation if running as root is undesirable.

> **The Pktmon output is in `.etl` format, which Wireshark won't read directly.**
> Use `pktmon etl2pcap <file>.etl` to convert. The result is a `.pcapng` file Wireshark reads natively.

---

## What this gets you

After this chapter:

- You can capture network traffic with `tcpdump` on Linux, Pktmon on Windows, and Wireshark in the GUI.
- You understand the difference between capture filters and display filters.
- You can read a TCP three-way handshake, four-way teardown, and recognize RSTs and retransmissions.
- You can identify a TLS handshake in a capture and pull SNI, cipher suite, and certificate information.
- You can follow a TCP stream to see the full conversation.
- You know what's visible in encrypted traffic (handshake metadata) and what isn't (application data).
- You have the verification habit applied to AI explanations of protocol fields.

This is functional literacy. Mastery comes from doing this in working contexts repeatedly. The skill compounds: every capture you read makes the next one faster.

---

## What's next

Chapter 6 is firewalls. Specifically: hands-on with host firewalls (Defender Firewall on Windows, ufw and iptables on Linux), and reading-only with network firewalls (pfSense rule sets). The packet-level fluency from this chapter is what makes firewall rules concrete: a rule says "allow TCP traffic where source is X, destination port is 443," and now you know what TCP, source, and port mean at the packet level.
