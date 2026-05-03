# Chapter 4: DNS as a Service

**You come in with:** the diagnostic pattern from Ch03 in your head. You've used `dig` and `Resolve-DnsName` enough to know they query DNS. You've heard "DNS is the answer to half the network problems" and you're not sure whether to believe it.
**You leave with:** the depth on DNS that working admins need. You know record types, the resolver chain, how caching works, what DNSSEC does. You can read DMARC/DKIM/SPF records and explain what they enforce. You recognize common DNS attacks and what they accomplish.

**Time:** 75 to 90 minutes.

**Security+ alignment:** Domain 2.4 (indicators of malicious activity: DNS attacks). Domain 4.5 (modify enterprise capabilities to enhance security: DNS filtering, email security with DMARC/DKIM/SPF, secure protocols). Domain 4.9 (data sources for investigation: DNS logs). The email security trio is direct exam content; this chapter is where you learn it for real rather than as flashcards.

---

## Why DNS deserves its own chapter

The "half of network problems are DNS" claim is real. Anything that involves a name (which is most things) has a DNS lookup behind it. If DNS is broken, fast, slow, or pointing at the wrong place, the symptoms look like the network is broken. The fix is DNS-specific.

DNS is also a massive security surface. Every name resolution is an information leak (the resolver knows what you're looking up). Every name resolution is potentially a redirection target (cache poisoning, hijacking). Email security is built almost entirely on DNS records. DNS-over-HTTPS changes the trust model. DNSSEC fixes some problems and creates others.

This chapter goes from "DNS resolves names to addresses" to a working understanding of what DNS actually does, how it's used for security, and how it gets attacked. By the end you can answer "is DNS the problem here" with confidence and read DNS-related security configurations like DMARC records without panicking.

---

## What DNS actually does

DNS is a distributed database keyed on names. The most common query is "give me the IPv4 address for this name," but there are many other record types and use cases.

### The resolver chain, in detail

When your browser fetches `https://github.com`, the resolution flow:

1. **Browser asks the OS to resolve `github.com`.**
2. **OS checks its local cache.** Hit? Return immediately.
3. **Cache miss. OS asks the configured resolver** (your gateway, or 8.8.8.8, or your enterprise DNS server).
4. **Resolver checks its cache.** Hit? Return.
5. **Cache miss at the resolver. The resolver does recursive resolution:**
   - Ask a root server: "where do I find `.com` records?"
   - Root server returns: ".com is handled by these TLD servers."
   - Ask a TLD server: "where do I find `github.com` records?"
   - TLD server returns: "github.com is handled by these authoritative servers."
   - Ask an authoritative server: "what's the A record for `github.com`?"
   - Authoritative server returns the answer.
6. **Resolver caches the answer for the TTL and returns to the OS.**
7. **OS caches the answer and returns to the browser.**

The TTL (Time to Live) is included in every response and tells caches how long to remember it. Short TTLs (60 seconds) are flexible but increase resolver load. Long TTLs (24 hours) reduce load but make changes propagate slowly. Common values: 300 seconds (5 minutes) for active names, 3600 seconds (1 hour) for stable names.

You can do steps 4-5 manually:

```
# Linux
dig +trace github.com

# Walk the chain step by step:
dig @a.root-servers.net github.com    # ask the root
dig @192.5.6.30 github.com            # ask a .com TLD server
dig @ns1.p16.dynect.net github.com    # ask github's authoritative server
```

```powershell
# Windows
Resolve-DnsName github.com -Server 8.8.8.8 -Type A
```

The first dig with `+trace` does the full walk and shows you each step. Worth running once to see how the recursion actually happens.

### Record types you should know

The records that matter for working admin and security work:

| Type | Returns | Used for |
|---|---|---|
| A | IPv4 address | Most common lookup. Name to address. |
| AAAA | IPv6 address | Same as A but for IPv6. |
| CNAME | Another name | Aliasing. `www.example.com` is a CNAME for `example.com`. |
| MX | Mail server names | Where to deliver email for this domain. |
| TXT | Arbitrary text | Used for SPF, DKIM, DMARC, domain ownership verification. |
| NS | Name server names | Which DNS servers are authoritative for this domain. |
| SOA | Zone metadata | Start of Authority, includes serial number for change tracking. |
| PTR | Name | Reverse DNS. IP to name. |
| SRV | Service location | Used by services like SIP, XMPP, AD. Specifies host and port. |
| CAA | Certificate authority | Restricts which CAs can issue certs for the domain. |

You query specific record types:

```
# Linux
dig example.com A
dig example.com MX
dig example.com TXT
dig example.com NS

# Reverse lookup
dig -x 8.8.8.8
```

```powershell
# Windows
Resolve-DnsName example.com -Type A
Resolve-DnsName example.com -Type MX
Resolve-DnsName example.com -Type TXT
Resolve-DnsName example.com -Type NS

# Reverse lookup
Resolve-DnsName 8.8.8.8
```

### Reading dig output

`dig` output has several sections. Walk through one:

```
$ dig github.com

; <<>> DiG 9.18.1 <<>> github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;github.com.                    IN      A

;; ANSWER SECTION:
github.com.             60      IN      A       140.82.112.4

;; Query time: 12 msec
;; SERVER: 192.168.50.1#53(192.168.50.1)
;; WHEN: Tue Oct 14 10:23:14 EDT 2025
;; MSG SIZE  rcvd: 55
```

The pieces:

- **HEADER:** the query metadata. `status: NOERROR` is what you want; `NXDOMAIN` means the name doesn't exist; `SERVFAIL` means the resolver couldn't get an answer.
- **QUESTION:** what you asked for.
- **ANSWER:** what you got back. The `60` is the TTL; this record will be cached for 60 seconds.
- **AUTHORITY:** which servers are authoritative for the answer (sometimes empty).
- **ADDITIONAL:** extra records the server thought might be useful.
- **Footer:** which server you queried, when, and the response size.

The Answer's TTL is critical for diagnosis. If you see TTL = 60 on a name that's not changing often, the admin set a short TTL on purpose (probably for failover flexibility). If you see TTL = 86400, changes will take a day to propagate globally.

---

## DNS server types and roles

Three roles to understand:

**Authoritative servers** hold the actual records for a zone. When you register `example.com`, you specify which name servers are authoritative; those servers are queried to answer questions about `example.com`. Authoritative servers don't recursively resolve other names; they only answer for what they're authoritative for.

**Recursive resolvers** (sometimes called "caching resolvers") do the work of walking the resolution chain. When your machine asks for `github.com`, your configured resolver does the recursive walk and returns the answer. Examples: Google's 8.8.8.8, Cloudflare's 1.1.1.1, your home router, your enterprise DNS server.

**Forwarders** are partial resolvers. They forward queries to another resolver instead of doing recursion themselves. Common in enterprise environments: internal DNS servers forward unknown names to the internal "main" resolver, which forwards to the internet.

Your machine can be configured to use any of these as its resolver. Most often, you point at a recursive resolver (the gateway, an internal server, or a public service). Sometimes you point at an authoritative server directly for a specific lookup, which is what `dig @<server>` does.

---

## Caching and TTL behavior

Caching is what makes DNS scalable. Every step in the resolution chain caches answers, which is what makes the public DNS infrastructure capable of handling billions of queries per day without melting.

Caches at:

- Your machine's stub resolver (small cache, configurable on most OSes).
- Your local resolver (medium cache, often the gateway or enterprise DNS).
- Public resolvers (large cache, multi-region distributed).
- Browser internal cache (some browsers maintain their own).

When something is wrong with DNS, the cache layers can mask the change. You update the record, but your local resolver still returns the old answer until its TTL expires. This is "DNS hasn't propagated yet" in colloquial terms. The fix is waiting (the TTL eventually expires) or flushing caches at each layer.

To flush caches:

```
# Linux (depends on resolver)
sudo systemd-resolve --flush-caches    # systemd-resolved
sudo /etc/init.d/dnsmasq restart       # dnsmasq

# Windows
Clear-DnsClientCache
# Or legacy:
ipconfig /flushdns
```

Browser caches:

- Chrome: visit `chrome://net-internals/#dns` and click "Clear host cache."
- Firefox: about:networking#dns, click Clear.

Caching is also a security topic. A cached answer is a trusted answer; if an attacker can poison a cache, they can redirect traffic until the TTL expires. We come back to this in the attacks section.

---

## DNS over UDP, TCP, and the modern privacy protocols

DNS traditionally uses UDP port 53 for most queries because UDP is fast and DNS responses are small. UDP is used for queries up to 512 bytes; larger responses (like DNSSEC-signed responses or some TXT records) fall back to TCP port 53.

Modern privacy protocols:

**DNS over TLS (DoT):** DNS queries inside a TLS tunnel, port 853. Encrypts the queries so observers on the network can't see what you're looking up.

**DNS over HTTPS (DoH):** DNS queries inside HTTPS, port 443. Same privacy goal as DoT but harder to block (looks like normal HTTPS traffic). Used by Firefox by default in some configurations, by Chrome optionally, by Cloudflare's 1.1.1.1.

**DNS over QUIC (DoQ):** Newer, similar to DoT but using QUIC transport (the protocol behind HTTP/3).

For working admin work today: most DNS still goes over UDP/53 inside organizations because it's fast, simple, and the network is already trusted. DoT and DoH show up at the network edge (your home machine talking to 1.1.1.1 over DoH) and increasingly inside browsers.

The security implication: if your environment uses DNS filtering (blocking known-malicious domains at the resolver), DoH bypasses it because the queries don't go through your filter. Modern enterprise environments often disable DoH at the OS or browser level for this reason.

---

## DMARC, DKIM, SPF: email security via DNS

Email security is built on three DNS records that publishers add to their domains. Together they answer "is this email actually from this domain" for receiving mail servers.

### SPF: who's allowed to send mail from this domain

SPF (Sender Policy Framework) is a TXT record that lists which servers are authorized to send mail for the domain. Receiving mail servers check SPF when an email arrives.

Example SPF record:

```
$ dig example.com TXT

example.com. 3600 IN TXT "v=spf1 ip4:192.0.2.0/24 include:_spf.google.com -all"
```

Reading this:

- **v=spf1:** SPF version 1. Identifies this as an SPF record (which is just a TXT record with this format).
- **ip4:192.0.2.0/24:** servers in this CIDR range are authorized.
- **include:_spf.google.com:** also authorize whatever SPF includes from Google's record (because this domain uses Google for some mail).
- **-all:** any other server is unauthorized. The `-` prefix means "fail" (reject the email). `~all` means "soft fail" (mark as suspicious but accept).

When mail arrives at a receiving server, it checks the sender's IP against the SPF record. If the IP matches, SPF passes. If it doesn't match and the policy is `-all`, SPF fails.

SPF alone has weaknesses: it only checks the envelope sender (the SMTP-level "MAIL FROM"), not the visible "From:" header users see. An attacker can forge the From header without affecting SPF.

### DKIM: cryptographic signing of mail

DKIM (DomainKeys Identified Mail) signs outbound mail with a private key; the corresponding public key is published in DNS. Receivers verify the signature against the published key.

The DKIM record is a TXT record at a specific subdomain (called a "selector"):

```
$ dig selector1._domainkey.example.com TXT

selector1._domainkey.example.com. 3600 IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4..."
```

Reading this:

- **v=DKIM1:** DKIM version 1.
- **k=rsa:** the key type (RSA).
- **p=...:** the public key in base64.

When an outbound mail server sends mail, it adds a DKIM-Signature header that includes the selector name, the algorithm, and a cryptographic signature over selected headers and the message body. The receiver looks up the public key (using the selector and domain), verifies the signature, and concludes whether the mail was actually sent by an authorized sender for this domain (whoever has the private key).

DKIM proves "this message came from someone with the private key" but doesn't say anything about the visible From: header by itself.

### DMARC: the policy that ties them together

DMARC (Domain-based Message Authentication, Reporting, and Conformance) is the policy record that tells receivers what to do when SPF and/or DKIM fail. DMARC also requires "alignment": the domain in the visible From: header must match (or align with) the domain that passed SPF or DKIM. This is what makes DMARC actually prevent spoofing.

The DMARC record is a TXT record at `_dmarc.<domain>`:

```
$ dig _dmarc.example.com TXT

_dmarc.example.com. 3600 IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com; ruf=mailto:dmarc-forensics@example.com; pct=100; aspf=s; adkim=s"
```

Reading this:

- **v=DMARC1:** DMARC version.
- **p=quarantine:** if a message fails DMARC, quarantine it (put in spam). Other options: `none` (just monitor, don't act), `reject` (block entirely).
- **rua=mailto:...:** send aggregate reports here. These are summary reports from receiving mail servers about what they saw.
- **ruf=mailto:...:** send forensic reports here. These are sample failed messages.
- **pct=100:** apply policy to 100% of failing messages. Lower percentages are used during DMARC rollouts.
- **aspf=s:** strict SPF alignment (the domain must exactly match, not just a parent domain).
- **adkim=s:** strict DKIM alignment.

DMARC starts at `p=none` for monitoring (collect data without affecting delivery), then moves to `p=quarantine` (mark failures as suspicious), then to `p=reject` (block failures entirely). The progression takes weeks to months in production; you don't jump straight to reject because legitimate mail sources you didn't know about will fail until you fix them.

### Why this matters for working admins

DMARC/DKIM/SPF are direct Sec+ content. They're also direct job content: when an admin asks "why is our mail going to spam at customer X" or "we got a phishing email pretending to be from us," the answer almost always involves these three records.

The skill: read a domain's DMARC/DKIM/SPF records and explain what they enforce. Practice on real domains:

```
# A domain with mature DMARC
dig google.com TXT | grep spf
dig _dmarc.google.com TXT
dig google.com MX

# A small business that might not have DMARC
dig <some-small-domain> TXT | grep -i spf
dig _dmarc.<some-small-domain> TXT
```

You'll find that most large organizations have aggressive DMARC (`p=reject`), some mid-sized organizations have `p=quarantine`, and many small businesses have either no DMARC or just `p=none`. The pattern of adoption is itself informative.

---

## DNS attacks and defenses

DNS is a high-value target for attackers because controlling DNS means controlling where users go. The major attack categories:

### Cache poisoning

The classic DNS attack. The attacker convinces a recursive resolver to cache a fake answer. Future queries return the attacker's answer until the TTL expires.

The Kaminsky attack (2008) showed that cache poisoning was much easier than previously thought. The fix was randomizing source ports for resolver queries (so attackers can't easily forge responses). Most modern resolvers also support DNSSEC, which cryptographically signs answers and prevents acceptance of forged responses.

### DNS hijacking

The attacker takes control of the authoritative server (or registrar account) for a domain and changes records. Different from cache poisoning: the attacker isn't faking responses, they actually own the records temporarily.

Defense: registrar lock, two-factor authentication on registrar accounts, monitoring for unauthorized record changes.

### DNS tunneling

Encoding data inside DNS queries and responses to bypass network egress filtering. Useful for attackers because most networks allow DNS out by default.

The pattern: attacker controls a domain. Compromised host on the inside makes DNS queries with the data encoded in subdomain labels (e.g., `BASE64ENCODED.attacker.com`). The attacker's authoritative server receives the query, decodes the data, and can respond with more data encoded in TXT records.

Defense: DNS query log monitoring (look for unusually long queries or unusually high query volume to specific domains), DNS filtering services, DPI (Deep Packet Inspection) firewalls that look inside DNS traffic.

### DGA (Domain Generation Algorithms)

Malware uses an algorithm to generate hundreds or thousands of random-looking domain names per day. The attacker registers one or two of those names. The malware tries each generated name until it gets a response, contacting the C2 (command and control) server.

Defense: detection of DGA-style domains in logs (long random-looking strings). Many security products do this automatically.

### Fast flux

The attacker uses a DNS record with very short TTL pointing at a rapidly rotating set of compromised hosts. As soon as defenders block one IP, the next query returns a different one.

Defense: pattern recognition in DNS logs, threat intelligence feeds that identify fast-flux infrastructure.

### Typosquatting and homograph attacks

Not strictly DNS attacks, but DNS-adjacent. The attacker registers a domain similar to a legitimate one (`paypa1.com` vs `paypal.com`, or homograph using Cyrillic characters that look like Latin). Users mistype or misread, end up at the attacker's site.

Defense: user awareness, browser/email warnings about suspicious domains, DNS filtering services that block known-malicious lookalikes.

### What DNSSEC does

DNSSEC (DNS Security Extensions) cryptographically signs DNS records. Resolvers can verify that an answer came from the authoritative source and wasn't tampered with.

DNSSEC prevents cache poisoning and hijacking-via-resolver. It does not encrypt DNS traffic (that's DoT/DoH). It does not prevent DGA, tunneling, or hijacking-via-registrar.

DNSSEC adoption is mixed. Major TLDs have DNSSEC. Some country code TLDs lag behind. Many domain registrars don't make DNSSEC easy. Recursive resolvers vary in whether they validate signatures (Google's 8.8.8.8 does, Cloudflare's 1.1.1.1 does, your home router probably doesn't).

For Sec+ purposes: know that DNSSEC exists, what it protects (integrity), what it doesn't (confidentiality, registrar attacks). For job purposes: if you're configuring DNS, enable DNSSEC if your registrar supports it. It's not a silver bullet but it raises the bar.

---

## DNS filtering as a security control

Many organizations and services filter DNS at the resolver level: blocking known-malicious domains by returning NXDOMAIN or a sinkhole address.

Examples:

- **Cisco Umbrella (formerly OpenDNS):** commercial DNS filtering with threat intelligence.
- **Quad9 (9.9.9.9):** free, blocks known-malicious domains based on community threat intel.
- **Pi-hole:** open-source self-hosted DNS filter, often used in homes for ad-blocking and basic threat blocking.
- **DNS-based parental controls:** various consumer offerings.

The mechanism: when your machine queries a blocked domain, the filter returns NXDOMAIN (the name doesn't exist) or returns a sinkhole IP (an internal page that says "blocked"). The malicious domain is never reached.

This is one of the highest-value security controls per dollar in many environments because the threat intel is constantly updated and the latency overhead is minimal.

The attacker's response: DoH bypasses DNS filtering by encrypting queries to a different resolver. This is part of why DoH is contentious in enterprise environments.

---

## DNS in the diagnostic pattern

Looping back to Chapter 3: when DNS is the problem in your diagnostic walk, what does it look like?

**Symptom: name resolution fails, but you can ping the IP if you know it.** That's classic DNS broken. Run `dig <name>` and `Resolve-DnsName <name>` to see what's happening:

- **Times out:** your resolver isn't responding. Try a different resolver (`dig @1.1.1.1 <name>`) to confirm.
- **Returns SERVFAIL:** your resolver got an error from upstream. Could be DNSSEC validation failure, could be the authoritative servers are down, could be your resolver has a problem.
- **Returns NXDOMAIN:** the name doesn't exist (according to whoever you asked). Could be a typo, could be the domain expired, could be your view of DNS is different from the public view (split DNS).

**Symptom: name resolves but to the wrong IP.** Cache poisoning, hijacking, or split DNS misconfiguration. Compare what `dig` returns to what an external resolver returns:

```
dig <name>                    # what your local resolver says
dig @1.1.1.1 <name>           # what Cloudflare's public resolver says
```

If they differ, something is interesting. Maybe you have legitimate split DNS (internal name resolves differently inside the corporate network). Maybe you're seeing an attack.

**Symptom: name resolves slowly.** Your resolver might be doing recursion every time (no caching). Check resolver configuration. Or your resolver might be far away (high latency to the resolver itself).

---

## A practical exercise

Pick three real domains you use regularly: a large tech company, a mid-sized business, a small personal site if you have one.

For each:

1. Query A, MX, and TXT records. Read the output.
2. Query the SPF record (it's a TXT record starting with `v=spf1`). Decode it: which servers can send mail for this domain?
3. Query the DMARC record (`_dmarc.<domain>` TXT). What's the policy? `none`, `quarantine`, or `reject`?
4. Compare the maturity of email security across the three. Which has the strongest DMARC?
5. Run `dig +trace <domain>`. How many steps did the recursive resolution take? Note the authoritative servers at the bottom.
6. Use AI to ask "What would happen if a phishing email pretended to be from <domain>?" Verify the answer against the domain's actual DMARC record.

Submit your answers (or just keep them; the exercise is for your own understanding).

---

## Common stumbling blocks

> **`dig` returns no answer for a name I know exists.**
> Two possibilities. First, your default resolver might be broken; try `dig @8.8.8.8 <name>`. Second, the name might exist but not have the record type you queried; try `dig <name> ANY` to get all record types.

> **`Resolve-DnsName` works but `dig` doesn't, or vice versa.**
> Different default resolvers. Windows and Linux can have different DNS configurations. Check what each is using as its default.

> **My `dig +trace` output looks weird with extra hops.**
> Some networks intercept DNS queries (corporate networks, hotel WiFi, etc.) and inject their own responses. The trace might show the interception. If you see results that don't match what `dig @<authoritative-server>` returns, suspect interception.

> **The TTL in my `dig` answer keeps decreasing.**
> That's expected. The answer was cached at some point with the original TTL; subsequent queries return the cached answer with the remaining TTL. When you see TTL = 0 or close to 0, the cache is about to expire and the next query will trigger a fresh resolution.

> **DMARC says `p=reject` but I'm still receiving spoofed mail from this domain.**
> Most likely the receiving mail server isn't enforcing DMARC. Many smaller mail systems still don't. DMARC is only as strong as the receivers honoring it; major providers (Gmail, Outlook 365) do, smaller systems sometimes don't.

> **I can resolve internal names but not external, or vice versa.**
> Split-horizon DNS configuration issue. Your machine is using a resolver that handles one set but not the other. Check resolv.conf or DNS server config.

> **My machine ignores `/etc/resolv.conf` changes on Linux.**
> Modern Linux distributions often have `systemd-resolved` or `NetworkManager` managing DNS. Edits to `/etc/resolv.conf` get overwritten. The fix depends on which resolver: `resolvectl dns <interface> <server>` for systemd-resolved, NetworkManager GUI for NetworkManager, or edit the source config rather than `/etc/resolv.conf`.

---

## What this gets you

After this chapter:

- You understand the DNS resolver chain and can walk it manually with `dig +trace`.
- You know the major record types and what each is for.
- You can read DMARC, DKIM, and SPF records and explain what they enforce.
- You recognize the major DNS attack categories and what defenses exist for each.
- You can diagnose DNS-specific problems within the broader network diagnostic pattern.
- You know what DNSSEC provides and what it doesn't.
- You understand DNS filtering as a security control and why DoH bypasses it.

The DMARC/DKIM/SPF section is where this chapter pays off most for Sec+ alignment. The exam tests these directly. Real working knowledge of these (not flashcard memorization) is also one of the most valuable email-security skills you can have.

---

## What's next

Chapter 5 is packet capture. We've talked about queries and responses; the next chapter is where we look at what's actually on the wire. Wireshark, tcpdump, Pktmon. Reading TCP handshakes. Recognizing TLS. By the end of Chapter 5 you can capture network traffic and pull useful information from it.
