# Chapter 5: Services and Scheduled Tasks

**You come in with:** workshop-level service control (Get-Service, Start/Stop/Restart). You have created one scheduled task.
**You leave with:** the ability to read any service definition on a Windows host, configure scheduled tasks for real automation including long-running and event-driven patterns, choose deliberately between services and scheduled tasks, and recognize when an attacker has abused either as a persistence mechanism.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 4.1 (hardening: secure baselines, removal of unnecessary software, applying security techniques to servers). Domain 4.4 (alerting and monitoring of services and dependencies). Domain 4.7 (automation: enabling/disabling services and access through scheduled tasks). Domain 2.4 (indicators of malicious activity: persistence techniques including services and scheduled tasks). The service-account hygiene introduced here is the practical version of Domain 4.6 (least privilege applied to running processes).

---

## Why this chapter matters

Services and scheduled tasks are the two main ways code runs on a Windows host without a user being logged in. They are also two of the three main categories of Windows persistence (the third is registry-based autoruns, which we cover in Chapter 11). Knowing these mechanisms cold matters for both administration and security work.

For administration: when a service refuses to start, when a scheduled task does not fire, when a third-party application installs itself as a service running as SYSTEM, the diagnostic skill is "read the configuration, understand the components, identify the failing piece." That skill applies once and pays off forever.

For security: services and scheduled tasks are the most common persistence mechanisms on Windows. An attacker who compromises a box and wants to survive a reboot will probably install a service or schedule a task. Reading these mechanisms adversarially is the foundation of Chapter 11.

---

## Services in depth

A Windows service is a program managed by the Service Control Manager (SCM). *The Service Control Manager is a Windows component that starts services at boot, monitors them while they run, and provides the API that PowerShell, sc.exe, and services.msc use to interact with them.*

Services are configured in the registry under `HKLM:\SYSTEM\CurrentControlSet\Services`. Each service is a key with values describing what it is and how to run it. The PowerShell, sc.exe, and GUI tools are friendly views of those registry entries.

### What a service definition looks like

```
ls 'HKLM:\SYSTEM\CurrentControlSet\Services' | Select-Object -First 10
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\W32Time' | Select-Object Description, ImagePath, Start, Type, ObjectName, ServiceDll
```

Reading the values:

- **Description**: human-readable description.
- **ImagePath**: the executable that runs the service. For most services this is `svchost.exe -k <group>`, where svchost is the shared service host.
- **Start**: 0 = boot, 1 = system, 2 = automatic, 3 = manual, 4 = disabled.
- **Type**: 16 = own process, 32 = shared process, others.
- **ObjectName**: the account the service runs as. `LocalSystem` (= SYSTEM), `NT AUTHORITY\NetworkService`, `NT AUTHORITY\LocalService`, or a domain or local account.
- **ServiceDll** (in `\Parameters` subkey): for services hosted in svchost, the DLL that implements the service.

For day-to-day work, you do not edit this registry directly. The PowerShell cmdlets and sc.exe handle it. But when you investigate a service, knowing where the configuration lives lets you cross-check.

### Reading service configuration

The PowerShell `Get-Service` returns basic state. For full configuration, use `Get-CimInstance`:

```
Get-CimInstance Win32_Service -Filter "Name='W32Time'" |
    Select-Object Name, DisplayName, State, StartMode, StartName, PathName, ProcessId
```

The fields:

- **Name**: the short name (W32Time).
- **DisplayName**: the friendly name (Windows Time).
- **State**: Running, Stopped, etc.
- **StartMode**: Auto, Manual, Disabled.
- **StartName**: the account it runs as. `LocalSystem` is the most common; others are listed below.
- **PathName**: the executable. For shared services, includes the svchost group.
- **ProcessId**: the PID of the running process. 0 if not running.

A useful query: every service running with non-default user accounts:

```
Get-CimInstance Win32_Service |
    Where-Object { $_.StartName -notin 'LocalSystem','NT AUTHORITY\NetworkService','NT AUTHORITY\LocalService','NT AUTHORITY\SYSTEM' } |
    Select-Object Name, StartName, PathName
```

That returns services whose start account is not one of the standard system accounts. Anything in the result is a service running as a specific user, which is often a custom service (sometimes legitimate, sometimes a finding).

### Service accounts

The standard Windows service accounts:

| Account | What it has |
|---------|-------------|
| **LocalSystem** | Highest local privilege. Roughly equivalent to root on Linux. Most services run here. |
| **LocalService** | Limited privileges. Cannot access network resources with credentials. |
| **NetworkService** | Limited privileges. Can access network resources as the computer account. |
| **<custom>** | A specific user account, configured by an administrator. |

The hardening principle: services should run with the minimum privilege they need. LocalSystem is overkill for most services; LocalService or NetworkService is enough for a service that does not need administrative access. A real-world admin reviewing services for hardening looks at every service running as LocalSystem and asks "does this one really need it."

### Recovery options

Services have a recovery configuration: what should happen if the service crashes. The defaults are typically "do nothing" or "restart once."

```
sc.exe qfailure W32Time
```

Output:

```
[SC] QueryServiceConfig2 SUCCESS

SERVICE_NAME: W32Time
RESET_PERIOD (in seconds)    : 0
REBOOT_MESSAGE               :
COMMAND_LINE                 :
FAILURE_ACTIONS              : RESTART -- Delay = 60000 milliseconds.
```

To configure recovery via PowerShell, the simplest path is sc.exe (PowerShell does not have a clean cmdlet for this):

```
sc.exe failure ServiceName reset= 86400 actions= restart/60000/restart/60000/restart/60000
```

Reading this: reset the failure counter after 86400 seconds (24 hours). On first failure, restart after 60 seconds. Same for second and third. This is a reasonable default for a service that should auto-recover.

Be aware: `sc.exe failure` requires literal spaces after the equals signs (`reset= 86400`, not `reset=86400`). This is sc.exe's quirk.

---

## Why we are not installing a custom service today

The next obvious question, given everything you have read about services so far: how do you install one of your own?

The honest answer for a beginner Windows admin: you probably do not. Or rather, you do not in the way most people first reach for. Walking through the reasoning is genuinely useful.

### PowerShell is not a service

A Windows service must respond to service control messages from the Service Control Manager: start, stop, pause, continue. The SCM speaks a specific protocol over a named pipe. An executable that does not speak this protocol cannot be a service. Bare `powershell.exe -File script.ps1` does not speak it; the SCM will start the process but the process will not respond, and within a few seconds the SCM will mark the service as "failed to start" and kill it.

This is why "make my PowerShell script a service" is not a one-command operation. It requires either:

1. **A real compiled application** that implements the service protocol (using .NET's ServiceBase class, or equivalent in Go, Rust, or another language).
2. **A wrapper executable** that translates SCM messages into something a script can respond to.

### Wrapper utilities exist; they are not where you start

Several third-party utilities exist to wrap arbitrary executables as services. The most common is NSSM (the Non-Sucking Service Manager). It is widely used in production, has been around for years, and is the de facto answer to "how do I make this thing a service."

This unit does not teach NSSM. Reasons:

- **Adding third-party software to a clean Windows box runs against the discipline this unit teaches.** Chapter 4 introduced "remove unnecessary software" as a CIS hardening principle. Chapter 12 will harden the box against unsigned executables. Installing a service-wrapper to make a script run as a service contradicts that posture.
- **The pattern almost always indicates the wrong tool was chosen.** A script you want to run continuously is almost always better as a scheduled task. A script you want to run on a schedule is always better as a scheduled task. The "wrap PowerShell as a service" pattern is what beginners reach for when they have not yet learned that scheduled tasks cover most of the territory.
- **Real long-running services are real applications.** If you find yourself wanting a continuous PowerShell process, that is a signal you should be writing a real application in a compiled language. .NET, Go, Rust, or even Python with a service wrapper. The PowerShell-wrapped-as-service pattern survives in production but rarely deserves to.

### The honest decision tree

When you find yourself wanting to "install a service" for a script, walk this tree first:

1. **Does the work need to run on a schedule?** Use a scheduled task with `New-ScheduledTaskTrigger -Daily` or similar.
2. **Does the work need to run continuously, starting at boot?** Use a scheduled task with `New-ScheduledTaskTrigger -AtStartup` and a script that loops.
3. **Does the work need to respond to specific events?** Use a scheduled task with an event-based trigger. We cover these in the next section.
4. **Do you genuinely need a real Windows service?** Then write a real application, in a compiled language, that implements the service protocol natively. This is intermediate cohort or beyond.

For 95% of "I want this to be a service" cases on a working admin's plate, the answer is one of the first three: a scheduled task. We focus the rest of this chapter on doing scheduled tasks well, including the patterns that look service-like.

### Removing a custom service

Even though you are not installing custom services, you may need to remove one someone else installed. The pattern:

```
Get-Service -Name <name>
Stop-Service -Name <name>
sc.exe delete <name>
```

`sc.exe delete` removes the registry entry that defines the service. The next reboot fully removes it from the system. This is the complement to "install a service": it is the part you might genuinely need.

---

## Scheduled Tasks in depth

The workshop introduced scheduled tasks. Now we go deeper.

### The components of a task

A scheduled task has four required components:

- **Action**: what to run. The executable, arguments, and working directory.
- **Trigger**: when to run. A schedule (daily, hourly), an event (at logon, at startup), or in response to an event log.
- **Principal**: as whom to run. The user account, the run level, the logon type.
- **Settings**: how to behave. Stop if running too long, run only if on AC power, retry on failure, etc.

Each is a separate object in PowerShell, assembled with `Register-ScheduledTask`.

### Triggers in depth

The full trigger types:

```
New-ScheduledTaskTrigger -Once -At <DateTime>
New-ScheduledTaskTrigger -Daily -At <DateTime>
New-ScheduledTaskTrigger -Weekly -At <DateTime> -DaysOfWeek Monday, Wednesday
New-ScheduledTaskTrigger -AtLogOn
New-ScheduledTaskTrigger -AtLogOn -User student
New-ScheduledTaskTrigger -AtStartup
```

Plus repetition modifiers:

```
New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 5)
```

For event-based triggers (run when a specific event log entry appears), the syntax is more verbose and uses CIM directly:

```powershell
$EventTrigger = New-CimInstance -ClassName MSFT_TaskEventTrigger -Namespace Root\Microsoft\Windows\TaskScheduler -ClientOnly -Property @{
    Subscription = '<QueryList><Query Id="0" Path="Security"><Select Path="Security">*[System[(EventID=4625)]]</Select></Query></QueryList>'
}
```

That trigger fires whenever event 4625 (failed logon) is recorded. Useful for "alert me when my account fails to log in." Event-driven scheduled tasks are a real automation pattern; the syntax is verbose but the capability is powerful.

### Principals in depth

```
New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
New-ScheduledTaskPrincipal -UserId "student" -LogonType Interactive -RunLevel Limited
New-ScheduledTaskPrincipal -UserId "student" -LogonType S4U -RunLevel Limited
```

The interesting parameters:

- **UserId**: the account. Built-in or specific.
- **LogonType**: how the task authenticates.
  - `ServiceAccount`: for system accounts (SYSTEM, LocalService, NetworkService).
  - `Password`: requires a stored password. Risky if the password changes.
  - `Interactive`: runs only when the user is logged in.
  - `S4U`: "Service for User," runs without a stored password but cannot access network resources.
- **RunLevel**: `Limited` (UAC-restricted) or `Highest` (elevated, full admin token).

For most automation that needs to run regardless of user state, the combination `SYSTEM` + `ServiceAccount` + `Highest` is the right answer.

### Settings worth knowing

```
$Settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -DontStopOnIdleEnd `
    -ExecutionTimeLimit (New-TimeSpan -Hours 1) `
    -MultipleInstances IgnoreNew `
    -RestartCount 3 `
    -RestartInterval (New-TimeSpan -Minutes 5)
```

Reading these:

- **StartWhenAvailable**: if the trigger fires while the box is off, run the task when the box comes back. Equivalent to `Persistent=true` in systemd timers.
- **DontStopOnIdleEnd**: keep running even if the system becomes idle.
- **ExecutionTimeLimit**: kill the task if it runs longer than this.
- **MultipleInstances IgnoreNew**: if a new trigger fires while the task is already running, ignore the new one (do not start a second instance).
- **RestartCount**: if the task fails, retry this many times.
- **RestartInterval**: how long to wait between retries.

Most of these are good defaults for production-grade tasks. The default settings (just `Register-ScheduledTask` with no `-Settings`) are fine for casual use but lacking for automation that needs to survive failure.

### A complete scheduled task

Putting it all together:

```powershell
$Action = New-ScheduledTaskAction `
    -Execute "$env:SystemRoot\System32\WindowsPowerShell\v1.0\powershell.exe" `
    -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\AppData\webops\check.ps1"

$Trigger = New-ScheduledTaskTrigger -Daily -At "03:00"

$Principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

$Settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 30) `
    -MultipleInstances IgnoreNew `
    -RestartCount 2 `
    -RestartInterval (New-TimeSpan -Minutes 5)

Register-ScheduledTask -TaskName "DailyCheck" `
    -Action $Action `
    -Trigger $Trigger `
    -Principal $Principal `
    -Settings $Settings `
    -Description "Daily system check"
```

Confirming:

```
Get-ScheduledTask -TaskName DailyCheck | Format-List *
Get-ScheduledTaskInfo -TaskName DailyCheck
```

`Get-ScheduledTaskInfo` shows the runtime status: when it last ran, what the last result was, when it next runs.

### The service-like pattern: a long-running task at boot

Earlier in this chapter we said that "I want my script to run continuously starting at boot" is one of the cases people incorrectly reach for a service. Here is how you do it correctly with a scheduled task.

The pattern: a script that loops forever, triggered by `AtStartup`, running as SYSTEM, with `ExecutionTimeLimit` set to "no limit."

First, the script. Save as `C:\AppData\webops\heartbeat-loop.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
$LogPath = 'C:\AppData\webops\heartbeat-loop.log'

while ($true) {
    try {
        Add-Content -Path $LogPath -Value "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') heartbeat from PID $PID"
    } catch {
        # If the log write fails, keep trying. A real service should not die on a transient write error.
    }
    Start-Sleep -Seconds 30
}
```

Now the scheduled task:

```powershell
$Action = New-ScheduledTaskAction `
    -Execute "$env:SystemRoot\System32\WindowsPowerShell\v1.0\powershell.exe" `
    -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\AppData\webops\heartbeat-loop.ps1"

$Trigger = New-ScheduledTaskTrigger -AtStartup

$Principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

$Settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -ExecutionTimeLimit ([TimeSpan]::Zero) `
    -MultipleInstances IgnoreNew `
    -RestartCount 3 `
    -RestartInterval (New-TimeSpan -Minutes 1) `
    -DisallowHardTerminate:$false

Register-ScheduledTask -TaskName "HeartbeatLoop" `
    -Action $Action `
    -Trigger $Trigger `
    -Principal $Principal `
    -Settings $Settings `
    -Description "Continuous heartbeat (service-like pattern)"
```

Reading the differences from a normal scheduled task:

- **`-AtStartup` trigger**: fires once when the system boots.
- **`ExecutionTimeLimit ([TimeSpan]::Zero)`**: no time limit. Without this, scheduled tasks default to a 72-hour cap and would be killed.
- **`MultipleInstances IgnoreNew`**: if the task is somehow triggered while already running (it should not be, with only AtStartup), do not start a second copy.
- **`RestartCount 3` and `RestartInterval`**: if the script exits unexpectedly, restart it up to 3 times.

To start it without rebooting:

```
Start-ScheduledTask -TaskName HeartbeatLoop
```

Wait a minute, then check:

```
Get-Content C:\AppData\webops\heartbeat-loop.log -Tail 5
Get-ScheduledTaskInfo -TaskName HeartbeatLoop
```

You should see heartbeat entries every 30 seconds and the task showing as Running.

This is the correct pattern for "I want a continuous PowerShell process that starts at boot." It uses only built-in Windows tooling, requires no third-party software, and gives you the same operational result that wrapping PowerShell as a service would, with cleaner mechanics.

To clean up:

```
Stop-ScheduledTask -TaskName HeartbeatLoop
Unregister-ScheduledTask -TaskName HeartbeatLoop -Confirm:$false
```

### Reading existing scheduled tasks

```
Get-ScheduledTask | Where-Object State -eq Ready | Select-Object TaskPath, TaskName, State -First 10
```

The TaskPath is the directory in the Task Scheduler hierarchy. `\` is the root. `\Microsoft\Windows\` holds the OS-installed tasks. Custom tasks usually go to `\` unless explicitly placed elsewhere.

To read a task's full configuration:

```
Get-ScheduledTask -TaskName "Heartbeat" | Format-List *
```

The output is verbose. Worth focusing on:

- **Actions**: the underlying executable and arguments.
- **Triggers**: the schedule.
- **Principal**: the account.
- **Author**: who created it. Often "MicrosoftWindowsCorporation" for OS tasks, or the user who registered it for custom ones.

When investigating, the **Actions** field is the highest-value field. A task whose action is "powershell.exe -EncodedCommand <base64>" or "cmd.exe /c curl http://... | iex" is suspicious. We come back to this in Chapter 11.

### Running a task on demand

```
Start-ScheduledTask -TaskName DailyCheck
```

Useful for testing a task without waiting for the trigger.

### Disabling and removing

```
Disable-ScheduledTask -TaskName DailyCheck
Enable-ScheduledTask -TaskName DailyCheck
Unregister-ScheduledTask -TaskName DailyCheck -Confirm:$false
```

Disable keeps the task definition but prevents the trigger from firing. Unregister removes it entirely.

---

## Reading services and tasks adversarially

The workshop and the chapters so far have built the operational skill. The security framing is: when investigating, every service and every scheduled task is a question.

### Questions to ask of every service

For each service on the box (or each one you do not recognize):

1. **Is it from a known publisher?** Check the executable path. Microsoft, NVIDIA, Intel, named vendor software in `C:\Program Files\<vendor>` is usually expected. Executables in `C:\Users\<name>` or `C:\Temp` or `C:\ProgramData` warrant questions.

2. **What account does it run as?** LocalSystem is fine for most things. A specific user account is unusual and worth understanding why.

3. **Was it installed recently?** Check the install date if you can determine it. Recently-installed services that you did not install are findings.

### Questions to ask of every scheduled task

For each task in the root or non-Microsoft path:

1. **What does the action run?** Read the Execute and Argument fields. Is the path normal? Does the argument string look right?

2. **What triggers it?** A task triggered at logon, every minute, or in response to an event is more aggressive than one that runs nightly. Aggressive triggers paired with suspicious actions are findings.

3. **What account does it run as?** SYSTEM at high run level is the strongest combination; tasks with this combination warrant the most scrutiny.

### Useful queries for triage

```
# Services not running as one of the standard accounts
Get-CimInstance Win32_Service |
    Where-Object { $_.StartName -notin 'LocalSystem','NT AUTHORITY\NetworkService','NT AUTHORITY\LocalService' } |
    Select-Object Name, StartName, PathName

# Services with executable paths outside Program Files or Windows
Get-CimInstance Win32_Service |
    Where-Object { $_.PathName -notmatch '^"?[Cc]:\\(Windows|Program Files)' } |
    Select-Object Name, StartName, PathName

# Scheduled tasks not in the Microsoft path, running as SYSTEM
Get-ScheduledTask | Where-Object { $_.TaskPath -notlike "\Microsoft*" -and $_.Principal.UserId -eq "SYSTEM" } |
    Select-Object TaskPath, TaskName, @{N='Action';E={$_.Actions.Execute + ' ' + $_.Actions.Arguments}}

# Recently-modified scheduled tasks
Get-ChildItem 'C:\Windows\System32\Tasks' -Recurse -File |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 FullName, LastWriteTime
```

These queries are the start of an audit. Anything they return goes into your "investigate further" list. Most results on a clean system will be expected; on a less-clean system, you find things.

---

## Try this

**1. Inventory services running as non-default accounts.**

Run the query for services running as something other than LocalSystem/LocalService/NetworkService. On a clean Windows 11 lab box, you might see one or two (typically related to Windows Defender or the Edge updater, both legitimate). For each one, identify the publisher and the purpose.

**2. Build the long-running scheduled task pattern.**

Follow the "service-like pattern" procedure earlier in the chapter. Create the heartbeat-loop script and the AtStartup scheduled task. Use `Start-ScheduledTask` to start it without rebooting. Confirm:

- The task shows State=Running (`Get-ScheduledTask -TaskName HeartbeatLoop`).
- The log file is being appended every 30 seconds (`Get-Content -Tail 5 -Wait C:\AppData\webops\heartbeat-loop.log`).
- The task is configured correctly (`Get-ScheduledTask -TaskName HeartbeatLoop | Format-List Settings`). Verify `ExecutionTimeLimit` is `PT0S` (zero), confirming it has no time limit.

Then clean up: `Stop-ScheduledTask` and `Unregister-ScheduledTask`.

This exercise is the practical replacement for "install a service for my script." Same outcome, built-in tooling, no third-party software added to the box.

**3. Build a periodic scheduled task with proper settings.**

Build a scheduled task that runs the `check.ps1` script you wrote in the workshop, every 15 minutes, with these properties:

- Runs as SYSTEM at Highest run level.
- Has StartWhenAvailable enabled.
- Has an ExecutionTimeLimit of 5 minutes (so it cannot run forever).
- Has RestartCount 2 with a 1-minute interval.

Confirm with `Get-ScheduledTaskInfo`.

**4. Read three OS scheduled tasks.**

Pick three tasks from `\Microsoft\Windows\` and read their full configuration. Examples: `Microsoft\Windows\Defrag\ScheduledDefrag`, `Microsoft\Windows\UpdateOrchestrator\Schedule Scan`, `Microsoft\Windows\BitLocker\BitLocker Encrypt All Drives`.

For each, identify: what does it run, when, as whom, and why. Some of these reveal interesting facts about how Windows operates internally.

**5. Inspect a service's underlying registry.**

Pick any service. Look at its registry entry directly:

```
$svc = "Spooler"  # or any other
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\$svc" | Format-List
```

Identify the ImagePath, Start, Type, ObjectName. Compare to what `Get-CimInstance Win32_Service` shows. Confirm both views agree.

---

## Common stumbling blocks

> **`Register-ScheduledTask` with SYSTEM as principal fails with "Access denied."**
> You need to run PowerShell as administrator. SYSTEM is a privileged identity; only admins can register tasks that run as it.

> **My scheduled task is registered but never fires.**
> Common causes: the task is disabled (`Disable-ScheduledTask`), the trigger is in the past with no repeat, the action's executable path is wrong, or the user account specified does not have "Log on as a batch job" rights. Run `Get-ScheduledTaskInfo -TaskName <name>` to see the LastTaskResult; the result code tells you what went wrong.

> **A service I installed will not start.**
> Read the System event log: `Get-WinEvent -LogName System -MaxEvents 20 | Where-Object Id -in 7000,7001,7009,7011`. These IDs are service-related errors. The Message field includes the service name and the error that caused the start failure.

> **`sc.exe failure` rejects my command line.**
> The literal-spaces-after-equals quirk. The correct syntax is `sc.exe failure ServiceName reset= 86400 actions= restart/60000` with a space after each equals sign. This is sc.exe's specific argument parsing.

> **`Get-ScheduledTask` returns "Catastrophic failure" or similar generic error.**
> The Task Scheduler service may be hung. Restart it: `Restart-Service Schedule`. This is rare but does happen, especially on systems that have been up for a long time without reboot.

> **My task's action runs but the script does not produce expected output.**
> Common cause: the working directory is not set, and the script uses relative paths. Set the action's working directory: `New-ScheduledTaskAction ... -WorkingDirectory C:\AppData\webops`. Or use absolute paths inside your script.

---

## What this gets you

After this chapter:

- You can read any service definition on a Windows host and explain it.
- You know why "wrap a script as a service" is rarely the right answer and what to do instead.
- You can build a scheduled task with all four components (action, trigger, principal, settings) using full options.
- You can build the long-running, service-like scheduled task pattern (AtStartup trigger plus indefinite ExecutionTimeLimit).
- You can read existing scheduled tasks adversarially and identify suspicious patterns.
- You know the rough mapping of Linux scheduling concepts to Windows ones, including when each platform's tools overlap and when they do not.
- You can audit services and scheduled tasks for unusual accounts, paths, and triggers.

The "audit services and tasks" pattern is the bridge to Chapter 11. By the end of this unit, every service and scheduled task on the box is a question you can answer or escalate.

---

## What's next

Chapter 6 is Windows Event Logs and audit policy. The chapter where Get-WinEvent becomes serious skill, where the Security log event IDs from the workshop get filled in with context, and where Sysmon's pre-loaded telemetry gets read fluently. About 70-100 minutes; budget two sittings if needed.
