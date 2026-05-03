# Chapter 4: Software Installation and Updates

**You come in with:** workshop-level winget. You can install, upgrade, and list packages.
**You leave with:** the ability to inventory installed software on a Windows host, install software from any of the standard sources (winget, MSI, EXE, Microsoft Store), choose between them deliberately, and configure Windows Update so it does what you want rather than what it wants.

**Time:** 45 to 75 minutes including the exercises.

**Security+ alignment:** Domain 2.5 (mitigation techniques: patching). Domain 4.1 (hardening: removal of unnecessary software, patching). Domain 4.3 (vulnerability response and remediation: patching, validation of remediation). Domain 5.1 (procedures: change management). Patch management is a recurring theme on Security+ and this chapter is the practical version on Windows.

---

## Why this chapter matters

Patching is the single most effective security control most organizations implement, and Windows patching is operationally harder than Linux patching for several reasons. Windows mixes OS updates with application updates with driver updates, each on their own schedule. The default behavior tries to do the right thing automatically and sometimes does the wrong thing at the wrong time. Working admins need to understand what is happening and how to control it.

The other reason this chapter exists: a real working admin will need to install software that is not in winget. Some of it comes from Microsoft (older Visual C++ runtimes, optional features), some from vendor sites (specialized tools), some from internal sources (line-of-business apps). Knowing the sane patterns for each saves you from reinventing the wheel under time pressure.

---

## What gets installed and where

When you install software on Windows, the installer puts files in several places. A typical installation touches:

- **`C:\Program Files`** or **`C:\Program Files (x86)`**: the application binaries.
- **`C:\ProgramData\<vendor>`**: shared application state.
- **`%USERPROFILE%\AppData\<Local|Roaming>\<vendor>`**: per-user state, created on first run.
- **The registry**: hundreds of keys, including the uninstall entry that lets `winget list` and Add/Remove Programs find the application.
- **Sometimes services**: the application may register one or more Windows services that start at boot.
- **Sometimes scheduled tasks**: the application may register update checkers or background tasks.
- **Sometimes drivers**: low-level applications (security tools, virtualization, hardware utilities) install kernel-mode drivers.

The takeaway: "installing an application" is more invasive than "extracting a folder of files." A working admin should be able to inventory each of these locations to understand what an application has touched.

---

## Inventorying installed software

There are at least three ways to list installed software, and they return different sets.

### winget list

```
winget list
winget list --source winget
winget list --upgrade-available
```

`winget list` shows everything winget knows about, including software it did not install. winget reads the same registry uninstall entries that Add/Remove Programs reads, so it sees most installed software. The `--upgrade-available` filter is the start of any "what needs patching" conversation.

### Get-Package

```
Get-Package | Select-Object Name, Version, ProviderName | Sort-Object Name
```

`Get-Package` is the PowerShellGet cmdlet that abstracts package providers. Out of the box it returns MSI, Programs (registry-based detection), and PowerShellGet packages. Different output than winget, often complementary.

### The registry directly

```
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' |
    Where-Object DisplayName |
    Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
    Sort-Object DisplayName

Get-ItemProperty 'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
    Where-Object DisplayName |
    Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
    Sort-Object DisplayName

Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
    Where-Object DisplayName |
    Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
    Sort-Object DisplayName
```

Three queries, three locations:

- `HKLM:\Software\...\Uninstall` is the 64-bit installed-for-all-users list.
- `HKLM:\Software\WOW6432Node\...\Uninstall` is the 32-bit list (note the `WOW6432Node` redirect).
- `HKCU:\Software\...\Uninstall` is the per-user install list (some apps install per-user).

The registry is the authoritative source. winget and Get-Package read from these locations and present them in friendlier form. When triaging, knowing where the data actually lives lets you cross-check.

### A unified one-liner

```powershell
$paths = @(
    'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*'
)
Get-ItemProperty $paths -ErrorAction SilentlyContinue |
    Where-Object DisplayName |
    Select-Object DisplayName, DisplayVersion, Publisher, InstallDate, InstallLocation |
    Sort-Object DisplayName
```

That is the practitioner one-liner. It walks all three locations, filters to entries with names, and shows the useful columns. Save it as a function in your `$PROFILE` and you have a real software inventory at hand.

---

## winget in depth

The workshop covered the basics. A few things worth knowing now.

### Package IDs and sources

```
winget search powershell
```

Output (abridged):

```
Name                            Id                                Version    Source
PowerShell                      Microsoft.PowerShell              7.4.6.0    winget
PowerShell Preview              Microsoft.PowerShell.Preview      7.5.0.5    winget
```

The **Id** column is the canonical identifier for installation. Use IDs in scripts; names sometimes match multiple packages. The **Source** column is the source repository. By default, winget has two sources:

- `winget`: the community-curated repository.
- `msstore`: the Microsoft Store.

```
winget source list
```

Most installations should come from `winget`. Microsoft Store packages have stricter sandboxing, automatic updates, and (for some applications) different feature sets.

### Install with explicit ID and silent

```
winget install --id Microsoft.PowerShell --silent --accept-package-agreements --accept-source-agreements
```

The flags worth knowing:

- `--id <id>`: install by exact ID, avoiding name ambiguity.
- `--silent`: no UI prompts (still uses the package's own installer; some installers respect this, some do not).
- `--accept-package-agreements`: accept license agreements.
- `--accept-source-agreements`: accept source repository agreements.
- `--scope user` or `--scope machine`: install for current user only or all users.
- `--exact`: require exact name/id match, no fuzzy matching.

For scripted automation, all four `--accept` and `--silent` flags are essential. Without them, winget pauses for prompts.

### Upgrading

```
winget upgrade                        # list available upgrades
winget upgrade --all                  # upgrade everything
winget upgrade --all --silent --accept-package-agreements --accept-source-agreements
```

The `--all` form is the equivalent of `apt upgrade`. For controlled environments, you would script this with logging:

```powershell
$Log = "C:\Logs\winget-upgrade-$(Get-Date -Format yyyy-MM-dd).log"
winget upgrade --all --silent --accept-package-agreements --accept-source-agreements 2>&1 |
    Tee-Object -FilePath $Log
```

### Pinning a version

Sometimes you do not want a specific package upgraded automatically. winget supports pinning:

```
winget pin add --id Notepad++.Notepad++ --version 8.6.*
winget pin list
winget pin remove --id Notepad++.Notepad++
```

`8.6.*` keeps the package on the 8.6.x branch but allows updates within it. The version string supports wildcards and ranges.

This is the equivalent of `apt-mark hold` from the Linux unit. Same discipline applies: every pin should have a documented reason and an expiration plan.

### Importing a list of packages

```
winget export --output packages.json
winget import --import-file packages.json --accept-source-agreements --accept-package-agreements
```

`export` produces a JSON file describing every winget-installed package. `import` installs from such a file. This is the rough equivalent of an Ansible playbook for a Windows endpoint and is genuinely useful for reproducible setup.

---

## MSI and EXE installers

Not everything is in winget. When you have an .msi or .exe file from a vendor, here is how to install it sanely.

### MSI files

An MSI file is a Windows Installer package. *MSI is Microsoft's standardized installer format. MSI installs are tracked centrally, can be repaired, and can be uninstalled cleanly.*

To install:

```
msiexec /i C:\path\to\package.msi /quiet /qn /norestart /l*v C:\Logs\install.log
```

Reading the flags:

- `/i` is install. (`/x` is uninstall, `/u` is also uninstall.)
- `/quiet` suppresses UI.
- `/qn` is "no UI at all." Combine with `/quiet` for fully unattended.
- `/norestart` prevents automatic reboot.
- `/l*v <path>` writes a verbose log to the path. Essential for debugging failed installs.

After the install, the MSI is registered in the registry under the Uninstall key. winget list, Get-Package, and Add/Remove Programs can all see it.

### EXE installers

EXE installers are arbitrary executables. Each one is a snowflake. The command-line flags depend entirely on which installer toolkit the vendor used.

The most common silent install flags:

- `/S` (capital S): silent install. NSIS-built installers.
- `/silent` or `/SILENT`: silent install. Inno Setup, others.
- `/quiet`: silent install. Some MSI-wrapped EXEs.
- `/passive`: minimal UI but progress shown. Some MSI-wrapped EXEs.

Look up the specific installer's documentation. Vendors usually publish silent-install flags in their deployment guides.

```
.\setup.exe /S
.\Setup.exe /silent /norestart
```

EXE installers are riskier than MSI because they can do anything. They are not centrally tracked the way MSI is, the install does not always create a clean uninstall entry, and the silent flags may not work as documented. Prefer MSI when given a choice.

### When to use msiexec versus winget

If the package is in winget, use winget. Always.

If you have a vendor .msi file and the package is not in winget, use msiexec.

If you have a vendor .exe and the package is not in winget, look up the silent install flags for that specific installer and use them. If you cannot find them, run the installer interactively and document what you did.

---

## The Microsoft Store

Microsoft Store apps are sandboxed, signed, and update automatically. Some applications you might think of as "Windows apps" (Windows Terminal, Notepad on Windows 11, the new Snipping Tool) are actually Store apps.

```
Get-AppxPackage | Select-Object Name, Version, Publisher | Sort-Object Name | Select-Object -First 20
```

`Get-AppxPackage` lists every installed Store app for the current user. To see all users' Store apps:

```
Get-AppxPackage -AllUsers | Select-Object Name, PackageUserInformation | Select-Object -First 10
```

To install a Store app from the command line:

```
winget install --id 9NVDKW0PM02D --source msstore --accept-package-agreements --accept-source-agreements
```

The `9N...` prefix is a Store app ID. winget can install from msstore source.

To uninstall a Store app:

```
Get-AppxPackage -Name "*Solitaire*" | Remove-AppxPackage
```

This is also the technique for removing Microsoft's bundled apps that you may not want (Solitaire Collection, Xbox Game Bar, etc.). Treat it as the modern equivalent of "remove unnecessary software," which is a CIS hardening item we cover in Chapter 12.

---

## Windows Update: how it works and how to control it

Windows Update is the service that delivers OS patches, driver updates, and feature updates to Windows endpoints.

The components:

- **`wuauserv`**: the Windows Update service.
- **`UsoSvc`**: the Update Orchestrator service that coordinates updates.
- **`%SystemRoot%\SoftwareDistribution`**: the directory where Windows Update caches downloads.

To check Windows Update state:

```
Get-Service -Name wuauserv, UsoSvc | Format-Table Name, Status, StartType
```

Both should be Running (or AutomaticDelayedStart) on a healthy Windows 11 install.

### Triggering an update check

Windows does not provide a clean PowerShell cmdlet for this in the box. The standard tooling is the `PSWindowsUpdate` module from the gallery (which we touched on in Chapter 2):

```
Install-Module PSWindowsUpdate -Scope CurrentUser -Force
Import-Module PSWindowsUpdate

Get-WindowsUpdate
Get-WindowsUpdate -Verbose
```

`Get-WindowsUpdate` queries Windows Update for available updates. Output is the list of pending updates with KB numbers, sizes, and categories.

To install everything:

```
Install-WindowsUpdate -AcceptAll -AutoReboot:$false
```

`-AcceptAll` accepts EULAs without prompting. `-AutoReboot:$false` prevents automatic reboot. For most server-like workloads, this is the right combination.

To install only specific updates:

```
Install-WindowsUpdate -KBArticleID KB5031356 -AcceptAll
```

For the lab box, do not run `Install-WindowsUpdate` unless you want a long wait and a reboot. The exercises later in this chapter use `Get-WindowsUpdate` (read-only) instead.

### Pause updates

You can pause Windows Update for a period:

```
# Via the registry (effective immediately):
$pause = (Get-Date).AddDays(7).ToString('yyyy-MM-ddT00:00:00Z')
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings' -Name 'PauseUpdatesExpiryTime' -Value $pause
```

That pauses updates until the date you set. The maximum is 35 days (Microsoft's policy).

For longer-term control, you use Group Policy or registry-based deferral policies. We cover the local Group Policy hardening in Chapter 12.

### Active hours

By default, Windows tries not to reboot during "active hours" (8 AM to 5 PM). To set:

```
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings' -Name 'ActiveHoursStart' -Value 8
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings' -Name 'ActiveHoursEnd' -Value 18
```

Active hours is a soft control. In practice, you also want a deferral policy for feature updates and some discipline around when restarts happen. Group Policy and Group Policy via Intune are how organizations handle this at scale.

### WSUS at recognition depth

In a corporate environment, Windows endpoints typically do not pull updates directly from Microsoft. They pull from a local WSUS server. *WSUS, Windows Server Update Services, is a Microsoft server product that mirrors and curates Windows updates for an organization, allowing centralized control of which updates are deployed when.*

Your lab box does not have WSUS configured. If you joined a corporate domain, registry settings under `HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate` would point you at the WSUS server. To check:

```
Get-ItemProperty 'HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate' -ErrorAction SilentlyContinue
```

On your lab box, this returns nothing or an empty result. On a managed corporate workstation, it returns the WSUS server URL and policy details.

For this chapter, recognition is enough. WSUS is intermediate cohort territory.

---

## Drivers

Windows drivers are kernel-mode code. *A driver is software that runs in kernel-mode to control hardware or provide a privileged service to user-mode applications.* Driver updates are part of Windows Update by default but can be controlled separately.

To list drivers:

```
Get-WindowsDriver -Online | Select-Object -First 10 Driver, OriginalFileName, Date, Version
```

To list signed drivers and their publishers:

```
Get-CimInstance Win32_PnPSignedDriver |
    Select-Object DeviceName, DriverProviderName, DriverVersion |
    Sort-Object DeviceName |
    Select-Object -First 20
```

The DriverProviderName tells you who signed the driver. Microsoft drivers are usually safe by definition. Third-party drivers from known hardware vendors (Intel, NVIDIA, etc.) are usually fine. Unknown publishers warrant attention.

Driver issues are one of the cleanest categories of "this Windows machine is unstable." When troubleshooting blue screens, recently-installed or recently-updated drivers are the first suspects. Reverting a driver in Device Manager is the recovery path.

For the chapter scope, recognition is enough: drivers exist, they run in kernel-mode, they can be inventoried, they are an important hardening surface (since attackers signing malicious drivers is a real threat). Chapter 12 covers the relevant CIS items.

---

## Removing unnecessary software

Once you have an inventory, the next question is what should not be there.

The CIS hardening principle: remove software that is not needed. The Microsoft Store apps that ship with Windows include several that are pure consumer fluff (Solitaire, Xbox Game Bar, various ad-supported widgets). For a workstation focused on real work, removing them reduces attack surface.

To remove a Store app for the current user:

```
Get-AppxPackage *Solitaire* | Remove-AppxPackage
```

To remove for all users (requires admin):

```
Get-AppxPackage -AllUsers *Solitaire* | Remove-AppxPackage -AllUsers
```

To prevent reinstallation when new users are created:

```
Get-AppxProvisionedPackage -Online | Where-Object DisplayName -like "*Solitaire*" | Remove-AppxProvisionedPackage -Online
```

The `Get-AppxProvisionedPackage` query finds the staged version that gets installed for new users. Removing it prevents the "I removed this app and it came back when a new user logged in" surprise.

For traditional MSI/EXE applications:

```
winget uninstall --id <id>
```

Or from the registry, find the UninstallString and run it:

```powershell
$pkg = Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' |
    Where-Object DisplayName -eq "Some Application"
$pkg.UninstallString
```

The UninstallString is the command the application's uninstaller registered. Running it (with appropriate silent flags) uninstalls.

A pragmatic list of categories to consider removing on a workstation:

- Pre-installed games (Solitaire, Xbox apps).
- Marketing apps (LinkedIn, TikTok, news widgets).
- Manufacturer "system optimizers" (often unnecessary, sometimes harmful).
- Third-party browser toolbars (rare on Windows 11 but persistent on older systems).

For each one, the question is "does any user actually use this." If not, remove.

---

## Try this

**1. Inventory installed software using all three methods.**

Run:

```
winget list | Out-File C:\AppData\inventory-winget.txt
Get-Package | Out-File C:\AppData\inventory-getpackage.txt
# The registry one-liner from earlier in this chapter
```

Compare the three lists. Are there packages in one that the others miss? winget tends to miss installer-only software; Get-Package can miss winget Store apps. The registry sees everything that registers properly.

**2. Install a package three different ways.**

Pick a small utility (like Notepad++ if not installed, or 7zip). Install it via winget. Confirm with `winget list`. Uninstall it. Install it via the registry one-liner (find the package's installer in their docs, run msiexec with appropriate flags). Confirm with the registry inventory query. Uninstall it.

The point is to feel the difference between the install paths. winget is friction-free; MSI is more verbose; EXE installers are vendor-specific.

**3. Pin a package version.**

Pick any installed package. Pin it to its current major version using `winget pin add`. Run `winget pin list` to confirm. Run `winget upgrade --all --dry-run` (or just `winget upgrade`) and confirm the pinned package is not in the upgrade list.

Then unpin it.

**4. Inventory drivers.**

Run:

```
Get-CimInstance Win32_PnPSignedDriver |
    Select-Object DeviceName, DriverProviderName, DriverVersion, DriverDate |
    Sort-Object DriverProviderName, DeviceName
```

Read the output. For each unique provider, identify whether you recognize it (Microsoft, Intel, AMD, etc.). Anything from a publisher you do not recognize warrants further investigation. (On a clean Windows 11 lab VM, you likely see only Microsoft and the virtualization provider.)

**5. Check Windows Update state.**

Run:

```
Get-Service -Name wuauserv, UsoSvc | Format-Table Name, Status, StartType
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings' -ErrorAction SilentlyContinue
```

If you have not run it yet, install PSWindowsUpdate and run `Get-WindowsUpdate`. Read the list. Note: do not run `Install-WindowsUpdate` on the lab box during the workshop unless instructed; it can take a long time.

---

## Common stumbling blocks

> **`winget` returns "No applicable update found" but I see updates in the GUI.**
> The Windows Update GUI and `winget upgrade` show different update sources. winget shows app upgrades from its repositories; the Settings > Windows Update GUI shows OS updates. `Get-WindowsUpdate` from PSWindowsUpdate is what bridges them.

> **An MSI install logs success but the application is broken.**
> Read the verbose log (`/l*v <path>`). MSI installers can succeed at the file-copying step but fail at custom actions, leaving a half-installed application. The log shows which step failed and why.

> **`Get-AppxPackage | Remove-AppxPackage` works for me but not for new users.**
> You also need to remove the provisioned package: `Get-AppxProvisionedPackage -Online | Where-Object DisplayName -like "*Name*" | Remove-AppxProvisionedPackage -Online`. The provisioned version is the template that new user profiles get. Without removing it, the app comes back when a new user logs in.

> **`winget upgrade` upgrades a package I did not want upgraded.**
> If you wanted to exclude it, you needed to pin it first (`winget pin add`). Pins are persistent and survive winget operations. For one-off "skip this," you can also pass `--exclude <id>` to `winget upgrade`.

> **PSWindowsUpdate cmdlets fail with "Not signed in to a Microsoft account" or similar.**
> PSWindowsUpdate sometimes needs to register itself with Windows Update. Run `Add-WUServiceManager -MicrosoftUpdate` once to register, then retry.

> **A driver update from Windows Update broke my hardware.**
> In Settings > Windows Update > Update history > Uninstall updates, you can roll back the most recent update. From PowerShell: `Get-CimInstance Win32_QuickFixEngineering | Sort-Object InstalledOn -Descending | Select-Object -First 5` shows recently installed hotfixes and updates. The reverse is `wusa /uninstall /kb:<number>`.

> **`Install-WindowsUpdate` succeeded but Windows still shows updates pending.**
> Some updates require a restart to complete. `Get-WURebootStatus` (from PSWindowsUpdate) tells you whether a reboot is pending. Pending-but-not-applied updates are visible in `wuauclt /detectnow` output.

---

## What this gets you

After this chapter:

- You can inventory installed software using winget, Get-Package, or the registry directly.
- You know what the registry uninstall keys are and can read them.
- You can install software from any of the major Windows sources: winget, MSI, EXE, Microsoft Store.
- You can pin a package version when you have a reason.
- You can reproduce a setup using `winget export`/`import`.
- You can configure Windows Update behavior, pause it, and trigger updates programmatically with PSWindowsUpdate.
- You know about WSUS and drivers at recognition depth.
- You can remove unnecessary software, including Microsoft Store apps and provisioned packages.

Patch management is one of the fundamental disciplines of working IT. Security+ tests it. Working admins live it. The patterns from this chapter (inventory > update > validate > document) are what scale to managing 5,000 endpoints.

---

## What's next

Chapter 5 is Services and Scheduled Tasks. The chapter where you stop just running services and start writing your own service definitions, where Scheduled Tasks become a real automation tool rather than a curiosity, and where you learn to read the configuration of services you did not install.
