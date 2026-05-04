# Ch00: Active Directory Foundations Workshop

**Audience:** You finished M1 (Linux), M2 (Windows), and M3 (Networking). You can drive a Linux box and a Windows box, you can read a packet capture, and you understand how networks are segmented. What you have not done yet is operate the authentication and authorization plane that ties hosts together in a real environment.

**You come in with:** working command-line fluency on Linux and Windows, networking foundations, a verification habit for AI-assisted work.

**You leave with:** the ability to navigate an existing Active Directory environment with thousands of objects. Hands-on experience reading users, groups, OUs, and GPOs. A working understanding of what AD is and what it does. The PowerShell ActiveDirectory module as your primary tool.

**Time:** 5 hours. Three blocks of about 75 minutes plus opening and wrap.

**Security+ alignment:** Domain 4.5 (LDAP, SSO, Kerberos, group policy concepts). Domain 4.6 (identity and access management). Domain 4.1 (hardening targets: domain controllers). Heavy alignment, particularly for the operational AD content. Per the program's stance: we teach the work, and surface where it ties back to the exam.

---

## You're inheriting an environment

This workshop is different from M1, M2, and M3 in one important way. In those units, you started with default-installed systems and built skills on top. In this unit, you start with an environment that already exists.

That's not a curriculum convenience. It's how almost every junior practitioner first encounters Active Directory. AD environments live for years, sometimes decades. The directory you walk into on day one of your first IT job has been shaped by every admin who came before you, with users from people who left five years ago, groups created for projects nobody remembers, GPOs from three reorganizations ago, and OU structures that made sense to whoever was running things at the time.

Your job as a junior is to operate that environment. Reset passwords. Add users for new hires. Disable accounts when people leave. Apply GPO changes that someone above your pay grade approved. Read what's there and figure out what it means.

That's what this unit teaches. You won't install AD from scratch in this workshop. (The instructor will demo what install looks like so you've seen it, but you won't run the wizard.) Instead, you'll spend five hours on the directory you'll actually operate: a populated AD environment with thousands of users, groups, computers, OUs, and GPOs. Realistic mess. Real-shaped data.

The unit that comes after, M5, takes the same environment and applies security thinking: tier model, privileged access workstations, delegation, audit. The intermediate cohort eventually covers the architecture and design work that the previous admin who built this environment did. That's senior work, properly placed later.

Today: you're a junior on day one. You've been given the keys to an AD environment. Let's see what's there.

---

## What we're doing today

Three blocks. Each block has teaching content followed by hands-on lab work.

**Block 1: Orientation (~75 minutes).** What AD is, conceptually. What's in the environment you've inherited. Demo of what installing AD looks like (instructor demo, not student work). Hands-on exploration of the AD environment: walking the OU tree, finding the default users and groups, identifying what's customized vs default.

**Block 2: Managing users, groups, and OUs (~75 minutes).** The day-to-day work of an AD admin. Creating users, modifying group memberships, moving objects between OUs. PowerShell as the primary tool, ADUC for context. AI integration with the verification habit applied to PowerShell queries.

**Block 3: Group Policy basics (~75 minutes).** What GPOs are, how they apply, reading the policies in your environment, modifying one policy and verifying it took effect.

We open with a 15-minute orientation. We close with a 30-minute wrap. Two 15-minute breaks between blocks.

---

## Your lab environment

Today's labs run on a CourseStack-hosted environment with three components:

- **Domain controller (NW-DC01):** Windows Server 2025 with Active Directory Domain Services installed. The forest root domain is `corp.acme.local`. The directory is pre-populated with about 2,500 users, 500 groups, 100 organizational units, and a smattering of GPOs. This is what your inherited environment looks like.

- **Domain-joined Windows 11 workstation:** a workstation already joined to the `corp.acme.local` domain. You'll do most of your work from here, using RSAT (Remote Server Administration Tools) to manage AD without RDPing to the DC.

- **Non-joined Windows 11 workstation:** the same client you used in M2 and M3, still there, still not joined. This box stays around because some tools (security auditors like PingCastle, plus M3 networking work) make sense from a non-joined position. You'll see this client come back in Ch09 and Ch10.

You connect the same way you have all along: through CourseStack's lab interface in your browser. Your instructor will provide the specific lab URL.

### A note on RSAT

RSAT is the package that lets you manage Server features (including AD) from a workstation. It's not installed by default on Windows 11 but it's available as an optional feature. On the domain-joined client, RSAT is already installed; you'll have ADUC, GPMC, DNS Manager, and the ActiveDirectory PowerShell module available without extra setup.

This matters because **working AD admins don't RDP into the DC to manage users**. They manage from their workstation, with RSAT installed. RDPing to the DC every time you need to add a user is a security anti-pattern (more on that in M5) and operationally clumsy. From day one, you do AD management from your workstation.

---

## A note on AI tools

The verification habit from M3 carries forward. AI tools are useful for explaining unfamiliar PowerShell output, drafting first-pass queries, translating between Linux and Windows commands. They are bad at your specific environment, recently changed product details, and confidently wrong claims about specific log message meanings.

For AD work specifically, the verification matters because AD has many object types, many attribute names, and many conventions that look interchangeable but aren't. An LLM might give you `Get-ADUser -Filter "displayname -eq 'Alice'"` when the correct attribute is `DisplayName` (case sensitivity matters in some PowerShell versions and contexts) or might suggest `Get-ADUser -Identity` when you actually need `-Filter`. These are subtle wrongnesses that fail in production.

We'll see this concretely in Block 2 when we use AI to help write queries against the directory. The verification step is what makes AI useful instead of misleading.

---

## Block 1: Orientation to AD

### Block 1 learning objectives

By the end of this block, you can:

- Explain what Active Directory is, in your own words.
- Describe the relationship between a forest, a domain, and an OU.
- Use ADUC to navigate the OU tree and read object properties.
- Use PowerShell ActiveDirectory module to query basic information.
- Identify the difference between built-in/default objects and customized objects.

### What Active Directory is

Active Directory is several things at once:

**A directory service.** A specialized database optimized for read-heavy workloads, holding information about users, computers, groups, and configuration. The protocol it speaks is LDAP (Lightweight Directory Access Protocol).

**An authentication service.** The directory holds credentials and the AD server validates them. The protocol it speaks for authentication is Kerberos (with NTLM as legacy fallback).

**A central management plane.** Group Policy Objects (GPOs) stored in AD are applied to domain-joined machines automatically, providing a single place to manage configuration across many systems.

**A trust boundary.** Machines and users that are part of an AD domain are part of the same trust zone; the domain's authority is recognized across all of them.

In a small business or mid-sized enterprise environment, AD is the answer to the question: "How do we manage this many users and computers without it becoming chaos?" Without AD, every workstation has its own local users (the M2 model). With AD, users and groups are defined once and used everywhere.

### Forests, domains, and OUs

A few terms that get used interchangeably in casual speech but mean specific things:

**Forest:** the top-level boundary of an AD installation. A forest contains one or more domains. The forest is the security boundary; objects in different forests don't trust each other unless explicit forest trusts are configured.

**Domain:** a partition of the forest. Has its own database, its own admins, its own policies. A domain controller (DC) is a server that hosts a copy of the domain database and answers authentication requests.

**Organizational Unit (OU):** a container inside a domain for organizing objects. OUs are not security boundaries; they're administrative boundaries. OUs are where you apply GPOs and delegate management.

For this lab:

- The forest is `acme.local`.
- The domain is `corp.acme.local` (a child of the forest root, in this lab the only domain).
- The OUs inside `corp.acme.local` were generated by BadBlood and look like a real-world environment with department-shaped OU structure.

### A quick demo of what install looks like

Your instructor will walk through what `Install-ADDSForest` does, with a live demo or recorded video. Watch this once. Notice:

- The prerequisites (static IP, DNS configuration, hostname).
- The wizard prompts (forest functional level, DSRM password, DNS settings).
- What gets created (the domain, the SYSVOL share, the NTDS database, the default users, default groups, default OUs).
- The promotion happening: the server going from a regular Windows Server to a DC.

You don't run this yourself. The DC in your lab is already promoted. The point of the demo is so you've seen what install looks like; M5 and intermediate cohort go deeper on production-grade installs.

### Block 1 lab: Explore the environment

Log into the domain-joined workstation. Open ADUC: Start menu, search for "Active Directory Users and Computers."

You should see `corp.acme.local` in the left pane with a tree of OUs and containers underneath.

**Exercise 1.** Walk the OU tree. Click each top-level OU and look at what's inside. Note the rough structure: what departments are represented, how deep does the OU tree go, what kinds of objects live where.

**Exercise 2.** Open the "Users" container (the built-in one at the top, not an OU). What accounts are there? Identify Administrator, Guest, and krbtgt. Note that this is a Container (different icon from an OU); built-in containers can't have GPOs applied to them.

**Exercise 3.** Open PowerShell on your workstation. Run:

```powershell
# Verify the AD module is available
Get-Module -ListAvailable ActiveDirectory

# Connect to the domain
Get-ADDomain

# Count users
(Get-ADUser -Filter *).Count

# Count groups
(Get-ADGroup -Filter *).Count

# Count computers
(Get-ADComputer -Filter *).Count

# Count OUs
(Get-ADOrganizationalUnit -Filter *).Count
```

Note the numbers. This is the scale of the environment you're working with. Compare to M1 and M2 where the user count was usually under 10. The scale is what makes querying important; you can't manage thousands of objects by clicking through ADUC.

**Exercise 4.** Look at one user in detail. Pick any user from one of the populated OUs. In PowerShell:

```powershell
Get-ADUser -Identity <username> -Properties *
```

Read the output. There are dozens of attributes. Note `DistinguishedName`, `UserPrincipalName`, `SamAccountName`, `Enabled`, `LockedOut`, `LastLogonDate`. These are the attributes you'll work with most.

### Block 1 debrief

The most common stumbling block in Block 1 is the OU vs Container distinction. Containers (capital C, like the built-in "Users") look like OUs in the tree but behave differently. You can't apply GPOs to Containers. Most user-creation operations end up in Containers by default, which is one of the reasons OU design matters (you usually want users in OUs, not Containers).

The second is reading the tree as if it were the org chart. OU structure should reflect administrative boundaries, not org structure. A 50-person company doesn't need 50 OUs. The BadBlood-generated environment may have OU structure that looks org-shaped because that's a common (if not always best) practice.

The third is overestimating what Distinguished Names tell you. The DN is just the path to the object in the directory; it's how the directory identifies the object internally. For most management work you'll use the SamAccountName or UPN, not the DN.

---

*Break: 15 minutes.*

---

## Block 2: Managing users, groups, and OUs

### Block 2 learning objectives

By the end of this block, you can:

- Create a user account with `New-ADUser`.
- Add and remove users from groups.
- Move objects between OUs.
- Read group memberships and identify nested groups.
- Use AD PowerShell filters to find specific subsets of objects.
- Apply the verification habit to AI-suggested AD operations.

### The everyday admin work

If you take an AD admin job, the bulk of your daily work is:

- New hire? Create a user account, add to appropriate groups, place in the right OU, assign initial password.
- Role change? Update group memberships, possibly move OU.
- Termination? Disable the account, remove from groups (or leave for retention purposes; depends on policy).
- Lockout? Unlock the account, possibly reset password.
- Password reset? Reset password.
- "I can't access X." Check group memberships, check whether the user is in the right place in the OU tree, check whether GPO scoping covers them.

This is the maintenance loop. It's not glamorous. It's most of the job.

### Creating users with PowerShell

The basic user creation:

```powershell
$securePassword = Read-Host -AsSecureString -Prompt "Initial password"

New-ADUser `
    -Name "Test User" `
    -GivenName "Test" `
    -Surname "User" `
    -SamAccountName "tuser" `
    -UserPrincipalName "tuser@corp.acme.local" `
    -Path "OU=Users,OU=IT,DC=corp,DC=acme,DC=local" `
    -AccountPassword $securePassword `
    -ChangePasswordAtLogon $true `
    -Enabled $true
```

Reading this:

- `-Name` is the display name (what shows in ADUC and most GUIs).
- `-GivenName` and `-Surname` populate the structured fields.
- `-SamAccountName` is the legacy NetBIOS-style logon name. Limited to 20 characters. This is what users typically type in classic logon dialogs.
- `-UserPrincipalName` is the modern logon name. Looks like an email address.
- `-Path` is the OU where the user goes. Specified as a Distinguished Name. **Get this right; the user goes wherever you tell it.**
- `-AccountPassword` is the initial password as a SecureString.
- `-ChangePasswordAtLogon` forces the user to set a real password at first logon.
- `-Enabled $true` is necessary or the account won't work.

Run this carefully. You can't easily undo it; you have to delete the account and recreate it. Working admins reach for this pattern often enough that they typically have a script or a function that wraps it.

### Adding to groups

```powershell
# Add a user to a group
Add-ADGroupMember -Identity "FinanceUsers" -Members "tuser"

# Add multiple users at once
Add-ADGroupMember -Identity "FinanceUsers" -Members "tuser","alice","bob"

# Verify
Get-ADGroupMember -Identity "FinanceUsers"
```

### Reading nested groups

This is where AD gets interesting. Groups can contain users, computers, and other groups. The "effective" membership requires walking the chain.

```powershell
# Direct members of a group
Get-ADGroupMember -Identity "FinanceUsers"

# Recursive: walk nested groups too
Get-ADGroupMember -Identity "FinanceUsers" -Recursive

# All groups a user is a member of (including nested)
Get-ADUser -Identity "tuser" -Properties MemberOf |
    Select-Object -ExpandProperty MemberOf
```

In a real environment, the `-Recursive` flag matters. Users get added to a group like "FinanceManagers" which is a member of "FinanceUsers" which is a member of "AllStaff." Without recursive enumeration, you don't see the actual effective membership.

### Block 2 lab: Hands-on user and group work

**Exercise 1.** Create a test user. Use `New-ADUser` to create a user named with your name (or initials) plus "test", placed in the IT OU (or wherever the BadBlood structure put one). Set a temporary password. Set ChangePasswordAtLogon. Enable the account. Verify by querying the user with `Get-ADUser -Identity <yourname>test`.

**Exercise 2.** Find an existing group with at least 5 members. Add your test user. Verify the membership took:

```powershell
Get-ADGroupMember -Identity "<groupname>" -Recursive
```

Make sure your test user appears in the output.

**Exercise 3.** Find a user with a deep group nesting. Use the recursive query to enumerate all groups they're effectively in:

```powershell
$user = "<some-username>"
$directGroups = Get-ADUser $user -Properties MemberOf |
    Select-Object -ExpandProperty MemberOf
Write-Host "Direct groups for $user :"
$directGroups | ForEach-Object { (Get-ADGroup $_).Name }
```

Then expand recursively. (You'll need a script that walks the chain; ask AI for a starting point and apply the verification habit.)

**Exercise 4 (the AI exercise).** Ask an LLM to write you a PowerShell one-liner that finds all users whose accounts are enabled and who haven't logged on in the last 90 days. The natural query uses `LastLogonDate` and `Enabled`.

Take the LLM's answer. Don't run it yet. Walk through:

1. Does the syntax look right? Read it carefully.
2. Does `LastLogonDate` exist as an attribute? Verify with `Get-ADUser -Identity <someuser> -Properties *` and look for it.
3. What's the exact filter syntax? AD PowerShell uses an unusual filter language; AI sometimes generates filter syntax that almost-works.

Then run it. Compare the output to your expectations. Note what the LLM got right and wrong.

### Block 2 debrief

The most common stumbling block is the AD filter syntax. PowerShell's normal filter (`Where-Object`) works but is slow because it pulls all results before filtering. The native AD filter (`-Filter`) is faster but uses different syntax: `'Enabled -eq $true'` (string with embedded `-eq`) rather than `{ $_.Enabled -eq $true }` (script block). AI confuses these consistently.

The second is forgetting that `LastLogonDate` is replicated only every 14 days, while `lastLogon` is per-DC and not replicated. For finding stale accounts, `LastLogonDate` is good enough. For precise per-DC last logon, you query each DC. Don't worry about this distinction today; just know that "last logon" has nuance.

The third is the OU vs Container issue from Block 1, biting again. If you don't specify `-Path` when creating a user, it defaults to the Users Container, where you can't apply GPOs. Always specify `-Path`.

---

*Break: 15 minutes.*

---

## Block 3: Group Policy basics

### Block 3 learning objectives

By the end of this block, you can:

- Open GPMC and navigate to the GPOs in your environment.
- Read what an existing GPO does (settings, scope, links).
- Apply a simple GPO modification.
- Verify the GPO took effect on a target machine.
- Read `gpresult` output to understand what applied to a client and why.

### What Group Policy is

Group Policy is the mechanism that lets you apply configuration centrally to domain-joined machines. A GPO (Group Policy Object) is a collection of settings stored in AD; when a machine or user has the GPO applied to them, the settings take effect.

GPOs apply by **link**: a GPO is linked to a site, domain, or OU. Machines and users in the linked location get the GPO. The order of application is LSDOU:

1. **L**ocal policy (on the machine itself, configured via `gpedit.msc` for non-domain or as part of domain processing).
2. **S**ite policy (linked to AD sites).
3. **D**omain policy (linked to the domain root).
4. **OU policy (deepest first).**

Later policies override earlier ones for the same setting. So domain policies override site policies, OU policies override domain, and a deeper-nested OU policy overrides shallower OU policies. Block Inheritance and Enforced flags can change this; we won't cover them today.

### GPMC: where you read GPOs

Group Policy Management Console (GPMC) is the GUI tool. On your domain-joined workstation: `gpmc.msc` from Run, or find it in the Start menu (might be under Administrative Tools).

The top-level structure:

- **Forest:** at the top.
- **Domains:** below the forest. Click the domain to expand.
- **The domain itself:** linked GPOs at the domain level.
- **Sites:** site-linked GPOs (rare in small environments).
- **OUs:** linked GPOs at OU level.
- **Group Policy Objects:** the master list of all GPOs in the domain (not necessarily linked).

When you expand a domain or OU, you see the GPOs linked at that level. When you click on the "Group Policy Objects" container, you see every GPO that exists, whether linked or not. Some GPOs exist but are unlinked (orphan policies that nobody got around to deleting; this is common in real environments).

### Reading a GPO's settings

In GPMC, click a linked GPO. The right pane has tabs:

- **Scope:** where this GPO applies (links, security filtering, WMI filters).
- **Details:** modification dates, GUIDs, status.
- **Settings:** an HTML report of every setting configured. This is what the GPO actually does.
- **Delegation:** who can manage this GPO.

The Settings tab is the important one for "what does this GPO do." It generates a readable report of every configured setting in Computer Configuration and User Configuration. Read through it.

For the lab environment, the BadBlood-populated domain has some GPOs already. Some are default (like Default Domain Policy and Default Domain Controllers Policy); some are custom. Click through a few and see what they configure.

### Block 3 lab: Read and modify a GPO

**Exercise 1.** Open GPMC. Find the Default Domain Policy. Click the Settings tab. Read what it configures. Most of what you see will be Computer Configuration / Account Policies (password policy, account lockout policy, Kerberos policy). These are the policies applied to the domain root by default.

**Exercise 2.** Find a custom GPO (one created by BadBlood or by the lab setup). Read its settings, scope, and delegation. Identify:

- Where is it linked?
- What does it configure?
- Who can edit it?

**Exercise 3.** Create a new GPO. In GPMC, expand the domain, find an OU you can experiment in. Right-click the OU > Create a GPO in this domain, and Link it here.

Name it something memorable like "Lab Test - Desktop Wallpaper" (replace "Lab" with your initials so multiple students can do this).

Edit the GPO: right-click > Edit. Navigate to User Configuration > Policies > Administrative Templates > Desktop > Desktop > Desktop Wallpaper. Enable the policy and specify a path (any UNC path or local path; you don't need it to actually exist for the lab purposes).

**Exercise 4.** Verify the GPO scope. Click the GPO in GPMC. Scope tab. Make sure it's linked where you expect, and that the security filtering covers the users you'd expect.

**Exercise 5.** Force a refresh on a target client. From the Windows 11 client (the domain-joined one), open PowerShell and run:

```powershell
gpupdate /force
```

This forces a Group Policy refresh. After it completes:

```powershell
gpresult /h C:\gpresult.html
```

This generates a report of what applied to the user. Open `C:\gpresult.html` in a browser. Find your test GPO in the list of applied GPOs. Confirm it shows up.

If your test GPO is in the report, the link and scope worked. If it's not, check why:

- Is the user actually in the OU where the GPO is linked?
- Does the GPO have security filtering that excludes this user?
- Is the GPO disabled (Computer or User Configuration disabled)?

### Block 3 debrief

The most common stumbling block is the GPO replication delay. After you create a GPO, it takes a few seconds to replicate from the DC to the client's view. If `gpupdate /force` runs immediately after GPO creation, sometimes the client doesn't see the new GPO yet. Wait 30 seconds and try again.

The second is the user vs computer scope. User Configuration applies to users when they log on (the user's Authenticated Users group needs the GPO applied). Computer Configuration applies to computers at boot. If you put a user-config policy on a computer-only OU, it won't apply.

The third is security filtering. By default, GPOs apply to "Authenticated Users" which means everyone. If someone has changed the security filtering to apply only to specific groups, you have to be a member of those groups to see the policy.

---

## Wrap

### What you can now do

- Navigate an existing AD environment with thousands of objects.
- Read users, groups, computers, and OUs in PowerShell and ADUC.
- Create, modify, and remove user accounts.
- Manage group memberships, including reading nested groups recursively.
- Read and write basic GPOs.
- Understand the LSDOU policy application order.
- Verify policy application with `gpupdate` and `gpresult`.
- Apply the verification habit to AI-suggested AD operations.

That's a meaningful skill expansion in five hours. The post-workshop unit goes deeper.

### What's deferred to the post-workshop unit

Ten chapters covering:

- **Ch01:** Why Active Directory exists. The conceptual frame in detail.
- **Ch02:** The AD object model. Users, groups, computers, OUs as objects with attributes.
- **Ch03:** Touring an existing environment. Deep walk of the BadBlood-populated `corp.acme.local`.
- **Ch04:** Users, groups, computers, OUs at depth. Lifecycle management.
- **Ch05:** Group Policy from the ground up. Beyond what we touched today.
- **Ch06:** Authentication. Kerberos at practitioner depth, NTLM legacy, reading auth event logs.
- **Ch07:** Querying the directory. PowerShell ActiveDirectory module deep dive.
- **Ch08:** Trusts, sites, and replication at recognition depth.
- **Ch09:** The AD-aware practitioner. Investigation patterns, finding abandoned accounts and suspicious group memberships.
- **Ch10:** AD hardening capstone. Audit and remediate this environment, with verification script.

### What's next in the program

- **M5:** Secure administration. Tier model, PAW, delegation, audit. Builds on this same environment.
- **M6:** Cloud and identity (Azure / Entra ID). RADIUS, 802.1X, EAP variants. Federation.
- **M7-M10:** SecOps, IR, Sec+ prep, capstone.

You're a third of the way through the program. The skills compound from here.

---

## Common stumbling blocks across the workshop

> **The directory has too many users; I can't find anything.**
> Welcome to real AD. Use PowerShell filters. `Get-ADUser -Filter "Department -eq 'Finance'"` is faster than scrolling. The volume is the point of this lab; if you could find things by clicking, the skill wouldn't transfer.

> **My PowerShell command syntax errors out.**
> AD PowerShell has its own filter syntax. The `-Filter` parameter takes a string with quoted operators (`-eq`, `-like`, `-and`). `Where-Object` after the fact takes a script block. Don't mix them.

> **My GPO isn't applying.**
> Run `gpresult /h <file>.html` and read what applied. The report shows every GPO considered and why it did or didn't apply. If your test GPO isn't there at all, the link or security filtering is wrong.

> **I'm not sure what's "default" vs what BadBlood added.**
> Default users (Administrator, Guest, krbtgt, DefaultAccount) live in the Users container. Default groups (Domain Admins, Enterprise Admins, etc.) also in Users container. Default OUs are Domain Controllers. Almost everything else (the populated department OUs, the user accounts in them, the security groups) is BadBlood. As the chapters progress we'll get more precise; for today the default vs custom distinction is recognition-only.

> **I created a user but they can't log on.**
> Three common causes: password complexity rejection (the password didn't meet domain policy requirements), account creation succeeded but Enable was missed, or account was created but ChangePasswordAtLogon is set and you didn't actually set the password. Verify by `Get-ADUser -Identity <name> -Properties Enabled, PasswordExpired, LockedOut`.

> **gpresult shows GPOs I don't recognize.**
> Default Domain Policy and Default Domain Controllers Policy are domain defaults; they always show. Other GPOs may be linked at the domain root or apply through OU inheritance. Read the report carefully; it tells you where each GPO came from.

---

## What's next

For the next session, read Ch01 (Why Active Directory exists). It deepens the conceptual frame from this morning and sets up the rest of the unit.

If you want to get ahead, the post-workshop unit reads in order. Ch01-Ch04 are foundational; Ch05-Ch08 are operational; Ch09 is the AD-aware practitioner chapter; Ch10 is the capstone.

Skill, not talent. The people who get good at AD are the people who keep practicing on real-shaped environments.
