# Chapter 11: Windows Artifacts and What Attackers Leave Behind

**You come in with:** Chapter 6 (event logs and Sysmon), Chapter 9 (reading PowerShell adversarially), Chapter 10 (process inspection). You know how to read a Windows system; you have not yet read one adversarially end-to-end.
**You leave with:** the ability to do basic IR triage on a Windows host you did not build, with concrete artifact-level checks tied to MITRE ATT&CK techniques.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** This chapter is the densest single mapping to Security+ in the unit. Domain 2.4 (indicators of malicious activity: account takeover, persistence, defense evasion, command and control). Domain 4.4 (alerting and monitoring concepts and tools: file integrity monitoring). Domain 4.8 (incident response: detection, analysis, root cause analysis, threat hunting). Domain 4.9 (data sources to support an investigation: log data, file metadata, command history, host artifacts). MITRE ATT&CK technique IDs are cited throughout this chapter; ATT&CK alignment is implicitly tested via Security+ Domain 2 and explicitly tested in many follow-on certifications (CySA+, GCFE, GCIH).

---

## Why this is the audience-defining chapter

Same role as the Linux Chapter 11. Up to this point, the unit has been "Windows for sysadmins and security pros" with a security flavor. This is the chapter that makes the security focus explicit. Without it, the unit reads as a generalist Windows intro. With it, the unit is foundational training for someone heading into security operations, incident response, or threat hunting.

The pattern this chapter teaches:

1. Establish what normal looks like for the kind of system you are looking at.
2. Look for specific artifacts known to be left by specific attacker techniques.
3. Decide which findings are real and which are noise.
4. Document what you found, with reproducible commands, in a way an investigator coming after you can follow.

This is most of what junior incident response analysts do. The Windows version is denser than the Linux version because Windows has more places to plant things, more layers of telemetry, and a richer attack surface.

---

## A note on lab setup

This chapter assumes you have a "compromised" lab box prepared by your instructor. The box will have planted artifacts representing common attacker behaviors. The exercises walk you through finding and characterizing them.

If your instructor has not set up the compromised box yet, you can still work through the chapter conceptually. The artifact patterns and commands work on any Windows box; on a clean system, the queries will return empty results, which is informative on its own (it shows you what "no findings" looks like).

The placeholder artifacts described at the end of this chapter are the recommended set for the instructor to plant. The exercises will tell you what categories to look for; you will find the specific instances on your lab box.

---

## The mental model: assume nothing

Same framing as the Linux unit, with Windows specifics.

When you walk up to a Windows box you suspect is compromised, fight the impulse to trust the system's own tools without cross-checking:

- Standard Windows binaries (`tasklist`, `whoami`, `reg`) are signed by Microsoft. You can verify the signatures, which is more than you usually have on Linux. Use this.
- The event log can be partially cleared (Event 1102 records the clearance, but the cleared content is gone).
- Registry values can be modified to hide entries from common tools.
- Process Explorer and Sysmon catch things `tasklist` does not.

Cross-check tools when results matter. When `Get-Process` and Sysmon Event 1 disagree, that disagreement is itself information.

For most real-world investigations, the attacker is not sophisticated enough to backdoor Windows binaries (Microsoft's signing infrastructure makes this hard). But the model still serves you: assume nothing, verify what you can, document what you find.

---

## Establishing baseline: what normal looks like

The first task on any new Windows host is figuring out what is supposed to be there.

### Local users and admins

```
Get-LocalUser | Select-Object Name, SID, Enabled, LastLogon
Get-LocalGroupMember -Group Administrators
Get-LocalGroupMember -Group "Remote Desktop Users"
```

For each user, ask:

- Should this user exist?
- Is the user enabled?
- When did they last log on?

For administrators, the question is "should this account have admin." Anything outside your expected list is a finding.

A particular pattern to look for: a user account named to look like a system account (like `support`, `mssql_admin`, `service_user`) that is in the Administrators group. The naming is camouflage; the membership is the finding.

### Listening services and ports

```
Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess
```

For each listening port, identify the owning process. Anything you cannot identify is a question. We covered this in Chapter 8; the security application is the same query, read adversarially.

### Recently modified files in System32

```
Get-ChildItem C:\Windows\System32 -File -ErrorAction SilentlyContinue |
    Where-Object LastWriteTime -gt (Get-Date).AddDays(-30) |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 20 LastWriteTime, Name, Length
```

System32 changes during Windows updates and feature installs. A recently-modified file in System32 outside an update window is a finding. Cross-reference with Get-WinEvent for Update Orchestrator events to confirm or deny "this changed because of Windows Update."

### Scheduled tasks not from Microsoft

```
Get-ScheduledTask | Where-Object TaskPath -notlike "\Microsoft*" |
    Select-Object TaskPath, TaskName, State, Author
```

The `\Microsoft\` path is reserved for OS tasks. Anything outside it is custom. For each, identify what it does and why.

### Services not from Microsoft

```
Get-CimInstance Win32_Service |
    Where-Object { $_.PathName -notmatch '^"?[Cc]:\\(Windows|Program Files( \(x86\))?)' } |
    Select-Object Name, StartName, PathName, State
```

That returns services whose executable path is outside the standard locations. Anything in the result warrants attention.

These four checks plus the running-process signature audit from Chapter 10 constitute the first 30 seconds of triage. Run them on every Windows box you investigate.

---

## Persistence: how attackers stay

Windows has more persistence mechanisms than Linux, and they are spread across more locations. The chapter walks the major categories with one or two queries per technique.

### Run keys (T1547.001)

The classic Windows persistence. Anything in a Run key launches when the user logs in (HKCU) or when any user logs in (HKLM).

```powershell
$paths = @(
    'HKLM:\Software\Microsoft\Windows\CurrentVersion\Run',
    'HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce',
    'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Run',
    'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\RunOnce',
    'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run',
    'HKCU:\Software\Microsoft\Windows\CurrentVersion\RunOnce'
)
foreach ($p in $paths) {
    Write-Host "`n=== $p ===" -ForegroundColor Yellow
    Get-ItemProperty $p -ErrorAction SilentlyContinue | Format-List
}
```

For each value found, ask: what is this, why is it here, do I recognize the publisher of the executable it launches?

A common attacker pattern: a Run key value with a name like `Updater` or `SecurityScan` that launches a PowerShell command from a hidden directory. The naming is camouflage; the command is the finding.

### RunOnce keys (T1547.001)

`RunOnce` keys delete themselves after running. Used for one-time install steps and (sometimes) for one-time persistence that runs once at login. Less common than `Run` for persistence but worth checking.

The query above includes RunOnce paths.

### Image File Execution Options (T1546.012)

A more sophisticated persistence mechanism. *Image File Execution Options is a registry key intended to support debugging applications, but it can be abused: setting a Debugger value for an executable causes Windows to run that debugger instead of the original executable when someone tries to launch it.*

```
Get-ChildItem 'HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options' |
    Where-Object { $_.Property -contains 'Debugger' } |
    ForEach-Object {
        $name = $_.PSChildName
        $debugger = (Get-ItemProperty $_.PSPath -Name Debugger).Debugger
        [PSCustomObject]@{Image=$name; Debugger=$debugger}
    }
```

Most entries here are legitimate (Microsoft's own AeDebug is one, for example). Any entry where the Debugger value is `cmd.exe`, `powershell.exe`, or a binary in a non-standard location is a finding.

The classic abuse: setting `Debugger` for `sethc.exe` (Sticky Keys) to `cmd.exe`. When the user hits Shift five times at the login screen, instead of Sticky Keys popping up, a SYSTEM-level cmd.exe appears. ATT&CK T1546.008.

### AppInit_DLLs (T1546.010)

A historical persistence mechanism. *AppInit_DLLs is a registry value that lists DLLs to be loaded into every process that loads user32.dll, which is most processes.* On modern Windows with Secure Boot, AppInit_DLLs is largely deprecated, but checking it is a one-line audit.

```
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Windows' |
    Select-Object AppInit_DLLs, LoadAppInit_DLLs
```

`AppInit_DLLs` should be empty on modern Windows. Anything in it is a finding. `LoadAppInit_DLLs` should be 0 (disabled) on Windows 10/11 with Secure Boot.

### Services (T1543.003)

We covered this in Chapter 5. The artifact:

```powershell
Get-CimInstance Win32_Service |
    Where-Object { $_.PathName -notmatch '^"?[Cc]:\\(Windows|Program Files( \(x86\))?)\\' } |
    Select-Object Name, StartName, PathName, State
```

Plus the registry view, which catches some things `Get-CimInstance` does not:

```
Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services' |
    ForEach-Object {
        $img = (Get-ItemProperty $_.PSPath -Name ImagePath -ErrorAction SilentlyContinue).ImagePath
        if ($img -and $img -notmatch '^"?[\\?]+(\\??\\)?[Cc]:\\(Windows|Program Files)') {
            [PSCustomObject]@{Name=$_.PSChildName; ImagePath=$img}
        }
    }
```

For each service found, read the PathName/ImagePath. Anything not in `C:\Windows` or `C:\Program Files` is a question.

### Scheduled tasks (T1053.005)

```powershell
Get-ScheduledTask | Where-Object {
    $_.TaskPath -notlike "\Microsoft*" -and $_.State -ne 'Disabled'
} | ForEach-Object {
    [PSCustomObject]@{
        Path = $_.TaskPath
        Name = $_.TaskName
        Action = $_.Actions[0].Execute
        Args = $_.Actions[0].Arguments
        Principal = $_.Principal.UserId
    }
}
```

For each task, read the action. Tasks that launch PowerShell with `-EncodedCommand`, `-WindowStyle Hidden`, or `-ExecutionPolicy Bypass` are particularly worth investigating. Tasks that fetch from URLs are findings.

### Startup folder (T1547.001)

The Startup folder is a per-user (or all-users) directory whose contents launch at logon.

```powershell
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" -Force -ErrorAction SilentlyContinue
Get-ChildItem "$env:ProgramData\Microsoft\Windows\Start Menu\Programs\Startup" -Force -ErrorAction SilentlyContinue
```

The first is the current user's Startup folder. The second is the all-users version. Anything in either is auto-launched at logon. Most Windows installs have nothing in either.

### WMI Event Subscription (T1546.003)

A more advanced persistence technique. WMI supports event consumers that run code when specific events fire (e.g., "every 30 minutes," "when a USB drive is inserted"). Attacker code planted as a WMI event consumer survives reboots and runs without showing up in scheduled tasks or services.

```
Get-CimInstance -Namespace root\subscription -ClassName __EventConsumer
Get-CimInstance -Namespace root\subscription -ClassName __EventFilter
Get-CimInstance -Namespace root\subscription -ClassName __FilterToConsumerBinding
```

On a clean Windows 11 box, all three return empty or only built-in entries. Anything else is a high-confidence finding. WMI persistence is rare in commodity malware but common in targeted attacks.

For this chapter, recognition is enough. The intermediate cohort goes deeper.

---

## Defense evasion: what attackers hide

After persistence, attackers try to hide. Each technique leaves a different artifact.

### Cleared event logs (T1070.001)

The Windows event log is harder to selectively edit than Linux text logs, but it can be cleared. When it is, the event log records its own clearance.

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=1102} -ErrorAction SilentlyContinue
Get-WinEvent -FilterHashtable @{LogName='System'; Id=104} -ErrorAction SilentlyContinue
```

Event 1102 in Security is "Audit log was cleared." Event 104 in System is "Log file cleared" for non-Security logs. Both events include the user who did the clearing. Both are themselves a finding.

A cleared log plus a missing 1102/104 event is also possible (advanced attackers can disable Eventlog service before clearing). The triage: if the Security log has very few events and you cannot find a 1102, the absence is itself suspicious.

### PowerShell history clearance (T1070.003)

The PowerShell history file is the artifact behind "what did this user run."

```
$histPath = "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
Test-Path $histPath
if (Test-Path $histPath) {
    (Get-Item $histPath).Length
    Get-Content $histPath | Select-Object -Last 50
}
```

A missing or empty history file for a user who has clearly used PowerShell is a finding. As is a history file with `Set-PSReadlineOption -HistorySaveStyle SaveNothing` or `Clear-History` near the end.

### PSReadLine option tampering (T1562)

```powershell
Get-PSReadLineOption | Select-Object HistorySaveStyle, HistorySavePath
```

`HistorySaveStyle` should be `SaveIncrementally` (the default). `SaveNothing` is the explicit "do not save my history" setting. Found in attack scripts.

### Files in unusual locations (T1564)

```powershell
Get-ChildItem $env:TEMP -File -ErrorAction SilentlyContinue |
    Where-Object Extension -in '.exe','.dll','.ps1','.vbs','.bat','.scr' |
    Sort-Object LastWriteTime -Descending |
    Select-Object Name, Length, LastWriteTime
```

Executable content in temp directories is unusual. Some legitimate installers cache MSI files in temp during install; that is normal. Standalone executables in temp, especially with recent timestamps, warrant investigation.

Other suspicious locations:

```
Get-ChildItem $env:LOCALAPPDATA -Recurse -Include *.exe,*.dll,*.ps1 -ErrorAction SilentlyContinue |
    Where-Object { $_.FullName -notmatch '\\Microsoft\\|\\Programs\\|\\Mozilla\\|\\Google\\' }
```

That walks the user's LocalAppData looking for executable content outside known-good vendor directories. The pattern: legitimate apps have their own subdirectories; standalone executables in unusual paths are findings.

### Alternate data streams (T1564.004)

Touched on in Chapter 7. The query for hidden content in alternate streams:

```
Get-ChildItem 'C:\Users\student' -Recurse -ErrorAction SilentlyContinue |
    ForEach-Object { Get-Item $_.FullName -Stream * -ErrorAction SilentlyContinue } |
    Where-Object { $_.Stream -notin ':$DATA','Zone.Identifier' }
```

That walks a user directory and finds files with alternate streams that are not the standard ones. Most files have only `:$DATA`. Files downloaded from the internet have a `Zone.Identifier`. Anything else is a finding.

### Timestomping (T1070.006)

Attackers sometimes change file timestamps to make their files look old or unchanged. The classic check:

```powershell
Get-ChildItem $suspect_path -File | Select-Object Name, CreationTime, LastWriteTime, LastAccessTime
```

A file where CreationTime is much later than LastWriteTime is suspicious (the file was modified before it was created, in the timestamps). Sysmon Event 2 captures explicit timestamp manipulations if Sysmon is configured to log them.

### Disabled or modified Defender (T1562.001)

```powershell
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled, OnAccessProtectionEnabled, AMServiceEnabled
Get-MpPreference | Select-Object DisableRealtimeMonitoring, ExclusionPath, ExclusionExtension, ExclusionProcess
```

The first should report `True` for all four. Any `False` is a finding.

The second shows Defender exclusions. Exclusions are sometimes legitimate (specific business applications need them), but they are also a common attacker target: if an attacker can add `C:\Temp` to ExclusionPath, anything they put there is invisible to Defender. The exclusion list is itself a finding to review.

---

## Command and control: what attackers talk to

Persistence and evasion put the attacker in. C2 is how the attacker uses access. The artifacts are the network connections and the configuration that enables them.

### Active connections from unusual processes

```powershell
Get-NetTCPConnection -State Established |
    ForEach-Object {
        $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
        [PSCustomObject]@{
            Process = $proc.Name
            Path = $proc.Path
            Local = "$($_.LocalAddress):$($_.LocalPort)"
            Remote = "$($_.RemoteAddress):$($_.RemotePort)"
        }
    } | Where-Object { $_.Path -and $_.Path -notmatch 'Windows|Program Files' }
```

That returns connections owned by processes whose binaries are outside standard locations. PowerShell.exe in `C:\Windows\System32` is normal; a process running from `C:\Users\<user>\Temp` with active connections is not.

### DNS query history from Sysmon

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=22} -MaxEvents 100 |
    ForEach-Object {
        if ($_.Message -match 'QueryName: (\S+)') { $matches[1] }
    } | Sort-Object | Get-Unique
```

That extracts every unique hostname queried via DNS in the recent Sysmon events. Read the list. Anything you do not recognize is a question. DGA-style names (random-looking strings) are a particular pattern.

### PowerShell network activity (T1059.001)

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=3} -MaxEvents 200 |
    Where-Object { $_.Message -match 'powershell.exe' } |
    Select-Object -First 10 TimeCreated, Message
```

Sysmon Event 3 captures network connections. Filtered for PowerShell, you get every PowerShell-initiated outbound. Most of these are legitimate (Get-Module from Gallery, Update-Help, etc.); anything connecting to an IP without a hostname or to a non-Microsoft destination is worth understanding.

---

## A complete triage walkthrough

Putting it all together. The scenario: you are told "something is wrong" with the lab box. Walk the triage.

### Phase 1: situational awareness (5 minutes)

```
$env:COMPUTERNAME; Get-Date; (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
Get-LocalUser | Select-Object Name, SID, Enabled
Get-LocalGroupMember -Group Administrators
Get-Process | Where-Object Path | Sort-Object Path | Select-Object Name, Id, Path
Get-NetTCPConnection -State Listen | Select-Object LocalPort, OwningProcess
```

Know who you are looking at. Know who has admin. Know what is running. Know what is listening.

### Phase 2: persistence checks (10 minutes)

```
# Run keys
$paths = @('HKLM:\Software\Microsoft\Windows\CurrentVersion\Run', 'HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce', 'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Run', 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run', 'HKCU:\Software\Microsoft\Windows\CurrentVersion\RunOnce')
foreach ($p in $paths) {
    Get-ItemProperty $p -ErrorAction SilentlyContinue
}

# Startup folder
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" -Force -ErrorAction SilentlyContinue
Get-ChildItem "$env:ProgramData\Microsoft\Windows\Start Menu\Programs\Startup" -Force -ErrorAction SilentlyContinue

# Non-Microsoft scheduled tasks
Get-ScheduledTask | Where-Object TaskPath -notlike "\Microsoft*"

# Non-standard service paths
Get-CimInstance Win32_Service | Where-Object { $_.PathName -notmatch '^"?[Cc]:\\(Windows|Program Files( \(x86\))?)\\' }

# IFEO debuggers
Get-ChildItem 'HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options' |
    Where-Object { $_.Property -contains 'Debugger' }
```

For each output, decide: expected or not.

### Phase 3: signature audit (5 minutes)

```powershell
Get-Process | Where-Object Path | ForEach-Object {
    $sig = Get-AuthenticodeSignature -FilePath $_.Path -ErrorAction SilentlyContinue
    if ($sig.Status -ne 'Valid') {
        [PSCustomObject]@{
            Process = $_.Name
            Path = $_.Path
            Signed = $sig.Status
            Signer = $sig.SignerCertificate.Subject
        }
    }
}
```

Anything not Valid is a finding.

### Phase 4: evasion checks (5 minutes)

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=1102} -ErrorAction SilentlyContinue
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled
Get-MpPreference | Select-Object ExclusionPath, ExclusionProcess
$histPath = "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
if (Test-Path $histPath) { (Get-Item $histPath).Length } else { "MISSING" }
```

Cleared logs, disabled Defender, missing history: each is a finding.

### Phase 5: PowerShell adversarial review (10 minutes)

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104; StartTime=(Get-Date).AddDays(-7)} -ErrorAction SilentlyContinue |
    Where-Object { $_.Message -match 'EncodedCommand|FromBase64|DownloadString|Invoke-Expression|IEX|webclient' } |
    Select-Object -First 5 TimeCreated, Message
```

That returns PowerShell script blocks containing patterns associated with malicious PowerShell. False positives exist (legitimate scripts use IEX), but these warrant reading. Each match is a small investigation.

### Phase 6: timeline (5 minutes)

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4720,4732,4733,4738; StartTime=(Get-Date).AddDays(-30)}
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7045; StartTime=(Get-Date).AddDays(-30)}
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624; StartTime=(Get-Date).AddDays(-7)} | Select-Object -First 20 TimeCreated, Message
```

Account changes in the last month. Service installs in the last month. Logons in the last week.

That is roughly 40 minutes of triage. It is not exhaustive. It is the first pass that sets up the deeper investigation.

---

## Try this

These exercises assume the instructor has prepared a compromised lab box with planted artifacts. If you are working through the chapter without that box, do the exercises on a clean box and confirm the queries return empty (which is what a clean baseline should look like).

**1. Run the full triage walkthrough on the prepared lab box.**

Follow the six phases from the walkthrough above. Document your findings as you go: write each command you ran, what it returned, and your interpretation. Aim for a written triage report at the end.

**2. Find the planted persistence.**

The instructor has planted at least one persistence mechanism. Find it. Likely categories: a Run key value, a backdoor account, a malicious scheduled task, a malicious service, an IFEO debugger entry, or a Startup folder file.

For each finding, document:
- The exact location (registry path, file path, or task path).
- The relevant artifact content.
- The MITRE ATT&CK technique it represents.

**3. Find the planted evasion.**

The instructor has planted at least one evasion technique. Find it. Likely categories: cleared event log, missing or tampered PowerShell history, disabled Defender setting, suspicious Defender exclusion, file in an unusual location, or alternate data stream content.

Document as before, with ATT&CK references.

**4. Find the unsigned process or DLL, if planted.**

Run the bulk signature audit. If anything appears, identify the process, its parent, its command line, and what it is doing. This is the same investigation pattern from Chapter 10 applied to the suspicious finding.

**5. Read the PowerShell script block log adversarially.**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104} -MaxEvents 50 |
    Select-Object -First 10 | Format-List Message
```

Read the script blocks. Some may be legitimate (PowerShell modules loading themselves, PSReadLine setup). Others may be the planted content. Identify any that look suspicious and explain why.

**6. Write the report.**

Take your findings from exercises 1-5 and write a one-page IR report. Format:

- **Summary**: one paragraph stating what you found and your assessment.
- **Findings**: numbered list, one per artifact, with the location, evidence, and ATT&CK reference.
- **Recommendations**: what you would do to remediate, prioritized by what to do first.

The report is the deliverable. Working IR analysts produce these constantly, and the discipline of writing one is itself the skill.

---

## Common stumbling blocks

> **I cannot find anything suspicious on the lab box.**
> Either the artifacts are not yet planted (instructor has not configured the box), or you need to run with elevation. Most of the queries in this chapter need admin to read the relevant registry keys, scheduled tasks, and event logs. Confirm you are using elevated PowerShell.

> **The PowerShell history file is missing for a user.**
> Two possibilities: the user has not used PowerShell yet (legitimate, but rare on a long-running machine), or someone removed it (finding). Cross-reference with PowerShell Operational log Event 4104; if you see PowerShell activity but no history file, the file was deleted or redirected.

> **The "non-Microsoft scheduled task" query returns Microsoft tasks.**
> Some Microsoft tools place their tasks in `\` rather than `\Microsoft\` (Edge updater, OneDrive). The task's Author field tells you who registered it. Microsoft-signed binaries with Author='Microsoft Corporation' running from `C:\Program Files` are legitimate; the same binary running from `C:\Users\<user>\Temp` is not.

> **My signature audit returns "NotSigned" for several legitimate processes.**
> Some Windows components are signed via catalog files, not embedded signatures. `Get-AuthenticodeSignature` does not always check catalogs. Cross-reference with `sigcheck.exe` (Sysinternals) for these specific files. The tool does check catalogs.

> **The IFEO query returns many entries.**
> Some IFEO entries are legitimate (Microsoft's debugging helpers, .NET, anti-malware integration). The finding is specifically a Debugger value pointing at cmd.exe, powershell.exe, or a non-standard path. Other IFEO values (CFG flags, ASLR settings) are normal.

> **`Get-LocalUser` does not show some accounts I expect.**
> Domain accounts do not appear in Get-LocalUser. On a non-domain workstation, the only relevant accounts are local. If your investigation involves domain accounts, you need different tools (covered in Month 3).

> **A finding I documented disappeared on re-check.**
> Some artifacts (active connections, running processes) are point-in-time. If you do not capture them when you find them, they may not be visible later. The discipline: when you find something interesting, capture the full output to a file immediately. Do not assume you can re-run the query.

---

## What this gets you

After this chapter:

- You can do basic IR triage on a Windows host you did not build.
- You can run the six-phase triage walkthrough as a repeatable checklist.
- You can identify the most common Windows persistence techniques and find their artifacts: Run keys, RunOnce, IFEO, services, scheduled tasks, Startup folder, AppInit_DLLs, WMI subscriptions.
- You can identify the most common Windows evasion techniques and find their artifacts: cleared logs, history clearance, suspicious Defender exclusions, files in unusual locations.
- You can produce an IR report that an investigator coming after you can follow.
- You know enough MITRE ATT&CK to map findings to techniques and discuss them in the vocabulary the field uses.

This chapter is the bridge from "I can administer Windows" to "I can investigate Windows." Both skills are valuable on their own. The combination is what makes someone effective in a security operations or IR role.

The intermediate cohort builds significantly on this chapter, with deeper artifact analysis, memory forensics introduction, EDR-mediated detection (LimaCharlie), and Windows-specific threat hunting. This chapter is where that path begins on the Windows side.

---

## Placeholder: artifacts the instructor will plant

The compromised lab box should have a subset of the following artifacts planted. The exercises will tell you which categories to look for; this list is the recommended set for the instructor to build against. Each artifact maps to a category and a MITRE ATT&CK technique.

**1. Backdoor administrator account.** A user named to look legitimate (`support`, `mssql_admin`, `webservice_op`) added to the local Administrators group. The naming makes the account look like a service account; the membership makes it root-equivalent. ATT&CK T1136.001 plus T1078.003.

**2. Run key persistence.** A value in `HKCU:\Software\Microsoft\Windows\CurrentVersion\Run` named to look like a system service (`Updater`, `SystemCheck`, `WindowsService`) launching `powershell.exe -WindowStyle Hidden -ExecutionPolicy Bypass -File <path>`. ATT&CK T1547.001.

**3. Scheduled task with hidden PowerShell.** A scheduled task at the root TaskPath (`\`) named innocuously (`Microsoft Telemetry`, `Windows Update Helper`) whose action is `powershell.exe -EncodedCommand <base64>` or `powershell.exe -WindowStyle Hidden -File <path>`. Triggered AtLogOn or at a regular interval. Running as SYSTEM at high run level. ATT&CK T1053.005.

**4. IFEO Debugger abuse.** An entry in `HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\<some.exe>` with a `Debugger` value pointing at `cmd.exe` or `C:\Users\<user>\AppData\Local\Temp\<something>.exe`. The classic pattern is `sethc.exe` (Sticky Keys) but any common executable works. ATT&CK T1546.012.

**5. Startup folder file.** A file in `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup` named innocuously (`update.ps1`, `system-init.bat`) that runs at logon. ATT&CK T1547.001.

**6. Cleared Security log.** Run `wevtutil cl Security` from the lab box at some point, generating Event 1102. Students should find the 1102 event itself plus notice that the Security log is sparse. ATT&CK T1070.001.

**7. PowerShell history clearance.** Add `Set-PSReadlineOption -HistorySaveStyle SaveNothing` and `Clear-History` to the user's PowerShell history right before they run, plus delete the existing history file. The pattern that should be caught: history file is empty or contains only the disabling commands. ATT&CK T1070.003.

**8. Suspicious Defender exclusion.** Add `C:\Temp` or `C:\Users\<user>\AppData\Local\Temp` to ExclusionPath. Or add `powershell.exe` to ExclusionProcess. The exclusions disable Defender for those locations or processes. ATT&CK T1562.001.

**9. File in unusual location.** A `.ps1` or `.exe` in `C:\Users\student\AppData\Local\Temp` with a recent timestamp, named innocuously (`update.ps1`, `installer.exe`). Possibly referenced by one of the persistence mechanisms above. ATT&CK T1564.

**10. Unsigned process running with active connection.** A small unsigned executable (or PowerShell process running an unsigned script) with an active outbound connection to a stub destination. Caught by the bulk signature audit plus Get-NetTCPConnection. ATT&CK T1071.001 plus T1059.001.

The instructor should plant 4-6 of the 10 for the chapter exercises, mixed across persistence, evasion, and C2 categories. The exercises ask students to find at least one persistence and one evasion artifact; the rest are stretch goals or material for instructor-led debrief.

The instructor should also have a debrief session after students complete the exercises, walking through every planted artifact (whether students found it or not) with the ATT&CK reference and the reasoning. Students learn as much from the artifacts they missed as from the ones they found.

---

## What's next

Chapter 12 is Windows hardening with CIS alignment. The capstone of the unit. By the end you will have hardened your lab box against a curated subset of the CIS Microsoft Windows 11 Benchmark, with a documented before-and-after diff. That diff is your portfolio piece.
