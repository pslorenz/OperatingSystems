# Chapter 3: Local Users, Groups, and NTFS Permissions

**You come in with:** workshop-level user creation and basic ACL setting via icacls.
**You leave with:** the ability to audit who has access to what on a Windows host you did not build, configure NTFS permissions correctly with proper inheritance, and use the principle of least privilege when designing service account access.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Strong alignment, this is a Domain 4 chapter. Domain 1.2 (fundamental security concepts: authorization models, least privilege). Domain 4.1 (hardening: account management). Domain 4.5 (operating system security including Group Policy). Domain 4.6 (access controls: discretionary access control, role-based access, least privilege). Domain 2.4 (indicators of malicious activity: account creation, privilege escalation). The "audit who has admin" pattern in this chapter is the practical version of "verify least privilege" on the cert.

---

## Why this chapter matters

On Windows, most production issues that look like "the application is broken" turn out to be permissions issues. The same is true on Linux. The Windows specifics differ enough to need their own treatment: SIDs instead of UIDs, ACLs instead of rwx, inheritance rules that propagate (or do not), the implicit Administrators group that grants more than people realize.

For security work, this chapter is foundational to Chapter 11 (artifacts) because the most common Windows persistence techniques involve abusing accounts and permissions, and to Chapter 12 (hardening) because the largest CIS benchmark category is account-related.

This chapter goes deeper than the workshop on the same topics. The workshop showed you the verbs (`New-LocalUser`, `icacls`); this chapter teaches you the nouns and the model behind them.

---

## Security identifiers (SIDs)

When you create a user, Windows assigns it a SID. *A SID, security identifier, is a variable-length number that uniquely identifies a security principal: a user, a group, a service, or a system identity.*

Every access control decision Windows makes uses SIDs, not names. The username `student` is a label for the user with SID `S-1-5-21-12345-67890-...`. If you delete `student` and create a new user also called `student`, that new user has a different SID and is, from the kernel's perspective, a different account. This is why renaming a user is cheap (the SID is unchanged) but recreating one is consequential (the SID is new).

To see your own SID:

```
whoami /user
```

Output:

```
USER INFORMATION
----------------
User Name        SID
================ ============================================
labbox\student   S-1-5-21-1234567890-1234567890-1234567890-1001
```

Reading the SID:

- `S-1-5-` is the standard prefix.
- `21-...-` is the domain or local machine identifier.
- The last segment is the relative ID (RID). 1001 is the first non-built-in user; 1002 is the second; etc.

Some SIDs are well-known and constant across all Windows installs:

| SID | What it represents |
|-----|---------------------|
| S-1-1-0 | Everyone |
| S-1-5-18 | LocalSystem (SYSTEM) |
| S-1-5-19 | LocalService |
| S-1-5-20 | NetworkService |
| S-1-5-32-544 | Built-in Administrators group |
| S-1-5-32-545 | Built-in Users group |
| S-1-5-32-546 | Built-in Guests group |

When investigating, recognizing the well-known SIDs saves time. An ACL entry for `S-1-5-18` is "SYSTEM has this access," regardless of how the GUI displays it.

To enumerate the local users with their SIDs:

```
Get-LocalUser | Select-Object Name, SID, Enabled
```

To enumerate local groups:

```
Get-LocalGroup | Select-Object Name, SID, Description
```

---

## The local Administrators group

Membership in the local Administrators group is the largest single privilege boundary on a Windows endpoint.

```
Get-LocalGroupMember -Group Administrators
```

On a default Windows 11 install, this returns:

- The built-in Administrator account (disabled by default).
- The first user created during install (your `student` account, enabled).
- Possibly `Administrator@<computername>` (legacy reference).

That is it for a fresh install. On a managed corporate machine, it might also include domain groups (Domain Admins, etc.) but you do not see those on a non-domain-joined system.

Anyone in this group is, effectively, root on this box. They can:

- Modify any file on the system (subject to the TrustedInstaller exception we discuss below).
- Install drivers (kernel-mode code).
- Add or remove users.
- Disable security features.
- Read all user data on the system.

The principle of least privilege says: most users should not be in this group. The corollary: when investigating a system, the membership of the local Administrators group is one of the first things you check.

### The Administrator vs. local admin distinction

Two terms get confused:

- **The built-in Administrator account** (RID 500) is a specific account. Disabled by default on Windows 11.
- **Local admin** is shorthand for "any account that is a member of the local Administrators group."

`student` is a local admin (member of the group) but is not "the Administrator." The built-in Administrator account is special in a few ways: it cannot be locked out, its UAC behavior differs slightly, and it has RID 500 which some tools and attacks specifically look for.

For most purposes, treat the built-in Administrator as "another local admin that should stay disabled."

---

## Inspecting current permissions: Get-Acl

The workshop touched on `Get-Acl`. Now we read its output more carefully.

```
Get-Acl 'C:\Windows' | Format-List
```

Output (abridged):

```
Path   : Microsoft.PowerShell.Core\FileSystem::C:\Windows
Owner  : NT SERVICE\TrustedInstaller
Group  : NT SERVICE\TrustedInstaller
Access : NT AUTHORITY\SYSTEM Allow  FullControl
         BUILTIN\Administrators Allow  Modify, ChangePermissions, TakeOwnership
         BUILTIN\Users Allow  ReadAndExecute, Synchronize
         CREATOR OWNER Allow  GenericAll
         APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES Allow  ReadAndExecute, Synchronize
```

Reading this:

- **Owner** is the account that "owns" the directory. Owners have implicit rights to change permissions even if not granted explicitly. `TrustedInstaller` is a special service identity that owns most OS-protected files; this is why even Administrators cannot modify some files in `C:\Windows`.
- **Access** is the actual ACL: each line is one access control entry (ACE).

Each ACE has:

- An identity (who).
- An effect: Allow or Deny.
- A set of rights.
- (Not visible in this format, but present internally) inheritance flags.

To see the access entries with more detail:

```
Get-Acl 'C:\Windows' | Select-Object -ExpandProperty Access | Format-Table IdentityReference, AccessControlType, FileSystemRights -AutoSize
```

To see the access entries with all properties:

```
Get-Acl 'C:\Windows' | Select-Object -ExpandProperty Access | Format-List
```

The `Format-List` form shows IsInherited, InheritanceFlags, and PropagationFlags for each entry. We come back to inheritance below.

---

## NTFS rights in detail

The workshop introduced common rights (FullControl, Modify, etc.). Here is the complete picture.

NTFS has 13 fundamental rights. Most of the named "rights" you see in the GUI are combinations of these. The common combinations:

| Combination | Includes |
|-------------|----------|
| Read | List directory, Read attributes, Read extended attributes, Read permissions |
| Write | Create files, Create directories, Write attributes, Write extended attributes |
| ReadAndExecute | Read + Execute file (for files) or Traverse directory (for directories) |
| Modify | ReadAndExecute + Write + Delete |
| FullControl | All rights including Change permissions and Take ownership |

For most practical work, you use Modify and FullControl (with Read and ReadAndExecute for read-only access). The fundamental rights matter when you need to grant something specific without granting more (e.g., "this user can write files but not delete them").

### The fundamental rights

For reference, the 13 are: ReadData, WriteData, AppendData, ReadExtendedAttributes, WriteExtendedAttributes, ExecuteFile, DeleteSubdirectoriesAndFiles, ReadAttributes, WriteAttributes, Delete, ReadPermissions, ChangePermissions, TakeOwnership. You will rarely set these individually; the named combinations cover almost every realistic case.

### Special rights worth understanding

A few rights are worth understanding individually because they control important actions:

- **ChangePermissions**: lets the holder modify the ACL itself. Granting this is roughly granting full control through indirection.
- **TakeOwnership**: lets the holder take ownership of the file. The new owner can then change permissions. This is how Administrators can recover access to a file they were locked out of.
- **Delete**: the right to delete the file itself. Surprisingly tricky because parent directory permissions also affect delete.

### Allow versus Deny

ACEs can be Allow or Deny. Deny entries take precedence over Allow entries when they conflict.

```
# Real ACE: webrunner is denied write
$Acl.Access | Where-Object AccessControlType -eq Deny
```

In practice, Deny entries are rare and powerful. They override even FullControl Allow entries from the same user being a member of multiple groups. Use Deny only when the rights you want to remove cannot be expressed as "do not grant in the first place."

A common (mis)use: a user is a member of the Modify group on a directory, but you want to specifically deny them writing one file inside it. You can add a Deny ACE on the file. This works but is a maintenance burden; a better pattern is usually to put the file outside the directory or grant specific Allow entries instead of group-wide Allow.

---

## Inheritance

Permissions on a parent directory propagate to child files and directories by default. *Inheritance is the mechanism by which an ACE on a directory automatically applies to items inside it.*

When you create `C:\AppData\webops\config.txt`, that file inherits the ACEs from `C:\AppData\webops`. You can see this:

```
mkdir C:\test
echo "test" | Out-File C:\test\file.txt
Get-Acl C:\test\file.txt | Select-Object -ExpandProperty Access | Format-Table IdentityReference, FileSystemRights, IsInherited
```

The `IsInherited` column is True for entries that were inherited and False for entries set explicitly.

### Disabling inheritance

You can break inheritance on a directory:

```
$Acl = Get-Acl C:\test
$Acl.SetAccessRuleProtection($true, $true)
# First $true: protect from inheritance (block parent's ACEs from applying)
# Second $true: copy the currently inherited ACEs to be explicit (so existing access is preserved)
Set-Acl C:\test $Acl
```

After this, the directory has its own copy of the previously-inherited ACEs as explicit ACEs, and changes on the parent no longer affect this directory. This is the right pattern when you want to lock down a directory differently from its surroundings.

The icacls equivalent:

```
icacls C:\test /inheritance:r           # remove inheritance, do NOT copy
icacls C:\test /inheritance:d           # disable inheritance, DO copy current ACEs
icacls C:\test /inheritance:e           # re-enable inheritance
```

The `r` versus `d` distinction matters. `r` (remove) leaves the directory with no ACEs except whatever you have set explicitly; `d` (disable) leaves the previously-inherited ACEs as explicit ACEs. Use `d` when you want to start from the current effective permissions and adjust; use `r` when you want a clean slate.

### Inheritance flags

When setting an ACE, you specify how it propagates. The two flags:

- **InheritanceFlags**: which child types inherit. `None`, `ContainerInherit` (subfolders inherit), `ObjectInherit` (files inherit), or both.
- **PropagationFlags**: how the inheritance behaves. `None` is the usual case. `InheritOnly` means the ACE only affects children, not the directory itself. `NoPropagateInherit` means the ACE inherits one level down and stops.

For most cases: `ContainerInherit, ObjectInherit` is "applies to this directory and everything inside it forever," which is what you usually want.

The PowerShell syntax for creating an inheritable ACE:

```powershell
$Rights = [System.Security.AccessControl.FileSystemRights]::Modify
$Inheritance = [System.Security.AccessControl.InheritanceFlags]::ContainerInherit, [System.Security.AccessControl.InheritanceFlags]::ObjectInherit
$Propagation = [System.Security.AccessControl.PropagationFlags]::None
$Type = [System.Security.AccessControl.AccessControlType]::Allow
$Identity = New-Object System.Security.Principal.NTAccount("webrunner")

$Rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $Identity, $Rights, $Inheritance, $Propagation, $Type)

$Acl = Get-Acl C:\AppData\webops
$Acl.AddAccessRule($Rule)
Set-Acl C:\AppData\webops $Acl
```

Yes, that is verbose. icacls is shorter for one-off changes:

```
icacls C:\AppData\webops /grant 'webrunner:(OI)(CI)M'
```

Reading the icacls notation:

- `(OI)` is ObjectInherit: files inside inherit.
- `(CI)` is ContainerInherit: subfolders inherit.
- `M` is Modify (one of the named combinations).

For scripts and one-off work, icacls is faster. For complex permission setups or when integrating with other PowerShell logic, the verbose Set-Acl form is better.

---

## A practical permission setup

Combining the concepts. Build a directory structure where:

- `C:\AppData\webops` is the root, accessible to admins and to the webops group.
- Inside it, `private` subdirectory is accessible only to webrunner and admins.
- Inside it, `public` subdirectory is readable by all authenticated users.

```powershell
# Create the structure
New-Item -ItemType Directory -Path C:\AppData\webops -Force | Out-Null
New-Item -ItemType Directory -Path C:\AppData\webops\private -Force | Out-Null
New-Item -ItemType Directory -Path C:\AppData\webops\public -Force | Out-Null

# Ensure the webops group exists
if (-not (Get-LocalGroup -Name "webops" -ErrorAction SilentlyContinue)) {
    New-LocalGroup -Name "webops" -Description "Web operations group"
}

# Top-level: remove inheritance, set explicit ACL
icacls 'C:\AppData\webops' /inheritance:r
icacls 'C:\AppData\webops' /grant 'Administrators:(OI)(CI)F'
icacls 'C:\AppData\webops' /grant 'webops:(OI)(CI)RX'
icacls 'C:\AppData\webops' /grant 'SYSTEM:(OI)(CI)F'

# Private subdir: tighter
icacls 'C:\AppData\webops\private' /inheritance:r
icacls 'C:\AppData\webops\private' /grant 'Administrators:(OI)(CI)F'
icacls 'C:\AppData\webops\private' /grant 'webrunner:(OI)(CI)M'
icacls 'C:\AppData\webops\private' /grant 'SYSTEM:(OI)(CI)F'

# Public subdir: read for authenticated
icacls 'C:\AppData\webops\public' /inheritance:r
icacls 'C:\AppData\webops\public' /grant 'Administrators:(OI)(CI)F'
icacls 'C:\AppData\webops\public' /grant 'webops:(OI)(CI)M'
icacls 'C:\AppData\webops\public' /grant '*S-1-5-11:(OI)(CI)RX'      # Authenticated Users
icacls 'C:\AppData\webops\public' /grant 'SYSTEM:(OI)(CI)F'

# Confirm the result
icacls 'C:\AppData\webops' /T
```

Walk through what happened:

- We removed inheritance on each directory so the permissions we set are not affected by parent changes.
- We granted Administrators FullControl and SYSTEM FullControl on every level (always include SYSTEM; many system services need it).
- The webops group gets ReadAndExecute on the parent (so they can traverse) and Modify on `public` (so they can write there).
- webrunner specifically gets Modify on `private` (so they can write there but other webops members cannot).
- We used the SID `*S-1-5-11` to reference Authenticated Users without depending on the localized name.

The `/T` flag on the final icacls walks the tree, showing the permissions on every item. Read it to confirm the structure is what you intended.

This setup is the realistic version of "set up a service account's working directory." It is verbose because permissions are detailed; it is correct because the principle of least privilege is followed (each identity has only what it needs).

---

## Auditing: who has what on this box

The practitioner skill the chapter builds toward. When you walk into a Windows box, what is the access landscape?

### Who is a local admin

```
Get-LocalGroupMember -Group Administrators
```

This is the first thing you check. Anything you do not expect is a finding.

### Who has access to a specific directory

```
icacls 'C:\Users\student' | Where-Object { $_ -notmatch '^Successfully' }
```

For each entry, evaluate: should this account or group have this access? Inherited entries that look reasonable are usually fine; explicit entries on user data directories warrant explanation.

### Find world-writable items in places they should not be

```
Get-Acl 'C:\Program Files' -ErrorAction SilentlyContinue |
    Where-Object { $_.Access | Where-Object { $_.IdentityReference -eq 'Everyone' -and $_.AccessControlType -eq 'Allow' } }
```

The Everyone group with Allow rights in `C:\Program Files` is a finding. Most legitimate software does not need it.

A more thorough scan walks a directory tree:

```powershell
Get-ChildItem 'C:\Program Files' -Recurse -ErrorAction SilentlyContinue |
    ForEach-Object {
        $acl = Get-Acl $_.FullName -ErrorAction SilentlyContinue
        if ($acl.Access | Where-Object { $_.IdentityReference -eq 'Everyone' -and $_.AccessControlType -eq 'Allow' -and ($_.FileSystemRights -match 'Write|Modify|FullControl') }) {
            $_.FullName
        }
    }
```

That can take a while on large directory trees. For real audit work, this kind of query is what compliance tools automate. Knowing the principle is what matters.

### Find non-default users

```
Get-LocalUser | Where-Object SID -notmatch 'S-1-5-21-.*-(500|501|503|504)$'
```

That excludes the well-known built-in accounts (RIDs 500-504 are Administrator, Guest, DefaultAccount, WDAGUtilityAccount). What is left is custom users. On a fresh Windows 11, you should see one user (yours). Anything else warrants explanation.

### Find non-default group members of Administrators

```
$expected = @('Administrator', 'student')   # known good list
Get-LocalGroupMember -Group Administrators |
    Where-Object Name -notin $expected
```

Adjust the expected list for your environment. Anything in the result is unexpected admin access.

These four queries are the start of a permission audit. They do not catch sophisticated abuse but they catch the common cases. The discipline of running them on every box you investigate is what builds intuition over time.

---

## Try this

**1. Read the SID for every local account.**

```
Get-LocalUser | Select-Object Name, SID, Enabled
Get-LocalGroup | Select-Object Name, SID
```

For each user, identify the RID (last segment). Confirm which built-in accounts are present (RID 500 Administrator, RID 501 Guest, etc.) and which are user-created.

**2. Audit the local Administrators group.**

Run `Get-LocalGroupMember -Group Administrators`. For each member, identify whether it is:

- A built-in account.
- A user you created.
- Something else.

If "something else" appears, that is a finding to investigate.

**3. Build the practical permission setup.**

Follow the example in the chapter to create `C:\AppData\webops` with the public/private structure. Then test:

- Can you (as `student`, who is a local admin) write to all three directories? (Yes, FullControl.)
- Can `webrunner` (the service account from the workshop) write to `private`? (Yes, Modify.)
- Could a user not in any relevant group access `private`? (No, only admins, SYSTEM, and webrunner.)

To test as a different user without logging out, use:

```
Start-Process powershell -Credential webrunner
```

You will be prompted for webrunner's password. The new PowerShell window runs as webrunner.

**4. Audit your own profile directory.**

Run:

```
Get-Acl $env:USERPROFILE | Select-Object -ExpandProperty Access | Format-Table -AutoSize
```

Read it. For each entry, identify:

- The identity.
- Whether it is an Allow or Deny.
- The rights granted.
- Whether it is inherited.

A typical Windows user profile has ACEs for: the user themselves (FullControl), Administrators (FullControl), SYSTEM (FullControl). Anything else warrants attention.

**5. Find the implicit risks.**

Run this query on your lab box:

```
Get-ChildItem 'C:\' -Recurse -Depth 2 -ErrorAction SilentlyContinue |
    ForEach-Object {
        try {
            $acl = Get-Acl $_.FullName -ErrorAction Stop
            $bad = $acl.Access | Where-Object { $_.IdentityReference -eq 'Everyone' -and $_.AccessControlType -eq 'Allow' -and $_.FileSystemRights -match 'Write|Modify|FullControl' }
            if ($bad) { 
                [PSCustomObject]@{Path=$_.FullName; Access=$bad.FileSystemRights -join ','}
            }
        } catch {}
    }
```

That walks the top two levels of `C:\` looking for Everyone-writable items. On a clean Windows 11 install, this typically returns nothing (or only known-public locations). On a less clean install, the results are informative.

---

## Common stumbling blocks

> **icacls says "successfully processed" but the permissions did not actually change.**
> Check the inheritance state first. If a directory inherits permissions and you grant something at a higher level, the lower level may show it as inherited but you cannot remove it from the lower level without breaking inheritance. Use `icacls <path> /inheritance:d` to convert inherited entries to explicit ones first, then modify.

> **I added a user to a group but Get-LocalGroupMember does not show it.**
> Group membership is sometimes cached in user sessions. The change is real but does not appear in already-running processes for that user. The user needs to log out and back in (or open a new PowerShell window) for the change to take effect.

> **`Get-Acl` returns "no information at this time" for some files.**
> The current user does not have ReadPermissions on the file. Even Administrators sometimes hit this on TrustedInstaller-owned files. Take ownership first if you must, but be careful: changing ownership of OS files breaks them.

> **An ACE I set has no effect because of inheritance from a parent.**
> Allow ACEs from parents combine with explicit Allow ACEs at the child. Deny ACEs from parents (or anywhere) override conflicting Allow. If your child-level Allow does nothing, look for an inherited Deny somewhere up the tree, or for a missing prerequisite right (e.g., you granted Write on the file but the parent does not let the user traverse).

> **`whoami /groups` shows me as a member of groups I did not join.**
> Some groups are special: "Authenticated Users" (S-1-5-11), "Everyone" (S-1-1-0), "INTERACTIVE" (S-1-5-4). These are computed at logon based on how you logged in, not assigned via group membership. They appear in the output for everyone.

> **icacls notation `(OI)(CI)F` does not propagate to existing files.**
> The notation specifies how new files inherit. Existing files at the time of the change do not get retroactively updated unless you also run `icacls <dir> /reset /T` to reset and re-apply, or use the `/grant` with `:r` to replace inheritance on existing items.

---

## What this gets you

After this chapter:

- You understand SIDs as the actual identity Windows uses, with username as a label.
- You can read a Get-Acl output and explain every entry.
- You can recognize the well-known SIDs (SYSTEM, LocalService, Administrators, Everyone, Authenticated Users).
- You can set NTFS permissions correctly with appropriate inheritance using either icacls or Set-Acl.
- You can audit who has admin rights on a box and find unexpected entries.
- You can audit who has access to a specific directory and recognize unusual ACEs.
- You can build a layered permission setup (root + private + public) using least privilege.
- You know the difference between Allow and Deny and when each is appropriate.

The "build a layered permission setup" exercise is the part of this chapter that pays off most. Real production setups have these patterns. Recognizing them and being able to construct them yourself is the differentiator between "I can run icacls" and "I can design permissions."

---

## What's next

Chapter 4 is Software installation and updates. The chapter where winget becomes your default tool, where you learn what MSI installers actually do, how Windows Update works and how to control it, and what to do about software that is not packaged. Includes the Microsoft Store-versus-winget tradeoff and WSUS at recognition depth.
