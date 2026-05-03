# Chapter 8: Host Networking on Windows

**You come in with:** workshop-level Get-NetTCPConnection and Resolve-DnsName, plus the four-step network diagnosis pattern from the Linux unit.
**You leave with:** the ability to read a Windows host's network configuration, troubleshoot "the network is broken from this box" with the four-step pattern translated to Windows tools, and configure Windows Defender Firewall via PowerShell rather than the GUI.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 4.4 (alerting and monitoring tools and concepts: scanning, traffic analysis). Domain 4.5 (modify enterprise capabilities to enhance security: firewall configuration, port and protocol selection). Domain 3.4 (security implications of architecture: network segmentation). Domain 4.9 (data sources to support an investigation: network logs, traffic flows). The four-step troubleshooting pattern in this chapter is the Windows version of the layer-by-layer model the cert tests at the conceptual level.

---

## Why this chapter matters

Half of "the application is broken" tickets a junior admin gets are network problems. On Windows, the diagnostic toolkit looks superficially different from Linux but follows the same shape: tools to read the config, tools to test each layer, tools to control the firewall. Once you know the Windows commands, the underlying skill is the same skill you built on Linux.

The Windows specifics matter because most working environments are Windows-shop-heavy. A help desk technician who can troubleshoot Linux but not Windows is unhelpful for 80% of tickets. The cert tests both platforms at the conceptual level; this chapter gives you the practical Windows fluency.

---

## The PowerShell networking modules

Windows has a set of networking modules built into PowerShell. The major ones:

- **NetAdapter**: physical and logical network interfaces.
- **NetTCPIP**: IP addresses, routes, IP-level configuration.
- **NetTCPConnection** (in NetTCPIP): TCP and UDP connections.
- **DnsClient**: DNS resolution.
- **NetSecurity**: Windows Defender Firewall.
- **NetConnection** (the Test-NetConnection cmdlet): connectivity testing.

These replace older tools like `ipconfig`, `route`, `netstat`, and `ping`. The legacy tools still work; the PowerShell cmdlets are the modern path and the one that scales to scripting.

Try a quick inventory:

```
Get-Module -ListAvailable Net*, Dns* | Select-Object Name
```

You see the modules available. They load on demand when you use commands from them.

---

## Network adapters and addresses

The Windows equivalent of Linux's `ip addr`.

### Get-NetAdapter

```
Get-NetAdapter
```

Output:

```
Name              InterfaceDescription          ifIndex Status     MacAddress           LinkSpeed
----              --------------------          ------- ------     ----------           ---------
Ethernet          Microsoft Hyper-V Network ...      4 Up         00-15-5D-12-34-56    10 Gbps
```

`Status` should be `Up` for active interfaces. `Name` is the friendly name; `InterfaceDescription` is the driver-level name; `ifIndex` is the integer ID used by other cmdlets to reference this adapter.

For more detail on a specific adapter:

```
Get-NetAdapter -Name Ethernet | Format-List *
```

The full output has dozens of properties. Useful ones include `MacAddress`, `LinkSpeed`, `MediaConnectionState`, `DriverProvider`, `DriverVersion`.

### Get-NetIPAddress

```
Get-NetIPAddress | Where-Object AddressFamily -eq IPv4 | Format-Table InterfaceAlias, IPAddress, PrefixLength
```

Output shows IPv4 addresses for each interface. The `PrefixLength` column is the CIDR length.

For IPv6:

```
Get-NetIPAddress | Where-Object AddressFamily -eq IPv6 | Select-Object InterfaceAlias, IPAddress, PrefixLength
```

Modern Windows enables IPv6 by default; you typically see link-local IPv6 addresses (`fe80::...`) on every interface, plus possibly globally-routable IPv6 addresses if your network supports them.

### Get-NetRoute

```
Get-NetRoute -AddressFamily IPv4 | Format-Table DestinationPrefix, NextHop, RouteMetric, InterfaceAlias
```

The route table. The most important entry is the default route, which is the one with `DestinationPrefix = 0.0.0.0/0`. Without a default route, the box cannot reach anything outside its local subnet.

To find just the default route:

```
Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Select-Object NextHop, InterfaceAlias, RouteMetric
```

`NextHop` is the gateway IP. `InterfaceAlias` is which adapter to send the traffic out. `RouteMetric` matters when there are multiple default routes (lower metric wins).

### Get-NetIPConfiguration

A summary view that combines several queries into one:

```
Get-NetIPConfiguration
Get-NetIPConfiguration -InterfaceAlias Ethernet
```

For most "what is this interface configured with" questions, this is the friendliest cmdlet. It shows the interface, IP, gateway, and DNS servers in one view.

### The legacy ipconfig

```
ipconfig
ipconfig /all
ipconfig /flushdns
ipconfig /release
ipconfig /renew
```

`ipconfig /all` is the all-in-one legacy view. Many admins still use it because it is fast to type and produces a complete snapshot.

`ipconfig /flushdns` clears the local DNS cache. Worth knowing because "the DNS change has not propagated" sometimes has the local cache as the culprit.

`ipconfig /release` and `/renew` work on DHCP-configured interfaces.

For scripting and automation, the PowerShell cmdlets are the right answer. For interactive "what's this box's IP," ipconfig is fine.

---

## Listening ports and active connections

The Windows equivalent of Linux's `ss`.

```
Get-NetTCPConnection -State Listen
Get-NetTCPConnection -State Established
```

The listening view shows what services are accepting connections. Most useful columns: `LocalAddress`, `LocalPort`, `OwningProcess`.

To resolve OwningProcess to a process name (the Windows equivalent of `ss -tlnp`):

```powershell
Get-NetTCPConnection -State Listen | ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        LocalAddress = $_.LocalAddress
        LocalPort = $_.LocalPort
        Process = $proc.Name
        ProcessId = $proc.Id
    }
} | Sort-Object LocalPort | Format-Table -AutoSize
```

That is the join pattern from the workshop. For something you run constantly, save it as a function:

```powershell
function Get-NetListening {
    Get-NetTCPConnection -State Listen | ForEach-Object {
        $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
        [PSCustomObject]@{
            LocalAddress = $_.LocalAddress
            LocalPort = $_.LocalPort
            Process = $proc.Name
            ProcessId = $proc.Id
            ProcessPath = $proc.Path
        }
    } | Sort-Object LocalPort
}
```

Add this to `$PROFILE` and you have a Windows `ss -tlnp`.

### Established connections

```
Get-NetTCPConnection -State Established |
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, OwningProcess
```

For "what is this box talking to right now." Cross-reference with running processes the same way as listening ports.

### UDP

```
Get-NetUDPEndpoint
```

UDP is connectionless, so there is no "established" state. Get-NetUDPEndpoint shows what is listening for UDP traffic.

### The legacy netstat

```
netstat -ano
netstat -anob
```

The flags: `-a` all sockets, `-n` no DNS resolution, `-o` show owning PID, `-b` show owning binary (requires admin).

netstat is still useful for quick interactive viewing. For scripts, the PowerShell cmdlets are better because their output is structured.

---

## DNS and name resolution

The Windows equivalent of Linux's `dig` is `Resolve-DnsName`.

```
Resolve-DnsName example.com
Resolve-DnsName example.com -Type A
Resolve-DnsName example.com -Type AAAA
Resolve-DnsName example.com -Type MX
Resolve-DnsName example.com -Type TXT
```

The output is structured PowerShell objects, so you can pipe and filter:

```
Resolve-DnsName example.com -Type A | Select-Object Name, IPAddress
```

To query a specific DNS server:

```
Resolve-DnsName example.com -Server 8.8.8.8
```

This is the equivalent of `dig @8.8.8.8 example.com` and serves the same purpose: confirming whether a resolution issue is at your local resolver versus upstream.

### The local DNS cache

Windows caches DNS responses locally. To inspect:

```
Get-DnsClientCache
ipconfig /displaydns       # legacy view, often more readable
```

To flush:

```
Clear-DnsClientCache
ipconfig /flushdns         # legacy form
```

When investigating "the DNS change has not propagated yet," clearing the local cache is the first step.

### The DNS client configuration

```
Get-DnsClientServerAddress
```

Shows which DNS servers each interface is configured to use. The output includes both IPv4 and IPv6 servers.

To set the DNS servers programmatically:

```
Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses "1.1.1.1", "8.8.8.8"
```

(Requires admin and is rare in scripted form; usually DNS is set by DHCP or Group Policy.)

---

## Connectivity testing: Test-NetConnection

The Windows equivalent of Linux's `ping` plus `nc -vz` plus a bit of `traceroute`.

```
Test-NetConnection example.com
Test-NetConnection example.com -Port 443
Test-NetConnection example.com -TraceRoute
Test-NetConnection 8.8.8.8 -CommonTCPPort HTTP
```

Reading these:

- Bare `Test-NetConnection example.com` does an ICMP ping. The output includes `PingSucceeded: True/False` and `RemoteAddress`.
- With `-Port`, it does a TCP connection test to that port. The output adds `TcpTestSucceeded: True/False`.
- With `-TraceRoute`, it does a traceroute. Slow but informative when you need to see where in the network path the failure happens.
- With `-CommonTCPPort`, it tests well-known ports by name (HTTP=80, HTTPS=443, RDP=3389, SMB=445).

For most diagnosis, the `-Port` form is what you want. It tells you in one command whether the host is reachable, whether DNS resolves, and whether the specific port is open.

### A common alias

The shorter alias `tnc` works the same:

```
tnc example.com -Port 443
```

`tnc` is a Microsoft-provided alias for `Test-NetConnection`. Use it freely; it is shorter for interactive use.

### When tnc lies

Test-NetConnection's ICMP behavior is sometimes misleading. ICMP is often blocked or rate-limited at firewalls, so `PingSucceeded: False` does not necessarily mean the host is unreachable. The TCP test (`-Port`) is more reliable for "is the service actually reachable."

When in doubt, use `-Port` against the specific service port. If the TCP test succeeds, the box is reachable for that service regardless of what ping says.

---

## HTTP testing: Invoke-WebRequest

The Windows equivalent of curl.

```
Invoke-WebRequest -Uri https://example.com
Invoke-WebRequest -Uri https://example.com -UseBasicParsing
Invoke-WebRequest -Uri https://example.com -UseBasicParsing -Method Head
Invoke-WebRequest -Uri https://example.com -UseBasicParsing -OutFile C:\Temp\page.html
```

The `-UseBasicParsing` flag is worth understanding. By default, Invoke-WebRequest tries to load the response into the legacy IE rendering engine. On Windows 11, IE is gone, so this fails with cryptic errors. `-UseBasicParsing` skips the IE engine and returns the raw response. Always use `-UseBasicParsing` on modern Windows.

For a quick "is this URL alive" check:

```powershell
$response = Invoke-WebRequest -Uri https://example.com -UseBasicParsing -Method Head
$response.StatusCode
```

For a full curl-equivalent with verbose output:

```powershell
$response = Invoke-WebRequest -Uri https://example.com -UseBasicParsing
$response.StatusCode
$response.Headers
$response.Content.Length
```

To follow redirects (which Invoke-WebRequest does by default):

```
Invoke-WebRequest -Uri https://shorturl.example -UseBasicParsing | Select-Object StatusCode, BaseResponse
```

`BaseResponse.ResponseUri` shows the final URL after redirects.

### When you need real curl

Invoke-WebRequest covers most cases. For some use cases (specifically: scripts that need to work identically on Windows and Linux, or scripts that need exact curl behavior), winget's `curl.exe` is available:

```
winget install cURL.cURL
```

Modern Windows 10 and 11 also ship a `curl.exe` that lives in `C:\Windows\System32\curl.exe`. It is real curl, not an alias.

```
where.exe curl
curl --version
```

For scripts targeting cross-platform use, real curl is more predictable. For Windows-only scripts, Invoke-WebRequest is fine.

---

## The four-step diagnosis pattern, on Windows

Putting it all together. The Linux pattern was:

1. Can I reach the host? (`ping`)
2. Does DNS work? (`dig`)
3. Can I open TCP to the port? (`nc -vz`)
4. Does the application respond? (`curl -v`)

The Windows version, using PowerShell:

```
# Step 1: Can I reach the host?
Test-NetConnection example.com

# Step 2: Does DNS resolve?
Resolve-DnsName example.com -Type A

# Step 3: Can I open TCP to the port?
Test-NetConnection example.com -Port 443

# Step 4: Does the application respond?
Invoke-WebRequest -Uri https://example.com -UseBasicParsing -Method Head
```

The same four layers, the same logic. Each step that succeeds rules out that layer as the problem.

A practical tip: combine into a function for fast diagnosis:

```powershell
function Test-Site {
    param(
        [Parameter(Mandatory)][string]$Host,
        [int]$Port = 443,
        [string]$Url
    )

    if (-not $Url) { $Url = "https://$Host" }

    Write-Host "1. Reachability:" -ForegroundColor Yellow
    $ping = Test-NetConnection $Host -InformationLevel Quiet
    Write-Host "   Ping succeeded: $ping"

    Write-Host "2. DNS resolution:" -ForegroundColor Yellow
    try {
        $dns = Resolve-DnsName $Host -Type A -ErrorAction Stop
        Write-Host "   $($dns[0].Name) -> $($dns[0].IPAddress)"
    } catch {
        Write-Host "   FAILED: $_" -ForegroundColor Red
    }

    Write-Host "3. TCP to port ${Port}:" -ForegroundColor Yellow
    $tcp = Test-NetConnection $Host -Port $Port -InformationLevel Quiet
    Write-Host "   TCP succeeded: $tcp"

    Write-Host "4. HTTP response:" -ForegroundColor Yellow
    try {
        $resp = Invoke-WebRequest -Uri $Url -UseBasicParsing -Method Head -ErrorAction Stop
        Write-Host "   Status: $($resp.StatusCode)"
    } catch {
        Write-Host "   FAILED: $_" -ForegroundColor Red
    }
}
```

Save in `$PROFILE`. Use as `Test-Site google.com` or `Test-Site example.com -Port 8080 -Url https://example.com:8080/health`.

---

## Windows Defender Firewall

Windows ships with a built-in stateful firewall. *Windows Defender Firewall is a host-based firewall integrated into Windows that filters inbound and outbound traffic based on rules.* It is on by default and blocks most inbound connections by default.

### Profile concept

Windows Firewall has three profiles:

- **Domain**: applied when the box is on a domain network (joined to AD and the DC is reachable).
- **Private**: applied when the box is on a "trusted" network (typically home).
- **Public**: applied when the box is on an "untrusted" network (typically coffee shops, airports).

Each profile has its own set of rules and its own default action. The active profile depends on which network the system is currently connected to.

```
Get-NetFirewallProfile | Format-Table Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

A typical output:

```
Name    Enabled DefaultInboundAction DefaultOutboundAction
----    ------- -------------------- ---------------------
Domain     True                Block                 Allow
Private    True                Block                 Allow
Public     True                Block                 Allow
```

All three profiles are enabled. Inbound is blocked by default; outbound is allowed by default. This is the conventional posture.

To see the active profile:

```
Get-NetConnectionProfile | Select-Object Name, NetworkCategory
```

### Reading firewall rules

```
Get-NetFirewallRule -Enabled True | Select-Object DisplayName, Direction, Action, Profile -First 10
```

Rules can be inbound or outbound, allow or block, applied to specific profiles or all. The default Windows install has hundreds of rules, mostly for built-in OS services (file sharing, Cortana, Edge browser, etc.).

For more detail on a specific rule:

```
Get-NetFirewallRule -DisplayName "Remote Desktop - User Mode (TCP-In)" |
    Get-NetFirewallPortFilter
```

The mechanic: rules have associated filters for ports, addresses, services, and applications. The `Get-NetFirewallRule` cmdlet returns the rule object; the `Get-NetFirewallPortFilter` (and similar) cmdlets retrieve the filter components.

This separation makes the firewall API verbose but precise. For most working admin tasks, the higher-level `New-NetFirewallRule` and `Get-NetFirewallRule` cmdlets are sufficient.

### Adding a rule

```
New-NetFirewallRule -DisplayName "Allow HTTP from local subnet" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 80 `
    -RemoteAddress 192.168.1.0/24 `
    -Profile Private
```

Reading this: an inbound rule named "Allow HTTP from local subnet," allowing TCP port 80 from the 192.168.1.0/24 subnet, applied only when the firewall is in the Private profile.

### Disabling and removing rules

```
Disable-NetFirewallRule -DisplayName "Allow HTTP from local subnet"
Enable-NetFirewallRule -DisplayName "Allow HTTP from local subnet"
Remove-NetFirewallRule -DisplayName "Allow HTTP from local subnet"
```

### Common scenarios

**Allow inbound RDP from a specific subnet:**

```
New-NetFirewallRule -DisplayName "RDP from admin subnet" `
    -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3389 `
    -RemoteAddress 10.0.5.0/24 -Profile Domain, Private
```

**Block outbound traffic to a specific IP:**

```
New-NetFirewallRule -DisplayName "Block bad-host outbound" `
    -Direction Outbound -Action Block -RemoteAddress 198.51.100.42 `
    -Profile Domain, Private, Public
```

**Block all inbound except RDP and ICMP:**

This requires setting the default action plus explicit allow rules. The general pattern is "default deny, then explicitly allow what is needed," same as ufw on Linux.

### A safety note

Modifying firewall rules carries the same risk as modifying ufw on Linux: it is possible to lock yourself out. Specifically: if you are connected to the box via RDP and you remove or break the RDP allow rule, your session continues but you cannot reconnect.

The discipline:

1. Always have a known-good way to reach the box that does not depend on the rule you are changing (console access, separate network, IPMI).
2. Test changes by adding an explicit allow rule first, then verify that path works, then change the default.
3. For risky changes, consider using PowerShell to schedule a "revert" action that runs in 5 minutes if you do not cancel it.

---

## Network shares: SMB at recognition depth

Windows uses SMB for file sharing. *SMB, Server Message Block, is the protocol Windows uses for file shares, printer sharing, and named pipe IPC.* Modern versions (SMB3) are encrypted by default; legacy SMBv1 is insecure and deprecated.

To list SMB shares on the local box:

```
Get-SmbShare
```

To list SMB connections (other machines connected to your shares):

```
Get-SmbSession
```

To list SMB connections from this box outbound:

```
Get-SmbConnection
```

For most workstation roles, you do not host SMB shares. If you do, the relevant security considerations are:

- SMBv1 should be disabled. (CIS recommendation; covered in Chapter 12.)
- SMB Signing should be required.
- SMB Encryption should be enabled for sensitive shares.

To check SMB protocol version support:

```
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol, EncryptData, RequireSecuritySignature
```

The SMBv1 deprecation is one of the highest-impact hardening items on Windows. Chapter 12 walks through it.

---

## Try this

**1. Map your network configuration.**

Run all four core queries:

```
Get-NetAdapter
Get-NetIPAddress | Where-Object AddressFamily -eq IPv4
Get-NetRoute -DestinationPrefix 0.0.0.0/0
Get-DnsClientServerAddress -AddressFamily IPv4
```

For your lab box, identify the adapter, its IP and prefix length, the default gateway, and the DNS servers. Compare with `ipconfig /all` output to confirm the same picture.

**2. List what is listening on the box.**

Run the listening-ports-with-process-names function (build it if you have not added it to your `$PROFILE` yet). Identify each listening port and the process owning it. On a fresh Windows 11 lab box you typically see a few standard services: SSH (if installed), RDP (3389), SMB (445), and various ephemeral ports for system services.

For each port, ask: should this be listening? On a workstation, the answer for inbound RDP and SMB depends on use. The exercise builds the habit of "every listener is a question."

**3. Walk the four-step diagnosis on a known target.**

Pick a public service (google.com works). Run all four steps:

```
Test-NetConnection google.com
Resolve-DnsName google.com -Type A
Test-NetConnection google.com -Port 443
Invoke-WebRequest -Uri https://google.com -UseBasicParsing -Method Head
```

Confirm each succeeds. Read the output carefully. The point is to know what success looks like before you have to recognize failure.

**4. Walk the four-step diagnosis on a known-broken target.**

Pick a target that will fail. `Test-NetConnection 192.168.99.99 -Port 9999` (an IP and port that probably do not exist on your network). Watch what each tool does on failure.

For an even cleaner failure, try `Resolve-DnsName not-a-real-domain-12345.example`. The DNS failure mode is its own thing.

**5. Inspect Windows Firewall.**

Run:

```
Get-NetFirewallProfile | Format-Table Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Get-NetConnectionProfile
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True | Measure-Object
```

The first shows the profile state. The second shows which profile is active for each network. The third counts the active inbound allow rules.

For a deeper look, pick five rules from the inbound allow list and read their full configuration:

```
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True |
    Select-Object -First 5 |
    ForEach-Object {
        Write-Host "`n=== $($_.DisplayName) ===" -ForegroundColor Yellow
        $_ | Format-List DisplayName, Direction, Action, Profile, Description
        $_ | Get-NetFirewallPortFilter | Format-List Protocol, LocalPort, RemotePort
        $_ | Get-NetFirewallApplicationFilter | Format-List Program
    }
```

Identify what each rule allows and why. Some rules are for specific applications (Edge, Microsoft Store, etc.), others for specific protocols (mDNS, LLMNR, etc.).

---

## Common stumbling blocks

> **`Test-NetConnection` says PingSucceeded: False but I can reach the site in a browser.**
> ICMP is often blocked or rate-limited at firewalls. The TCP test (`-Port`) is more reliable for "is this service reachable." For sites behind CDNs or major cloud providers, ICMP is frequently dropped while HTTP works fine.

> **`Invoke-WebRequest` fails with "the response content cannot be parsed because the Internet Explorer engine is not available."**
> Add `-UseBasicParsing`. On Windows 11, IE is gone; legacy parsing modes do not work. `-UseBasicParsing` is the workaround and should be your default.

> **`Get-NetTCPConnection` shows ports owned by PID 4 (System).**
> PID 4 is the System process, which owns kernel-level networking endpoints. Most of these are SMB (445, 139), HTTPS Server APIs (443 for things like AD FS), and similar. They are not normal user-mode services; they are kernel services.

> **My new firewall rule has no effect.**
> Common causes. First, the rule is targeting the wrong profile (check with `Get-NetFirewallRule | Get-NetFirewallProfile`). Second, an existing rule is overriding yours; firewall rule precedence is "block beats allow, more-specific beats less-specific." Third, the rule is disabled (`Enabled: False` in the rule listing).

> **`Resolve-DnsName` returns a different IP than the browser uses.**
> The browser may use a different DNS resolver (DoH/DoT bypassing the OS resolver). Check with `Get-DnsClientServerAddress` to see what the OS uses. For browsers using DoH, the OS-level DNS does not match.

> **`Get-NetFirewallRule` is slow.**
> Default returns thousands of rules. Filter at the source: `Get-NetFirewallRule -Enabled True -Direction Inbound -Action Allow` is dramatically faster than retrieving all and filtering with `Where-Object`.

> **I changed the DNS server and ipconfig still shows the old one.**
> The active configuration may be DHCP-assigned; setting it manually with `Set-DnsClientServerAddress` works only if the interface is statically configured. To set DNS on a DHCP interface, you can either configure the DHCP server, set DNS manually with the cmdlet (which overrides DHCP for the session), or change the interface to static.

> **Invoke-WebRequest is much slower than curl.**
> `Invoke-WebRequest` initializes a lot of .NET infrastructure on each invocation. For scripts that make many web requests, use `Invoke-WebRequest` once to get a session, or use `[System.Net.HttpClient]` directly for speed-sensitive paths. For interactive use, the slowness is rarely meaningful.

---

## What this gets you

After this chapter:

- You can read a Windows host's network configuration in 30 seconds with the four core PowerShell cmdlets.
- You can identify what is listening on a box and what process owns each socket.
- You can troubleshoot "the network is broken" with the four-step pattern translated to Windows.
- You can read and write Windows Defender Firewall rules via PowerShell.
- You know about firewall profiles and how the active profile depends on the network.
- You know about SMB at recognition depth and why disabling SMBv1 matters.
- You can build practical functions (Test-Site, Get-NetListening) for daily use.

The four-step pattern is the part of this chapter that pays off most. It is the same pattern as Linux, and that consistency means a working admin who has both can do basic network diagnosis on either platform without thinking about which tools to use.

---

## What's next

Chapter 9 is PowerShell scripting. The chapter where you build on the workshop's small script and Chapter 2's foundations to write working scripts with proper error handling, parameters, and structure. Plus the audience-tuned framing: read scripts adversarially, including malicious ones, with PowerShell-specific patterns to recognize.
