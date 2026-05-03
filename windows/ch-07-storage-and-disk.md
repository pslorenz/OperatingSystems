# Chapter 7: Storage and the Disk Subsystem

**You come in with:** the workshop's brief tour of the filesystem and Chapter 1's directory map. You can navigate. You have not seriously thought about disks, partitions, or the storage layer underneath.
**You leave with:** the ability to read the disk and partition layout on a Windows host, answer "where did the disk space go" with a repeatable diagnostic pattern, and recognize what BitLocker and Volume Shadow Copy do at the level a working sysadmin needs.

**Time:** 50 to 75 minutes including the exercises.

**Security+ alignment:** Domain 3.3 (resilience and recovery: backups, recovery sites, restoration). Domain 4.1 (hardening: disk encryption, secure baselines). Domain 4.4 (monitoring: capacity monitoring as part of alerting). Domain 4.9 (data sources to support an investigation: file metadata, Volume Shadow Copy as a forensic source). The "where did the disk go" diagnostic pattern is the operational version of capacity-management monitoring.

---

## Why this chapter matters

Disk problems are the most reliably recurring source of production outages, on every platform. On Windows specifically, the symptoms tend to look different than on Linux: WinSxS bloat, shadow copy storage growing without bounds, hibernation files, page files, the Recycle Bin holding gigabytes of "deleted" files. The diagnostic pattern is the same shape as Linux ("which filesystem, which directory, what is in there"), but the specific places to look are Windows-shaped.

The other reason this chapter exists: BitLocker and Volume Shadow Copy are real Windows features that show up on Security+ and in real production environments. Knowing them at recognition depth means you understand what is happening when someone says "the laptop is encrypted with BitLocker" or "we recovered the file from a shadow copy."

This chapter is shorter than Chapter 6, with less depth and more practical focus. The skills it builds are diagnostic, not configuration-heavy.

---

## Disks, volumes, and partitions

Linux had drive letters as a non-thing; everything mounted into a single tree. Windows has drive letters as the primary user-facing concept and a more complicated relationship between physical disks, partitions, and what shows up as `C:`.

The hierarchy:

- **A disk** is a physical (or virtual) block device. On the lab box, you have one.
- **A partition** is a slice of a disk. A disk can have one partition or several.
- **A volume** is something with a filesystem mounted on it. Usually a partition, but can be a span across partitions or a logical volume.
- **A drive letter** is a label assigned to a volume. `C:` is conventionally the system volume.

The PowerShell cmdlets for each layer:

```
Get-Disk
Get-Partition
Get-Volume
```

Try them:

```powershell
Get-Disk | Format-Table Number, FriendlyName, Size, OperationalStatus, PartitionStyle
```

Output on a typical Windows 11 lab VM:

```
Number FriendlyName             Size OperationalStatus PartitionStyle
------ ------------             ---- ----------------- --------------
     0 Virtual Disk           64 GB Online            GPT
```

`PartitionStyle` is either `MBR` (Master Boot Record, legacy) or `GPT` (GUID Partition Table, modern). Modern Windows 11 installs use GPT.

```powershell
Get-Partition | Format-Table DiskNumber, PartitionNumber, DriveLetter, Size, Type
```

Output:

```
DiskNumber PartitionNumber DriveLetter   Size Type
---------- --------------- -----------   ---- ----
         0               1             100 MB System
         0               2              16 MB Reserved
         0               3 C            63 GB Basic
         0               4             574 MB Recovery
```

Reading this:

- Partition 1 (System): the EFI System Partition. Holds the bootloader.
- Partition 2 (Reserved): Microsoft Reserved partition, used for storing OS metadata.
- Partition 3 (Basic, drive letter C): the main Windows install.
- Partition 4 (Recovery): the Windows Recovery Environment partition.

```powershell
Get-Volume | Format-Table DriveLetter, FileSystem, FileSystemLabel, SizeRemaining, Size, HealthStatus
```

The Get-Volume view is what most working admins use day-to-day. It shows just the volumes (the things with filesystems), with their drive letters, types, and sizes.

### File systems on Windows

Windows volumes are typically formatted with NTFS. *NTFS, the New Technology File System, is Microsoft's primary file system since Windows NT. It supports access control lists, encryption, compression, journaling, and other features the older FAT family does not.* Removable media is sometimes FAT32 or exFAT for cross-platform compatibility.

To see filesystem details:

```
Get-Volume C | Select-Object FileSystem, FileSystemLabel, AllocationUnitSize, SizeRemaining, Size
```

For most workstations, the answer is "NTFS, allocation unit 4096, with X bytes free of Y." NTFS handles everything you need on a system volume.

ReFS (Resilient File System) is Microsoft's newer filesystem aimed at server workloads (large file servers, virtualization hosts). You will not see it on a workstation.

---

## "Where did the disk go": the diagnostic pattern

The most common operational question on every platform. The Windows version of the pattern.

### Step 1: Confirm the symptom

```
Get-Volume C
```

If `SizeRemaining` is small relative to `Size`, you have a disk-fill problem. The number tells you how desperate the situation is.

### Step 2: Find the heaviest top-level directories

```powershell
Get-ChildItem C:\ -Directory -Force -ErrorAction SilentlyContinue |
    ForEach-Object {
        $size = (Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue |
                 Measure-Object -Property Length -Sum).Sum
        [PSCustomObject]@{
            Path = $_.FullName
            SizeGB = [math]::Round($size / 1GB, 2)
        }
    } | Sort-Object SizeGB -Descending | Select-Object -First 10
```

That is verbose. Save it as a function in your `$PROFILE`. The output is one line per top-level directory under `C:\`, sorted largest to smallest.

A faster form for scripts:

```powershell
$dirs = Get-ChildItem C:\ -Directory -Force -ErrorAction SilentlyContinue
foreach ($d in $dirs) {
    $size = (Get-ChildItem $d.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    "{0,15:N2} GB  {1}" -f ($size / 1GB), $d.FullName
}
```

The format string `{0,15:N2} GB  {1}` produces a fixed-width 15-character right-aligned number with two decimals, then the path. Tidier output than the PSCustomObject form for terminal viewing.

On a typical Windows 11 box, the heavy directories are:

- **`C:\Windows`**: usually 30 to 50 GB. Most of that is in `WinSxS`.
- **`C:\Program Files`** and **`C:\Program Files (x86)`**: depends on what is installed.
- **`C:\Users`**: per-user data, Downloads folder is often the surprise.
- **`C:\ProgramData`**: shared application state. Defender's signatures and update caches live here.
- **`C:\$Recycle.Bin`**: the Recycle Bin. Sometimes surprisingly large.
- **`C:\hiberfil.sys`** (top-level file): hibernation file, sized to RAM.
- **`C:\pagefile.sys`** (top-level file): page file, default size is system-managed.

### Step 3: Drill into the heaviest directory

If `C:\Windows` is the heaviest, drill in:

```powershell
Get-ChildItem C:\Windows -Directory | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    "{0,12:N2} GB  {1}" -f ($size / 1GB), $_.Name
} | Sort-Object
```

The usual suspects under `C:\Windows`:

- **`WinSxS`**: the side-by-side store. Often 10 GB or more on a long-running install.
- **`Installer`**: cached MSI files. Can grow large.
- **`SoftwareDistribution`**: Windows Update download cache.
- **`Logs`**: various OS logs.
- **`Temp`**: system-wide temp.

### Step 4: Take action, with caution

Same rule as the Linux unit: do not delete what you cannot identify.

The safe targets, with the right way to address each:

**WinSxS bloat:**

```
DISM.exe /Online /Cleanup-Image /AnalyzeComponentStore
DISM.exe /Online /Cleanup-Image /StartComponentCleanup
DISM.exe /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

The `Analyze` step reports how much can be cleaned up. `StartComponentCleanup` does a normal cleanup. `ResetBase` is more aggressive: it removes the ability to uninstall some Windows updates in exchange for more space. Use `ResetBase` only on systems where you are confident you do not need to roll back updates.

**SoftwareDistribution cache:**

```powershell
Stop-Service wuauserv, bits -Force
Remove-Item C:\Windows\SoftwareDistribution\Download\* -Recurse -Force
Start-Service wuauserv, bits
```

That clears the Windows Update download cache. Safe to do; Windows Update will redownload anything it needs.

**Recycle Bin:**

```powershell
Clear-RecycleBin -Force
```

That empties the Recycle Bin. Self-explanatory; do not do this if you might need to recover something.

**User profile cleanup:**

The Downloads folder is the most reliable surprise. Each user's `~\Downloads` accumulates files until manually cleaned. `Get-ChildItem $env:USERPROFILE\Downloads -Recurse -File | Measure-Object Length -Sum | Select-Object -ExpandProperty Sum` tells you the size for the current user.

**Hibernation file:**

If the box does not need hibernation:

```
powercfg /hibernate off
```

That removes `hiberfil.sys` and reclaims its space (typically equal to RAM size). If you reactivate hibernation later, the file comes back.

**Don't touch these without thought:**

- `C:\System Volume Information` (System Restore data, Volume Shadow Copies).
- `C:\Users\<user>\AppData\Local\<application>` (application state).
- `pagefile.sys` (let the OS manage it).

### A practical one-liner

For a quick "what is heavy under C:\":

```powershell
Get-ChildItem C:\ -Directory -Force -ErrorAction SilentlyContinue |
    Sort-Object {(Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum} -Descending |
    Select-Object -First 5 Name,
        @{N='SizeGB'; E={[math]::Round((Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB, 2)}}
```

That is the kind of thing you save in your profile as a function. Call it `Get-DiskHogs` or similar.

---

## NTFS features worth knowing about

NTFS has a set of features beyond basic file storage. Most of them are invisible until they matter.

### Compression

NTFS supports per-file or per-folder compression. To enable on a directory:

```
compact /c /s C:\Logs
```

That walks `C:\Logs` and compresses every file. Useful for log archives and rarely-accessed data. Compressed files appear normal to applications; the compression and decompression happen transparently.

To check if a file is compressed:

```powershell
Get-ItemProperty C:\Logs\old.log | Select-Object Attributes
```

If `Compressed` is among the attributes, the file is compressed.

NTFS compression is convenient but not always a win. For active log files (frequently appended), the compression overhead can be noticeable. For archived data, compression is essentially free.

### Alternate Data Streams

NTFS files can have multiple "streams" of data attached. *An alternate data stream is a named secondary stream of bytes attached to a file. The default content of the file is the unnamed stream; alternate streams are addressed with `filename:streamname` syntax.*

This is mostly invisible to users but matters in security work because attackers sometimes hide payloads in alternate streams.

To list streams on a file:

```
Get-Item C:\path\to\file.txt -Stream *
```

Most files have only `:$DATA` (the default content). Files downloaded from the internet often have a `Zone.Identifier` stream that records where they came from (this is what powers the "this file came from the internet" warning).

```
Get-Item C:\Users\student\Downloads\* -Stream Zone.Identifier -ErrorAction SilentlyContinue |
    Select-Object FileName, Stream
```

Reading the Zone.Identifier stream tells you whether files have the "downloaded from internet" mark. We come back to alternate streams in Chapter 11 with the artifact angle.

### Junctions and symbolic links

NTFS supports several kinds of links:

- **Hard links**: multiple directory entries pointing at the same file. Like Linux hard links.
- **Junctions**: directory pointers within a volume. The Windows analog of `mount --bind`. Used internally by Windows for things like `C:\Users\All Users -> C:\ProgramData`.
- **Symbolic links**: like Linux symlinks. Can cross volumes and point at files or directories.

To create them:

```
mklink /H link.txt original.txt              # hard link
mklink /J junction-name target-dir           # junction
mklink /D symlink-name target-dir            # directory symlink
mklink symlink-name target-file              # file symlink
```

Symbolic link creation requires admin privileges by default (this can be relaxed via Local Security Policy or Developer Mode).

To detect links in a directory listing:

```
Get-ChildItem C:\Users | Where-Object LinkType
```

The `LinkType` property is non-null for items that are links. The `Target` property tells you where they point.

For most working admins, knowing junctions exist and what they look like is enough. Working with them daily is not common except in specific scenarios (some application install patterns, container image work, advanced scripting).

---

## The Recycle Bin

When a user deletes a file from File Explorer, it goes to the Recycle Bin. The Recycle Bin is per-volume: each volume has a `$Recycle.Bin` directory at the root, and inside it a folder for each user (named by SID).

```
Get-ChildItem 'C:\$Recycle.Bin' -Force -ErrorAction SilentlyContinue
```

You see one directory per user that has used the Recycle Bin on this volume. Each user's directory contains the deleted files (renamed) plus metadata about original locations.

The `$Recycle.Bin` is typically not user-readable for other users' content; only an admin can read another user's Recycle Bin.

To empty the current user's Recycle Bin:

```
Clear-RecycleBin -Force
```

To check the size:

```powershell
Get-ChildItem 'C:\$Recycle.Bin' -Recurse -Force -File -ErrorAction SilentlyContinue |
    Measure-Object Length -Sum |
    Select-Object @{N='SizeGB'; E={[math]::Round($_.Sum / 1GB, 2)}}
```

A surprisingly common cause of "the disk is full." Files dragged to the Recycle Bin still take space until the bin is emptied, and many users never empty it.

For forensics, the Recycle Bin is interesting: it is a record of what was deleted, when, and (sometimes) by whom. We do not go deep on this in the course; intermediate cohort covers it.

---

## Volume Shadow Copy

Volume Shadow Copy is Windows's snapshot mechanism. *VSS, the Volume Shadow Copy Service, creates point-in-time copies of files and entire volumes. It is used by System Restore, Windows Backup, and many third-party backup tools.*

VSS snapshots are read-only point-in-time views of a volume. They are stored efficiently (only changes are tracked, not full copies) and can be enumerated with `vssadmin`:

```
vssadmin list shadows
```

If shadows exist, the output shows their creation time, the volume they snapshot, and the storage they use. On a fresh Windows 11 lab box, this often returns "No items found that satisfy the query," because System Restore is off by default in some configurations.

To list the storage allocation:

```
vssadmin list shadowstorage
```

That shows which volumes have shadow storage allocated and how much.

### Why VSS matters

For working admins:

- It is what lets users right-click a folder and choose "Restore previous versions."
- It is how most Windows backup tools take consistent snapshots of running systems.
- It can fill up a disk if not managed; the default storage allocation grows over time.

For security analysts:

- Shadow copies are a forensic data source. Files that an attacker deleted from the live filesystem might still exist in shadow copies.
- Some ransomware specifically targets VSS to delete shadow copies before encrypting files. The command `vssadmin delete shadows` is on most ransomware behavior watchlists.

To create a shadow copy manually:

```
vssadmin create shadow /for=C:
```

(That requires admin and specific Windows editions; on Windows 11 Home, vssadmin's create command is restricted.)

For this chapter, recognition is enough. VSS exists, snapshots are how Windows does point-in-time recovery, deleting shadow copies is a recognizable attacker action.

---

## BitLocker

BitLocker is full-disk encryption built into Windows. *BitLocker is Microsoft's volume-level encryption feature that protects data at rest by encrypting the entire volume, with keys typically stored in the TPM (Trusted Platform Module) and unlocked at boot.*

To check BitLocker status:

```
Get-BitLockerVolume
manage-bde -status C:
```

On a typical lab VM, BitLocker is not enabled. On a corporate-managed laptop, it usually is.

The relevant fields:

- **VolumeStatus**: `FullyEncrypted`, `FullyDecrypted`, `EncryptionInProgress`, etc.
- **EncryptionMethod**: typically `XtsAes128` or `XtsAes256` on modern systems.
- **ProtectionStatus**: `On` (key sealed in TPM, drive auto-unlocks at boot) or `Off` (key not protected; encryption is not adding security).
- **KeyProtectors**: how the encryption key is secured. TPM, recovery password, smartcard, password.

### How BitLocker actually works

The volume's data is encrypted with a Volume Master Key (VMK). The VMK is itself encrypted with one or more Key Protectors. Common protectors:

- **TPM**: the key is sealed to the TPM and released only if the boot chain is intact.
- **TPM + PIN**: requires a PIN at boot in addition to TPM unlock.
- **Recovery password**: a 48-digit numerical password, used as fallback if TPM unlock fails.
- **External key**: a key file on a USB drive.

The recovery password is the critical piece for operations work. *The BitLocker recovery password is a 48-digit numerical key that can decrypt the drive if the primary unlock method fails.* If the TPM fails, the OS misboots, or the drive is moved to a new machine, the recovery password is what gets the data back.

In a managed environment, recovery passwords are typically escrowed: stored in Active Directory, Azure AD, Microsoft Intune, or an MDM. Without escrow, losing the password means losing the data.

### BitLocker for unmanaged users

For a user who turned on BitLocker themselves (Windows 11 Pro), the recovery password is saved either to a Microsoft account, printed, or saved to a file. If they did none of those, the recovery password is gone, and a TPM event that requires recovery becomes a data-loss event.

For this chapter: recognition. BitLocker exists, it encrypts the whole volume, the recovery password matters, the TPM is the typical protector. Chapter 12 covers enabling and configuring it.

---

## Page file and hibernation

Two large files at the root of `C:\` worth knowing about.

### pagefile.sys

The page file is virtual memory backed by disk. *The page file is a file on disk that Windows uses as virtual memory: when physical RAM is full, less-used pages are written to the page file to free RAM.*

```
Get-CimInstance Win32_PageFileSetting
```

The default behavior is "automatically managed by the system." Windows sizes the page file based on RAM and usage patterns. For most modern systems with adequate RAM, the page file rarely sees heavy use.

You can check current usage:

```
Get-Counter '\Paging File(_Total)\% Usage'
```

Persistent high page file usage suggests the system is RAM-constrained. The fix is more RAM, not a larger page file.

### hiberfil.sys

The hibernation file holds the contents of RAM when the system hibernates, so it can be restored on next boot.

```
Get-Item C:\hiberfil.sys -Force -ErrorAction SilentlyContinue | Select-Object Name, Length, Attributes
```

If the file exists, hibernation is enabled. Default size is roughly 40% of RAM.

For desktops and lab VMs that do not need hibernation:

```
powercfg /hibernate off
```

That deletes the file and reclaims the space. To re-enable: `powercfg /hibernate on`.

For laptops that benefit from hibernation, leave it alone. The space cost is the price of being able to fully suspend.

---

## Try this

**1. Tour your storage layout.**

Run all four core queries:

```
Get-Disk
Get-Partition
Get-Volume
Get-PhysicalDisk
```

For your single-disk lab VM, identify the disk, its size, the partitions on it, and the volumes formatted on those partitions. Confirm `C:` is on partition number 3 (or whatever it is) and is NTFS.

**2. Find the disk hogs.**

Run the "where is the heavy stuff" query for `C:\`. Identify the top 5 directories by size. For each one, hypothesize what is in it, then verify by drilling in one level.

The exercise: build a mental model of "where does Windows spend disk." On a clean install, you should be able to account for most of the disk usage with this exercise. On a system that has accumulated cruft, you find the cruft.

**3. Run the WinSxS analysis.**

```
DISM.exe /Online /Cleanup-Image /AnalyzeComponentStore
```

The output reports the size of WinSxS, how much could be cleaned up, and a recommendation on whether cleanup is suggested. Read it. (Do not run the actual cleanup unless the recommendation is to clean up; on a relatively new install, there is not much to gain.)

**4. Inspect alternate data streams on a downloaded file.**

If you downloaded any files in Block 2's winget install exercises, find one in `C:\Users\<you>\Downloads` (or wherever winget cached it) that came from the internet. Run:

```
Get-Item <path> -Stream *
```

If you see a `Zone.Identifier` stream, read it:

```
Get-Content <path> -Stream Zone.Identifier
```

The content is a small INI-format text noting the file's origin. This is the artifact behind "this file came from the internet" warnings.

**5. Check the Recycle Bin and BitLocker status.**

```
Get-ChildItem 'C:\$Recycle.Bin' -Recurse -Force -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum

Get-BitLockerVolume -ErrorAction SilentlyContinue
```

Note the Recycle Bin size and the BitLocker status. On a fresh lab VM, both are typically empty/disabled. On a long-lived managed system, both have content; the contrast itself is informative.

---

## Common stumbling blocks

> **`Get-ChildItem` walks of large directories take forever.**
> Three things help. Add `-File` so directories themselves are not enumerated as items. Add `-ErrorAction SilentlyContinue` to skip permission errors quickly. Use `Measure-Object` to compute the size in one pass rather than retrieving and processing each item.

> **`Get-Volume` shows `HealthStatus: Unknown`.**
> Some virtual disks (especially in nested virtualization or unusual storage stacks) report Unknown rather than Healthy. Cross-check with `Get-Disk` and `chkdsk /scan C:` (read-only scan). On a fresh lab VM it is usually fine; persistent Unknown on a real machine warrants investigation.

> **DISM /StartComponentCleanup runs forever.**
> WinSxS cleanup is genuinely slow on systems with many cumulative updates. It is not stuck; it is processing. Let it run. On a busy server, schedule cleanup during a maintenance window.

> **`Clear-RecycleBin` does not free space.**
> The Recycle Bin's storage is per-user per-volume. `Clear-RecycleBin` clears the current user's bin on the system volume by default. To clear all volumes for the current user: `Clear-RecycleBin -DriveLetter C, D -Force`. To clear another user's bin, you need to be admin and target their SID-named directory under `$Recycle.Bin`.

> **BitLocker reports `ProtectionStatus: Off` even though the volume shows encrypted.**
> The volume is encrypted (data is unrecoverable without keys), but the Volume Master Key is stored in the clear, so the protection is not "active." This happens when BitLocker is suspended (e.g., before a firmware update). Resume with `Resume-BitLocker -MountPoint C:` to seal the key back into the TPM.

> **`compact /c` ran but the directory is the same size.**
> Some files cannot be compressed in place (open files, files with active alternate streams, certain system files). NTFS compression is a heuristic; results vary. The compression is real on the files that compressed; failed files remain uncompressed. Read the compact output for per-file results.

> **The page file is still growing even though I have plenty of RAM.**
> Windows uses the page file even when RAM is plentiful, for crash-dump support and certain memory management features. This is normal. If you specifically do not need crash dumps, you can shrink or eliminate the page file with `Get-CimInstance Win32_ComputerSystem | Set-CimInstance -Property @{AutomaticManagedPagefile=$false}` plus appropriate page file settings. Most systems should leave it managed.

---

## What this gets you

After this chapter:

- You can read the disk, partition, and volume layout on a Windows host.
- You can answer "where did the disk space go" with a repeatable diagnostic pattern.
- You know the usual suspects: WinSxS, SoftwareDistribution, hiberfil.sys, the Recycle Bin, user Downloads folders.
- You can clean up safely with DISM, Clear-RecycleBin, and powercfg without breaking anything.
- You know what NTFS compression, alternate data streams, and junctions are at recognition depth.
- You know what Volume Shadow Copy is and why "ransomware deletes shadow copies" is a behavior signature.
- You know what BitLocker is, what protectors are, and why the recovery password matters.

The diagnostic pattern (Get-Volume to confirm, recursive directory size to drill, careful cleanup) is the part of this chapter that pays off most. You will use it in production. The BitLocker and VSS recognition pays off in interviews and in security work later.

---

## What's next

Chapter 8 is Host networking on Windows. The chapter where Get-NetAdapter, Test-NetConnection, and Resolve-DnsName become the daily-use tools, where Windows Defender Firewall configuration is approached with PowerShell, and where the four-step network diagnosis pattern from Linux gets translated to Windows commands.
