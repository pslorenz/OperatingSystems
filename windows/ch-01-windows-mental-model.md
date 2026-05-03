# Chapter 1: The Windows Mental Model

**You come in with:** workshop-level recognition of `C:\Windows`, `C:\Program Files`, and `C:\Users`. You can navigate in PowerShell.
**You leave with:** a working mental model of how Windows organizes itself: where things live, what the registry actually is, how user profiles are structured, and how the Windows architecture differs from the Linux one in ways that affect daily work.

**Time:** 45 to 75 minutes including the exercises.

**Security+ alignment:** No direct exam content. Foundation skill that supports Chapter 6 (event logs as data sources, Domain 4.9), Chapter 11 (Windows artifacts and indicators of malicious activity, Domain 2.4), and Chapter 12 (hardening, Domain 4.1). The registry orientation in this chapter is prerequisite knowledge for understanding registry-based persistence, which is the largest single category of Windows persistence techniques.

---

## Why this chapter matters

Last unit's first chapter was the Linux filesystem hierarchy. Same role here. The workshop taught you to navigate; this chapter teaches you where to navigate to.

The challenge with Windows is that "where things live" is more layered than on Linux. There is the filesystem (C:\Windows, C:\Program Files, etc.), the registry (a parallel hierarchy that holds configuration), and per-user profiles that scatter data across multiple locations. A working sysadmin knows where to look for what. A working security analyst knows what changes when something interesting happens.

This chapter is recognition-level, not memorization. You are not going to memorize every registry hive or every special folder. You are going to recognize them when you see them and know roughly where to start looking.

---

## The filesystem layout

Windows uses drive letters where Linux uses a single root. `C:` is the system drive. Other drives, USB sticks, and network shares get other letters. There is no mounting things into a single tree the way Linux does (technically there is, called junction points or mount points, but it is uncommon enough that you can ignore it for daily work).

The major directories under `C:\`:

### C:\Windows: the OS itself

```
ls C:\Windows | Select-Object -First 15
```

`C:\Windows` is the OS install. Inside it, the directories that matter most:

| Directory | What lives here |
|-----------|-----------------|
| `System32` | Most system binaries and drivers. Despite the name, this is the 64-bit directory on 64-bit Windows. |
| `SysWOW64` | 32-bit binaries on 64-bit Windows. The naming is intentionally backwards for compatibility reasons. |
| `WinSxS` | The "side-by-side" component store. Holds every version of every system component for rollback purposes. Often the largest directory on a Windows install. |
| `Temp` | System-wide temporary files. |
| `System32\drivers` | Device drivers, including the system-critical ones like `tcpip.sys`. |
| `System32\config` | The on-disk files that back the registry hives. We come back to this. |
| `System32\winevt\Logs` | The on-disk files that back the Windows event log. |

A few of these are worth knowing in detail.

**System32 vs SysWOW64.** This is one of the most counterintuitive design decisions in Windows. On a 64-bit Windows install, `System32` holds 64-bit binaries and `SysWOW64` holds 32-bit binaries. The naming is preserved from the 32-bit era; renaming it would break too many old applications. For daily work: 64-bit applications go to `System32`, 32-bit to `SysWOW64`. The Windows file system redirector handles routing 32-bit applications to the right place automatically.

**WinSxS.** *WinSxS, short for Windows Side-by-Side, is the component store that holds every version of every system file installed across cumulative updates.* It is often 10 GB or more on a long-running Windows install. You do not delete files from WinSxS by hand. The cleanup command is:

```
DISM.exe /Online /Cleanup-Image /StartComponentCleanup
```

Microsoft built WinSxS to support feature rollback. It is roughly the Windows equivalent of Linux's "old kernels still installed in /boot," but more aggressive about retention.

### C:\Program Files and C:\Program Files (x86)

Installed applications. The split mirrors System32/SysWOW64: 64-bit apps go to `Program Files`, 32-bit apps go to `Program Files (x86)`.

```
ls 'C:\Program Files' | Select-Object -First 10
ls 'C:\Program Files (x86)' | Select-Object -First 10
```

A typical Windows install has both populated. Notepad++ is 64-bit on modern systems and lives in `C:\Program Files`. Older third-party software is often still 32-bit and lives in `C:\Program Files (x86)`.

The space in the path is a real annoyance for scripts. When you reference these paths in PowerShell, single-quote them or use the environment variable: `$env:ProgramFiles` and `${env:ProgramFiles(x86)}`.

### C:\ProgramData

The Windows analog of `/var` on Linux. Application data shared across all users.

```
ls C:\ProgramData -Force
```

This directory is hidden by default (note the `-Force` flag to show hidden items, the equivalent of `ls -a`). Many applications store their global state, logs, and configuration here:

- `C:\ProgramData\Microsoft\Windows\WER` is the Windows Error Reporting cache.
- `C:\ProgramData\Microsoft\Windows Defender` holds Defender's runtime state.
- `C:\ProgramData\<vendor>` is where most third-party apps put cross-user state.

When investigating an unusual application's footprint, `C:\ProgramData` is one of the first places to look.

### C:\Users: profiles

Each user has a directory under `C:\Users`. Yours is `C:\Users\student`.

```
ls C:\Users
ls $env:USERPROFILE -Force
```

Inside a user's profile, the directory layout is roughly:

| Directory | What lives here |
|-----------|-----------------|
| `Desktop`, `Documents`, `Downloads`, etc. | The known folders. User content. |
| `AppData\Local` | Per-user, machine-specific data. Does not roam. |
| `AppData\LocalLow` | Lower-integrity per-user data. Used by sandboxed processes like browsers. |
| `AppData\Roaming` | Per-user data that follows the user when their profile roams to another machine in a domain environment. |

The `AppData` distinction matters. *AppData\Local holds settings that are tied to this specific machine. AppData\Roaming holds settings that should follow the user.* In a corporate environment with roaming profiles, only Roaming follows. PowerShell's profile (`$PROFILE`) is in Documents on most systems but related per-user state lives in AppData.

A few specific paths worth knowing inside the profile:

- `AppData\Local\Temp` is the per-user temp directory. The `%TEMP%` and `%TMP%` environment variables point here.
- `AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` is the PowerShell command history file. We come back to this in Chapter 11.
- `AppData\Local\Microsoft\Windows\INetCache` is Internet Explorer's old cache. Modern browsers have their own paths under AppData.

### C:\Windows\Temp vs the other temp directories

There are at least three temp directories on Windows:

- `C:\Windows\Temp`: system-wide, used by services running as SYSTEM.
- `C:\Users\<user>\AppData\Local\Temp`: per-user.
- `%TEMP%` and `%TMP%` environment variables: usually point at the per-user temp, but can be overridden.

```
$env:TEMP
$env:TMP
[System.IO.Path]::GetTempPath()
```

Each of those returns the temp directory the current process should use. They are typically the same. When investigating, check all three locations because applications running as different identities use different temp directories.

---

## The registry: the configuration store

The registry is one of the things that genuinely makes Windows different from Linux. *The Windows registry is a hierarchical key-value database that stores configuration for the operating system, services, applications, and individual users.*

On Linux, configuration lives in text files under `/etc`, with each application owning its own files in its own format. On Windows, almost all configuration lives in the registry, organized in a tree with standard top-level branches. Both designs have tradeoffs. The registry is faster for the system to read and easier to query programmatically; text files are easier to read with grep and easier to keep in version control.

For working admins and security analysts, the registry is non-optional. Many configuration settings, persistence mechanisms, and forensic artifacts are registry-based.

### The hives

The registry is divided into "hives." *A hive is a top-level branch of the registry, each backed by one or more files on disk.* The standard top-level hives:

| Hive | Short form | What lives here |
|------|------------|-----------------|
| HKEY_LOCAL_MACHINE | HKLM | System-wide configuration. The big one. |
| HKEY_CURRENT_USER | HKCU | Configuration for the currently logged-in user. |
| HKEY_USERS | HKU | All loaded user hives. HKCU is a subset. |
| HKEY_CLASSES_ROOT | HKCR | File associations and COM class registrations. Mostly a merged view of HKLM and HKCU data. |
| HKEY_CURRENT_CONFIG | HKCC | Hardware profile. Largely vestigial; rarely used today. |

The two you care about for daily work: HKLM and HKCU. The others are either subsets or merged views of those.

### Registry structure

Each hive has a tree of keys, and each key has values. *A key is a registry equivalent of a directory; a value is the registry equivalent of a file with a name and data.* A value has a name, a data type, and the data itself.

Common value types:

| Type | Holds |
|------|-------|
| REG_SZ | A string |
| REG_EXPAND_SZ | A string with environment variable expansion (e.g., `%SystemRoot%\System32\foo.exe`) |
| REG_DWORD | A 32-bit integer |
| REG_QWORD | A 64-bit integer |
| REG_BINARY | Arbitrary binary data |
| REG_MULTI_SZ | An array of strings |

Most values you encounter day-to-day are REG_SZ, REG_EXPAND_SZ, or REG_DWORD.

### Reading the registry from PowerShell

PowerShell exposes the registry as a virtual drive, so you can use the same commands you use on the filesystem.

```
ls HKLM:\
ls HKLM:\Software | Select-Object -First 10
ls HKCU:\Software\Microsoft | Select-Object -First 10
```

The `:` after the hive name is required. PowerShell treats `HKLM:` like `C:` for navigation purposes.

To read a value, use `Get-ItemProperty`:

```
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' | Select-Object ProductName, DisplayVersion, BuildLabEx
```

That reads the registry key that holds Windows version info. The output shows the product name, version, and build label.

A few specific reads worth practicing:

```
# What programs auto-start for the current user?
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run'

# What programs auto-start for the system?
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'

# What scheduled tasks are registered?
ls 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree' -Recurse | Select-Object -First 5

# Active environment variables for the system
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment'
```

Each of those queries answers a question that comes up in real work.

### Writing the registry

Writing requires admin elevation for HKLM and most of HKCU.

```
# Set a string value
Set-ItemProperty -Path 'HKCU:\Software\Test' -Name 'MyValue' -Value 'Hello'

# Create a key first if it does not exist
New-Item -Path 'HKCU:\Software\Test' -Force
Set-ItemProperty -Path 'HKCU:\Software\Test' -Name 'MyValue' -Value 'Hello'

# Read it back
Get-ItemProperty 'HKCU:\Software\Test' | Select-Object MyValue

# Delete the value
Remove-ItemProperty -Path 'HKCU:\Software\Test' -Name 'MyValue'

# Delete the key
Remove-Item 'HKCU:\Software\Test'
```

The pattern is the same as filesystem operations: `New-Item`, `Set-ItemProperty`, `Get-ItemProperty`, `Remove-ItemProperty`, `Remove-Item`. PowerShell's design choice to expose the registry through the same provider model as the filesystem pays off here.

### regedit and reg.exe

Two GUI/CLI alternatives to PowerShell:

**regedit.exe** is the GUI registry editor. Open it from the Start menu (type `regedit`, press Enter). It shows the same hives and keys that PowerShell sees, in a tree view. Useful for browsing. Useful when a scripted approach is overkill or when you want to see the full structure.

**reg.exe** is a CLI tool that predates PowerShell. It is still useful in scripts where PowerShell is unavailable (some recovery scenarios, batch files):

```
reg query 'HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion'
reg add 'HKCU\Software\Test' /v MyValue /t REG_SZ /d "Hello"
reg delete 'HKCU\Software\Test' /v MyValue /f
```

For a working admin, PowerShell is the default. reg.exe and regedit are the fallbacks.

### A note on safety

Registry editing is a power tool. It can break the system. The rule: do not change registry values you do not understand, do not import .reg files from untrusted sources, and back up before making changes you are unsure about.

The registry has no recycle bin. Deleting the wrong key requires either restoring from backup or System Restore. Be careful in regedit; the GUI is unforgiving.

---

## How the system actually starts

A simplified view of the Windows boot process, for orientation:

1. **Firmware (UEFI on modern systems)** loads the Windows Boot Manager.
2. **Boot Manager** loads the kernel (`ntoskrnl.exe`) and the boot drivers.
3. **The kernel** loads the rest of the drivers and starts the Session Manager (`smss.exe`).
4. **Session Manager** sets up the system environment, then starts the Local Security Authority (`lsass.exe`), the Service Control Manager (`services.exe`), and the Winlogon process.
5. **Service Control Manager** starts every service marked Automatic.
6. **Winlogon** displays the login prompt.
7. **You log in.** Winlogon launches `userinit.exe`, which sets up your environment and starts `explorer.exe` (the desktop shell).

Why this matters for working admins:

- When something goes wrong at boot, the failure happens at one of these stages, and the recovery path depends on which stage.
- When investigating persistence, attackers can plant code at several of these stages: as a service (step 5), as a Winlogon hook (step 6), via Userinit values (step 7), or in user-level autoruns (after step 7). Each leaves different artifacts.
- The Service Control Manager is what `services.msc` and `Get-Service` interact with.

We come back to this in Chapter 11 with the artifact angle. For now, the takeaway is that "Windows starts up" is a multi-stage process with distinct components, each of which is a potential ground for things going wrong or being compromised.

---

## Services, scheduled tasks, and where they overlap

Linux had two main scheduling mechanisms (cron and systemd timers). Windows has two main mechanisms too (services and scheduled tasks), but they serve different roles than the Linux pair.

**Services** are long-running background processes that the Service Control Manager starts at boot or on demand. They are the Windows equivalent of long-running Linux daemons (sshd, nginx). Services are configured in the registry under `HKLM:\SYSTEM\CurrentControlSet\Services`, and the SCM tracks their state.

**Scheduled tasks** are commands that run on a schedule or in response to events. They are the Windows equivalent of cron and systemd timers. Tasks are configured in the registry (under `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache`) and as XML files in `C:\Windows\System32\Tasks`.

The rough mapping:

| Linux | Windows |
|-------|---------|
| systemd service (long-running) | Windows service |
| systemd service (oneshot) | Scheduled task |
| systemd timer | Scheduled task |
| cron | Scheduled task |
| init scripts (legacy) | Windows service (legacy) |

In practice: if it should run continuously and stay up, it is a service. If it should run on a schedule or in response to an event, it is a scheduled task. We covered both in the workshop and they get their own chapter (Chapter 5).

---

## The architecture you cannot see: kernel-mode versus user-mode

A brief mental model. Windows divides code into kernel-mode and user-mode.

**Kernel-mode** code runs at the highest privilege level, with direct access to hardware and memory. The Windows kernel, drivers, and a small set of system services run here. Bugs in kernel-mode code can crash the system (the blue screen).

**User-mode** code runs at a lower privilege level. Every application you run, every service, your shell, all of it. User-mode code talks to kernel-mode through a defined API.

This matters for security work because:

- A driver running in kernel-mode can do almost anything. A malicious driver is the most dangerous Windows compromise.
- User-mode malware is the common case; kernel-mode rootkits are rarer but more dangerous.
- Process Explorer (Sysinternals) shows you which threads are running in kernel-mode versus user-mode. We come back to this in Chapter 10.

For daily admin work, you do not interact with the distinction directly. For security work, knowing it exists explains why "this driver is signed by Microsoft" matters and why kernel debugging is its own discipline.

---

## Try this

**1. Tour the directories.**

Open PowerShell. Visit each of the directories from this chapter and look at what is there:

```
ls C:\Windows\System32 -File | Select-Object -First 10
ls 'C:\Program Files' | Select-Object -First 10
ls C:\ProgramData -Force | Select-Object -First 10
ls C:\Users
ls $env:USERPROFILE\AppData -Force
ls $env:USERPROFILE\AppData\Local -Force | Select-Object -First 10
```

For each directory, write down (mentally or in notes): what kinds of things live here, what is one thing you notice.

**2. Find what makes your user profile yours.**

Run:

```
ls $env:USERPROFILE -Force
ls $env:USERPROFILE\AppData\Roaming\Microsoft\Windows\PowerShell -Force
```

Confirm `PSReadLine\ConsoleHost_history.txt` exists. Open it (`notepad $env:USERPROFILE\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`). You should see your PowerShell command history from the workshop. This is one of the artifacts a security analyst would look at first when investigating user activity.

**3. Read four registry keys.**

Run each of these and read the output:

```
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName, DisplayVersion, InstallationType

Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run' -ErrorAction SilentlyContinue

Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -ErrorAction SilentlyContinue

Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment' |
    Select-Object Path, PATHEXT, TEMP, TMP, OS
```

For each, identify what the values tell you:

- The first identifies the Windows version.
- The second and third are the system-wide and user-specific autorun lists.
- The fourth is the system-wide environment variables, including PATH.

**4. Write and read your own registry value.**

```
New-Item -Path 'HKCU:\Software\Lab' -Force | Out-Null
Set-ItemProperty -Path 'HKCU:\Software\Lab' -Name 'TestValue' -Value 'Hello from chapter 1'
Get-ItemProperty 'HKCU:\Software\Lab' | Select-Object TestValue
```

Confirm the value comes back. Then clean up:

```
Remove-Item 'HKCU:\Software\Lab' -Recurse
```

This exercise is the round-trip: write, read, delete. It confirms you can use the registry as a tool, not just look at one.

**5. Map a problem to a directory.**

For each of these scenarios, write down which directory you would investigate first. Then check your answers below.

a. The user reports that PowerShell tab completion is acting weird.
b. A service installed by a third-party application is misbehaving.
c. The system disk is filling up and you do not know what is using the space.
d. A user just logged in and you want to see their PowerShell command history.

Answers (after you have written yours):

a. PowerShell profile: `$PROFILE` and the directory it points to (usually under `Documents`).
b. The service's executable in `C:\Program Files\<vendor>` and possibly its data in `C:\ProgramData\<vendor>`.
c. Walk the disk usage. The usual suspects are `C:\Windows\WinSxS`, `C:\Windows\Temp`, and per-user temp directories. Use `Get-ChildItem` with size and Sort-Object.
d. `$env:USERPROFILE\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`.

---

## Common stumbling blocks

> **`ls C:\Windows\System32` returns "Access denied" on some files.**
> Some files in System32 are owned by SYSTEM or TrustedInstaller and require elevation to read or modify. Run PowerShell as admin. For files owned by TrustedInstaller specifically, even Administrators must take ownership first; this is intentional and protects core OS files.

> **`HKCU:` does not exist or is empty.**
> The user hive only loads when the user is logged in. If you are running as another user (e.g., via `runas` or a scheduled task as SYSTEM), HKCU points to that other user's hive. To explicitly read a specific user's hive, use HKU\<SID>. Find the SID with `Get-LocalUser <name> | Select-Object SID`.

> **I deleted a registry key and now an application does not work.**
> There is no registry recycle bin. Either restore from a backup (System Restore creates registry checkpoints) or reinstall the application. Be more careful next time. Specifically: when in regedit or PowerShell, prefer to export a key (`reg export <key> backup.reg`) before deleting it, so you have a way back.

> **A registry value's type matters and I set the wrong one.**
> Setting a REG_DWORD value where the application expects REG_SZ (or vice versa) usually causes the value to be ignored. The fix is to delete the value and recreate it with the right type. The third argument to `Set-ItemProperty` is the type: `Set-ItemProperty -Path X -Name Y -Value Z -Type DWord`.

> **The "Program Files (x86)" path with the parenthesis breaks my script.**
> Either single-quote the path: `'C:\Program Files (x86)\App'`, or use the environment variable: `${env:ProgramFiles(x86)}\App`. The parens confuse certain string-handling contexts.

> **AppData is hidden and `ls` does not show it.**
> Use `ls -Force` (or `dir /a` in CMD) to show hidden items. Hidden is just a flag; the directory is not actually concealed.

---

## What this gets you

After this chapter:

- You can read the major Windows directories and explain what each is for.
- You know the difference between System32 and SysWOW64 and why the naming is confusing.
- You know the registry's structure: hives, keys, values, types.
- You can read and write registry values from PowerShell.
- You know where user profile data lives and which parts roam.
- You know the rough Windows boot sequence and the components involved at each stage.
- You understand the kernel-mode versus user-mode distinction at the recognition level.

The recognition this chapter builds is what lets the rest of the unit go fast. Every time we mention "the autorun key" or "the user's PSReadLine history," you know roughly where to find it. The names stop being arbitrary.

---

## What's next

Chapter 2 is PowerShell foundations. The chapter where the workshop's basic PowerShell becomes serious skill: profiles, modules, error handling, parameters, and the patterns that distinguish "I can run a command" from "I can write working PowerShell." Including how to use it to actually edit files when notepad is not enough.
