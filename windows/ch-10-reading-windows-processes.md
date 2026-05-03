# Chapter 10: Reading Windows Processes

**You come in with:** workshop-level Get-Process. You can list processes and see what is using CPU. You have read a Sysmon Event 1 once.
**You leave with:** the ability to investigate a Windows process end-to-end ("what is it running, who is its parent, is it signed, what is it talking to") with a repeatable pattern, plus working recognition of Process Explorer and signed-binary verification as practitioner tools.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 2.4 (indicators of malicious activity: process injection, unusual parent-child relationships, unsigned binaries running from unusual locations). Domain 4.4 (alerting and monitoring concepts and tools: process monitoring). Domain 4.8 (incident response: analysis activities). The signed-binary verification pattern in this chapter is the practical version of "trust but verify" applied to running code.

---

## Why this chapter matters

This is the parallel of Linux Chapter 10. Same pedagogical role: the chapter where reading the system genuinely starts to look like security work. Until now, the unit has built operational fluency. Now you turn that fluency adversarial.

The skills here are the prerequisite for Chapter 11 (Windows artifacts). When you investigate a Windows process during incident response, every question you have ("what binary, signed by whom, started by what parent, with what arguments, talking to what") has an answer in the system. Knowing how to retrieve those answers is the foundation of process-level IR on Windows.

For sysadmin work, the same skills apply to mundane debugging. "Why is this svchost.exe using 90% CPU" or "why does my service take 30 seconds to start" both reduce to questions about what the process is actually doing.

---

## Get-Process in depth

The workshop showed you the basics. There is more.

### Useful properties

```
Get-Process -Name explorer | Select-Object Name, Id, Path, Company, CPU, WorkingSet64, StartTime
```

The fields that matter:

- **Name**: the process name (without extension).
- **Id**: the PID.
- **Path**: the full path to the executable. Often null for system processes you cannot read.
- **Company**: the publisher in the binary's metadata.
- **CPU**: total CPU time consumed since the process started (cumulative seconds, not "%").
- **WorkingSet64**: physical memory in use. The number that matters for "how much memory."
- **StartTime**: when the process started. Useful for "what just launched."

`CPU` is occasionally misleading: it is cumulative time, not instantaneous percentage. A process that has been running for a week may have 600 CPU seconds even at idle. To see "what is using CPU right now," sample twice with a delay:

```powershell
$snap1 = Get-Process | Select-Object Id, CPU
Start-Sleep -Seconds 2
$snap2 = Get-Process | Select-Object Id, CPU
$snap2 | ForEach-Object {
    $old = ($snap1 | Where-Object Id -eq $_.Id).CPU
    if ($old) {
        [PSCustomObject]@{
            Id = $_.Id
            CPUDelta = [math]::Round($_.CPU - $old, 2)
        }
    }
} | Sort-Object CPUDelta -Descending | Select-Object -First 10
```

That measures the CPU difference over 2 seconds and identifies the busiest processes during that window. More accurate than the cumulative number.

### Get-Process with paths

For investigations, the binary path is often the most important field.

```
Get-Process | Where-Object Path | Select-Object Id, Name, Path | Sort-Object Path | Format-Table -AutoSize
```

Reading the output: every process where you have permission to read the path. System processes (System, Idle, Memory Compression) typically show null paths.

For one specific process:

```
Get-Process -Id 1234 | Select-Object Name, Id, Path, MainModule
$proc = Get-Process -Id 1234
$proc.MainModule.FileName
$proc.MainModule.FileVersionInfo
```

The `MainModule.FileVersionInfo` includes the file's product name, original filename, internal name, and Authenticode signing info (we cover signing below).

### Sorting and filtering

```
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 Name, Id, WorkingSet64
Get-Process | Where-Object Path -like "*Temp*"
Get-Process | Where-Object Path -like "C:\Users\*"
```

The middle one finds processes running from temp directories: a finding worth investigating. The third finds processes running from user profile directories, which is unusual for legitimate software.

### Get-Process versus Get-CimInstance Win32_Process

Get-Process is convenient but missing some fields. For deeper queries, use the CIM class:

```
Get-CimInstance Win32_Process | Select-Object ProcessId, ParentProcessId, Name, CommandLine | Format-Table -AutoSize
```

The CIM class has CommandLine (the full command-line arguments) and ParentProcessId. Get-Process does not expose either directly. For investigation work, Win32_Process is often the right starting point.

A useful "process tree by parent" query:

```powershell
Get-CimInstance Win32_Process |
    Select-Object ProcessId, ParentProcessId, Name, CommandLine |
    Sort-Object ParentProcessId, ProcessId |
    Format-Table -AutoSize
```

For a more readable tree, build it programmatically:

```powershell
function Show-ProcessTree {
    $procs = Get-CimInstance Win32_Process
    $byParent = $procs | Group-Object ParentProcessId -AsHashTable

    function Render($pid, $depth) {
        $children = $byParent[$pid]
        foreach ($c in $children) {
            $indent = "  " * $depth
            "$indent$($c.Name) [$($c.ProcessId)]"
            Render $c.ProcessId ($depth + 1)
        }
    }

    Render 0 0
}
```

Save in `$PROFILE`. Call `Show-ProcessTree` to see the parent-child relationships visually. The output is the closest PowerShell-only analog of `pstree -p` from Linux.

---

## Stopping processes

```
Stop-Process -Id 1234
Stop-Process -Name notepad
Stop-Process -Name notepad -Force
```

`-Force` skips the confirmation prompt and is required for processes you do not own. Behaves like SIGKILL: the process is terminated without a chance to clean up.

`Stop-Process` is the right tool for "this thing is hung, kill it." For graceful shutdown of a service-like application, use `Stop-Service` or send the application its proper shutdown signal.

A common mistake: killing a process by name when multiple processes share a name. `Stop-Process -Name svchost -Force` would attempt to kill every svchost.exe on the system, which would crash Windows. Always check the PID first if the name is ambiguous.

---

## Signed binaries: the trust model

Windows uses Authenticode for binary signing. *Authenticode is Microsoft's code-signing standard. A signed executable carries a cryptographic signature from the publisher, plus a chain of trust to a root certificate authority Windows trusts.*

The trust model:

- A signed binary has been hashed and the hash signed by a publisher's private key.
- The signing publisher's certificate chains to a root CA Windows trusts.
- A signature can be verified offline (no network call) using the certificate chain.
- Some signatures are countersigned with a timestamp, so they remain valid even after the signing certificate expires.

Practical implication: when you see a process running, you can verify whether the binary was actually published by the company it claims to be from.

### Verifying a signature

```
Get-AuthenticodeSignature C:\Windows\System32\notepad.exe
```

Output:

```
SignerCertificate                         Status                                 Path
-----------------                         ------                                 ----
4F36AE13266F45869B8DE..                   Valid                                  C:\Windows\System32\notepad.exe
```

The Status field is the most important. Values:

- **Valid**: signature is good, certificate chains to a trusted root, file matches the signed hash.
- **NotSigned**: no signature.
- **HashMismatch**: file content does not match the signature. The file was modified after signing.
- **NotTrusted**: signature is structurally valid but the signing certificate does not chain to a trusted root.
- **UnknownError**: the system could not determine.

For full detail:

```
Get-AuthenticodeSignature C:\Windows\System32\notepad.exe | Format-List *
```

You see the signing certificate's Subject (who signed it), the timestamp certificate (if present), and the SignatureType. For Microsoft binaries, the Subject typically includes "Microsoft Windows" or "Microsoft Corporation."

### Bulk verification

For investigation, you often want to verify many binaries at once. The pattern:

```powershell
Get-Process | Where-Object Path | ForEach-Object {
    $sig = Get-AuthenticodeSignature -FilePath $_.Path -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        ProcessName = $_.Name
        ProcessId = $_.Id
        Path = $_.Path
        Signed = $sig.Status
        Signer = $sig.SignerCertificate.Subject
    }
} | Sort-Object Signed, ProcessName | Format-Table -AutoSize
```

That walks every running process, checks its binary's signature, and reports the result. On a clean Windows 11 lab box, almost every process is `Valid` and signed by Microsoft, with a few legitimate third-party signers (NVIDIA, Intel, etc. for hardware-related processes). Unsigned or NotTrusted binaries are findings.

This single query is one of the highest-value process-level audits on Windows. Run it on every box you investigate.

### sigcheck (the better tool when you can install it)

The Sysinternals `sigcheck` utility is the gold standard for signature verification. It does more than `Get-AuthenticodeSignature`: it checks against VirusTotal, validates timestamp signatures more rigorously, and supports multiple signatures on a single file.

```
sigcheck -accepteula -nobanner C:\Windows\System32\notepad.exe
```

For this chapter, recognition is enough. The Sysinternals suite (which includes Process Explorer, sigcheck, autoruns, and more) is the de facto investigation toolkit on Windows. Intermediate cohort goes deeper.

---

## Process Explorer at recognition depth

Process Explorer is a Sysinternals tool that does what Task Manager does, plus much more. *Process Explorer is a free Microsoft-published utility that shows running processes in a tree view, with detailed information about each one's owner, signed status, network connections, open handles, and loaded modules.*

It is the de facto interactive process inspector on Windows. SOC analysts use it constantly. It is not built into Windows; you download it from the Sysinternals page on Microsoft Learn.

For this unit, we do not install it (the chapter's writing exercises use only built-in PowerShell). But you should know:

- It is freely downloadable from Microsoft.
- It shows processes as a tree, with parent-child relationships visible at a glance.
- It has columns for "Verified Signer" so you can see signed status without running cmdlets.
- It can replace Task Manager (Options > Replace Task Manager) so it pops up when you press Ctrl+Shift+Esc.
- It shows handles, modules, threads, and network connections per process.
- It has a "Find Window" tool: drag a target onto a window on the desktop, and it identifies the owning process.

The intermediate cohort installs it as part of the standard toolkit. For now, recognition of "Process Explorer is the tool you reach for when you need a real process inspector on Windows" is enough.

---

## Network state per process

You did this in Chapter 8 with the listening-ports query. The same pattern applies to investigating a specific process:

```powershell
$pid = (Get-Process -Name powershell)[0].Id
Get-NetTCPConnection -OwningProcess $pid
```

Or to find every process with active outbound connections:

```powershell
Get-NetTCPConnection -State Established |
    Group-Object OwningProcess |
    ForEach-Object {
        $proc = Get-Process -Id $_.Name -ErrorAction SilentlyContinue
        [PSCustomObject]@{
            Process = $proc.Name
            ProcessId = $_.Name
            Path = $proc.Path
            ConnectionCount = $_.Count
        }
    } | Sort-Object ConnectionCount -Descending
```

This is the network-side complement to "what is this process doing." For incident response, "process X has 50 outbound connections to a single IP" is a different signature than "process X has 1 outbound connection." The query gives you both views at once.

---

## DLLs loaded into a process

A Windows process is composed of an EXE plus many DLLs. *A DLL, dynamic link library, is a code module that can be loaded into a process at runtime. Most of Windows's user-mode functionality is in DLLs that processes load as needed.*

Investigating which DLLs are loaded into a process is sometimes useful: process injection attacks place attacker DLLs into legitimate processes, and finding them is a real IR skill.

```
Get-Process -Name explorer | Select-Object -ExpandProperty Modules | Select-Object ModuleName, FileName | Format-Table -AutoSize
```

That returns every DLL loaded into explorer.exe. A typical browser process loads hundreds of DLLs; explorer loads dozens.

For a specific question ("is this DLL loaded into this process"):

```
(Get-Process -Name explorer).Modules | Where-Object FileName -like "*custom-name*"
```

For investigation, the deeper question is "which DLLs are signed and which are not." Process Explorer shows this column directly. In PowerShell, you would walk Get-AuthenticodeSignature on every loaded module:

```powershell
$proc = Get-Process -Name explorer
$proc.Modules | ForEach-Object {
    $sig = Get-AuthenticodeSignature -FilePath $_.FileName -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Module = $_.ModuleName
        Path = $_.FileName
        Signed = $sig.Status
    }
} | Where-Object Signed -ne 'Valid'
```

That returns only the DLLs that are not validly signed. On a clean process, this is empty. On a process with injected code, you might see a finding.

This is power-tool territory for routine work but real IR practice. We come back to it in Chapter 11.

---

## A practical investigation

Walking through a question end-to-end, mirroring the Linux Chapter 10 example. The question: a process named `runner` is using 80% CPU on the box. What is it doing.

**Step 1.** Find the process.

```
Get-Process -Name runner
```

You see something like `runner` with a PID and the typical columns.

**Step 2.** Get the binary path and signature.

```powershell
$proc = Get-Process -Name runner
$proc | Select-Object Id, Name, Path, StartTime
Get-AuthenticodeSignature $proc.Path
```

If the path is `C:\Program Files\Acme\runner.exe` and the signature is Valid with a known publisher, the binary is probably legitimate. If the path is `C:\Users\student\AppData\Local\Temp\runner.exe` with an unsigned status, that is a finding.

**Step 3.** Find the parent process.

```powershell
$cimProc = Get-CimInstance Win32_Process -Filter "Name='runner.exe'"
$parent = Get-CimInstance Win32_Process -Filter "ProcessId=$($cimProc.ParentProcessId)"
$parent | Select-Object ProcessId, Name, CommandLine, ExecutablePath
```

The parent context tells you a lot. If `runner.exe` was started by `services.exe`, it is a service. If it was started by `explorer.exe`, a user clicked something. If it was started by `cmd.exe` with an unusual command line, that is the trail back to whoever launched it.

**Step 4.** Get the command line and current directory.

```powershell
$cimProc | Select-Object CommandLine, ExecutablePath
```

The command line is the most useful single field for investigation. `runner.exe` with no arguments is one thing; `runner.exe -c http://198.51.100.42` is another.

**Step 5.** Find what files and network connections it has open.

```powershell
Get-NetTCPConnection -OwningProcess $proc.Id
```

For full handle inspection, Process Explorer or the Sysinternals `handle.exe` tool is needed. PowerShell does not have a built-in cmdlet for the equivalent of Linux's `lsof`.

**Step 6.** Check for child processes.

```powershell
Get-CimInstance Win32_Process -Filter "ParentProcessId=$($proc.Id)" |
    Select-Object ProcessId, Name, CommandLine
```

If `runner` has spawned its own children (especially `cmd.exe` or `powershell.exe`), the children's command lines are often the actual interesting evidence.

This six-step pattern mirrors the Linux Chapter 10 pattern. The same investigation structure, the same logic, with Windows tools.

---

## Sysmon Event 1: the same investigation, captured

Everything you did manually in the practical investigation is captured by Sysmon Event 1, automatically, for every process that started.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1} -MaxEvents 100 |
    Where-Object { $_.Message -match 'runner.exe' }
```

The event includes the binary path, the command line, the parent process, the parent command line, the user, and the file hashes. All in one event.

This is why Sysmon is so valuable for IR: it captures the answer to "what was this process doing when it started" without you having to be there with a PowerShell prompt at the time.

The intermediate cohort builds on this with EDR (LimaCharlie) which captures the same telemetry plus continuous behavior over the process's lifetime. The detection rules they write live and die by the parent-child-command-line tuples that Sysmon Event 1 records.

For now: recognize that the manual investigation pattern from the previous section is what gets automated when an EDR or Sysmon is collecting telemetry. The skill of doing the investigation manually is what lets you read the telemetry intelligently.

---

## Looking at services and scheduled tasks as processes

A service, when running, is a process. A scheduled task, when running, is a process. The connection back to Chapters 5 and 6:

For services:

```powershell
Get-CimInstance Win32_Service -Filter "State='Running'" |
    Select-Object Name, ProcessId, PathName, StartName |
    Format-Table -AutoSize
```

The ProcessId column links each service to its running process. You can then `Get-Process -Id <id>` to see the process side.

For scheduled tasks running right now:

```powershell
Get-ScheduledTask | Get-ScheduledTaskInfo | Where-Object LastTaskResult -ne 0
```

Tasks with non-zero LastTaskResult failed at their last run. Combine with Sysmon Event 1 filtered by parent process to find the actual command that ran and failed.

The pattern: services and scheduled tasks are the configuration; processes are the runtime. A complete investigation reads both.

---

## Try this

**1. Take inventory of running processes.**

Run:

```
Get-Process | Where-Object Path | Sort-Object Path | Select-Object Name, Id, Path -First 20
```

For each unique Path, verify whether you recognize the publisher. Cross-reference with `Get-AuthenticodeSignature` for any path that looks unfamiliar.

**2. Run the bulk signature audit.**

Run the bulk verification query from this chapter. Pipe the output to `Where-Object Signed -ne 'Valid'`. On a clean Windows 11 lab box, the result should be empty or nearly empty (a few system processes whose path is not readable show as null). Anything else is a finding worth understanding.

**3. Build the process tree.**

Add the `Show-ProcessTree` function from this chapter to your `$PROFILE`. Run it. Read the output. Identify:

- The root process (typically idle/System).
- Where `services.exe` lives in the tree, and what its children are.
- Where `explorer.exe` lives, and what it has spawned.
- Your current PowerShell session: who is its parent, and what is the parent's parent?

Walk up the tree from your PowerShell to the root. Each step explains "what is responsible for me being here."

**4. Inspect a process you started.**

Open a second PowerShell window. Get its PID:

```
$pid
```

In your first PowerShell window:

```powershell
$target = $other_pid       # the PID you got from the second window
$cim = Get-CimInstance Win32_Process -Filter "ProcessId=$target"
$cim | Select-Object Name, CommandLine, ParentProcessId, CreationDate
$parent = Get-CimInstance Win32_Process -Filter "ProcessId=$($cim.ParentProcessId)"
$parent | Select-Object Name, CommandLine
```

Walk up to find what spawned the second PowerShell. The chain is typically: your second PowerShell -> explorer.exe (if you launched from Start menu) or your first PowerShell (if you ran `powershell` from inside it).

**5. Read Sysmon Event 1 for a process you just started.**

Start any process (open Notepad). Then:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1; StartTime=(Get-Date).AddMinutes(-5)} |
    Where-Object { $_.Message -match 'notepad' } |
    Select-Object -First 1 | Format-List Message
```

Read the message. Identify each field: Image, CommandLine, User, ParentImage, ParentCommandLine, Hashes. This is the telemetry an EDR captures and a SOC analyst reads. You just generated and read your own.

---

## Common stumbling blocks

> **`Get-Process` does not show paths for some processes.**
> System processes like `System`, `Idle`, `Memory Compression`, and processes running as a different user often have null paths because PowerShell cannot read them without elevation. Run from elevated PowerShell to see more.

> **`Get-AuthenticodeSignature` returns Status: NotSigned for what should be a signed binary.**
> Two possibilities. First, the binary really is unsigned (not all software is signed; older third-party utilities sometimes are not). Second, the signature is in the .cat catalog files for Windows components, not embedded in the binary; `Get-AuthenticodeSignature` may not check catalog signatures by default. Use `sigcheck.exe` (Sysinternals) for the more rigorous check.

> **The MainModule property is null on some processes.**
> Same elevation issue as null Path. PowerShell cannot read the main module without permission to read the process. Elevate.

> **My process tree function is missing some processes.**
> If a parent process has exited but its children are still running, the children's ParentProcessId points at a PID that no longer exists. They are "orphaned" and the tree-builder will not place them. To handle this, treat any unparented process as a root for tree-building purposes.

> **Stop-Process killed more than I intended.**
> `Stop-Process -Name <name>` kills every process with that name. For svchost.exe specifically, this is system-fatal. Always identify the specific PID first with `Get-Process -Id <id>` before killing.

> **`Get-CimInstance Win32_Process` is much slower than `Get-Process`.**
> CIM queries hit a different layer (WMI). For pure listing, `Get-Process` is faster. For queries that need ParentProcessId or CommandLine, you must use Win32_Process. Filter at the source: `Get-CimInstance Win32_Process -Filter "Name='powershell.exe'"` is faster than retrieving all and filtering with Where-Object.

> **`Get-NetTCPConnection -OwningProcess` returns nothing for a process I know has connections.**
> The process may be using UDP (use Get-NetUDPEndpoint) or named pipes (no PowerShell built-in for this; use Process Explorer or handle.exe). Also: if the process exited between when you got its PID and when you queried connections, the result is empty.

---

## What this gets you

After this chapter:

- You can read Get-Process output fluently and explain every column.
- You can use `Get-CimInstance Win32_Process` to get the parent and command line for any process.
- You can build a process tree showing parent-child relationships.
- You can verify signatures on running binaries with `Get-AuthenticodeSignature` and recognize the Status values.
- You can run a bulk signature audit across every running process.
- You can investigate a process end-to-end with the six-step pattern: PID, signature, parent, command line, network, children.
- You know what Process Explorer is and what it does, even though we did not install it.
- You know that DLL injection is a thing and how to inspect loaded modules.
- You know that Sysmon Event 1 is the automated capture of the manual investigation pattern.

The bulk signature audit is the part of this chapter that pays off most. Run it on every Windows box you investigate. The signature status of running processes is one of the most reliable single-pass indicators of host integrity on Windows.

---

## What's next

Chapter 11 is Windows artifacts and what attackers leave behind. The chapter that pulls together everything from the previous chapters and applies it adversarially. By the end, you can do basic IR triage on a Windows host you did not build, with concrete artifact-level checks tied to MITRE ATT&CK techniques.
