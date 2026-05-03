# Chapter 8: Host Networking Tools

**You come in with:** ability to SSH and `curl`. You know the box has an IP address. You probably do not know how to find it.
**You leave with:** the ability to read a host's network configuration, troubleshoot "the network is broken from this box" with a repeatable four-step pattern, and capture packets when the symptom needs more depth than `curl -v` provides.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 4.4 (alerting and monitoring tools and concepts: scanning, packet captures). Domain 4.5 (modify enterprise capabilities to enhance security: firewall configuration, port and protocol selection). Domain 3.4 (security implications of architecture: network segmentation). Domain 4.9 (data sources to support an investigation: network logs, traffic flows). The four-step troubleshooting pattern in this chapter is the practical version of the layer-by-layer model the cert tests at the conceptual level.

---

## Why this chapter matters

This unit was originally going to defer most networking content to a separate Networking unit. The Networking unit was redesigned without a firewall lab, which means the host-side networking skills need to live somewhere. They live here.

Beyond the unit's structure, the practical case is strong. Half of all "the application is broken" tickets a junior admin gets turn out to be network problems: DNS not resolving, the firewall blocking the connection, the wrong network interface being used, a bad route. Without the host-side tools, every one of these tickets becomes "open a help desk ticket with the network team." With them, you solve most of them yourself in five minutes.

This chapter goes a little deeper than typical Linux intros. Worth it for this audience.

---

## The four tools

Most network problems on a host can be diagnosed with four tools:

- `ip`: shows and modifies network interfaces, addresses, and routes.
- `ss`: shows network sockets, including listening ports and active connections.
- `dig`: queries DNS.
- `curl`: makes HTTP requests with detailed output.

A fifth, `tcpdump`, is for the cases where the four tools above are not enough. We cover it later in the chapter.

The legacy tools `ifconfig`, `netstat`, `route`, and `nslookup` still exist on some systems. They are deprecated. Tutorials that use them are still common. Use the modern tools; they are better and they are what production environments standardize on.

---

## ip: addresses, links, and routes

The `ip` command is the entry point to network configuration. It has subcommands: `ip addr`, `ip link`, `ip route`, `ip neigh`, and others. Each does one thing.

### ip addr: what addresses do I have

```
ip addr
ip a              # short form
```

Output on a typical lab VM:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

Reading this:

- Interface 1 is `lo`, the loopback. Always present. Address `127.0.0.1`.
- Interface 2 is `eth0`, the primary network interface. Address `172.17.0.2/16`.
- The `link/ether` line shows the MAC address (the interface's hardware identifier).
- The `inet` line shows the IPv4 address with its CIDR mask (`/16` is a 65536-host network).

The `/16` notation is CIDR. *CIDR notation is a way to write a network mask as the number of leading 1 bits. `/16` means the first 16 bits of the address identify the network, and the remaining 16 bits identify the host within the network.* `/24` is the most common in small networks (256 addresses); `/16` is common in larger ones.

### ip link: physical and logical interfaces

```
ip link
```

This is `ip addr` without the addresses, showing just interface state. Useful for "is this interface up" and "what is the MAC address." For most beginners, `ip addr` covers what `ip link` covers plus more, but `ip link` is the right command when you specifically want to enable/disable an interface:

```
sudo ip link set eth0 down
sudo ip link set eth0 up
```

### ip route: where does traffic go

```
ip route
ip r              # short form
```

Output:

```
default via 172.17.0.1 dev eth0 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2 
```

Reading this:

- `default via 172.17.0.1 dev eth0` is the default route. Anything not matching a more specific route goes here.
- `172.17.0.0/16 dev eth0 proto kernel scope link` is the route for the local network. Anything in 172.17.0.0/16 is reached directly via eth0.

The default route is the most important thing in this output. If `ip route` does not show a default route, the box cannot reach anything outside its local subnet. That is one of the first things to check when "the network is broken."

### ip neigh: who does my host think the neighbors are

```
ip neigh
```

Output:

```
172.17.0.1 dev eth0 lladdr 02:42:99:88:77:66 REACHABLE
```

This is the ARP table: the mapping of IPs to MAC addresses that the kernel has learned for the local subnet. *ARP, the Address Resolution Protocol, is how a host finds the MAC address of an IP on the same network.* For most troubleshooting you do not need this, but when something is wrong at the local-network layer, `ip neigh` shows whether the kernel has learned the MAC of the gateway.

### Setting an address temporarily

```
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip addr del 192.168.1.10/24 dev eth0
```

These changes do not persist across reboots. For permanent changes on Ubuntu, use netplan (covered later in this chapter). The `ip` command is for testing or temporary configuration.

---

## ss: sockets, listening ports, and active connections

`ss` (socket statistics) is the modern replacement for `netstat`. It is faster (it reads kernel socket tables directly rather than parsing `/proc/net/`), and it has cleaner filter syntax.

### ss for listening ports

The most common use:

```
ss -tlnp
```

Reading the flags:

- `-t`: TCP only.
- `-l`: listening sockets only.
- `-n`: do not resolve port numbers to service names.
- `-p`: show the process owning each socket (requires root for non-own processes).

Output:

```
State    Recv-Q   Send-Q   Local Address:Port    Peer Address:Port   Process
LISTEN   0        128      0.0.0.0:22            0.0.0.0:*           users:(("sshd",pid=842,fd=3))
LISTEN   0        511      0.0.0.0:80            0.0.0.0:*           users:(("nginx",pid=1234,fd=6))
LISTEN   0        128      [::]:22               [::]:*              users:(("sshd",pid=842,fd=4))
```

Reading this:

- ssh is listening on TCP port 22, on every interface (`0.0.0.0`).
- nginx is listening on TCP port 80, on every interface.
- ssh also listens on IPv6 (`[::]`).

This single command answers "what is listening on this box and what process owns it." It is a fundamental security audit step.

### ss for active connections

```
ss -tunap
```

`-u` adds UDP. `-a` includes both listening and established. The output now shows active connections in addition to listeners. Useful for "what is this box talking to right now."

### Filtering by state

```
ss state established
ss state listening
ss state time-wait
```

Each shows sockets in a specific TCP state.

### Counting connections

```
ss -tn state established | wc -l
```

That counts the established TCP connections. Useful for "how many things are talking to this box."

### Why ss is better than netstat

`ss` is roughly 10-100x faster on systems with many connections, because it queries the kernel directly. On a busy server, `netstat -an` can take seconds; `ss -an` is instant. If you find tutorials that still use `netstat`, mentally substitute `ss`.

---

## dig: when DNS is the problem

`dig` (domain information groper) queries DNS servers. It replaces `nslookup`, which still exists but is less informative.

### Basic lookup

```
dig example.com
```

Output (abridged):

```
;; ANSWER SECTION:
example.com.        2390    IN  A   93.184.216.34

;; Query time: 12 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
```

Reading this:

- `example.com` resolves to `93.184.216.34`.
- TTL of 2390 seconds. Cache lifetime.
- The server that answered was `127.0.0.53`, which is systemd-resolved (the local DNS resolver on Ubuntu).

### Useful flags

```
dig +short example.com
```

Just the answer, no metadata. Best for scripting.

```
dig example.com MX
dig example.com NS
dig example.com TXT
```

Query specific record types. MX for mail, NS for nameservers, TXT for text records.

```
dig @8.8.8.8 example.com
```

Query a specific DNS server (here, Google's 8.8.8.8). Useful for "is this a DNS issue at my resolver, or upstream?" If the local resolver returns one answer and Google returns another, your local resolver has stale or misconfigured data.

```
dig +trace example.com
```

Show the full resolution path from the root nameservers down. Verbose, but the right tool for "where is this resolution failing."

### The local resolver chain on Ubuntu

Ubuntu uses systemd-resolved as the default DNS resolver, which lives at `127.0.0.53`. It receives queries from applications, looks them up using a configured set of upstream DNS servers, and caches the results.

```
resolvectl status
```

That shows the current DNS configuration: which interfaces use which DNS servers, what the search domains are, and what the current state is. Useful for "is DNS even configured on this box."

The `/etc/resolv.conf` file you may have seen in older tutorials is now a symlink to a file managed by systemd-resolved. Editing it directly does not work the way it used to.

---

## curl: when HTTP is the problem

`curl` makes HTTP requests. The verbose flags are what make it a network diagnostic tool, not just a download tool.

### Verbose output

```
curl -v https://example.com
```

Output (abridged):

```
*   Trying 93.184.216.34:443...
* Connected to example.com (93.184.216.34) port 443
* TLSv1.3 handshake complete
> GET / HTTP/1.1
> Host: example.com
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Content-Type: text/html
< Content-Length: 1256
< 
<!doctype html>...
```

Reading this:

- DNS resolved (`93.184.216.34`).
- TCP connection succeeded.
- TLS handshake succeeded.
- Request was sent.
- Response came back: 200 OK with the HTML.

Each step that succeeds rules out that layer as the problem. If `curl -v` fails at "trying", DNS is fine but the connection is not happening. If it fails at "TLS handshake," DNS and TCP are fine but TLS is not.

### Headers only

```
curl -I https://example.com
```

Just the response headers. Fast check for "is the server responding at all."

### Following redirects

```
curl -L https://example.com
```

By default, curl does not follow HTTP redirects. `-L` makes it follow them. Useful when the URL redirects and you want the final response.

### Ignoring TLS errors during testing

```
curl -k https://localhost
```

`-k` skips TLS certificate validation. Useful for testing a service with a self-signed certificate. **Never use `-k` in production scripts.** The `k` is for "kindly ignore the security warning" and is the antipattern equivalent to chmod 777 in network testing.

### Output to stdin/files

```
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
```

Reading this:

- `-s` silences progress output.
- `-o /dev/null` discards the body.
- `-w "%{http_code}\n"` writes only the HTTP status code.

Output: `200`. That is "is this URL alive" condensed to one number. Useful in scripts and monitors.

---

## The four-step diagnosis pattern

When something on the box "cannot reach a service," the diagnosis is the same every time:

**Step 1. Can I reach the host at all? (`ping`)**

```
ping -c 3 example.com
```

The `-c 3` sends three packets and stops. If pings succeed, layer 3 (the IP layer) is working. If they fail with "Name or service not known," DNS is broken. If they fail with "Destination Host Unreachable" or just timeout, routing or a firewall is blocking.

**Step 2. Is DNS working? (`dig`)**

```
dig example.com
```

If `dig` returns an answer, DNS is fine. If not, the resolver is the problem. Try `dig @8.8.8.8 example.com` to confirm whether your resolver is the issue specifically (vs an upstream DNS issue).

**Step 3. Can I open a TCP connection to the right port? (`nc` or `curl`)**

```
nc -vz example.com 443
curl -v https://example.com
```

`nc -vz` (netcat in verbose, scan mode) just opens a TCP connection to confirm the port is reachable. If it succeeds, the network path to that port is clear. If it fails, a firewall is blocking, the service is not listening, or the port is wrong.

**Step 4. Is the application responding correctly? (`curl -v`)**

```
curl -v https://example.com
```

If curl gets a connection but the response is wrong (500, weird content, no response), the application is the problem, not the network.

The pattern: each step rules out one layer. ICMP (ping) tests routing. DNS (dig) tests name resolution. TCP (nc) tests transport. HTTP (curl) tests the application. Whatever fails first identifies which layer to investigate.

This is one of the most useful patterns in working with Linux. Practice it on real targets a few times and it becomes muscle memory.

---

## tcpdump: when curl is not enough

When the four-step pattern says "the connection completes but the response is weird," sometimes you need to see the actual packets. `tcpdump` is the tool.

### Basic capture

```
sudo tcpdump -i any -n port 80
```

Reading the flags:

- `-i any`: capture on all interfaces.
- `-n`: do not resolve IPs to names (faster, less noisy).
- `port 80`: only show traffic on TCP/UDP port 80.

The output is a stream of packet headers, one line each. Press Ctrl-C to stop.

### Useful filters

```
sudo tcpdump -i eth0 -n host 8.8.8.8
sudo tcpdump -i eth0 -n 'port 53'
sudo tcpdump -i eth0 -n 'tcp port 443'
sudo tcpdump -i eth0 -n 'src 192.168.1.10 and dst port 443'
```

The filter syntax is BPF (Berkeley Packet Filter). You can combine: `host`, `port`, `src`, `dst`, `tcp`, `udp`, `icmp`, with `and`/`or`/`not`.

### Capturing to a file for later analysis

```
sudo tcpdump -i any -n -w /tmp/capture.pcap port 443
```

`-w` writes the raw packet data to a pcap file instead of decoding to text. The pcap file can be opened in Wireshark for deeper analysis. This is the right pattern when you have a transient issue: capture broadly to a file, stop the capture once the issue happens, analyze the file at leisure.

### What you see

A typical tcpdump line:

```
14:23:01.234567 IP 172.17.0.2.54321 > 93.184.216.34.443: Flags [S], seq 1234, win 64240, length 0
```

Reading this:

- `14:23:01.234567` is the timestamp.
- `172.17.0.2.54321 > 93.184.216.34.443` is source > destination, with port numbers.
- `Flags [S]` is a SYN packet (start of TCP handshake).
- The rest is sequence numbers and window size.

You do not need to read every packet to use tcpdump effectively. You need to see whether packets are flowing and in which direction. "I sent a SYN but never got a SYN-ACK back" is "the firewall is blocking outbound" or "the server is not listening."

### tcpdump versus Wireshark

tcpdump is the command-line tool. Wireshark is a graphical tool that does the same job with deeper analysis features. For routine work on remote servers, tcpdump is what you have. For deep analysis, capture with tcpdump (`-w` to a file), copy the file to your laptop, open in Wireshark.

For this chapter, recognition-level familiarity with tcpdump is enough. Chapter 11 (and the intermediate cohort) goes deeper.

---

## netplan: configuring the network on Ubuntu

Ubuntu Server uses netplan to configure interfaces. *netplan is a YAML-based network configuration tool that translates declarative config into the active configuration for whichever backend (systemd-networkd or NetworkManager) the system uses.*

The config files live in `/etc/netplan/`:

```
ls /etc/netplan/
sudo cat /etc/netplan/*.yaml
```

A typical cloud-init-generated config:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```

That says "interface eth0 should use DHCP." That is it.

A static IP config looks like:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.10/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

To apply changes:

```
sudo netplan try
```

`try` applies the config, gives you 120 seconds to confirm it works (by pressing Enter), and reverts if you do not. This is the safety net for "I changed the network config and now I cannot SSH in." Always use `try` on a remote box. Never use `apply` directly until you have tested on console first.

```
sudo netplan apply
```

`apply` commits the change immediately. Use only when you are certain.

### Permission notes

Netplan files in some Ubuntu installs are world-readable, which can leak information about network configuration. Hardening tools recommend:

```
sudo chmod 600 /etc/netplan/*.yaml
```

We come back to this in Chapter 12.

---

## ufw: the host firewall front-end

Ubuntu ships with `ufw` (Uncomplicated Firewall), a friendly front-end for iptables/nftables. *ufw is a tool that translates simple commands like "allow port 22" into the underlying firewall rules.*

By default, ufw is installed but inactive. To enable:

```
sudo ufw status
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
```

Reading this sequence:

1. Check current status.
2. Enable ufw.
3. Set default policy: deny incoming, allow outgoing.
4. Explicitly allow SSH (port 22).
5. Explicitly allow HTTP and HTTPS.
6. Show the result with detail.

The pattern is "default deny, then explicitly allow what is needed." This matches Security+ Domain 4.5 (firewall configuration) and is the standard host-firewall approach.

### A critical warning

If you enable ufw on a remote box and forget to allow SSH, your next reconnect attempt will fail. Always allow SSH before enabling ufw. The order matters:

```
sudo ufw allow ssh
sudo ufw enable
```

Some versions of ufw catch this and warn you. Do not rely on the warning.

### Removing rules

```
sudo ufw status numbered
sudo ufw delete 3
```

Removes rule number 3 from the list shown by `status numbered`.

### When ufw is not enough

ufw covers the common cases. For complex needs (rate limiting beyond the basic, NAT, complex match conditions), you go to nftables directly. That is past beginner scope.

---

## A complete diagnosis: walking the four-step pattern

A scenario worth walking. The application on a box cannot reach a database server at `db.example.com:5432`. Walk the diagnosis.

```
# Step 1: can we reach the host?
ping -c 3 db.example.com

# Step 2: does DNS resolve?
dig +short db.example.com

# Step 3: can we open a TCP connection to the port?
nc -vz db.example.com 5432

# Step 4: does the application protocol work?
# (Postgres-specific; this would be psql in real use)
```

The result of each step:

- If step 1 fails with "Name or service not known," DNS is broken. Skip to step 2.
- If step 1 fails with timeouts, network or firewall. Try `traceroute db.example.com` to see where it stops.
- If step 2 returns no answer, DNS is broken. Try `dig @8.8.8.8 db.example.com` to see if upstream DNS has it.
- If step 3 fails with "Connection refused," the host is reachable but nothing is listening on that port. Either the service is down or you have the wrong port.
- If step 3 fails with timeouts, a firewall (host or network) is blocking.
- If steps 1-3 succeed and step 4 fails, the network is fine; the problem is the application.

This pattern is the entirety of host-side network troubleshooting. Memorize it. Run it for everything. Most tickets resolve in the first 30 seconds of running through it.

---

## Try this

**1. Map the network configuration of your lab box.**

Run:

```
ip addr
ip route
resolvectl status
ss -tlnp
```

Without running anything else, write down: what is the IP address of the box, what is the default gateway, what DNS server is being used, and what TCP ports is the box listening on.

**2. Walk the four-step diagnosis on a known-good target.**

Pick a public service you know works (e.g., `google.com` on port 443). Run all four steps:

```
ping -c 3 google.com
dig +short google.com
nc -vz google.com 443
curl -v -o /dev/null https://google.com 2>&1 | head -20
```

Confirm each step succeeds. Read the output carefully. The point is to know what success looks like before you have to recognize failure.

**3. Walk the four-step diagnosis on a known-broken target.**

Pick a target that will fail. `ping -c 3 192.168.99.99` (a private IP that probably is not on your network). Or `dig +short does-not-exist-for-real-12345.example` (a name that does not resolve). Walk through the four steps and observe what each failure mode looks like.

**4. Capture some packets.**

Open two terminals. In one, run:

```
sudo tcpdump -i any -n port 53
```

In the other:

```
dig example.com
```

Watch the tcpdump terminal show the DNS query going out and the response coming back. This is the moment "what is happening on the network" stops being abstract.

**5. Configure ufw safely.**

On your lab box, with you SSHed in: run `sudo ufw status`. If it is inactive, do this carefully:

```
sudo ufw allow ssh        # CRITICAL: do this first, before enable
sudo ufw enable
sudo ufw status verbose
```

Confirm SSH still works (open a new SSH session in another terminal). Then:

```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status numbered
```

When you are done practicing:

```
sudo ufw disable
```

Or leave ufw enabled if you want it that way. The point of the exercise is the procedure, not the end state.

---

## Common stumbling blocks

> **`ip addr` shows my interface but no IPv4 address.**
> Either DHCP did not get an answer, or the interface is not configured. Check `journalctl -u systemd-networkd -n 50` (if using systemd-networkd) or `journalctl -u NetworkManager` (if using NetworkManager) for errors. On a cloud VM, this almost always means cloud-init did not finish; reboot may resolve it.

> **`ss -tlnp` does not show process names.**
> ss can only show process info if you have permission to read the socket's owning process. As a non-root user you see your own processes; for a complete picture, run with sudo.

> **`dig` returns NXDOMAIN but a browser can resolve the name.**
> Different resolvers may have different views, especially for internal/private domains. Try `dig @<your-corporate-DNS> hostname` if your network has internal DNS. Also check if the hostname is in `/etc/hosts`, which dig does not consult by default; use `getent hosts hostname` to see what the system would actually resolve.

> **`curl -v` connects but hangs forever after the request.**
> Either the server is not responding, or there is a middlebox (proxy, WAF, load balancer) that is dropping the connection. Try `curl -v --max-time 10` to bound the wait. The verbose output will show whether you got bytes back at all.

> **netplan apply succeeded but the network broke.**
> If you are SSHed in remotely, you may have lost connectivity at this point. The reason `netplan try` exists is that you can revert in 120 seconds without console access. Always `try` on a remote box. If you used `apply` and lost connectivity, you need console access (cloud serial console, IPMI, or physical) to fix it.

> **ufw is enabled but my service is still reachable from the internet.**
> Two possibilities. First, ufw is enabled but you also added an `allow` rule for that service. Check with `ufw status verbose`. Second, on cloud platforms, the cloud's network firewall (security groups in AWS, network security groups in Azure) is what actually controls inbound traffic to the VM; the host's ufw is layered on top. The cloud-level firewall may be allowing what you thought you blocked.

> **`tcpdump` shows no traffic when I expect some.**
> Check the interface name (`-i any` is safest), check the filter (a too-restrictive filter shows nothing), and check that the traffic is actually happening (run the test that should generate traffic in another terminal). On switched networks, you only see traffic that the switch sends to your interface; you would need a SPAN port to see traffic to other hosts.

---

## What this gets you

After this chapter:

- You can read a host's network configuration in 30 seconds (`ip addr`, `ip route`, `resolvectl status`).
- You can identify what is listening on a box and what process owns each socket.
- You can troubleshoot "the network is broken" with the four-step pattern.
- You can capture packets when the symptom needs more depth.
- You can configure a static IP with netplan, with the safety of `netplan try`.
- You can configure a host firewall with ufw, with the discipline of "allow SSH before enable."
- You know that DNS is its own subsystem and that resolution can fail independently of network connectivity.

The four-step diagnosis pattern (ping > dig > nc > curl) is the practical skill from this chapter. Memorize it. Use it on every "the network is broken" ticket. It will resolve most of them in the first 30 seconds and tell you which team to escalate to for the rest.

---

## What's next

Chapter 9 is Shell scripting. The chapter where you stop typing the same three commands repeatedly and start saving them to files. By the end you can write a wrapper script with proper error handling, and read someone else's script (including a slightly suspicious one) and figure out what it does.
