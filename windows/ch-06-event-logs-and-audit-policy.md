# Chapter 6: Windows Event Logs and Audit Policy

**You come in with:** workshop-level Get-WinEvent. You can read a single channel for recent events. You have seen the Microsoft-Windows-Sysmon/Operational channel.
**You leave with:** the ability to answer "what did this user do," "who logged in successfully and from where," "when did this thing fail," and "what changed in the last hour" using Get-WinEvent and the right filters. Plus working understanding of audit policy and Sysmon's telemetry as data sources.

**Time:** 70 to 100 minutes including the exercises. The longest chapter in the unit; budget two sittings if needed.

**Security+ alignment:** Heavy alignment, this is one of the most exam-relevant chapters in the unit. Domain 4.4 (alerting and monitoring concepts and tools: log aggregation, alerting, scanning, reporting, archiving). Domain 4.8 (incident response: detection, analysis, root cause analysis, threat hunting). Domain 4.9 (data sources to support an investigation: log data including OS-specific security logs, endpoint logs, application logs, metadata). The audit policy section maps to Domain 4.5 (operating system security). Reading the Security log adversarially is the practical version of Domain 2.4 (indicators of malicious activity).

---

## Why this chapter matters

Logs are the system's memory. On Windows, the event log is the primary log infrastructure: structured, queryable, with multiple channels for different concerns. For working admins, mastering Get-WinEvent is the difference between "the box is broken" and "the box is broken because at 14:32 service X failed with error Y, and here is the corresponding entry in the System log." For security analysts, the Security log and Sysmon channel are the foundation of incident response on Windows.

The chapter is the longest in the unit. The pattern matters: most of Get-WinEvent's power is in filters, and most filters become muscle memory only after you use them on real questions.

---

## What the Windows event log actually is

The Windows event log is a structured, indexed event store managed by the Event Log service. *The Event Log is a Windows component that captures structured events from the operating system, services, applications, and (with Sysmon) detailed system telemetry, storing them in queryable channels.* Each event has a numeric ID, a level (Critical, Error, Warning, Info, Verbose), a timestamp, and a structured payload.

The on-disk storage is `C:\Windows\System32\winevt\Logs`. Each log channel is a separate `.evtx` file. The format is binary; you do not read these directly with notepad.

```
ls C:\Windows\System32\winevt\Logs | Select-Object Name, Length -First 10
```

The largest files are the most active channels: Security, System, Application, and Microsoft-Windows-Sysmon%4Operational.

### Channels

A channel is a logical log destination. The Windows defaults:

| Channel | What lives here |
|---------|-----------------|
| Application | Application errors, warnings, informational messages. |
| System | Kernel, driver, and service events. The "is the OS healthy" log. |
| Security | Authentication and authorization events. The "who did what" log. Requires admin to read. |
| Setup | Windows Update and feature install events. |
| Microsoft-Windows-Sysmon/Operational | Sysmon's telemetry (pre-loaded on your lab box). |

Plus hundreds of more specialized channels under `Microsoft-Windows-*`. Most are quiet most of the time and only relevant to specific subsystems.

To list all channels:

```
Get-WinEvent -ListLog * | Select-Object LogName, RecordCount, LogMode | Sort-Object RecordCount -Descending | Select-Object -First 20
```

The output shows the busiest channels by count. On a typical Windows 11 box you see Security at the top, followed by Sysmon (because we have it pre-loaded), then System and Application.

---

## Get-WinEvent: filters from simple to advanced

The workshop introduced `-LogName` and `-MaxEvents`. There is much more.

### Filter by log and ID together

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 5
```

The hashtable form is the right pattern for any non-trivial query. It is dramatically faster than retrieving everything and filtering with `Where-Object`, because the filter happens at the kernel level rather than in PowerShell.

### Filter by time

```
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4624
    StartTime = (Get-Date).AddHours(-1)
}

Get-WinEvent -FilterHashtable @{
    LogName = 'System'
    StartTime = (Get-Date).AddDays(-1)
    EndTime = Get-Date
}
```

`StartTime` and `EndTime` accept any DateTime. Use `(Get-Date).AddHours(-N)` or `(Get-Date).AddDays(-N)` for relative times.

### Filter by level

```
Get-WinEvent -FilterHashtable @{
    LogName = 'System'
    Level = 2  # Error
    StartTime = (Get-Date).AddHours(-24)
}
```

Levels:

- 1: Critical
- 2: Error
- 3: Warning
- 4: Information
- 5: Verbose

For "what is broken on this box," `Level = 2` over the last 24 hours is the start.

### Filter by user

```
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4624
    StartTime = (Get-Date).AddDays(-1)
} | Where-Object { $_.Properties[5].Value -eq 'student' }
```

User filtering is harder because the username is in a property within the event data, not a top-level field. The pattern: get the events first, then filter by the property index. Different event IDs have different property layouts; you find the right index by reading one event with `Format-List *` and counting properties.

### Filter by data field (advanced)

For complex filters, you can use the FilterXPath form, which accepts an XPath query against the event XML:

```
Get-WinEvent -LogName Security -FilterXPath "*[EventData[Data[@Name='TargetUserName']='student']]"
```

That filters Security events where the TargetUserName field is "student." It is more powerful than the hashtable form but more verbose.

For this unit, the hashtable form is sufficient for most queries.

---

## The Security log

The Security log is where authentication and authorization events live. This is the log that matters most for security work.

### Reading requires admin

```
Get-WinEvent -LogName Security -MaxEvents 5
```

If you run this without admin, you get "Attempted to perform an unauthorized operation." Run from elevated PowerShell.

### The big event IDs

The Security log has hundreds of event IDs. These are the ones you should know cold:

| Event ID | What it means |
|----------|---------------|
| **4624** | Successful logon |
| **4625** | Failed logon |
| **4634** | Logoff |
| **4647** | User-initiated logoff |
| **4648** | Explicit credential use (e.g., `runas`) |
| **4672** | Special privileges assigned to new logon (admin token granted) |
| **4720** | User account created |
| **4722** | User account enabled |
| **4724** | Password reset attempted on user account |
| **4725** | User account disabled |
| **4726** | User account deleted |
| **4732** | Member added to a security-enabled local group |
| **4733** | Member removed from a security-enabled local group |
| **4738** | User account changed |
| **4768** | Kerberos authentication ticket (TGT) requested (domain) |
| **4769** | Kerberos service ticket requested (domain) |
| **4776** | NTLM authentication |
| **5140** | Network share accessed |

For a non-domain workstation, focus on 4624, 4625, 4672, 4720, 4732, and 4738. These cover account activity, logon events, and privilege use.

### Reading 4624: successful logon

```
$ev = Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 1
$ev | Format-List TimeCreated, Id, Message
```

The Message field is multiline and includes:

- **Subject**: who was already logged on when the new logon happened (often blank for fresh logons).
- **Logon Type**: how the logon happened. Values: 2 (interactive), 3 (network), 4 (batch), 5 (service), 7 (unlock), 8 (network cleartext), 9 (new credentials), 10 (remote interactive), 11 (cached interactive).
- **New Logon**: the account that just logged on. Includes the SID and account name.
- **Workstation Name**: the source computer.
- **Source Network Address**: the IP if remote.

Logon types matter. Type 2 is "someone sat at the keyboard." Type 3 is "network connection." Type 10 is RDP. Type 5 is service starting under that account. When triaging "who logged in," the type tells you how.

### Reading 4625: failed logon

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddHours(-24)} | Select-Object -First 5 TimeCreated, Message
```

The 4625 message includes the same fields as 4624 plus the **Failure Reason** and **Status**. Common failure reasons:

- **0xC000006A** (`STATUS_WRONG_PASSWORD`): wrong password.
- **0xC0000064** (`STATUS_NO_SUCH_USER`): username does not exist.
- **0xC0000234** (`STATUS_ACCOUNT_LOCKED_OUT`): account locked.
- **0xC0000071** (`STATUS_PASSWORD_EXPIRED`): password expired.

A pattern of 4625 events with `STATUS_NO_SUCH_USER` for many different usernames is a username-spray attack. A pattern with `STATUS_WRONG_PASSWORD` for one user is a password-spray or brute-force attempt.

### Useful 4624/4625 queries

Show successful logons in the last hour:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624; StartTime=(Get-Date).AddHours(-1)} |
    Select-Object TimeCreated,
        @{N='Account';E={$_.Properties[5].Value}},
        @{N='LogonType';E={$_.Properties[8].Value}},
        @{N='Source';E={$_.Properties[18].Value}}
```

The Properties index numbers come from the event schema. For 4624 specifically:
- Index 5: TargetUserName (the account that logged on)
- Index 8: LogonType
- Index 18: IpAddress

Failed logons grouped by source IP:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddDays(-1)} |
    Group-Object @{E={$_.Properties[19].Value}} |
    Sort-Object Count -Descending |
    Select-Object Count, Name -First 10
```

### Account activity events

Account creation: 4720. Account changes: 4738. Group membership changes: 4732 (add), 4733 (remove).

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4720,4722,4725,4726,4732,4733; StartTime=(Get-Date).AddDays(-30)}
```

These together are "account activity in the last 30 days." On a fresh workstation this returns a small set. On a real production machine that has been around for a while, the volume tells you about user lifecycle.

For each event, the Message includes who made the change, who was changed, and what changed. These are the events you read first when investigating "did someone create a backdoor account."

---

## Audit policy: what gets logged

Some Security log events are recorded by default. Many are not. The audit policy controls which events are emitted.

```
auditpol /get /category:*
```

Output shows each audit category and whether it is set to None, Success, Failure, or Success and Failure. The default policy on Windows 11 is reasonable but can be tightened.

### Categories that matter

For security work, the categories that matter most:

| Category | Event IDs | Default | Recommended |
|----------|-----------|---------|-------------|
| Logon | 4624, 4625, 4634, 4647 | Success | Success and Failure |
| Account Lockout | 4740 | None | Success |
| Account Management | 4720-4726, 4732-4733 | Success | Success and Failure |
| Special Logon | 4672 | Success | Success |
| Process Creation | 4688 | None | Success |
| Filtering Platform Connection | 5156, 5157 | None (loud) | Disabled (too noisy without filtering) |

Process Creation (4688) is particularly worth knowing about. It records every process started on the box. Built-in audit, no additional tooling required. We touch on it more in Chapter 11.

### Enabling Process Creation auditing

```
auditpol /set /subcategory:"Process Creation" /success:enable
```

After enabling, Event 4688 records every process. To verify:

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688; StartTime=(Get-Date).AddMinutes(-5)} | Select-Object -First 3 | Format-List Message
```

Also worth enabling: Process Creation includes the command line if you set the registry switch:

```
Set-ItemProperty -Path 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit' -Name 'ProcessCreationIncludeCmdLine_Enabled' -Value 1 -Type DWord
```

Without this, 4688 records the executable path but not the arguments. With it, the full command line is captured. This is a major detection win and is essentially free.

### auditpol vs Group Policy

The `auditpol` command-line tool sets the audit policy directly. In a managed environment, the policy is usually set via Group Policy, which writes the same configuration but through a centralized mechanism. For your lab box, `auditpol` is the right tool. We come back to Group Policy for hardening in Chapter 12.

---

## The Sysmon channel

Sysmon is pre-loaded on your lab box. The instructor has installed it; you do not need to. *Sysmon, System Monitor, is a Microsoft Sysinternals utility that adds detailed system telemetry to the event log: process creation, network connections, file modifications, and more.*

```
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 5 | Select-Object TimeCreated, Id
```

The Sysmon channel is one of the most useful data sources on Windows for security work. It captures what the default Security log does not: every process with parent context, every network connection, every DLL load.

### The Sysmon event IDs you should know

| Event ID | What it captures |
|----------|------------------|
| **1** | Process creation (with parent process and command line) |
| **2** | File creation time changed (timestomping) |
| **3** | Network connection |
| **4** | Sysmon service state change |
| **5** | Process terminated |
| **7** | Image (DLL) loaded into a process |
| **8** | CreateRemoteThread (process injection) |
| **10** | ProcessAccess (one process opening another's memory) |
| **11** | File create |
| **13** | Registry value set |
| **22** | DNS query |
| **23** | File delete (with archiving option) |

For most investigations, IDs 1, 3, 11, 13, and 22 cover the most ground.

### Reading Sysmon Event 1 (process creation)

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1} -MaxEvents 1 | Format-List TimeCreated, Message
```

The Message field of a Sysmon Event 1 is voluminous. Key fields:

- **UtcTime**: when the process started.
- **ProcessGuid**: a unique ID for this specific process instance.
- **ProcessId**: the PID.
- **Image**: the executable path.
- **CommandLine**: the full command line, including arguments. (Sysmon captures this without needing the audit policy registry tweak.)
- **CurrentDirectory**: the working directory.
- **User**: who launched the process.
- **ParentProcessGuid** and **ParentProcessId** and **ParentImage** and **ParentCommandLine**: the same fields for the parent process.
- **Hashes**: hashes of the executable.

Reading the parent context is the part that makes Sysmon Event 1 powerful. "powershell.exe ran with command X" is one piece of information. "powershell.exe with command X was launched by Y, which was launched by Z" is the call chain. Attack patterns often have characteristic parent-child relationships (a Word process spawning powershell, for example), and Event 1 captures this.

### Useful Sysmon queries

Process creations from a specific binary:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1; StartTime=(Get-Date).AddHours(-1)} |
    Where-Object { $_.Message -match 'powershell.exe' } |
    Select-Object -First 5 TimeCreated, Message
```

Network connections:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=3; StartTime=(Get-Date).AddMinutes(-30)} |
    Select-Object -First 10 TimeCreated, Message
```

DNS queries:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=22; StartTime=(Get-Date).AddMinutes(-30)} |
    Select-Object -First 10 TimeCreated, Message
```

Each of these is the start of a real investigation question. "Did anything connect to a strange IP?" -> Event 3. "Did anything resolve a suspicious hostname?" -> Event 22. "What did this PowerShell session actually launch?" -> Event 1 with parent matching the session's PID.

---

## A real investigation

The practical question that ties everything together: a user account is suspected of doing something wrong between 2 PM and 4 PM yesterday. Walk the answer.

**Step 1.** When did they log in?

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4624
    StartTime=(Get-Date).Date.AddDays(-1).AddHours(14)
    EndTime=(Get-Date).Date.AddDays(-1).AddHours(16)
} | Where-Object { $_.Properties[5].Value -eq 'student' } |
    Select-Object TimeCreated, @{N='LogonType';E={$_.Properties[8].Value}}, @{N='Source';E={$_.Properties[18].Value}}
```

You get the logon times, types, and source addresses.

**Step 2.** What processes did they run?

If Process Creation auditing is on:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4688
    StartTime=(Get-Date).Date.AddDays(-1).AddHours(14)
    EndTime=(Get-Date).Date.AddDays(-1).AddHours(16)
} | Where-Object { $_.Properties[1].Value -match 'student' } |
    Select-Object TimeCreated, @{N='NewProcess';E={$_.Properties[5].Value}}, @{N='CmdLine';E={$_.Properties[8].Value}}
```

If Sysmon is running (which it is on the lab box):

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=1
    StartTime=(Get-Date).Date.AddDays(-1).AddHours(14)
    EndTime=(Get-Date).Date.AddDays(-1).AddHours(16)
} | Where-Object { $_.Message -match 'student' } |
    Select-Object -First 20 TimeCreated, Message
```

Sysmon gives you more (parent process, command line) than 4688.

**Step 3.** What network activity?

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=3
    StartTime=(Get-Date).Date.AddDays(-1).AddHours(14)
    EndTime=(Get-Date).Date.AddDays(-1).AddHours(16)
} | Where-Object { $_.Message -match 'student' }
```

**Step 4.** Account changes?

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4720,4722,4724,4726,4732,4733,4738
    StartTime=(Get-Date).Date.AddDays(-1)
}
```

Reads as: any account changes, additions, deletions, group membership changes in the last day. If the user was investigating because of suspected privilege escalation, this is where to look.

**Step 5.** Logoff time.

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4634,4647
    StartTime=(Get-Date).Date.AddDays(-1).AddHours(14)
} | Where-Object { $_.Properties[1].Value -eq 'student' } |
    Select-Object -First 5 TimeCreated, Id
```

This sequence is the spine of an incident triage. It is also the spine of routine "what did this user do" investigations. The same five queries, every time.

---

## Log retention and forwarding

The default event log size is small. On a busy system, the Security log fills up and old events are overwritten. To check sizes and retention:

```
Get-WinEvent -ListLog Security, System, Application, 'Microsoft-Windows-Sysmon/Operational' |
    Select-Object LogName, MaximumSizeInBytes, RecordCount, LogMode
```

LogMode values:

- **Circular**: when full, oldest events are overwritten.
- **AutoBackup**: when full, the log is archived and a new one started.
- **Retain**: when full, no new events are recorded until manually cleared.

For most production systems, Circular with a large size is the default and is fine, as long as you forward logs off the box.

To increase the Security log size:

```
wevtutil sl Security /ms:1073741824
```

That sets the Security log to 1 GB. Adjust based on retention needs.

### Forwarding events

In a real environment, events should be forwarded to a central system (SIEM). Windows has built-in support called **Windows Event Forwarding (WEF)**. The basics: a WEF source (your endpoint) ships events to a WEF collector (a central server) over WS-Management/WinRM. The collector aggregates events from many endpoints.

For this chapter, recognition is enough. Configuring WEF is intermediate cohort territory. The important takeaway: events that exist only on the box where they happened are vulnerable to that box being compromised. Forwarding is what makes the events durable.

---

## Try this

**1. Run the Get-WinEvent inventory.**

```
Get-WinEvent -ListLog * | Where-Object RecordCount -gt 0 |
    Sort-Object RecordCount -Descending |
    Select-Object LogName, RecordCount, LogMode -First 15
```

Read the output. Identify the busiest channels. Note that Sysmon's channel is in the top group (because it captures every process and network connection).

**2. Read recent Security log events for your account.**

Run from elevated PowerShell:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624; StartTime=(Get-Date).AddHours(-1)} |
    Select-Object -First 5 TimeCreated, @{N='User';E={$_.Properties[5].Value}}, @{N='LogonType';E={$_.Properties[8].Value}}
```

Identify your own logon events. Note the LogonType values; type 2 is interactive, type 5 is service, etc.

**3. Enable Process Creation auditing and read 4688.**

```
auditpol /set /subcategory:"Process Creation" /success:enable
Set-ItemProperty -Path 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit' -Name 'ProcessCreationIncludeCmdLine_Enabled' -Value 1 -Type DWord
```

Then run a few commands in PowerShell. After a moment, read:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688; StartTime=(Get-Date).AddMinutes(-5)} |
    Select-Object -First 10 TimeCreated, Message
```

You see your own command activity in the Security log.

**4. Read Sysmon Event 1 for a recent PowerShell launch.**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1; StartTime=(Get-Date).AddMinutes(-10)} |
    Where-Object { $_.Message -match 'powershell' } |
    Select-Object -First 1 | Format-List Message
```

Read the message. Identify the Image, CommandLine, ParentImage, ParentCommandLine fields. You can see your own PowerShell session and what launched it (probably explorer.exe).

**5. Build the user-investigation query for your own account.**

Pretending you are investigating yourself, build the five-step query sequence from the "A real investigation" section, with your username as the target. Run each step. Confirm each returns sensible results for your own activity.

This exercise is the chapter's capstone: you are reading your own activity adversarially, the way an investigator would read someone else's. Comfort with this pattern is the foundation of incident response work.

---

## Common stumbling blocks

> **`Get-WinEvent` returns "No events were found that match the specified selection criteria."**
> Either the filter is too restrictive, the time window does not contain matching events, or you need elevation. Try widening the time window first; if that does not help, drop one filter at a time until you get results.

> **The Security log query is much slower than other channels.**
> The Security log is often the largest. Filter at the source with `-FilterHashtable` rather than retrieving and filtering with Where-Object. The hashtable form pushes the filter into the kernel.

> **Property indexes change between events.**
> Different event IDs have different property layouts. Index 5 for 4624 is TargetUserName; for 4688 it is NewProcessName. Verify with `Format-List *` on a single event before writing a query that uses Properties indexes.

> **Sysmon channel returns nothing.**
> Confirm Sysmon is running: `Get-Service Sysmon* | Select-Object Name, Status`. If the service exists but is stopped, start it. If it is not installed, contact your instructor; the lab box is supposed to have it pre-loaded.

> **Process Creation events do not include the command line even after enabling Process Creation auditing.**
> The registry tweak (ProcessCreationIncludeCmdLine_Enabled) is what enables command-line capture. Without it, only the executable name is logged. Both audit policy + registry tweak are needed.

> **The Security log filled up and is dropping events.**
> Either it is too small (increase with `wevtutil sl Security /ms:<size>`) or events are too verbose (reduce audit policy categories that are too noisy). Most production systems set Security to 1 GB or larger and tune audit policy carefully.

> **`auditpol` says "the system cannot find the path specified."**
> auditpol uses subcategory names that are localized. On English systems, "Process Creation" works. On a non-English Windows, the name is translated. Run `auditpol /list /subcategory:*` to see the names available on your system.

---

## What this gets you

After this chapter:

- You can read any channel of the Windows event log fluently using Get-WinEvent and FilterHashtable.
- You know the Security log event IDs that matter (4624, 4625, 4672, 4720, 4732, 4738) and what fields each contains.
- You can configure audit policy with auditpol to capture more than the defaults.
- You know about Sysmon and can read its channel for process creation, network connections, DNS queries, and other telemetry.
- You can do a real five-step user investigation: when did they log in, what did they run, what network activity, what account changes, when did they log off.
- You know that logs that exist only on the local box are vulnerable, and that forwarding to a SIEM is the production answer.

This chapter is the bridge to Chapter 11 and to the security path generally. The pattern of "read the logs adversarially" applies across every Windows incident response engagement. Mastering Get-WinEvent here pays off for the rest of a career in Windows-flavored security work.

---

## What's next

Chapter 7 is Storage and the disk subsystem. Shorter and more practical. By the end you can answer "where did the disk space go" on a Windows host, read the disk and partition layout, and know what BitLocker and Volume Shadow Copy do at recognition depth.
