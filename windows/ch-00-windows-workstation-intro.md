# Chapter 0: Windows Workstation Workshop

**You come in with:** a working Linux foundation from month 1. You can SSH, navigate, run a service, schedule a job. You have probably used Windows as a regular user but not as an admin.
**You leave with:** the ability to operate a Windows endpoint as an administrator, run real work in PowerShell instead of clicking through GUIs, manage local users and permissions, install software with winget, schedule tasks, and read the event log to figure out what happened. Five hours of typing, three blocks, one lab box.

**Time:** 5 hours, live with the cohort. No additional self-paced time required after the workshop, though re-reading this chapter as a reference is recommended.

**Security+ alignment:** No direct exam content for this workshop chapter. The workshop is foundation skill that supports later chapters mapping to Domain 2 (Threats and Vulnerabilities), Domain 3 (Security Architecture), and Domain 4 (Security Operations).

---

## How to use this document

This is the workshop chapter for month 2 of the program. It mirrors what we cover live in the room. If you are reading it during the workshop, follow along. If you are reading it after, this is your reference for what you did and the commands you used.

The document is structured the way the day runs: an open, three blocks separated by a break and lunch, then a wrap. Each block has explanatory text, the commands you typed, and the lab you ran. Use the slides as the visual; use this as the reference.

A few conventions:

`Commands look like this`. They are typed in monospace so you can copy them.

```powershell
Multi-line examples
look like this.
```

PowerShell prompts in examples appear as `PS C:\Users\student>`. We omit the prompt when showing only commands you would type.

---

## Open and orientation

Last month was Linux. This month is Windows. The two platforms are different in surface ways and similar in deeper ways. The same skills you built on Linux apply here: read the system, do not guess; small commands compose; the people who get good are the people who type.

A few things that are genuinely different about Windows that are worth flagging up front.

**The default user experience is GUI-mediated.** On Linux, the terminal is where serious work happens; the GUI is optional. On Windows, the GUI is where most users live, and admin work that does not happen in a GUI happens in PowerShell. Both apply. We use the GUI today to orient you; PowerShell is the tool we use to actually do things. That mix matches how working sysadmins run Windows.

**PowerShell is not bash.** It looks similar at first. The mental model is fundamentally different. Bash pipes text between commands. PowerShell pipes objects. That distinction changes everything you do with it, and we cover the implications today. *PowerShell is Microsoft's modern shell and scripting language. It is built on .NET, which means everything in PowerShell is a typed object with properties and methods.*

**The registry is the configuration store.** Linux has /etc with text files. Windows has the registry, a hierarchical key-value store with its own syntax and tooling. We do not become registry experts today. We learn enough to recognize what the registry is and how to read from it.

**Sysmon is pre-loaded on your lab box.** *Sysmon, short for System Monitor, is a Microsoft utility that adds detailed system telemetry to the Windows event log: process creation, network connections, file changes, image loads, and more.* The instructor has set it up. Block 3 covers reading what it captures. You will not configure or install Sysmon today; that is a real-world admin task with its own complexity and is intermediate cohort material.

**Skill, not talent.** Same as Linux. The people who type are the people who learn.

### Your lab box

Your CourseStack lab is a Windows 11 VM. You connect through the CourseStack web interface; the connection is mediated by Apache Guacamole, which means you click a button in the browser and get the Windows desktop in a browser tab. Everything happens inside that session.

A few small notes about working through Guacamole:

- **Clipboard works, but with a small ritual.** Look for the Guacamole menu (typically Ctrl+Alt+Shift opens a side panel). The clipboard pane in that menu is the bridge between your laptop's clipboard and the VM's clipboard. Paste text into the pane on your side, then paste from the VM's clipboard inside the VM.
- **Latency is fine for typing but visible for fast scrolling.** This is one reason today's workshop leans heavily on PowerShell rather than scrolling through GUI lists. PowerShell with filters renders instantly; a GUI list with thousands of entries through Guacamole is annoying.
- **The session persists across the workshop.** Anything you install or change stays. Closing the browser tab and reopening returns you to your session.

Once you are connected and at the desktop, open PowerShell. There are several ways. The fastest:

1. Press the Windows key.
2. Type `powershell`.
3. Press Enter.

A blue terminal window opens with a prompt. That prompt is where the workshop happens.

**Important note about PowerShell versions.** Windows ships with **Windows PowerShell 5.1** (the blue one). Microsoft also publishes **PowerShell 7+** (the black one), the modern cross-platform version. We use Windows PowerShell 5.1 today because it is what every Windows machine has by default, including in incident response scenarios where you cannot install anything. The skills transfer to PowerShell 7 with minor differences.

To confirm which version you are in:

```
$PSVersionTable
```

You should see `PSVersion: 5.1.x`. That is correct for today.

---

## Block 1: Getting around the modern Windows endpoint

By the end of this block, you can navigate a Windows machine in PowerShell, find your way around the directory structure, read files, and start to use PowerShell's pipeline as a real tool rather than a curiosity.

### The PowerShell prompt and basic navigation

The PowerShell prompt shows your current location. By default it is your home directory: `C:\Users\<username>`.

The basic navigation commands look familiar from Linux but use the Verb-Noun convention:

```
Get-Location              # equivalent to pwd
Set-Location C:\Windows   # equivalent to cd /etc
Get-ChildItem             # equivalent to ls
```

PowerShell ships with aliases for the bash equivalents:

```
pwd
cd C:\Windows
ls
```

Both forms work. The bash-style aliases are convenient and you will use them constantly. The full Verb-Noun forms are what scripts use because they are unambiguous and self-documenting. Today we mix freely.

Try a few:

```
cd C:\Users\student
pwd
ls
ls -Force                 # equivalent to ls -la, shows hidden
cd C:\Windows
ls *.exe | Select-Object -First 10
```

The last command introduces something important: the pipeline. We come back to it shortly.

### The Windows directory structure

Windows uses backslashes (`\`) as path separators. PowerShell tolerates either, but Windows applications expect backslashes. When you write paths in scripts, use backslashes. Stick the path in single quotes to avoid backslash escaping problems: `'C:\Program Files\Something'` is safer than `"C:\Program Files\Something"`.

The directories that matter most:

| Path | What it holds |
|------|--------------|
| `C:\Windows` | The OS itself. System binaries, libraries, drivers. |
| `C:\Windows\System32` | Most system binaries. Yes, even on 64-bit; the name is historical. |
| `C:\Windows\SysWOW64` | 32-bit binaries on 64-bit Windows. The naming is intentionally backwards. |
| `C:\Program Files` | Installed applications, 64-bit. |
| `C:\Program Files (x86)` | Installed applications, 32-bit. |
| `C:\ProgramData` | Application data shared by all users. The Windows analog to /var. |
| `C:\Users` | User profiles. Each user has a folder. |
| `C:\Users\<name>\AppData` | Per-user application data. Hidden by default. |
| `C:\Windows\Temp` | System temp space. |
| `C:\Users\<name>\AppData\Local\Temp` | Per-user temp space. |

A quick tour:

```
ls C:\Windows\System32 | Select-Object -First 5
ls 'C:\Program Files'
ls C:\Users
ls $env:USERPROFILE          # your home directory, as an environment variable
ls $env:USERPROFILE\AppData -Force
```

`$env:USERPROFILE` is how you reference environment variables in PowerShell. The `$env:` prefix is the namespace; the variable name follows. There are dozens of useful environment variables: `$env:COMPUTERNAME`, `$env:USERNAME`, `$env:PATH`, `$env:TEMP`. Try `Get-ChildItem env:` to see them all.

### Anatomy of a PowerShell command

PowerShell commands follow a Verb-Noun pattern: `Get-Process`, `Set-Location`, `Stop-Service`. The verb describes the action, the noun describes the target. Almost every command follows this convention.

```
Get-Process               # list processes
Get-Service               # list services
Get-ChildItem             # list directory contents
Get-Help Get-Process      # documentation for a command
```

The `Get-Help` command is your friend. To see real help (not just the brief summary):

```
Get-Help Get-Process -Full
Get-Help Get-Process -Examples
```

If help looks sparse, run `Update-Help` once (requires admin and an internet connection) to download the full documentation.

Tab completion works on commands, parameters, and values. Type `Get-Pr` and press Tab; PowerShell cycles through matching commands. Type `Get-Process -Na` and press Tab; PowerShell expands to `-Name`. Tab completion is the fastest way to discover what is available.

### Discovering commands

```
Get-Command                    # every available command, hundreds of them
Get-Command *service*          # commands related to services
Get-Command -Verb Get          # every Get-* command
Get-Command -Noun Service      # every command targeting services
```

The fact that you can ask the shell what it can do is genuinely useful. On Linux, you guess command names from your knowledge. In PowerShell, you ask. This is one of PowerShell's real wins over bash for new users.

### The pipeline: where PowerShell starts to feel different

In bash, `ps aux | grep nginx` works because both commands process text and grep filters lines. The pipe carries text bytes from one command to the next.

In PowerShell, the pipeline carries objects, not text. Each object has properties.

```
Get-Process | Where-Object CPU -gt 100
Get-Process | Sort-Object -Property CPU -Descending | Select-Object -First 10
Get-Process | Where-Object Name -like "*explorer*"
```

Reading the second one: get all processes, sort them by the CPU property descending, take the first 10. There is no text parsing because there is no text. Each process is an object with named properties (Id, Name, CPU, WorkingSet, etc.) and the pipeline hands them down.

To see what properties an object has:

```
Get-Process | Get-Member
Get-Process | Select-Object -First 1 | Format-List *
```

`Get-Member` shows every property and method of the objects flowing through the pipeline. This is the discoverability moment for new PowerShell users: any object you can produce, you can ask what it can tell you. Run `Get-Member` on the output of any command when you are not sure what you are looking at.

Try a few practical pipelines:

```
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 5 Name, WorkingSet
Get-Service | Where-Object Status -eq Running | Select-Object Name, DisplayName
Get-ChildItem C:\Windows -File | Sort-Object Length -Descending | Select-Object -First 5
```

Each one composes small commands into a useful result. This is the Windows version of the "small commands compose" idea you learned in Linux. The mechanism is different (objects instead of text) but the principle is the same.

### A note on case

Most things in PowerShell are case-insensitive. `Get-Process`, `get-process`, and `GET-PROCESS` are all the same. Variable names are case-insensitive too: `$Name` and `$name` reference the same variable. This differs from bash, where everything is case-sensitive.

The exception: string comparisons. `"Hello" -eq "hello"` is true (PowerShell defaults to case-insensitive string comparison). For case-sensitive comparison use the `-c` versions: `-ceq`, `-clike`, etc.

### The execution policy

Before we run any script later in the day, you will hit the execution policy. Run:

```
Get-ExecutionPolicy
```

If it returns `Restricted` (often the default), you cannot run script files yet. We change that in Block 3 when we write our first script. For now, just know that PowerShell has a per-machine setting that controls whether scripts can run; this is a security feature, not an obstacle.

### Lab: orient yourself on Windows

You connected to a Windows machine. You are in PowerShell. Find out what is on it.

1. Get the OS version and the computer name. Try `Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, CsName`.
2. List every running service whose name contains "Win". Try `Get-Service | Where-Object Name -like "*Win*"`.
3. Find the 10 largest files under `C:\Windows`. Use Get-ChildItem with `-Recurse`, Sort-Object on Length, and Select-Object. Suppress permission errors with `-ErrorAction SilentlyContinue`.
4. List the local users on the machine. Try `Get-LocalUser`.
5. Show the running processes sorted by memory use. Use Get-Process and the WorkingSet property.

If you finish early: pick one of the running services and use `Get-Service <name> | Format-List *` to see every property the service object has. Notice how many properties exist beyond what the default output shows. PowerShell shows you a summary by default; the full picture is one Format-List away.

---

## Block 2: Running and managing things

Block 1 was navigation. Block 2 is doing things to the system: managing local users, setting permissions, installing software, controlling services, and scheduling tasks.

### UAC and the elevation model

First, an essential concept. Windows has User Account Control. *UAC, User Account Control, is the Windows mechanism that prompts for explicit permission when an action requires administrative privilege, even if the current user is an administrator.*

This is why a PowerShell window opened normally cannot do everything an admin can do, even if you logged in as an admin. The UAC token strips administrative rights from your normal session and grants them only when you explicitly elevate.

To run PowerShell elevated:

1. Press the Windows key.
2. Type `powershell`.
3. **Right-click** the result and choose "Run as administrator."

The new window has `Administrator:` in its title. That window has the elevated token.

In an admin PowerShell window, you can see the difference:

```
whoami /groups | findstr "Mandatory"
```

In a normal window, the output shows `Mandatory Label\Medium Mandatory Level`. In an admin window, it shows `Mandatory Label\High Mandatory Level`. That is the elevation difference.

Almost everything in this block requires the admin window. Open one now.

### Local users and groups

Linux had `useradd`. Windows has `New-LocalUser`.

Look at the existing accounts:

```
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember -Group Administrators
```

`Get-LocalUser` shows all local accounts on the box. `Get-LocalGroup` shows the groups. `Get-LocalGroupMember Administrators` shows who is in the local Administrators group; on a fresh Windows 11, this is the default user (you, `student`) plus the built-in Administrator account (which is disabled).

Create a service account:

```
$Pass = Read-Host -AsSecureString -Prompt "New password"
New-LocalUser -Name "webrunner" -Password $Pass -Description "Service account" -PasswordNeverExpires -AccountNeverExpires
```

Walk through what happened:

- `Read-Host -AsSecureString` reads input but stores it as an encrypted string in memory, not plain text. This is the right pattern for any secret in PowerShell.
- `New-LocalUser` creates the account.
- `-PasswordNeverExpires` and `-AccountNeverExpires` are reasonable defaults for a service account that should not have its password expire automatically.

Confirm:

```
Get-LocalUser webrunner
```

Add the user to a group:

```
New-LocalGroup -Name "WebOps" -Description "Web service operators"
Add-LocalGroupMember -Group "WebOps" -Member "webrunner"
Get-LocalGroupMember -Group "WebOps"
```

The pattern is symmetric: New/Remove for the things themselves, Add/Remove for membership.

### Built-in accounts you should know

Windows has a small set of built-in accounts that are worth recognizing:

| Account | Role |
|---------|------|
| Administrator | The original local admin. Disabled by default on Windows 11. |
| Guest | Unauthenticated access. Disabled by default. Should stay disabled. |
| DefaultAccount | Reserved for system features. Leave alone. |
| WDAGUtilityAccount | Used by Windows Defender Application Guard. Leave alone. |

Plus pseudo-accounts (security principals that are not real interactive users):

- **`SYSTEM`** (or `LocalSystem`): the highest-privilege local account. Many services run as SYSTEM. Cannot log in interactively.
- **`LocalService`** and **`NetworkService`**: lower-privilege service accounts.
- **`<computer>\<username>`**: a local user. Note the computer name prefix.

When investigating later, you will see these names in event logs. Recognizing them matters.

### NTFS permissions

Windows uses NTFS as its filesystem. NTFS permissions are richer than the rwx model on Linux, closer to ACLs in their flexibility.

The basic operations:

```
Get-Acl C:\Users\student
Get-Acl C:\Users\student | Format-List
Get-Acl C:\Users\student | Select-Object -ExpandProperty Access
```

The output shows the Access Control List for the directory. Each access control entry is one rule: an identity, an inheritance setting, and a set of rights.

Common rights:

| Right | Meaning |
|-------|---------|
| FullControl | Read, write, execute, delete, change permissions, take ownership |
| Modify | Read, write, execute, delete |
| ReadAndExecute | Read and run, no write |
| Read | Read only |
| Write | Write only |

A practical example: create a directory and set permissions so only `webrunner` and admins can access it. The simpler tool is `icacls`:

```
New-Item -ItemType Directory -Path "C:\AppData\webops"
icacls "C:\AppData\webops" /inheritance:r
icacls "C:\AppData\webops" /grant webrunner:M
icacls "C:\AppData\webops" /grant Administrators:F
icacls "C:\AppData\webops" /remove Users
icacls "C:\AppData\webops"
```

Reading those:

- `/inheritance:r` removes inheritance, so the directory does not inherit from its parent.
- `/grant webrunner:M` grants webrunner Modify rights.
- `/grant Administrators:F` grants admins Full control.
- `/remove Users` removes the default Users entry.
- The last line shows the result.

A pure-PowerShell version using `Set-Acl` exists but is verbose. For one-off changes, `icacls` is faster. Scripts use one or the other; pick a consistent pattern. The chapter on permissions later in this unit goes deeper.

### Installing software with winget

Windows has had a fragmented software-install story for years. Each app had its own installer, downloaded from the vendor's website, signed differently, updated separately. winget is Microsoft's modern package manager that fixes this.

```
winget --version
winget search 7zip
winget install 7zip.7zip --accept-source-agreements --accept-package-agreements
winget list
winget upgrade
```

Walk through what happened:

- `winget --version` confirms winget is installed (Windows 11 ships with it).
- `winget search` finds packages by name.
- `winget install` installs by package ID. The `--accept-*` flags suppress interactive prompts.
- `winget list` shows what is installed.
- `winget upgrade` shows available upgrades; with no arguments it lists, with `--all` it actually upgrades.

For multiple installs:

```
winget install Notepad++.Notepad++ --silent
winget install Microsoft.PowerShell --silent
```

`--silent` runs the installer without UI. winget is the closest analog to apt. Same warning applies: do not download MSI/EXE installers from random sites when the package is in winget's repository.

### Services

Linux had systemd. Windows has the Service Control Manager. The PowerShell commands:

```
Get-Service                              # list every service
Get-Service -Name Spooler                # one specific service (Print Spooler)
Get-Service | Where-Object Status -eq "Running"
```

Manage a service:

```
Start-Service -Name Spooler
Stop-Service -Name Spooler
Restart-Service -Name Spooler
Set-Service -Name Spooler -StartupType Disabled
```

The Set-Service `-StartupType` values: Automatic, AutomaticDelayedStart, Manual, Disabled. `Disabled` means the service cannot start, even on demand.

For deeper service info there is `Get-CimInstance`:

```
Get-CimInstance -ClassName Win32_Service -Filter "Name='Spooler'" | Format-List Name, StartName, PathName, State
```

`StartName` is the account the service runs as. `PathName` is the executable plus arguments. These two fields matter for security work because a service running as a privileged account, with an executable path you do not recognize, is a finding.

### Scheduled Tasks

Linux had cron and systemd timers. Windows has the Task Scheduler.

```
Get-ScheduledTask | Where-Object State -eq Ready | Select-Object TaskName, TaskPath -First 10
Get-ScheduledTask -TaskName "GoogleUpdater" -ErrorAction SilentlyContinue | Get-ScheduledTaskInfo
```

To create a scheduled task that runs a command every 5 minutes, writing to a log:

```
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -Command `"Add-Content -Path C:\AppData\webops\heartbeat.log -Value (Get-Date).ToString()`""
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 5)
$Principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "Heartbeat" -Action $Action -Trigger $Trigger -Principal $Principal -Description "Heartbeat log writer"
```

Walk through what happened:

- The Action says what to run: PowerShell with `-NoProfile` (skip loading user profiles for speed) executing a one-line command that appends the current date to a file.
- The Trigger says when: starting now, every 5 minutes, indefinitely.
- The Principal says as whom: SYSTEM at the highest run level.
- Register-ScheduledTask saves the task.

To inspect or remove:

```
Get-ScheduledTask -TaskName "Heartbeat" | Get-ScheduledTaskInfo
Unregister-ScheduledTask -TaskName "Heartbeat" -Confirm:$false
```

### Lab: provision a service account, lock down a directory, schedule a task

Combining what we just covered.

1. **Create a local user** named `srvaccount` with a password you choose. Make the account `-PasswordNeverExpires` and disable interactive login by adding it to no privileged groups (it will be in Users by default).

2. **Create a directory** at `C:\AppData\heartbeat`. Use icacls to set the ACL: srvaccount has Modify, Administrators have Full, Users have nothing. Confirm with `icacls C:\AppData\heartbeat`.

3. **Install Notepad++** with winget if it is not already installed. Confirm with `Get-Command notepad++.exe -ErrorAction SilentlyContinue` and with `winget list Notepad++`.

4. **Schedule a task** that, every 2 minutes, writes the current date to `C:\AppData\heartbeat\beats.log`. Use SYSTEM as the principal. Confirm with `Get-Content C:\AppData\heartbeat\beats.log` after a few minutes.

5. **Read the task definition.** Run `Get-ScheduledTask -TaskName "Heartbeat" | Format-List *` and identify the Action's executable path, the trigger interval, and the principal.

If you finish early: change the scheduled task's principal to `srvaccount` instead of SYSTEM. You will hit a permissions issue (srvaccount cannot write to `C:\AppData\heartbeat\beats.log` because the directory is owned by Admins, even though the ACL grants srvaccount Modify). Diagnose and fix using icacls. This is the equivalent of the "permission denied because nginx runs as www-data" lesson from Linux Block 2.

---

## Block 3: Reading the system

Block 1 was navigation. Block 2 was managing things. Block 3 is reading the system: events, processes, network state, and a small script that ties it together.

### Event Viewer and Get-WinEvent

Linux had journalctl. Windows has Event Viewer (the GUI) and `Get-WinEvent` (PowerShell).

Open Event Viewer briefly to see the structure:

1. Press Windows key.
2. Type `event viewer`, press Enter.
3. Expand "Windows Logs" in the left tree. You see Application, Security, Setup, System.
4. Expand "Applications and Services Logs". You see many channels, including Microsoft > Windows > Sysmon > Operational (the channel Sysmon writes to).

Close Event Viewer. From here on we use PowerShell.

```
Get-WinEvent -ListLog * | Where-Object RecordCount -gt 0 | Sort-Object RecordCount -Descending | Select-Object -First 10
```

That lists the channels with the most events. The big ones on a typical Windows 11 box: System, Application, Security, Microsoft-Windows-Sysmon/Operational.

### Reading the System log

```
Get-WinEvent -LogName System -MaxEvents 20
Get-WinEvent -LogName System -MaxEvents 20 | Select-Object TimeCreated, Id, ProviderName, LevelDisplayName, Message
```

Each event has a numeric Id that identifies what happened, an Id-specific Message, and metadata. Filter by event ID:

```
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7045} -MaxEvents 10
```

Event 7045 is "a service was installed in the system." Recently installed services show up here. We come back to this in Chapter 11.

### Reading the Security log

The Security log is where authentication and authorization events live. Reading it requires admin rights.

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 10
```

Event 4624 is "an account was successfully logged on." This is the Windows analog to `grep "Accepted" /var/log/auth.log` from Linux.

The big Security log event IDs you should recognize:

| Event ID | Meaning |
|----------|---------|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4634 | Logoff |
| 4648 | Explicit credential use (e.g., `runas`) |
| 4672 | Special privileges assigned to new logon (admin token granted) |
| 4720 | User account created |
| 4732 | Member added to a security-enabled local group |
| 4738 | User account changed |
| 7045 | Service installed (System log, not Security) |

You do not memorize all of these today. You memorize 4624, 4625, 4720, and 7045, which are the four most useful in practice.

### Reading what Sysmon captures

Sysmon is pre-loaded on your lab box. Its events go to a separate channel:

```
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 10
```

Sysmon event IDs you should recognize:

| Event ID | Meaning |
|----------|---------|
| 1 | Process creation |
| 3 | Network connection |
| 7 | Image (DLL) loaded |
| 8 | CreateRemoteThread (often suspicious) |
| 11 | File create |
| 13 | Registry value set |
| 22 | DNS query |

Sysmon's Event 1 is particularly useful. It captures every process that starts on the box, with the parent process, the command line, and the user. This is the kind of detail the default Windows event log does not provide.

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1} -MaxEvents 5 |
    Select-Object TimeCreated, Message
```

Read one of those messages. You see fields for `ProcessId`, `Image`, `CommandLine`, `User`, `ParentImage`, `ParentCommandLine`. This level of detail is what makes Sysmon valuable for security work. The intermediate cohort uses an EDR (LimaCharlie) that captures similar telemetry; recognizing what Sysmon shows you now means you understand what your EDR is doing later.

### Filtering Get-WinEvent like a pro

Get-WinEvent's `-FilterHashtable` parameter lets you filter at the source rather than retrieving everything and filtering after. This is dramatically faster on busy logs.

```
# Failed logons in the last hour
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4625
    StartTime=(Get-Date).AddHours(-1)
}

# Process creations from PowerShell, ever
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'
    Id=1
} | Where-Object { $_.Message -match 'powershell.exe' } | Select-Object -First 5

# Service installs in the last 24 hours
Get-WinEvent -FilterHashtable @{
    LogName='System'
    Id=7045
    StartTime=(Get-Date).AddDays(-1)
}
```

The hashtable form is the right pattern for any non-trivial query. Memorize it.

### Processes and process trees

```
Get-Process | Where-Object Name -like "*svchost*"
Get-Process -Name svchost | Select-Object Id, Name, StartTime, Path
```

PowerShell does not have a built-in tree view for processes the way pstree does on Linux. The `Get-CimInstance Win32_Process` view includes ParentProcessId, which lets you build the tree:

```
Get-CimInstance Win32_Process | Select-Object ProcessId, ParentProcessId, Name, CommandLine | Format-Table -AutoSize
```

For a real tree view, the de facto standard tool is **Process Explorer** from the Sysinternals suite. We do not install it today (it requires a download from sysinternals.com and is intermediate cohort territory). Recognition-level: when you hear "Process Explorer," it is the GUI tool that shows process trees with the parent-child relationships and a lot more detail than Task Manager.

### Network state

```
Get-NetTCPConnection | Where-Object State -eq Listen | Select-Object LocalAddress, LocalPort, OwningProcess
Get-NetTCPConnection | Where-Object State -eq Established | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, OwningProcess
```

The `OwningProcess` field is a PID. To resolve to a process name:

```
Get-NetTCPConnection -State Listen | ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess
    [PSCustomObject]@{
        LocalAddress = $_.LocalAddress
        LocalPort = $_.LocalPort
        Process = $proc.Name
        ProcessId = $proc.Id
    }
}
```

That is verbose. The Linux version was `ss -tlnp`. The Windows version is more code but the same information. The pattern of "join two related queries with ForEach-Object" is genuinely useful in PowerShell.

For DNS:

```
Resolve-DnsName example.com
Resolve-DnsName example.com -Type MX
```

For HTTP:

```
Invoke-WebRequest -Uri https://example.com -UseBasicParsing | Select-Object StatusCode, Headers
```

`Invoke-WebRequest` is the Windows equivalent of curl. The `-UseBasicParsing` flag avoids loading the full HTML parser, which is faster and avoids issues on systems without IE installed (which Windows 11 is).

### A small PowerShell script

Linux had its tiny script with `#!/usr/bin/env bash`. Here is the Windows equivalent.

First, allow scripts to run on this machine. Open an elevated PowerShell:

```
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

`RemoteSigned` allows local scripts to run unconditionally and requires downloaded scripts to be signed. `-Scope CurrentUser` applies the change only to your account, not machine-wide. This is the right balance for a development environment.

Now write a script. Save the following as `C:\AppData\heartbeat\check.ps1`:

```powershell
# check.ps1: simple system check script
#Requires -Version 5.1

$ErrorActionPreference = 'Stop'

$LogPath = 'C:\AppData\heartbeat\check.log'
$Timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'

$BootTime = (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
$UptimeHours = [math]::Round(((Get-Date) - $BootTime).TotalHours, 1)

$RunningServices = (Get-Service | Where-Object Status -eq Running).Count

$Output = "$Timestamp | Uptime: $UptimeHours h | Running services: $RunningServices"

Add-Content -Path $LogPath -Value $Output
Write-Host $Output
```

A few things going on:

- `#Requires -Version 5.1` makes the script refuse to run on older PowerShell versions.
- `$ErrorActionPreference = 'Stop'` is the rough equivalent of `set -e` in bash. Errors abort the script instead of continuing.
- `Get-CimInstance Win32_OperatingSystem` is one of the Windows CIM classes; this one tells us about the OS, including last boot time.
- `[math]::Round(...)` is a static .NET method call. PowerShell can call .NET directly.
- `Add-Content` appends. `Set-Content` overwrites. (PowerShell's equivalents of `>>` and `>`.)

Run it:

```
C:\AppData\heartbeat\check.ps1
```

Confirm:

```
Get-Content C:\AppData\heartbeat\check.log
```

Now wire it to a scheduled task that runs every 10 minutes:

```
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\AppData\heartbeat\check.ps1"
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 10)
$Principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "SystemCheck" -Action $Action -Trigger $Trigger -Principal $Principal
```

The `-ExecutionPolicy Bypass` on the powershell.exe line makes the scheduled task ignore the system's execution policy for that one invocation. This is a common pattern and is safe because the task definition itself is the authorization boundary.

### Lab: investigate the system

The capstone lab. You wear the security analyst hat for this one.

1. Find the 5 most recently installed services. Use `Get-WinEvent` with `LogName='System'` and `Id=7045`. Read each event's Message and identify the service name and binary path.

2. Find every successful logon to your user account in the last hour. Use `Get-WinEvent` with `LogName='Security'`, `Id=4624`, and a time filter. The Message field includes the account name; filter for your user.

3. Find every process that PowerShell has launched in the last 30 minutes, using Sysmon. Use `Get-WinEvent` with `LogName='Microsoft-Windows-Sysmon/Operational'`, `Id=1`. Pipe through Where-Object to match `powershell.exe` in the Message field. (Hint: use `-Match` for regex matching against the multiline Message string.)

4. List every TCP connection your box has open, with the process name owning each. Use the join pattern from the Network state section.

5. Run your check.ps1 script. Confirm the log file appended a new line. Run `Get-WinEvent` to find the corresponding Sysmon Event 1 (process creation) for that PowerShell invocation. Read the CommandLine field. You can see your own script execution from the security telemetry.

That last exercise is the punchline of the workshop. You are reading your own activity in the security log, the way an investigator would read someone else's. It is the natural bridge to the post-workshop chapters and to the intermediate cohort's detection engineering work.

---

## Wrap

### What you can now do

Five hours ago this was a Windows 11 desktop. Now you can:

- Connect to a Windows machine through the CourseStack web UI and work in PowerShell.
- Navigate the Windows directory structure and recognize what lives where.
- Use the PowerShell pipeline to compose small commands into useful results.
- Manage local users, groups, and NTFS permissions.
- Install software with winget. Manage services with PowerShell. Schedule tasks.
- Read the Windows event log with Get-WinEvent and FilterHashtable.
- Recognize the major Security log event IDs and the Sysmon channel.
- Write a small PowerShell script with the safety equivalent of `set -e`.
- Investigate basic process and network state on a Windows host.

That is functional Windows administrator skill. Some of it has not stuck yet. The post-workshop chapters are where it sticks.

### What got cut

Five hours is enough to make you functional, not exhaustive. Here is what we deliberately skipped, with the chapter where it lives in the post-workshop unit.

- **The Windows mental model**: registry, profiles, services-vs-tasks, the deeper architecture. Chapter 1.
- **PowerShell foundations at depth**: profiles, modules, .NET interop, error handling. Chapter 2.
- **Permissions deep dive**: explicit ACLs, ownership, share permissions, EFS. Chapter 3.
- **Software installation deeper**: MSI internals, app deployment patterns, WSUS. Chapter 4.
- **Services and Scheduled Tasks deeper**: service accounts, trigger types, principal options. Chapter 5.
- **Event log deep dive**: audit policy, log forwarding (WEF), retention, Sysmon configuration. Chapter 6.
- **Storage and BitLocker**: disk management, BitLocker, Volume Shadow Copy. Chapter 7.
- **Host networking on Windows**: Windows Defender Firewall configuration, network profiles. Chapter 8.
- **PowerShell scripting at depth**: functions, modules, error handling, reading scripts adversarially. Chapter 9.
- **Reading processes on Windows**: signed binaries, DLLs, parent-child analysis. Chapter 10.
- **Windows artifacts**: registry persistence, scheduled task abuse, prefetch, the things that show up during incident response. Chapter 11.
- **Hardening**: CIS Microsoft Windows 11 Benchmark, Defender configuration, AppLocker, local Group Policy. Chapter 12.

### Where to keep going

**Tonight, the post-workshop unit opens.** Twelve self-paced chapters, around eleven hours of guided content, with the same lab box. Start with **Chapter 1: The Windows Mental Model**, which turns the directories and concepts you saw today into a working map of the platform.

**Practice between sessions.** A working knowledge of Windows comes from administering Windows. Spin up a Windows VM at home (Hyper-V is free and built into Windows; VirtualBox runs everywhere) and break things on it intentionally. The fastest way to get good at Windows is to own a Windows box you cannot mess up at work.

**Looking ahead.** Month 3 is Active Directory, where Windows starts to feel like a real business platform rather than a single endpoint. The skills you built today (PowerShell, services, event logs, NTFS permissions) all extend into AD. Show up to month 3 fluent in this material and the AD content lands faster.

---

## Common stumbling blocks

> **`Get-WinEvent` returns "No events were found."**
> Either the filter is too restrictive (try widening the time window or removing one filter), the log channel is empty (some channels are off by default), or you need elevation (the Security log requires admin). Run from an elevated PowerShell and try a less-restrictive filter.

> **My PowerShell command says "term is not recognized."**
> Either you typed it wrong (PowerShell is case-insensitive but spelling matters), or the command is in a module that is not loaded (try `Import-Module <name>`), or you are in PowerShell 5.1 trying to use a PowerShell 7-only feature. Double-check spelling first.

> **My script will not run, even though I saved it.**
> The execution policy is blocking. Check with `Get-ExecutionPolicy`. Set it to RemoteSigned at the user scope: `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`. For one-off bypass, run with `powershell.exe -ExecutionPolicy Bypass -File yourscript.ps1`.

> **`Get-LocalUser` says "command not found."**
> The Microsoft.PowerShell.LocalAccounts module needs to be imported. Try `Import-Module Microsoft.PowerShell.LocalAccounts`. On Windows 11 it is usually already available; if it is not, the older alternative is `net user`.

> **icacls says "access denied" even though I am admin.**
> File ownership matters. If a file is owned by SYSTEM or TrustedInstaller, even Administrators cannot modify the ACL without first taking ownership. Use `takeown /f <path> /a` to take ownership, then icacls to set permissions. (Use this carefully; some files are SYSTEM-owned for good reasons.)

> **Get-Process | Get-Member shows different methods than I expected.**
> Get-Member shows the type of object flowing through the pipeline at that point. If you piped through Select-Object first, the resulting objects are PSCustomObject, not the original type. To inspect the original, use Get-Member directly: `Get-Process | Get-Member` (no Select in between).

> **My scheduled task is registered but never runs.**
> Common causes: the trigger is in the past with no repeat (use a future StartBoundary or RepetitionInterval), the principal does not have logon-as-service rights (check Local Security Policy), or the action's executable path is wrong (check Get-ScheduledTask -TaskName | Format-List for the full action). Look at the task in Task Scheduler GUI to see "Last Run Result" for hints.

> **Through Guacamole, copy-paste from my laptop into the VM does not work.**
> Use the Guacamole side panel (typically Ctrl+Alt+Shift). Paste your text into the clipboard pane there, then paste from the VM clipboard inside the VM. The two clipboards are separate.

---

## Reference: every command from today

A condensed list, organized by block, for when you need to look something up.

**Block 1 (Getting around)**

```
$PSVersionTable
Get-Location  /  pwd
Set-Location  /  cd
Get-ChildItem  /  ls  /  dir
Get-ChildItem -Force                          # show hidden
Get-Help <command> -Full
Get-Help <command> -Examples
Get-Command
Get-Command *service*
Get-Command -Verb Get
Get-Command -Noun Service
Get-Member                                    # introspect pipeline objects
Get-Process
Get-Service
Where-Object <prop> -<op> <val>
Sort-Object -Property <prop> -Descending
Select-Object -First N <prop>, <prop>
Format-List *
Get-ChildItem env:                            # all environment variables
Get-ComputerInfo
Get-ExecutionPolicy
```

**Block 2 (Managing)**

```
whoami /groups
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember -Group <name>
New-LocalUser -Name <name> -Password <secure> ...
Add-LocalGroupMember -Group <name> -Member <user>
Remove-LocalGroupMember -Group <name> -Member <user>
Remove-LocalUser -Name <name>
New-LocalGroup -Name <name>
Get-Acl <path>
Set-Acl <path> $acl
icacls <path>
icacls <path> /inheritance:r
icacls <path> /grant <user>:M
icacls <path> /grant Administrators:F
icacls <path> /remove <user>
winget --version
winget search <term>
winget install <pkgid> --accept-source-agreements --accept-package-agreements
winget list
winget upgrade
Get-Service
Get-Service -Name <name>
Start-Service / Stop-Service / Restart-Service
Set-Service -Name <name> -StartupType Disabled
Get-CimInstance Win32_Service -Filter "Name='<name>'"
Get-ScheduledTask
Get-ScheduledTask -TaskName <name>
New-ScheduledTaskAction
New-ScheduledTaskTrigger
New-ScheduledTaskPrincipal
Register-ScheduledTask
Unregister-ScheduledTask
```

**Block 3 (Reading)**

```
Get-WinEvent -ListLog *
Get-WinEvent -LogName <name> -MaxEvents N
Get-WinEvent -FilterHashtable @{LogName='<name>'; Id=N; StartTime=<datetime>}
Get-Process
Get-Process | Where-Object ...
Get-CimInstance Win32_Process
Get-NetTCPConnection -State Listen
Get-NetTCPConnection -State Established
Resolve-DnsName <hostname>
Resolve-DnsName <hostname> -Type MX
Invoke-WebRequest -Uri <url> -UseBasicParsing
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
Add-Content -Path <path> -Value <text>
Set-Content -Path <path> -Value <text>
Get-Content <path>
$ErrorActionPreference = 'Stop'
```

---

## What's next

If you came here from the workshop, your next stop is **Chapter 1: The Windows Mental Model.** It expands the brief tour of `C:\Windows`, `C:\Program Files`, and the user profile into a working map of how Windows organizes itself, including the registry as the configuration store you saw in passing today.

If you are reading this before the workshop, see you in the room.
