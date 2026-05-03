# Chapter 3: Permissions Deep Dive

**You come in with:** workshop-level chmod and chown. You can read `-rw-r--r--` and roughly translate it to "owner can write, others can read."
**You leave with:** the ability to audit permissions on a system you did not build, knowledge of when to reach for ACLs versus standard permissions, and sudoers configured the way a real admin configures it.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** First chapter with real exam content. Domain 1.2 (fundamental security concepts: authorization models, least privilege). Domain 2.4 (indicators of malicious activity: privilege escalation, missing logs related to sudo). Domain 4.1 (hardening techniques: removal of unnecessary software, default password changes, host-based firewall foundations). Domain 4.5 (operating system security including SELinux and Group Policy parallels). Domain 4.6 (access controls: mandatory, discretionary, role-based, rule-based, least privilege). The 777 antipattern conversation in this chapter is the practical version of "least privilege" on the cert.

---

## Why this chapter matters

Most production Linux issues that look like "the application is broken" turn out to be permissions issues. A web server cannot read its own config. A backup script cannot write to its target directory. A scheduled job runs as the wrong user and silently produces output nobody can find.

Beyond reliability, permissions are the foundation of security on Linux. The principle of least privilege, which Security+ tests and which every real security framework requires, is enforced through file permissions, group membership, and sudoers. If you cannot read a `ls -l` output and explain why each character is there, you cannot audit a system someone else built.

This is also the chapter where the 777 antipattern stops being acceptable. The workshop introduced it. This chapter teaches you the patterns to use instead.

---

## The standard model in depth

Every file on a Linux system has three pieces of metadata that govern who can do what:

- An **owner**: a single user.
- A **group**: a single group.
- A **mode**: a set of read, write, and execute bits for the owner, the group, and everyone else.

The workshop covered the basics. Now we go deeper.

### Reading the long output

```
ls -l /etc/ssh/sshd_config
```

Output:

```
-rw-r--r-- 1 root root 3265 Jan 17 09:14 /etc/ssh/sshd_config
```

The first ten characters break down as follows:

| Position | Character | Meaning |
|---|---|---|
| 1 | `-` | File type. `-` regular file, `d` directory, `l` symlink, `c` character device, `b` block device, `s` socket, `p` named pipe |
| 2 to 4 | `rw-` | Owner permissions: read yes, write yes, execute no |
| 5 to 7 | `r--` | Group permissions: read yes, write no, execute no |
| 8 to 10 | `r--` | World permissions: read yes, write no, execute no |

After the mode, the columns are: link count, owner, group, size in bytes, modification time, name. Most beginners ignore the link count. It is the number of hardlinks pointing at this file. For regular files it is almost always 1.

### Symbolic versus numeric chmod

There are two ways to write permissions. Both work. Most admins end up using numeric (octal) for daily work because it is shorter, but symbolic is more readable when you want to make a single change.

**Numeric (octal):**

Each of the three groups (owner, group, world) is one digit. Each digit is the sum of:
- 4 for read
- 2 for write
- 1 for execute

So `chmod 644` means owner gets 6 (read+write), group gets 4 (read), world gets 4 (read).

Common modes:

| Mode | Means | Use for |
|---|---|---|
| 644 | rw-r--r-- | Standard config files, web pages |
| 600 | rw------- | Private files, SSH keys |
| 640 | rw-r----- | Config files with secrets, group can read |
| 755 | rwxr-xr-x | Executables and directories |
| 700 | rwx------ | Private directories like `~/.ssh` |
| 750 | rwxr-x--- | Group-shared directories with restricted access |
| 444 | r--r--r-- | Read-only, locked-down references |

Memorize 644, 600, 755, and 700. The rest follow the pattern.

**Symbolic:**

`chmod u+x file` means "user (owner) plus execute." The actors are `u` (user/owner), `g` (group), `o` (other/world), `a` (all). The operations are `+` (add), `-` (remove), `=` (set to exactly). The permissions are `r`, `w`, `x`.

Examples:

```
chmod +x script.sh                # add execute for everyone
chmod u+x,go-w file               # owner gets execute, group and other lose write
chmod g=r file                    # group gets exactly read, nothing else
chmod o-rwx file                  # remove all permissions for world
```

Use symbolic when you want to change one thing without touching the rest. Use numeric when you want to set the whole mode at once.

### Recursive changes

Both `chmod` and `chown` take `-R` for recursive. *Recursive means the operation applies to a directory and everything inside it, all the way down.*

```
sudo chown -R webrunner:webops /var/www/html/
sudo chmod -R 640 /var/www/html/
```

The second command is a problem. It sets mode 640 on every file *and* every directory, which means directories get 640 too. Directories need execute permission to be entered. After that command, nobody can `cd` into the html directory, including nginx.

The fix is to handle files and directories separately:

```
sudo find /var/www/html -type f -exec chmod 640 {} \;
sudo find /var/www/html -type d -exec chmod 750 {} \;
```

Or use the symbolic capital-X, which adds execute only where it makes sense:

```
sudo chmod -R u=rwX,g=rX,o= /var/www/html/
```

Capital `X` means "execute, but only for directories or files that already have at least one execute bit." That is the trick that makes recursive permission setting work without breaking directories.

This single distinction (lowercase `x` versus capital `X`) is one of the most useful things in this chapter. Most permission disasters in scripts come from a recursive `chmod -R 644` that broke every directory.

---

## The execute bit on directories

The execute bit means different things on files and directories. Beginners often miss this.

**On a file**, execute means "this can be run as a program." Without it, the file cannot be invoked even if it is readable.

**On a directory**, execute means "you can enter this directory and access its contents." Read on a directory means "you can list what is in it" (`ls`), and execute means "you can use the names of files inside it." A directory with read but not execute lets you run `ls` and see filenames, but every operation on those files (`cat`, `cd`, anything) fails. A directory with execute but not read lets you `cat` a file inside if you already know its name, but `ls` returns "permission denied."

Set up a quick demo on your lab box:

```
mkdir -p /tmp/perm_test
echo "secret" > /tmp/perm_test/file.txt
chmod 644 /tmp/perm_test/file.txt
chmod 644 /tmp/perm_test       # read but not execute
```

Then try:

```
ls /tmp/perm_test               # works, you can see the filename
cat /tmp/perm_test/file.txt     # fails, no execute on the directory
```

Now flip it:

```
chmod 711 /tmp/perm_test        # execute but not read
```

Try:

```
ls /tmp/perm_test               # fails, no read on the directory
cat /tmp/perm_test/file.txt     # works if you know the name
```

This is the gotcha that produces the "but the file permissions look right, why does this not work" question. The answer is almost always a directory in the path that lacks execute.

---

## umask: the permission default

When you create a new file, what mode does it get?

```
touch /tmp/newfile
ls -l /tmp/newfile
```

You probably see `-rw-r--r--` (mode 644). When you create a directory, you get 755. These defaults come from `umask`. *Umask is a value the shell subtracts from a maximum mode to determine the mode of new files.* 

Run:

```
umask
```

You see something like `0022`. The first digit is for special bits (see below). The next three are subtracted from 666 for files and 777 for directories. So with umask 022:

- New files: 666 minus 022 = 644.
- New directories: 777 minus 022 = 755.

If you want new files to be private to you by default:

```
umask 077
touch /tmp/private
ls -l /tmp/private
```

You see `-rw-------` (mode 600). New directories get 700.

The umask is set per-shell. To make it permanent for your shell, put it in `~/.bashrc`. Service accounts and system processes have their own umask, often set in `/etc/login.defs` or in the service unit file.

This matters in security contexts. A misconfigured umask of 000 means every new file is world-writable by default, which is a serious finding. Auditing the umask of service accounts is one of the things hardening checklists check.

---

## The special bits

Beyond rwx, there are three special permission bits. Each one solves a specific problem.

### Setuid (4xxx)

When you run a setuid binary, the process runs as the owner of the file, not as you. The classic example:

```
ls -l /usr/bin/passwd
```

You see something like:

```
-rwsr-xr-x 1 root root 68208 Mar 23 17:10 /usr/bin/passwd
```

Notice the `s` where the owner's `x` would normally be. That is setuid. When you run `passwd` as a regular user, the process runs as root, because root owns the binary and setuid is set. That is how a non-root user can change their password, which requires writing to `/etc/shadow` (a root-owned file).

Setuid is necessary for programs like `passwd`, `sudo`, and `su`. It is also a privilege escalation primitive that attackers love. Any setuid binary is a potential root path. Auditing setuid binaries is a basic forensic step:

```
find / -perm -4000 -type f 2>/dev/null
```

That command finds every setuid binary on the system. On a default Ubuntu 24.04 install, you should see a small set of well-known binaries: `passwd`, `chsh`, `chfn`, `gpasswd`, `mount`, `umount`, `su`, `sudo`, `pkexec`, `newgrp`, plus a few others. Anything beyond that warrants explanation. *A setuid binary in a user's home directory or in `/tmp` is one of the most reliable indicators that a system has been compromised.*

To set or remove setuid:

```
sudo chmod u+s /path/to/file
sudo chmod u-s /path/to/file
sudo chmod 4755 /path/to/file
```

The leading 4 is the setuid bit in numeric mode.

### Setgid (2xxx)

On a file, setgid behaves like setuid but for the group: the process runs with the file's group, not the user's group. Less common than setuid.

On a *directory*, setgid does something different and very useful. Files created inside a setgid directory inherit the directory's group, regardless of the creating user's primary group. This is the standard way to set up a shared directory where multiple users collaborate.

```
sudo mkdir /srv/shared
sudo chown root:webops /srv/shared
sudo chmod 2775 /srv/shared        # the leading 2 is setgid
```

Now any file created in `/srv/shared` by any member of webops gets the webops group automatically. No more "Steven created the file but it has Steven's primary group, not the shared one."

To find setgid binaries (the file version, which is the security-relevant one):

```
find / -perm -2000 -type f 2>/dev/null
```

### Sticky bit (1xxx)

On a directory, the sticky bit means "only the owner of a file can delete it, even if others have write permission to the directory." This is what makes `/tmp` work safely for everyone.

```
ls -ld /tmp
```

You see `drwxrwxrwt`. The `t` at the end is the sticky bit. Mode 1777. Anyone can write to `/tmp` (create files), but they can only delete their own files.

Without the sticky bit, anyone with write access to `/tmp` could delete anyone else's files there. The sticky bit fixes this for shared writable directories.

You will rarely set the sticky bit yourself. You will see it on `/tmp` and `/var/tmp`. Recognizing the `t` in a long listing matters for audits.

---

## ACLs: when standard permissions are not enough

The standard model has three actors: owner, group, everyone. What if you need to give one specific user access to a file without putting them in the group? Or give two different groups different levels of access?

That is what ACLs are for. *An Access Control List, or ACL, is an additional permission layer that lets you grant specific permissions to specific users or groups beyond the standard owner-group-other model.* ACLs are part of the filesystem and have been on Linux for two decades, but most beginners never learn them.

### Reading ACLs

```
getfacl /etc/passwd
```

Output:

```
# file: etc/passwd
# owner: root
# group: root
user::rw-
group::r--
other::r--
```

That is a file with no extra ACL entries; the output just confirms the standard mode. Now create one:

```
sudo touch /tmp/aclfile
sudo setfacl -m u:webrunner:rw /tmp/aclfile
getfacl /tmp/aclfile
```

You now see:

```
# file: tmp/aclfile
# owner: root
# group: root
user::rw-
user:webrunner:rw-
group::r--
mask::rw-
other::r--
```

The line `user:webrunner:rw-` is the ACL entry. webrunner can now read and write the file, even though it is not the owner and not in the file's group.

### When to use ACLs

The standard model handles most cases. Use ACLs when:

- One specific user needs access that does not match the file's owner or group.
- A directory needs different permissions for two different groups.
- You are integrating with a Windows or NFS environment that uses ACLs natively.

Avoid ACLs when:

- The standard model would work. ACLs add complexity that anyone reviewing the system has to understand.
- You can solve it by adjusting group membership instead. Adding webrunner to the file's group is usually cleaner than adding an ACL entry.

A file with active ACLs shows a `+` at the end of the mode in `ls -l`:

```
$ ls -l /tmp/aclfile
-rw-rw-r--+ 1 root root 0 May 03 12:34 /tmp/aclfile
```

That `+` is the visual cue that "the actual permissions are more nuanced than the mode shows; run `getfacl` to see the full picture." Auditors look for this.

### Removing ACLs

```
sudo setfacl -x u:webrunner /tmp/aclfile      # remove one entry
sudo setfacl -b /tmp/aclfile                  # remove all ACL entries
```

---

## sudo done correctly

Sudo is the most common privilege escalation mechanism on Linux. Configuring it wrong is the most common privilege escalation vulnerability on Linux. Both things are true.

### What sudo actually does

`sudo command` runs `command` as another user, by default root, after checking that the invoking user is allowed to do so. The check happens against `/etc/sudoers` and the files in `/etc/sudoers.d/`. The action is logged.

Run:

```
sudo -l
```

That shows you what sudo allows your user to do. On a default Ubuntu install with you in the `sudo` group, you see something like:

```
User student may run the following commands on labbox:
    (ALL : ALL) ALL
```

Translation: as user `student`, you can run any command, as any user, on any host. The `sudo` group has a wildcard rule.

### Editing sudoers correctly

Never edit `/etc/sudoers` directly with vi or nano. *Always use `visudo`, which validates the syntax before saving and prevents you from breaking sudo with a typo.* If you break sudoers, you lose your ability to fix it, because fixing it requires sudo. The system can become unrecoverable without booting from a recovery image.

```
sudo visudo
```

That opens `/etc/sudoers` in your default editor, but with a syntax-checking save step. If you make a mistake, visudo refuses to save and gives you the option to fix it.

For new rules, do not edit the main file. Add a drop-in file:

```
sudo visudo -f /etc/sudoers.d/myrule
```

Drop-in files are read in alphabetical order after the main sudoers. They are easier to manage with configuration management tools and easier to remove cleanly.

### Writing a sudoers rule

The basic syntax of a sudoers line:

```
who    where = (as_whom)  what
```

Examples:

```
student    ALL = (ALL) ALL                  # student can do anything
deploy     ALL = (root) /usr/bin/systemctl restart nginx
backup     ALL = (root) NOPASSWD: /usr/local/bin/backup.sh
%webops    ALL = (root) /usr/bin/journalctl -u nginx
```

Breaking those down:

- `student ALL = (ALL) ALL` means user student, on any host, as any user, can run any command. This is the default-admin rule.
- `deploy ALL = (root) /usr/bin/systemctl restart nginx` means the deploy user can run exactly `systemctl restart nginx` as root. Nothing else.
- `backup ALL = (root) NOPASSWD: /usr/local/bin/backup.sh` means the backup user can run that one script as root, without entering a password. NOPASSWD is necessary for scripted automation but should be tightly scoped.
- `%webops ALL = (root) /usr/bin/journalctl -u nginx` means anyone in the webops group can run journalctl filtered to nginx as root. The `%` prefix means "this is a group, not a user."

The principle: grant the smallest possible action to the smallest possible identity. Avoid `ALL=(ALL) ALL` rules for service accounts. Avoid NOPASSWD wildcards. The setup that matches a security audit is "deploy can run exactly these three commands, with passwords, logged."

### Common sudoers anti-patterns to avoid

These are real patterns that show up in audits as findings:

- `service_user ALL=(ALL) NOPASSWD: ALL`: gives a service account full root with no auth. The same as making the service account root.
- `user ALL=(ALL) NOPASSWD: /bin/bash`: explicitly allows running a shell as root. Identical to the above, just sneakier.
- `user ALL=(ALL) NOPASSWD: /usr/bin/vi`: vi can shell out via `:!command`, so this allows arbitrary command execution as root. Same problem with editors generally, with `find -exec`, with anything that can run subprocesses.

The pattern: any command that can spawn another command should not be granted via sudoers without careful thought.

### Logging

By default, sudo logs to `/var/log/auth.log` and to the journal:

```
sudo journalctl -u sudo
sudo grep sudo /var/log/auth.log | tail
```

Every sudo invocation is logged with who ran it, what was run, and whether it succeeded. This is one of the first things a security investigator looks at. We come back to this in Chapter 6 (Logs and auditing) and Chapter 11 (Linux artifacts).

---

## Auditing permissions on a system you did not build

The practitioner skill this chapter builds toward: walk into a system someone else set up and assess the permissions posture. The questions to answer:

**What setuid binaries exist?**

```
sudo find / -perm -4000 -type f 2>/dev/null
```

Compare the result against the default Ubuntu set. Anything unusual gets investigated.

**What world-writable files exist?**

```
sudo find / -perm -o+w -type f 2>/dev/null | grep -v "^/proc\|^/sys"
```

The `grep -v` excludes virtual filesystems where world-writable is normal. Real world-writable files in `/etc`, `/var`, or anywhere a user's home directory should not exist on a healthy system.

**Who has sudo access, and to what?**

```
sudo cat /etc/sudoers
sudo ls /etc/sudoers.d/
sudo cat /etc/sudoers.d/*
```

Check the wildcard rules. Check NOPASSWD entries. Note any rules granting access to specific commands and ask whether each one is necessary.

**Are there ACLs on critical files?**

```
sudo getfacl -R /etc 2>/dev/null | grep -B1 "^user:.*:" | grep "# file"
```

That finds every file under `/etc` with non-default ACL entries. Anything that comes back deserves a look.

**Are home directories private?**

```
ls -ld /home/*
```

Each home directory should be 700 or 750 with the user as owner. Mode 755 means every user on the system can read every other user's home, which is rarely what you want.

These five queries are the start of a permissions audit. They are also what Security+ Domain 4 calls "monitoring" and "scanning" at the conceptual level, applied to file permissions.

---

## Try this

Five exercises. They build on each other. Do them in order.

**1. Read a permission and explain it.**

Run:

```
ls -l /etc/shadow
```

Without searching, write down: who owns this file, what permissions does each actor have, and why is this safer than `/etc/passwd`. Then check yourself by reading `/etc/passwd` and comparing.

**2. Build a shared directory.**

Create `/srv/teamdocs` such that:

- It is owned by root.
- Its group is webops.
- Members of webops can read, write, and create files in it.
- Files created inside automatically inherit the webops group.
- Users not in webops cannot enter the directory.

The fix involves setgid and a specific mode. Hint: `chmod 2770`.

Confirm by creating a file as a webops member and running `ls -l` to see the inherited group.

**3. Find and analyze setuid binaries.**

Run the setuid finder. Compare the output against this list of expected Ubuntu 24.04 setuid binaries:

```
/usr/bin/su
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/umount
/usr/bin/pkexec
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/policykit-1/polkit-agent-helper-1
```

Anything on your box that is not on this list deserves explanation. (On a fresh Ubuntu install, you might also see `fusermount3` or distro-specific helpers; do not assume malicious without checking what the binary is.)

**4. Write a sudoers drop-in.**

Create `/etc/sudoers.d/webops_journal` that lets members of webops run exactly `journalctl -u nginx` and `journalctl -u nginx -f` as root, without other journalctl options, without other commands, and with passwords required.

Use `visudo -f` to create and edit it. Test it by adding your `student` user to webops temporarily (`sudo usermod -aG webops student`, then start a new shell), and confirming that `sudo journalctl -u nginx` works while `sudo journalctl` (without arguments) is denied.

**5. Recursive permissions, done correctly.**

Create `/srv/site` with the following structure:

```
/srv/site/
├── public/
│   ├── index.html
│   └── style.css
└── private/
    └── secrets.conf
```

Set permissions so that:

- `public/` and its contents are mode 644 for files, 755 for directories, owned by `webrunner:www-data`.
- `private/secrets.conf` is mode 640, owned by `webrunner:webops`.
- Everything is set in one command per piece, using either `find` or the symbolic capital-X trick.

The point of this exercise is to do it without breaking the directory traversal. Verify with `ls -lR /srv/site` and by trying to access the files as both webrunner (should work) and as a user with no relevant group memberships (should fail in expected ways).

---

## Common stumbling blocks

> **`chmod -R 644 /var/www/html` made the site unreachable.**
> Recursive 644 strips execute from directories, which means nginx (or any process) cannot enter them. The fix is `chmod -R u=rwX,go=rX /var/www/html` using capital-X, or split into two commands as shown in the chapter.

> **I added the user to a group with `usermod -aG`, but the new group does not work.**
> Group membership is loaded when a session starts, not retroactively. Open a new shell or log out and back in. Confirm with `id`. The flag matters too: always use `-a` (append). Without it, `usermod -G newgroup` removes the user from every other group.

> **`visudo` saves my file but the rule does not work.**
> Two common causes: a syntax error that visudo did not catch (rare but possible with edge cases), or the rule is shadowed by an earlier rule that grants broader access. Check the order in `/etc/sudoers` and verify your drop-in is named so it sorts where you expect. Drop-in files are read in alphabetical order.

> **`sudo` keeps asking for my password every time.**
> That is the default and it is correct behavior. There is a brief grace period (usually 15 minutes) after a successful sudo. If you want longer, increase `timestamp_timeout` in sudoers. If you want never (NOPASSWD), think hard about whether you really do; for an interactive admin user, password every time is the right answer.

> **I cannot read a file even though the mode looks right.**
> Check the directories above it. Every directory in the path needs execute for the user trying to read the file. Run `namei -l /path/to/file` to see the permission of every component in the path; the broken one is usually obvious.

> **`getfacl` shows `mask` in the output and the file's effective permissions are less than I set.**
> The mask is an upper bound on group and named-user ACL entries. If you set `setfacl -m u:webrunner:rwx file` on a file with mask `r--`, webrunner gets effective `r--`, not `rwx`. Set the mask explicitly: `setfacl -m m::rwx file` or use `setfacl --no-mask`.

---

## What this gets you

If you walked through everything:

- You can read a `ls -l` output and explain every character.
- You know when to use 644, 600, 755, 700, 640, and why.
- You understand that execute on a directory means traversal, not running, and you can debug the "but the file mode is right" problem.
- You can find and triage setuid binaries on a system, which is a basic forensic skill.
- You can configure sudo to grant specific actions to specific users without giving root.
- You can read existing sudoers and identify weak rules.
- You know about ACLs, when they help, and when they make things harder to audit.

The 777 antipattern, introduced in the workshop, should now be replaced by a thinking pattern: "what do I actually need this user or process to do, and what is the smallest set of permissions that gets there?" That question is the entirety of access control.

---

## What's next

Chapter 4 is Package management deeper. The chapter where you learn what `apt update` actually does, how to add a third-party repository safely, the difference between snap and apt and when each is right, and how to configure unattended-upgrades so it does not reboot your box at 3 AM.
