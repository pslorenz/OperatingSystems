# Chapter 12: CIS-aligned Windows Hardening

**You come in with:** Chapter 3 (users and permissions), Chapter 6 (logs and audit policy), Chapter 8 (host networking and firewall), Chapter 10 (process inspection), Chapter 11 (artifacts you would not want left behind on your own systems).
**You leave with:** a hardened lab box. You took a default Windows 11 install and made it materially more resistant to compromise, with a documented before-and-after diff. The diff is your portfolio piece.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Heavy alignment, this is the capstone chapter. Domain 4.1 (hardening techniques: encryption, configuration enforcement, host-based firewall, default password changes, removal of unnecessary software, applying security techniques to operating systems and servers). Domain 1.4 (cryptographic solutions: BitLocker, signed binaries). Domain 2.5 (mitigation techniques: hardening, patching, configuration enforcement). Domain 4.5 (operating system security: Group Policy applied locally). Domain 4.6 (access controls: least privilege applied across the host). Domain 5.4 (security awareness practices: documenting security configurations).

---

## Why the unit ends here

Twelve chapters in, you can operate a Windows box, audit a Windows box, and investigate a Windows box. The natural last question: how do you make a Windows box less likely to need investigation in the first place.

Hardening is the answer. Configuration changes that reduce the attack surface, that follow the principle of least privilege, that align with industry-recognized standards. The work is mostly small, individually unimpressive changes that combine into a meaningfully different security posture.

The CIS Benchmarks are the most widely referenced hardening standards. *CIS, the Center for Internet Security, publishes detailed configuration baselines for operating systems, applications, and cloud platforms.* The CIS Microsoft Windows 11 Enterprise Benchmark has hundreds of items, organized into Profile Level 1 (essential, low operational impact) and Profile Level 2 (more aggressive, higher operational cost). A real production hardening exercise typically targets Level 1 across the board, with selected Level 2 items based on the system's role.

This chapter does not work through hundreds of items. It works through a curated subset of about 20, chosen to:

- Cover the major CIS sections (account policies, audit policy, Defender, firewall, logging, hardening surface).
- Reinforce skills from previous chapters.
- Produce a measurable change in posture.
- Fit in 60 to 90 minutes.

Your output is a before-and-after comparison. That diff is what you can show in an interview as evidence of what you learned.

---

## A note on benchmark numbers and Local Group Policy

The CIS items in this chapter reference the structure of the CIS Microsoft Windows 11 Enterprise Benchmark v3.x. CIS occasionally renumbers items between releases. If the specific number on your benchmark differs slightly, the principle is what matters; look up the corresponding rule by name in your benchmark version. The benchmark itself is freely available at cisecurity.org with registration.

We use Local Group Policy (`gpedit.msc`) for some of the configuration in this chapter. *Local Group Policy is the in-box Windows mechanism for applying group-policy-style configuration to a single machine, even when not domain-joined.* Many CIS items are easier to apply via GPO than via individual registry edits, and the GPO surface is what real production environments use (typically pushed from Active Directory, but the local equivalent works the same way).

For each CIS item, we show both the GPO path and the equivalent registry change where useful. In a real environment, GPO is the durable answer; in a lab or scripted environment, registry edits are scriptable.

---

## Setting up the before-state baseline

Before you change anything, capture the starting state. Without a baseline, you cannot describe what you changed.

Create a working directory:

```
mkdir C:\hardening
cd C:\hardening
```

Capture the relevant config and outputs:

```powershell
# Audit policy
auditpol /get /category:* > auditpol.before.txt

# Firewall state
Get-NetFirewallProfile | Format-List > firewall.before.txt

# Defender state
Get-MpPreference | Format-List > defender.before.txt
Get-MpComputerStatus | Format-List > defender-status.before.txt

# Account policies
net accounts > accounts.before.txt

# SMB config
Get-SmbServerConfiguration | Format-List > smb.before.txt

# Registry exports for the keys we will modify
reg export 'HKLM\System\CurrentControlSet\Services\LanmanServer\Parameters' lanmanserver.before.reg /y
reg export 'HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System' policies-system.before.reg /y
reg export 'HKLM\Software\Policies\Microsoft\Windows\PowerShell' powershell-policies.before.reg /y 2>&1 | Out-Null

# UAC settings
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System' | Format-List > uac.before.txt

# Local users and admin members
Get-LocalUser | Select-Object Name, Enabled, PasswordExpires, PasswordLastSet > users.before.txt
Get-LocalGroupMember -Group Administrators > admins.before.txt

# BitLocker status
Get-BitLockerVolume -ErrorAction SilentlyContinue | Format-List > bitlocker.before.txt
```

You now have a snapshot. Every change in the rest of the chapter will be compared to this snapshot at the end.

---

## Section 1: Account policies (CIS 1.1)

Account policies set rules for password complexity, account lockout, and similar.

### CIS 1.1.x: Password and lockout policy

Open Local Security Policy: press Windows key, type `secpol.msc`, press Enter. Navigate to Account Policies > Password Policy and Account Policies > Account Lockout Policy.

Or use the command line:

```
net accounts /minpwlen:14
net accounts /maxpwage:60
net accounts /minpwage:1
net accounts /lockoutthreshold:10
net accounts /lockoutwindow:15
net accounts /lockoutduration:15
```

Reading these:

- **Minimum password length 14**: CIS recommendation. Real environments often go higher.
- **Maximum password age 60 days**: passwords expire every 60 days.
- **Minimum password age 1 day**: prevents users from cycling through passwords to reuse an old one.
- **Lockout threshold 10**: account locks after 10 failed attempts.
- **Lockout window 15 minutes**: the count resets after 15 minutes without a failure.
- **Lockout duration 15 minutes**: locked accounts auto-unlock after 15 minutes.

These are CIS Profile Level 1 defaults. Adjust based on your organization's risk tolerance.

For password complexity (must contain mix of character types), the setting is in `secpol.msc` under Account Policies > Password Policy > "Password must meet complexity requirements." Or via registry/GPO in an exported config.

### CIS 2.3.7.x: Disable Guest account

```
Get-LocalUser -Name Guest | Disable-LocalUser
```

The Guest account is disabled by default on modern Windows, but worth confirming.

### CIS 2.3.1.1: Rename Administrator

The built-in Administrator (RID 500) has a known SID and well-known username. Renaming it raises the bar slightly for attacks that target it specifically:

```
Rename-LocalUser -Name Administrator -NewName "AdmAccount"
```

The rename does not change the SID. RID 500 remains RID 500. But the obvious-target name is gone.

(On a personal lab box this is more theatrical than substantive. On a production server, it is a small additional layer.)

---

## Section 2: User Account Control (CIS 2.3.17.x)

UAC is the mechanism that prompts for elevation when administrative actions are attempted. The CIS recommendations tighten its behavior.

The key UAC settings live in `HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System`. To set them via PowerShell:

```powershell
$path = 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System'

# CIS 2.3.17.1 - Admin Approval Mode for the built-in Administrator
Set-ItemProperty -Path $path -Name 'FilterAdministratorToken' -Value 1 -Type DWord

# CIS 2.3.17.2 - UI Access apps cannot prompt for elevation without secure desktop
Set-ItemProperty -Path $path -Name 'EnableUIADesktopToggle' -Value 0 -Type DWord

# CIS 2.3.17.3 - Behavior of elevation prompt for administrators (prompt for consent on secure desktop)
Set-ItemProperty -Path $path -Name 'ConsentPromptBehaviorAdmin' -Value 2 -Type DWord

# CIS 2.3.17.4 - Behavior of elevation prompt for standard users (automatically deny)
Set-ItemProperty -Path $path -Name 'ConsentPromptBehaviorUser' -Value 0 -Type DWord

# CIS 2.3.17.5 - Detect application installations and prompt for elevation
Set-ItemProperty -Path $path -Name 'EnableInstallerDetection' -Value 1 -Type DWord

# CIS 2.3.17.6 - Only elevate UIAccess applications installed in secure locations
Set-ItemProperty -Path $path -Name 'EnableSecureUIAPaths' -Value 1 -Type DWord

# CIS 2.3.17.7 - Run all administrators in Admin Approval Mode (UAC must be on)
Set-ItemProperty -Path $path -Name 'EnableLUA' -Value 1 -Type DWord

# CIS 2.3.17.8 - Switch to secure desktop when prompting
Set-ItemProperty -Path $path -Name 'PromptOnSecureDesktop' -Value 1 -Type DWord

# CIS 2.3.17.9 - Virtualize file and registry write failures
Set-ItemProperty -Path $path -Name 'EnableVirtualization' -Value 1 -Type DWord
```

After all of these, UAC is at the most restrictive practical setting. Some workflows that worked before may now require explicit elevation more often. That is the intended cost of the hardening.

A reboot is required for some of these to fully take effect.

---

## Section 3: Audit policy (CIS 17.x)

We touched on this in Chapter 6. Now we apply the full CIS recommendations.

```
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable
auditpol /set /subcategory:"Other Account Logon Events" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable
auditpol /set /subcategory:"Account Lockout" /success:enable
auditpol /set /subcategory:"Audit Policy Change" /success:enable /failure:enable
auditpol /set /subcategory:"Authentication Policy Change" /success:enable
auditpol /set /subcategory:"Authorization Policy Change" /success:enable
auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable
auditpol /set /subcategory:"Security State Change" /success:enable
auditpol /set /subcategory:"Security System Extension" /success:enable
auditpol /set /subcategory:"System Integrity" /success:enable /failure:enable
```

Plus the registry switch for command-line capture in 4688:

```powershell
Set-ItemProperty -Path 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit' `
    -Name 'ProcessCreationIncludeCmdLine_Enabled' -Value 1 -Type DWord
```

After this, your Security log captures dramatically more than the default. You will see Event 4688 for every process started, with full command lines. This is one of the highest-impact hardening changes available.

The cost: more events in the Security log, which increases its size requirements. Bump the Security log size to compensate:

```
wevtutil sl Security /ms:1073741824
```

That sets the Security log to 1 GB, which is more than enough for a workstation.

---

## Section 4: PowerShell logging (CIS 18.9.x)

PowerShell script block logging captures the actual content of every script block executed, even if obfuscated or downloaded. We mentioned this in Chapter 9; now we configure it.

```powershell
$path = 'HKLM:\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging'
if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
Set-ItemProperty -Path $path -Name 'EnableScriptBlockLogging' -Value 1 -Type DWord

# Module logging
$path = 'HKLM:\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging'
if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
Set-ItemProperty -Path $path -Name 'EnableModuleLogging' -Value 1 -Type DWord
$modulePath = Join-Path $path 'ModuleNames'
if (-not (Test-Path $modulePath)) { New-Item -Path $modulePath -Force | Out-Null }
Set-ItemProperty -Path $modulePath -Name '*' -Value '*' -Type String

# Transcription
$path = 'HKLM:\Software\Policies\Microsoft\Windows\PowerShell\Transcription'
if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
Set-ItemProperty -Path $path -Name 'EnableTranscripting' -Value 1 -Type DWord
Set-ItemProperty -Path $path -Name 'IncludeInvocationHeader' -Value 1 -Type DWord
Set-ItemProperty -Path $path -Name 'OutputDirectory' -Value 'C:\Logs\PSTranscripts' -Type String
New-Item -ItemType Directory -Path 'C:\Logs\PSTranscripts' -Force | Out-Null
```

Reading these:

- **EnableScriptBlockLogging**: every script block executed is logged as Event 4104 in Microsoft-Windows-PowerShell/Operational. The decoded content of the block is captured, even for `-EncodedCommand` payloads.
- **EnableModuleLogging** with `'*'` for all modules: pipeline invocations are logged.
- **Transcription**: every PowerShell session writes a transcript to `C:\Logs\PSTranscripts`.

After this, your PowerShell activity is heavily instrumented. For investigation work, this is one of the most valuable changes you can make to a Windows endpoint.

The cost: disk space for transcripts (rotate or archive periodically) and slightly more CPU on PowerShell-heavy workloads. Both are minor for a single workstation.

---

## Section 5: Windows Defender (CIS 18.10.x)

Defender comes on by default. The CIS recommendations tighten its configuration.

```powershell
# Real-time protection on (should already be)
Set-MpPreference -DisableRealtimeMonitoring $false
Set-MpPreference -DisableBehaviorMonitoring $false
Set-MpPreference -DisableIOAVProtection $false
Set-MpPreference -DisableScriptScanning $false
Set-MpPreference -DisableArchiveScanning $false
Set-MpPreference -DisableEmailScanning $false
Set-MpPreference -DisableRemovableDriveScanning $false

# Cloud-delivered protection
Set-MpPreference -MAPSReporting Advanced
Set-MpPreference -SubmitSamplesConsent SendSafeSamples

# Network Inspection System
Set-MpPreference -DisableIntrusionPreventionSystem $false

# Quarantine items after a reasonable time
Set-MpPreference -QuarantinePurgeItemsAfterDelay 30

# Disable PUA detection: ON
Set-MpPreference -PUAProtection Enabled

# Controlled folder access (ransomware protection)
Set-MpPreference -EnableControlledFolderAccess Enabled
```

Walk through what each does:

- **Realtime/Behavior/IOAV/Script/Archive/Email/RemovableDrive Monitoring**: each one is a different scanning mode. All should be enabled.
- **MAPSReporting Advanced**: sends telemetry to Microsoft so cloud-based detection can work.
- **PUAProtection**: detects "Potentially Unwanted Applications" (browser extensions that hijack search, junk software bundled with installers).
- **Controlled folder access**: prevents non-trusted apps from modifying files in Documents, Desktop, etc. Ransomware-defense feature.

Confirm:

```
Get-MpPreference | Select-Object DisableRealtimeMonitoring, PUAProtection, EnableControlledFolderAccess
```

All should reflect the values you set.

### Attack Surface Reduction rules

ASR rules are specific behaviors Defender blocks. The CIS-aligned set:

```powershell
# Block executable content from email and webmail
Add-MpPreference -AttackSurfaceReductionRules_Ids 'BE9BA2D9-53EA-4CDC-84E5-9B1EEEE46550' -AttackSurfaceReductionRules_Actions Enabled

# Block all Office applications from creating child processes
Add-MpPreference -AttackSurfaceReductionRules_Ids 'D4F940AB-401B-4EFC-AADC-AD5F3C50688A' -AttackSurfaceReductionRules_Actions Enabled

# Block Office applications from creating executable content
Add-MpPreference -AttackSurfaceReductionRules_Ids '3B576869-A4EC-4529-8536-B80A7769E899' -AttackSurfaceReductionRules_Actions Enabled

# Block Office applications from injecting code into other processes
Add-MpPreference -AttackSurfaceReductionRules_Ids '75668C1F-73B5-4CF0-BB93-3ECF5CB7CC84' -AttackSurfaceReductionRules_Actions Enabled

# Block JavaScript or VBScript from launching downloaded executable content
Add-MpPreference -AttackSurfaceReductionRules_Ids 'D3E037E1-3EB8-44C8-A917-57927947596D' -AttackSurfaceReductionRules_Actions Enabled

# Block execution of potentially obfuscated scripts
Add-MpPreference -AttackSurfaceReductionRules_Ids '5BEB7EFE-FD9A-4556-801D-275E5FFC04CC' -AttackSurfaceReductionRules_Actions Enabled

# Block Win32 API calls from Office macros
Add-MpPreference -AttackSurfaceReductionRules_Ids '92E97FA1-2EDF-4476-BDD6-9DD0B4DDDC7B' -AttackSurfaceReductionRules_Actions Enabled

# Block credential stealing from LSASS
Add-MpPreference -AttackSurfaceReductionRules_Ids '9E6C4E1F-7D60-472F-BA1A-A39EF669E4B2' -AttackSurfaceReductionRules_Actions Enabled
```

Each GUID identifies a specific ASR rule. Together they block many of the most common attack chains: malicious Office documents, obfuscated scripts, credential dumping, etc. CIS recommends setting all of these to Enabled.

To confirm:

```
Get-MpPreference | Select-Object -ExpandProperty AttackSurfaceReductionRules_Ids
```

For full details on each rule, see Microsoft's ASR documentation. The intermediate cohort goes deeper into rule tuning.

---

## Section 6: Network and SMB hardening (CIS 18.x)

### CIS 18.3.x: Disable SMBv1

SMBv1 is the deprecated, insecure version of SMB. Famously the vector for WannaCry. It should be off everywhere.

```powershell
# Check current state
Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol

# Disable
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

### CIS 2.3.10.x: Limit anonymous access

```powershell
$path = 'HKLM:\System\CurrentControlSet\Control\Lsa'
Set-ItemProperty -Path $path -Name 'RestrictAnonymous' -Value 1 -Type DWord
Set-ItemProperty -Path $path -Name 'RestrictAnonymousSAM' -Value 1 -Type DWord
Set-ItemProperty -Path $path -Name 'EveryoneIncludesAnonymous' -Value 0 -Type DWord
```

These prevent anonymous users from enumerating accounts, shares, or other information.

### CIS 9.x: Windows Defender Firewall on all profiles

```powershell
Set-NetFirewallProfile -Profile Domain, Private, Public -Enabled True
Set-NetFirewallProfile -Profile Domain, Private, Public -DefaultInboundAction Block
Set-NetFirewallProfile -Profile Domain, Private, Public -DefaultOutboundAction Allow
Set-NetFirewallProfile -Profile Domain, Private, Public -LogAllowed False
Set-NetFirewallProfile -Profile Domain, Private, Public -LogBlocked True
Set-NetFirewallProfile -Profile Domain, Private, Public -LogIgnored False
Set-NetFirewallProfile -Profile Domain, Private, Public -LogMaxSizeKilobytes 16384
```

Reading these:

- All three profiles enabled.
- Default inbound = Block, default outbound = Allow (the standard host-firewall posture).
- Log dropped (blocked) packets but not allowed ones (to avoid log volume).
- 16 MB log size, which is enough for a workstation.

This was largely covered in Chapter 8; this is the CIS-blessed baseline.

---

## Section 7: BitLocker (CIS 18.9.x)

BitLocker enables full-disk encryption. We covered the recognition in Chapter 7; here we enable it.

**Note:** BitLocker requires a TPM. On a virtual lab box, the VM may have a virtual TPM (vTPM) configured, or it may not. Check before attempting:

```
Get-Tpm
```

If `TpmReady` is `False` or the cmdlet errors, the VM does not have a TPM and BitLocker cannot be enabled in its standard form. Skip this section if so.

If TPM is ready:

```powershell
# Enable BitLocker on C: with TPM protector
Enable-BitLocker -MountPoint C: -EncryptionMethod XtsAes256 -TpmProtector

# Add a recovery password
Add-BitLockerKeyProtector -MountPoint C: -RecoveryPasswordProtector

# View the recovery password
(Get-BitLockerVolume -MountPoint C:).KeyProtector | Where-Object KeyProtectorType -eq RecoveryPassword | Select-Object -ExpandProperty RecoveryPassword
```

**Critical:** save the recovery password somewhere off the box. If anything goes wrong with the TPM unlock, this 48-digit number is what gets you back in. Without it, the encrypted volume becomes unrecoverable.

For the lab, write the recovery password to a file outside `C:` (on a USB stick if you have one connected, or copy it out of the VM via clipboard):

```powershell
(Get-BitLockerVolume -MountPoint C:).KeyProtector |
    Where-Object KeyProtectorType -eq RecoveryPassword |
    Select-Object KeyProtectorId, RecoveryPassword |
    Export-Csv -Path bitlocker-recovery.csv -NoTypeInformation
```

Move that file off the box to be safe.

After enabling, encryption begins in the background. It takes hours on a real disk; on a small VM disk, it can be fast. Monitor:

```
manage-bde -status C:
```

For a lab, BitLocker is more theatrical than substantive (the lab box is short-lived). For a production laptop, it is one of the highest-impact security controls available.

---

## Section 8: Local Group Policy as a hardening surface

We have been making changes via PowerShell and registry edits. Many of the same changes can be made via Local Group Policy, and there are some changes that are easier in GPO than in PowerShell.

To open Local Group Policy:

```
gpedit.msc
```

The tree:

- **Computer Configuration**: applies to the machine.
  - **Windows Settings**: security settings, scripts, etc.
  - **Administrative Templates**: registry-backed policies. This is the bulk of GPO content.
- **User Configuration**: applies per user.

A few items easier in GPO than in PowerShell:

### Restrict the use of removable storage

Computer Configuration > Administrative Templates > System > Removable Storage Access. Multiple settings to deny read/write to specific device classes. The registry equivalent works but is less discoverable.

### Restrict the use of LM and NTLMv1

Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options > "Network security: LAN Manager authentication level."

Set to "Send NTLMv2 response only. Refuse LM & NTLM."

CIS recommends the most restrictive value (Level 5: Send NTLMv2 response only. Refuse LM & NTLM). Some legacy applications break under this, so test.

### Configure auditing in detail

The auditpol commands earlier in this chapter set the same things, but the GPO view (Computer Configuration > Windows Settings > Security Settings > Advanced Audit Policy Configuration) shows the full hierarchy of categories at once. For learning what the categories are, the GPO browser is more discoverable than the auditpol command line.

### A practical exercise: explore the policy tree

Open `gpedit.msc`. Spend 5 minutes browsing the tree. Specifically:

- Computer Configuration > Administrative Templates > Windows Components > Windows Defender Antivirus
- Computer Configuration > Administrative Templates > Windows Components > Windows PowerShell
- Computer Configuration > Windows Settings > Security Settings > Local Policies > Audit Policy
- Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options

Read a few policy descriptions. Notice that each policy has explanatory text that explains what it does and what the recommended values typically are. The GPO surface is documented; the registry version is not.

### When to use which

For automation and scripting: PowerShell or registry edits. Idempotent, version-controllable, automatable.

For interactive review and configuration of a single machine: gpedit.msc. More discoverable, less error-prone, easier to undo.

For a real environment: domain-joined GPO from Active Directory. Centralized, auditable, applied consistently across thousands of machines. (Intermediate cohort.)

For this chapter: we mix. Both surfaces are valuable to know.

---

## Section 9: Removing unnecessary software

Touched on in Chapter 4. The CIS principle: remove what you do not use.

A reasonable subset to consider removing on a workstation:

```powershell
# Microsoft Store games
$bloat = @(
    '*Microsoft.SolitaireCollection*',
    '*Microsoft.XboxGamingOverlay*',
    '*Microsoft.XboxGameCallableUI*',
    '*Microsoft.XboxIdentityProvider*',
    '*Microsoft.GamingApp*',
    '*Microsoft.BingNews*',
    '*Microsoft.BingWeather*',
    '*Microsoft.GetHelp*',
    '*Microsoft.Getstarted*',
    '*Microsoft.MicrosoftStickyNotes*',
    '*Microsoft.WindowsAlarms*',
    '*Microsoft.WindowsFeedbackHub*',
    '*Microsoft.WindowsMaps*',
    '*Microsoft.WindowsSoundRecorder*',
    '*Microsoft.YourPhone*',
    '*Microsoft.ZuneMusic*',
    '*Microsoft.ZuneVideo*'
)

foreach ($pattern in $bloat) {
    Get-AppxPackage -AllUsers -Name $pattern | Remove-AppxPackage -AllUsers
    Get-AppxProvisionedPackage -Online | Where-Object DisplayName -like $pattern | Remove-AppxProvisionedPackage -Online
}
```

Each app removed is one less attack surface and one less thing to update. The list is conservative; CIS does not specifically mandate removing these, but the principle of "remove unnecessary software" applies.

For business-critical apps, evaluate before removing. Some seemingly removable apps are integration points that other apps depend on.

---

## Capturing the after-state

You have made the changes. Now capture the after-state for the diff:

```powershell
cd C:\hardening

auditpol /get /category:* > auditpol.after.txt
Get-NetFirewallProfile | Format-List > firewall.after.txt
Get-MpPreference | Format-List > defender.after.txt
Get-MpComputerStatus | Format-List > defender-status.after.txt
net accounts > accounts.after.txt
Get-SmbServerConfiguration | Format-List > smb.after.txt
reg export 'HKLM\System\CurrentControlSet\Services\LanmanServer\Parameters' lanmanserver.after.reg /y
reg export 'HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System' policies-system.after.reg /y
reg export 'HKLM\Software\Policies\Microsoft\Windows\PowerShell' powershell-policies.after.reg /y
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System' | Format-List > uac.after.txt
Get-LocalUser | Select-Object Name, Enabled, PasswordExpires, PasswordLastSet > users.after.txt
Get-LocalGroupMember -Group Administrators > admins.after.txt
Get-BitLockerVolume -ErrorAction SilentlyContinue | Format-List > bitlocker.after.txt
```

Generate diffs (PowerShell does not have built-in diff like Linux; use Compare-Object):

```powershell
function Diff-File {
    param($Before, $After, $Output)
    $b = Get-Content $Before
    $a = Get-Content $After
    Compare-Object $b $a | Out-File $Output
}

Diff-File auditpol.before.txt auditpol.after.txt auditpol.diff.txt
Diff-File defender.before.txt defender.after.txt defender.diff.txt
Diff-File firewall.before.txt firewall.after.txt firewall.diff.txt
Diff-File uac.before.txt uac.after.txt uac.diff.txt
Diff-File smb.before.txt smb.after.txt smb.diff.txt
Diff-File accounts.before.txt accounts.after.txt accounts.diff.txt
```

Read each diff file. Confirm each change matches what you intended. The collection of diff files is your portfolio piece.

---

## A simple validation script

Write a script that confirms the hardening took. Save as `C:\hardening\verify.ps1`:

```powershell
#Requires -Version 5.1
#Requires -RunAsAdministrator

$ErrorActionPreference = 'Continue'

Write-Host "== CIS verification (selected items) ==" -ForegroundColor Cyan

function Check {
    param($Label, $Test, $Expected)
    if ($Test -eq $Expected -or ($Expected -is [string] -and $Test -like "*$Expected*")) {
        Write-Host "PASS: $Label" -ForegroundColor Green
    } else {
        Write-Host "FAIL: $Label (got: $Test, expected: $Expected)" -ForegroundColor Red
    }
}

# UAC
$uac = Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System'
Check "UAC enabled (EnableLUA = 1)" $uac.EnableLUA 1
Check "UAC secure desktop (PromptOnSecureDesktop = 1)" $uac.PromptOnSecureDesktop 1
Check "Admin Approval Mode for built-in (FilterAdministratorToken = 1)" $uac.FilterAdministratorToken 1

# Audit
$audit = (auditpol /get /subcategory:"Process Creation" /r) -join "`n"
Check "Audit Process Creation: Success" ($audit -match 'Success') $true

# Firewall
Get-NetFirewallProfile | ForEach-Object {
    Check "Firewall $($_.Name) enabled" $_.Enabled $true
}

# SMB
Check "SMBv1 disabled" (Get-SmbServerConfiguration).EnableSMB1Protocol $false

# Defender
Check "Defender realtime protection on" (Get-MpComputerStatus).RealTimeProtectionEnabled $true
Check "Defender PUA protection enabled" (Get-MpPreference).PUAProtection 1
Check "Controlled Folder Access enabled" (Get-MpPreference).EnableControlledFolderAccess 1

# PowerShell logging
$pslog = Get-ItemProperty 'HKLM:\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' -ErrorAction SilentlyContinue
Check "PowerShell ScriptBlockLogging enabled" $pslog.EnableScriptBlockLogging 1

# Anonymous restrictions
$lsa = Get-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Lsa'
Check "RestrictAnonymous = 1" $lsa.RestrictAnonymous 1
Check "RestrictAnonymousSAM = 1" $lsa.RestrictAnonymousSAM 1

# Process Creation command line capture
$audit_cmd = Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit' -ErrorAction SilentlyContinue
Check "Process Creation command line capture" $audit_cmd.ProcessCreationIncludeCmdLine_Enabled 1

Write-Host "`n== End of verification ==" -ForegroundColor Cyan
```

Run it from elevated PowerShell:

```
C:\hardening\verify.ps1
```

Each PASS line is one item validated. Each FAIL line tells you what is not yet configured as expected. This is the simplest version of "infrastructure-as-code with verification" applied to Windows hardening.

---

## What you did not do

This chapter covered about 20 CIS items. The full CIS Microsoft Windows 11 Enterprise Benchmark has hundreds. Items intentionally not covered, with brief explanations:

- **AppLocker / Windows Defender Application Control (CIS 9.3.x and 18.9.x)**: application allow-listing. Genuinely powerful, with significant configuration work to deploy without breaking legitimate apps. Intermediate cohort.
- **Detailed firewall rule sets per service**: the chapter set the profiles correctly but did not write specific service-level allow/block rules. Real production environments do.
- **Specific Group Policy hardening for Microsoft Office**: large category covered separately by CIS (Microsoft Office benchmark), out of scope for OS hardening.
- **Windows Hello / certificate-based authentication**: genuinely good but beyond beginner scope.
- **Specific service disable list (CIS 5.x)**: dozens of services CIS recommends disabling on most systems. Worth doing in a real environment, but each one needs evaluation against actual use; a blanket script is risky.
- **DNSSEC and DoH configuration (CIS 18.5.x)**: improving DNS security at the OS level. Real but specialized.

The full benchmark is worth reading. Working through it on a real production system is a multi-day project the first time; subsequent systems use automation (Group Policy, Intune, Ansible/PowerShell DSC).

---

## Try this

**1. Capture before, harden, capture after, generate diffs.**

This is the chapter's main exercise. Walk through every section, applying the changes. Capture before-state at the start. Capture after-state at the end. Generate the diffs.

The deliverable: a folder containing your before-state files, your after-state files, and the diff files. The diffs are your portfolio piece.

**2. Run the verification script.**

Use the script from the chapter. Run it after applying all the changes. Every line should PASS. If any FAIL, walk back to that section and identify what you missed.

**3. Add your own check.**

The verification script covers about 12 items. Pick two more CIS items from the chapter and add them as `Check` lines in the script. Run again. The skill: you can extend the validation as you extend the hardening.

**4. Document any deviations.**

If you chose to skip an item (for example, because BitLocker is not available on your VM, or because removing certain Microsoft Store apps would break workflows), document why. The format: a markdown file in your hardening folder titled `deviations.md`, with one entry per skipped item:

```
## CIS 18.9.85.1 (Enable BitLocker on the OS drive)
Skipped because: VM does not have a TPM available
Compensating control: Documented for production deployment; not relevant for ephemeral VM
```

Real-world hardening always involves deviations from the standard. The discipline of documenting them is what separates "I followed a checklist" from "I made informed security decisions."

**5. Run the Chapter 11 triage queries against the hardened box.**

Run the bulk signature audit. Run the persistence checks. Run the artifact queries from Chapter 11. Confirm: the hardened box should still pass cleanly. The hardening should not have introduced any artifacts that look like compromise.

**6. Use Local Group Policy to set one item.**

Pick one CIS recommendation and set it via gpedit.msc instead of PowerShell. Suggested: Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options > "Interactive logon: Do not display last user name." Set to Enabled.

After saving, run `gpupdate /force`. Verify the change took:

```
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System' | Select-Object DontDisplayLastUserName
```

The exercise: you can set any of the chapter's items via either the registry or GPO. Production environments use GPO at scale; lab work often uses registry/PowerShell. Both are real workflows.

---

## Common stumbling blocks

> **`gpedit.msc` does not exist on Windows 11 Home.**
> Group Policy Editor is a Pro/Enterprise feature. On Home edition, you set the underlying registry values directly. The CIS benchmarks assume Enterprise or Pro; Home is not the right baseline for serious hardening.

> **`auditpol /set` returns "the system cannot find the path specified."**
> auditpol uses subcategory names that are localized. On English systems, the names in this chapter work. On a non-English Windows, they are translated. Run `auditpol /list /subcategory:*` to see the names available on your system.

> **PowerShell logging settings do not seem to take effect.**
> Two things to check. First, run `gpupdate /force` to apply policy. Second, the Microsoft-Windows-PowerShell/Operational log may need to be enlarged: `wevtutil sl Microsoft-Windows-PowerShell/Operational /ms:1073741824`. Default size is small and fills quickly with script block events.

> **Set-MpPreference returns "Operation failed with the following error: 0x800106ba."**
> Defender service may be disabled (rare; usually means another endpoint protection product is installed). Check: `Get-Service WinDefend | Format-List Status, StartType`. If a third-party AV is present, configure that product instead.

> **Enable-BitLocker fails with "the volume cannot be encrypted because it is in use."**
> Some files are locked by running services. Reboot and try again, or use the `-Used` parameter (`Enable-BitLocker -UsedSpaceOnly`) which only encrypts used space and is faster but slightly weaker.

> **My CIS verification script says PASS for things I did not configure.**
> Many CIS settings have CIS-aligned defaults on Windows 11 Pro/Enterprise. The verify script confirms current state, not whether you set it explicitly. If you want to confirm you applied the change, look at the before/after diff for that specific setting.

> **Disabling SMBv1 broke a printer/scanner/legacy device.**
> Some legacy devices only support SMBv1. Either replace the device, isolate it on a separate network with SMBv1 enabled only there, or accept the residual risk. Real production hardening involves these tradeoffs constantly.

---

## What this gets you

After this chapter:

- Your lab box is materially more resistant to compromise than it was at the start.
- You have produced a documented diff between the default and hardened states. That diff is portfolio-quality work.
- You have walked through about 20 CIS Benchmark items, enough to understand the structure and read the full benchmark on your own.
- You have a verification script that confirms the changes took. The pattern (config + verification) is what real production hardening looks like.
- You know how to use Local Group Policy as a configuration surface.
- You know about ASR rules, Controlled Folder Access, PowerShell logging, and the modern Windows hardening surface generally.
- You can talk about Windows hardening in the vocabulary the field uses (CIS, ASR, GPO, AppLocker) rather than vaguely.

The verification script is the part of this chapter that pays off the longest. Anyone can apply settings once. The discipline of writing tests for your security configuration is what separates one-off work from sustainable hardening practice. Working admins should have a verify.ps1 for every system they own.

---

## What's next

Nothing. This is the last chapter of the unit.

You came in 12 chapters ago with workshop-level Windows skills: navigate, install software, manage users, run a service, schedule a task, read the event log. You leave able to operate, audit, investigate, and harden a Windows endpoint. That is roughly the skill set of a junior sysadmin or junior security analyst on a Windows-shop tour.

Where to go from here:

**Immediate next step:** finish anything in the unit you skipped. The post-workshop chapters are designed to be re-readable. The diff you produced in this chapter is a real artifact; keep it.

**For the security path:** the next month is Active Directory, where Windows starts to feel like a real business platform rather than a single endpoint. The skills you built here all extend into AD. Then on through the broader program: networking and AI integration in Month 3, Windows Server and AD in Month 4, secure admin foundations in Month 5, then SecOps, IR, and Security+ exam prep.

**For the sysadmin path:** practice. Spin up a Windows VM at home (Hyper-V is built in and free; VirtualBox runs everywhere). Break things on it intentionally. Run a small service for yourself: a personal file share, a homelab automation script, a hobby project. The boxes you administer in your spare time are the boxes you are best on.

**For Security+:** the exam-prep block (Month 9 of the broader program) fills in the test-specific vocabulary and the small set of topics this unit did not cover (governance, risk frameworks, specific compliance details). The foundations from this unit are now solid.

You are done with the unit. Take 10 minutes to read your hardening diff once more and notice what you built. Then go do something. The reading is over.
