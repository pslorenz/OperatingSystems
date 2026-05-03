# Chapter 0: Linux Foundations Workshop

**You come in with:** maybe nothing. Maybe a little command-line experience. Either is fine.
**You leave with:** the ability to SSH into a Linux box you have never seen before, find your way around it, run a service on it, push files to it, and schedule a job. Five hours of typing, three blocks, one lab box.

**Time:** 5 hours, live with the cohort. No additional self-paced time required after the workshop, though re-reading this chapter as a reference is recommended.

**Security+ alignment:** No direct exam content. The workshop is foundation skill that supports later chapters mapping to Domains 2 (Threats and Vulnerabilities), 3 (Security Architecture), and 4 (Security Operations). If you are working toward Security+, do this workshop first, then the rest of the unit.

---

## How to use this document

This is the workshop chapter. It mirrors what we cover live in the room. If you are reading it during the workshop, follow along. If you are reading it after, this is your reference for what you did and the commands you used.

The document is structured the way the day runs: an open, three blocks separated by a break and lunch, then a wrap. Each block has explanatory text, the commands you typed, and the lab you ran. Use the slides as the visual; use this as the reference.

A few conventions:

`Commands look like this`. They are typed in monospace so you can copy them.

```
Multi-line examples
look like this.
```

Output, when shown, follows the command. Long output is usually trimmed.

---

## Open and orientation

Linux is the operating system that runs most of the internet. Every web server you visit, every cloud workload, every SIEM, every log collector, most container infrastructure, and almost every attacker's toolkit runs on Linux. Even in a heavy Windows shop, you will read a Linux box this year.

That is the practical case. The personal case is different: Linux feels intimidating because nobody explains the basics out loud. The way to get good at Linux is to use Linux. By the end of this five hour block, you will have used Linux. You will not be an expert. You will be functional, which is a much higher bar than most people start at.

A few framing notes before we start.

This is **skill, not talent.** People who are good at Linux were once people who were not good at Linux. They typed enough that the commands became muscle memory. The people in this room who get the most out of today are the ones who type along, not the ones who watch.

You will get **stuck** at some point today. Probably more than once. That is the point. The lab box is yours. Break it. Fix it. The instructor will help when you ask. Stuck is where the learning happens.

Today is not exhaustive. Five hours is enough to make you functional, not complete. The post-workshop chapters in this unit cover what we cut. The day ends with a slide listing what we skipped and where to find it.

### Your lab box

Your CourseStack lab is a single Ubuntu 24.04 LTS server. You connect to it from your laptop with SSH:

```
ssh student@<your-lab-box-address>
```

The instructor will give you the address. The username is `student`. You have sudo. The box has internet access for installing packages. It persists across the workshop, so anything you install or change stays.

If you have never used SSH before: it is a program that opens a terminal session on a remote machine. *SSH stands for Secure Shell. It is the encrypted protocol that lets you log into a Linux box across the network.* After the connection is made, your local terminal sends keystrokes to the remote machine and shows you what it sends back. You are typing on a computer that is not in front of you.

When you see a prompt that ends in `$`, you are connected. That is where the workshop starts.

---

## Block 1: Getting around

By the end of this block you can SSH into a Linux machine you have never seen before and figure out what is on it. That is the skill we are building. Every command in this block serves that scenario.

### The terminal

A terminal is a program that lets you type commands and read replies. That is it. No magic.

When you click on an icon in the desktop to open Firefox, the desktop runs the same kind of command you would type. The terminal removes the desktop and lets you talk to the system directly.

Compared to a GUI:

- **GUI**: you click, the system guesses what you meant. Easy for one box, slow for fifty. Hard to script, hard to share, easy to misclick.
- **Terminal**: you type, the system does exactly what you said. Same commands work on every Linux box you will ever touch. Easy to script, easy to share, easy to repeat.

The terminal is the window. The shell is the program inside it that reads your commands. *A shell is the program that interprets what you type and runs the right command in response.* On Ubuntu, the default shell is bash. We are using bash today. You do not need to think about the distinction yet.

### Anatomy of a command

Every Linux command has the same shape. Three parts, in order:

```
ls -la /etc
```

- **command**: the verb. `ls` means list.
- **options**: flags that change behavior. Usually start with a dash. `-la` means "long format and show hidden."
- **argument**: what you act on. `/etc` is the directory you want listed.

When you do not know what a command does, ask it. Almost everything supports `--help`:

```
ls --help
```

When `--help` is not enough, read the manual:

```
man ls
```

Press `q` to quit the manual. You will use `q` to quit a lot of things in Linux. It is worth remembering.

### Where am I, what is here

Three commands. You will use them constantly.

```
pwd
```

`pwd` prints your current directory. The "working directory" is where you are. Most commands act on the working directory by default.

```
ls
ls -la
```

`ls` lists what is in the current directory. `ls -la` lists everything (including hidden files) in long format (with permissions, owner, size, modification time).

```
cd /var/log
cd ~
cd /etc
cd ..
cd -
```

`cd` changes directory. `~` is shorthand for your home directory (`/home/student`). `..` is the parent directory. `-` is the previous directory you were in. Try them.

### The filesystem tree

Every file on a Linux system lives somewhere under `/`. There is one root directory. No drive letters.

```
ls /
```

The directories you care about most:

- `/etc`: configuration files for the system.
- `/var/log`: logs.
- `/home/student`: your stuff.
- `/usr/bin`: most installed programs.
- `/tmp`: scratch space, cleared on reboot.

You do not need to memorize the rest. You need to recognize them. Chapter 1 of the post-workshop content walks every directory in detail.

### Reading files

Four tools, picked by what you need to see:

```
cat /etc/os-release
less /var/log/syslog
head /var/log/syslog
tail /var/log/syslog
tail -f /var/log/syslog
```

- `cat` dumps the whole file to the screen. Good for short files.
- `less` pages through a file. `q` to quit, `/` to search.
- `head` shows the first ten lines.
- `tail` shows the last ten lines. `tail -f` follows the file as it grows. Press Ctrl-C to stop.

### Searching

Two tools. Different jobs.

`find` looks for files by name, type, age, or owner:

```
find /etc -name "*.conf"
find /var/log -mtime -1
```

The first finds every file under `/etc` ending in `.conf`. The second finds files in `/var/log` modified in the last day.

`grep` looks for text inside files:

```
grep "Failed" /var/log/auth.log
grep -i "failed" /var/log/auth.log
grep -r "PermitRoot" /etc/ssh/
```

`-i` is case-insensitive. `-r` is recursive into a directory.

### Pipes and redirection

This is the conceptual idea that makes Linux feel different from clicking around.

A pipe `|` sends the output of one command into the input of the next. A redirect `>` sends output to a file instead of the screen.

```
grep "Failed" /var/log/auth.log | wc -l
grep "Failed" /var/log/auth.log > ~/failed.txt
cat ~/failed.txt | wc -l
```

The first counts the failed login lines. The second saves them to a file. The third reads that file and counts its lines.

Two more redirects worth knowing:

- `>` overwrites the destination file every time.
- `>>` appends to the destination file.

You will use append in Block 3.

### Lab: orient yourself

You SSH into a box. You do not know what is running. Find out.

1. Identify the OS version and the hostname. Try `cat /etc/os-release` and `hostname`.
2. Find every file in `/etc` that ends in `.conf`. Try `find /etc -name "*.conf"`.
3. Read the most recent ten lines of `/var/log/auth.log` without opening it in an editor. Try `tail /var/log/auth.log` (you may need `sudo` to read it).
4. Count how many failed login attempts are in that log. Use `grep` and a pipe.
5. List who is currently logged in. Try `who`.

If you finish early: use `grep` and a pipe to find every line in `/etc/ssh/sshd_config` that is not a comment. Hint: `grep -v "^#"` excludes lines starting with `#`.

---

## Block 2: Running a service

Block 1 was navigation. Block 2 is doing something. By the end of this block you will have a web server running on your lab box, with a page you put there, locked down by permissions you set.

Five concepts, then the lab.

### Users and groups

Every action on Linux is done by some user, who belongs to some groups. The system tracks this for everything.

```
whoami
id
```

`whoami` returns your username. `id` returns the full picture: your numeric user ID, your primary group, every other group you belong to. The number 1000 is the convention for the first regular user on a system.

When you run a command, Linux does not ask "are you Steven", it asks "are you UID 1000". The username is a label.

To create a service account that can own files but cannot log in:

```
sudo groupadd webops
sudo useradd -r -s /usr/sbin/nologin -g webops webrunner
id webrunner
```

`-r` makes it a system account. `-s /usr/sbin/nologin` means the account cannot log in interactively. `-g webops` puts it in the webops group.

This is the first piece of security hygiene in the workshop. Service accounts should not be able to log in. The webrunner user exists to own files and run a process. Nothing else.

### Permissions

Every file has an owner, a group, and a permission set. The permission set has three slots: read, write, execute. Three actors: owner, group, everyone else.

```
ls -l /etc/ssh/sshd_config
```

The output starts with something like `-rw-r--r--`. Reading left to right:

- The first character is the file type. `-` is regular file, `d` is directory, `l` is symlink.
- Then three groups of three: owner, group, everyone else.
- Each group is read, write, execute. A dash means "no."

So `-rw-r--r--` reads as: owner can read and write, group can read, everyone else can read. Nobody runs it (because it is not a program).

To change ownership and permissions:

```
sudo chown webrunner:webops /var/www/html/index.html
sudo chmod 640 /etc/myapp/secret.conf
```

`640` in numeric form: owner read+write (4+2=6), group read (4), everyone none (0). The right permission for a config file with secrets in it.

A warning that matters: when you see a permission error during the lab, do not reach for `chmod 777`. 777 means everyone on the box can do anything to that file. It is the same as posting your password on the front door. If you find yourself reaching for it, stop and ask the instructor instead. We call this the "777 antipattern" and you will hear it again in later courses.

### Processes

Anything running on the box is a process. Programs you run, services in the background, the shell itself. Each one has a number, a PID. *A PID, or process ID, is the number Linux uses to refer to a running process. PIDs are unique while the process is running and get reused after.*

```
ps aux | head
ps aux | grep ssh
top
```

`ps aux` is the inventory of every process. Pipe through `grep` to find one. `top` is the live view (press `q` to quit).

If you want a nicer live view:

```
sudo apt install htop
htop
```

To stop a process:

```
kill 12345
```

Where 12345 is the PID from `ps`. There is also `kill -9` which is a stronger version that skips the polite shutdown. Use it only when normal `kill` fails.

### systemd

A service is a program the system runs in the background and watches. systemd is the program that does the watching. systemctl is how you talk to it.

```
systemctl status ssh
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
sudo systemctl disable nginx
```

`status` shows what a service is doing. `start` and `stop` are obvious. `restart` is start after stop. `enable` makes the service start automatically when the box boots. `disable` turns that off.

There is also `enable --now` as a shorthand: enable plus start in one command. You will use this in Block 3.

### apt

`apt` is how you install software on Ubuntu. It downloads from Ubuntu's mirrors, resolves dependencies, and installs everything correctly.

```
sudo apt update
sudo apt install nginx
sudo apt remove nginx
sudo apt purge nginx
sudo apt upgrade
apt search htop
```

`update` refreshes the package list. `install` installs. `remove` uninstalls but keeps configs. `purge` uninstalls and removes configs. `upgrade` installs all available updates. `search` finds packages by keyword.

A warning: do not download `.deb` files from random websites and install them with `dpkg -i`. apt's whole job is to install from sources Ubuntu trusts, with dependencies handled. Sideloading random files breaks both of those things.

### Lab: stand up nginx

Install a web server. Make it serve a page. Lock it down.

1. Install nginx with apt. Confirm it is running with `systemctl status nginx`.

2. Replace the default page in `/var/www/html/index.html` with your own line of text. You have not learned an editor yet, so use this trick:

   ```
   echo "Hello from <your name>" | sudo tee /var/www/html/index.html
   ```

   `tee` writes its input to a file. `sudo tee` is how you write to a file you do not own.

3. Open the page in your browser by visiting your lab box address. You should see your line of text.

4. Create a webops group. Set the index file to be owned by `webrunner:webops` with mode `640`:

   ```
   sudo groupadd webops    # may already exist from earlier
   sudo chown webrunner:webops /var/www/html/index.html
   sudo chmod 640 /var/www/html/index.html
   ```

5. Confirm nginx still serves the page. It will not. The page returns 403 Forbidden.

   This is the lesson. nginx runs as the `www-data` user, which is neither webrunner nor in webops, so mode 640 means nginx cannot read the file. Read the journalctl error and fix it:

   ```
   sudo journalctl -u nginx -n 20
   ```

   Multiple right answers exist. The simplest fix is to change the group:

   ```
   sudo chown webrunner:www-data /var/www/html/index.html
   ```

   Or add www-data to webops:

   ```
   sudo usermod -aG webops www-data
   sudo systemctl restart nginx
   ```

   The point is: you read the log, found the error, chose a fix. That is the loop you use forever.

If you finish early: enable nginx so it starts at boot, then reboot the box (`sudo reboot`), SSH back in, and confirm nginx came back up.

---

## Block 3: Connecting and shipping

Last block. By the end of it, this lab box is something you actually work with: you authenticate without a password, push files from your laptop, read system logs, write a small script, and schedule it to run on a clock.

### SSH keys

Stop typing your password every time you connect. Start authenticating with math.

An SSH key is a pair of files. The private key stays on your laptop. It is a secret. The public key goes on every Linux box you want to log into. When you connect, the box challenges you to prove you hold the private key without ever sending it. If you can, you are in.

On your laptop (not the lab box):

```
ssh-keygen -t ed25519 -C "laptop"
```

Press Enter to accept the default file location. At the passphrase prompt, you have a real choice. A passphrase encrypts the private key on disk, so a stolen laptop does not equal stolen access. The tradeoff is you have to type the passphrase to use the key. For today, leaving it empty is fine. In real life, a passphrase plus an SSH agent is the right answer.

Copy the public key to the lab box:

```
ssh-copy-id student@<your-lab-box-address>
```

Type the password one last time. After that, plain SSH should work without a password:

```
ssh student@<your-lab-box-address>
```

### ssh config and file transfer

`~/.ssh/config` on your laptop is a list of named hosts. After you set this up, plain `ssh labbox` connects without typing the address.

Edit `~/.ssh/config` (create the file if it does not exist) and add:

```
Host labbox
    HostName <your-lab-box-address>
    User student
    IdentityFile ~/.ssh/id_ed25519
```

Now `ssh labbox` works. So does `scp labbox:` and `rsync to labbox:`.

`scp` copies a single file:

```
scp report.txt labbox:~/
```

That copies `report.txt` from your laptop to the lab box's home directory. The colon separates the hostname from the path on that host.

`rsync` is the better tool for anything bigger:

```
rsync -av ./site/ labbox:/var/www/html/
```

`-a` preserves permissions, timestamps, and recurses into directories. `-v` shows what gets copied. The trailing slash on `./site/` matters: it copies the contents of site, not the site directory itself. This trailing-slash rule is the rsync gotcha that catches everyone once.

### journalctl

Every service that systemd manages writes its output to the journal. journalctl reads it. The right filter turns a noisy stream into the answer to your question.

```
sudo journalctl -u nginx
sudo journalctl -u ssh -b
sudo journalctl -fu nginx
```

- `-u` filters to one service unit.
- `-b` filters to "since the most recent boot."
- `-fu` is follow-mode for one unit, like `tail -f` for systemd services.

Time-based filters:

```
sudo journalctl --since "1 hour ago"
sudo journalctl --since "1 hour ago" -p warning
```

`-p warning` filters to priority warning or worse. The priority levels match syslog: emerg, alert, crit, err, warning, notice, info, debug.

This is the tool you will use for "is the service running, what did it say, is there an error" for the rest of your career.

### A tiny shell script

A shell script is a text file that holds commands. The first line, the shebang, says which interpreter to use. *The shebang is the `#!` at the start of a script that tells the system which program should run the file. `#!/usr/bin/env bash` means "run this with bash."* Make the file executable, run it like any other program.

```bash
#!/usr/bin/env bash
set -euo pipefail

STAMP=$(date +%F_%H%M)
echo "backup started at $STAMP" >> /var/log/myapp.log

tar -czf /tmp/site-$STAMP.tgz /var/www/html
echo "backup done" >> /var/log/myapp.log
```

The `set -euo pipefail` line is non-negotiable for any script that matters. It makes bash fail loud and fail early instead of silently continuing through errors.

`$(date +%F_%H%M)` is command substitution. The shell runs `date +%F_%H%M`, takes its output (something like `2026-05-03_1245`), and uses it as the value of STAMP.

`>>` appends. `>` would overwrite the log file every time, which would make the log useless. Append is what you want.

To use a script: save it to a file, make it executable with `chmod +x scriptname`, then run it with `./scriptname`.

### systemd timers

Two unit files. One says what to run. One says when. systemd handles the rest.

A service unit at `/etc/systemd/system/sitebackup.service`:

```
[Unit]
Description=Back up the website

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

A timer unit at `/etc/systemd/system/sitebackup.timer`:

```
[Unit]
Description=Run sitebackup nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

`Type=oneshot` means it runs to completion, not a long-running service. `OnCalendar` is the schedule. `*-*-* 02:00:00` reads as "any year, any month, any day, at 02:00:00." You can also write `daily`, `hourly`, `weekly`, or specific times.

To activate:

```
sudo systemctl daemon-reload
sudo systemctl enable --now sitebackup.timer
sudo systemctl list-timers
```

`daemon-reload` tells systemd to read the new files. `enable --now` enables and starts in one command. `list-timers` shows every timer on the box.

Why timers and not cron? Timers integrate with the rest of systemd. Logs go to journalctl. Status comes from systemctl. Cron still works on older boxes. New work goes to timers.

### Lab: ship and schedule

From your laptop, to the box, on a schedule, with proof.

1. Generate an SSH key pair on your laptop and copy the public key to the lab box. (You may have already done this in the walkthrough above.)

2. Add a `Host labbox` entry to your `~/.ssh/config` so plain `ssh labbox` connects.

3. Use `rsync` to copy a small folder from your laptop to `/home/student/uploads`. You will have to make the folder first:

   ```
   ssh labbox 'mkdir -p ~/uploads'
   rsync -av ./somefolder/ labbox:~/uploads/
   ```

4. Write a two-line shell script that appends the current timestamp to `/var/log/checkin.log`. Save it as `/usr/local/bin/checkin.sh`. Make it executable:

   ```bash
   #!/usr/bin/env bash
   echo "$(date) checked in" >> /var/log/checkin.log
   ```

   ```
   sudo chmod +x /usr/local/bin/checkin.sh
   ```

5. Build a service unit and a timer unit that run your script every two minutes. Enable the timer.

   `/etc/systemd/system/checkin.service`:
   ```
   [Unit]
   Description=Check in to the log

   [Service]
   Type=oneshot
   ExecStart=/usr/local/bin/checkin.sh
   ```

   `/etc/systemd/system/checkin.timer`:
   ```
   [Unit]
   Description=Run checkin every two minutes

   [Timer]
   OnUnitActiveSec=2min
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```

   Activate:

   ```
   sudo systemctl daemon-reload
   sudo systemctl enable --now checkin.timer
   ```

6. Confirm. Two views:

   ```
   sudo journalctl -u checkin.service
   sudo tail -f /var/log/checkin.log
   ```

   The journalctl output should show the service starting and exiting cleanly, every two minutes. The log file should have a timestamp line per run.

If you finish early: run `systemctl list-timers --all` to see every timer on the box, including the OS's own timers like `apt-daily.timer`.

---

## Wrap

### What you can now do

Five hours ago this was a closed box. Now you can:

- Open a terminal, SSH into a Linux box, find your way around its filesystem.
- Read files, search for content, chain commands together with pipes.
- Identify users and groups. Set permissions and ownership without panicking.
- Install software with apt. Start, stop, and check services with systemctl.
- Authenticate with SSH keys. Move files with scp and rsync.
- Read logs with journalctl. Schedule jobs with systemd timers.

Some of it has not stuck yet. That is normal. The post-workshop chapters in this unit are where it sticks. The workshop's job was to make sure you have a place to put the content when it lands.

### What got cut

Five hours is enough to make you functional, not exhaustive. Here is what we deliberately skipped, and where it lives:

- **Vi survival**: editing files when nano is not installed. Chapter 2.
- **The shell environment**: PATH, dotfiles, aliases, why your alias did not stick. Chapter 2.
- **Permissions deep dive**: ACLs, sudoers done right, the special bits like setuid. Chapter 3.
- **systemd unit files at depth**: reading and writing your own service definitions. Chapter 5.
- **Disk and filesystem**: df, du, mount, fstab, recovering when fstab eats your boot. Chapter 7.
- **Host networking tools**: ip, ss, dig, tcpdump for the four out of five outage tickets. Chapter 8.
- **Shell scripting**: variables, conditionals, loops, exit codes. Chapter 9.
- **Reading the system**: /proc, lsof, the workflow for what is this process really doing. Chapter 10.

### Where to keep going

Three honest paths:

**1. Finish the unit.** Tonight your CourseStack opens up the rest of this unit. Twelve self-paced chapters, around eleven hours of guided content, with the same lab box. Start with **Chapter 1: The Filesystem Hierarchy.** It is the lightest chapter and turns the directories you saw today into a working map of any Linux box you will encounter.

**2. HackTheBox Starting Point.** Free, beginner tier, walks you through using a Linux box for security work. The first twenty boxes will turn today's commands into muscle memory.

**3. Run your own.** An Ubuntu VM on your laptop costs nothing. VirtualBox, UTM (Apple Silicon Macs), or Hyper-V (Windows). Break it. Fix it. There is no faster path to comfort with Linux than owning a box you cannot mess up at work.

The truth: the people who get good at Linux are the people who use Linux. Not the people who took the most courses. Whatever path keeps you typing into a terminal twice a week is the right path.

---

## Common stumbling blocks

A few real failure modes from previous workshops, with the fix.

> **The SSH connection times out or refuses.**
> Confirm the address is correct. Confirm your laptop has network access to it. If your lab platform exposes the box only through a web terminal, use that instead of trying to SSH from your laptop. The instructor can confirm what is supported in your environment.

> **I get "permission denied" trying to read /var/log/auth.log.**
> Some logs require root to read. Use `sudo tail /var/log/auth.log` or `sudo grep ... /var/log/auth.log`. Without sudo, the system protects auth-related logs from non-privileged users.

> **I am stuck inside `less` (or `man`) and cannot get out.**
> Press `q`. That is the universal quit for paged viewers in Linux. If `q` does not work, try `Esc` first, then `q`.

> **My nginx page loads default content, not my edited file.**
> Browser cache. Force-reload (Ctrl-Shift-R on most browsers, Cmd-Shift-R on Mac) or open the page in a private window.

> **I get "permission denied" trying to write to /var/www/html/index.html.**
> The file is owned by root. Use the `sudo tee` trick: `echo "text" | sudo tee /var/www/html/index.html`. The pipe-to-tee pattern is how you write to a file you do not own.

> **My systemd timer is enabled but does not seem to fire.**
> Three usual causes: forgot to run `sudo systemctl daemon-reload` after writing the unit files, the script is not executable (`chmod +x` it), or there is a typo in the `ExecStart` path. Read `journalctl -u <yourservice>` and the error will be in there.

---

## Reference: every command from today

A condensed list, in order of appearance, for when you need to look something up:

**Block 1**

```
ssh student@<address>
pwd
ls
ls -la
cd /var/log
cd ~
cd ..
cd -
ls /
cat /etc/os-release
less <file>          # q to quit, / to search
head <file>
tail <file>
tail -f <file>       # Ctrl-C to stop
find /etc -name "*.conf"
find /var/log -mtime -1
grep "Failed" /var/log/auth.log
grep -i "failed" /var/log/auth.log
grep -r "PermitRoot" /etc/ssh/
grep "Failed" /var/log/auth.log | wc -l
grep "Failed" /var/log/auth.log > ~/failed.txt
hostname
who
```

**Block 2**

```
whoami
id
sudo groupadd webops
sudo useradd -r -s /usr/sbin/nologin -g webops webrunner
ls -l <file>
sudo chown user:group <file>
sudo chmod 640 <file>
ps aux | head
ps aux | grep <name>
top                  # q to quit
htop
sudo apt install htop
kill <PID>
systemctl status <service>
sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl restart <service>
sudo systemctl enable <service>
sudo systemctl disable <service>
sudo systemctl enable --now <service>
sudo apt update
sudo apt install <package>
sudo apt remove <package>
sudo apt purge <package>
sudo apt upgrade
apt search <keyword>
echo "text" | sudo tee <file>
sudo journalctl -u <service> -n 20
sudo usermod -aG <group> <user>
```

**Block 3**

```
ssh-keygen -t ed25519 -C "laptop"
ssh-copy-id student@<address>
ssh labbox
scp <file> labbox:~/
rsync -av ./<dir>/ labbox:<remote-dir>/
sudo journalctl -u <service>
sudo journalctl -u <service> -b
sudo journalctl -fu <service>
sudo journalctl --since "1 hour ago"
sudo journalctl --since "1 hour ago" -p warning
chmod +x <script>
./<script>
sudo systemctl daemon-reload
sudo systemctl enable --now <timer>
sudo systemctl list-timers
sudo systemctl list-timers --all
```

---

## What's next

If you came here from the workshop, your next stop is **Chapter 1: The Filesystem Hierarchy.** It expands the brief tour of `/etc`, `/var/log`, and `/home` into a working map of every directory on a Linux box.

If you are reading this before the workshop, see you in the room.
