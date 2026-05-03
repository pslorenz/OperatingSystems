# Chapter 5: systemd Units and Timers

**You come in with:** workshop-level systemctl. You can start, stop, restart, enable, and disable services. You ran a timer once.
**You leave with:** the ability to read any unit file on a system you did not build, write a working service unit, write a timer that runs reliably, and configure user units for things that should run as you rather than as root.

**Time:** 60 to 90 minutes including the exercises.

**Security+ alignment:** Domain 4.1 (hardening: secure baselines, removal of unnecessary software, applying security techniques to servers). Domain 4.4 (alerting and monitoring of services, dependencies, system logs). Domain 4.7 (automation: enabling/disabling services and access through scheduled units). The User= directive on a service unit is the practical version of "least privilege" applied to running processes.

---

## Why this chapter matters

Every service on a modern Ubuntu box is managed by systemd. Knowing how to read the systemd unit for ssh, nginx, cron, or anything else is how you understand what the system is doing.

Beyond reading, you will eventually need to write a unit. A custom service that runs your application. A timer that runs a maintenance script on a schedule. A user-level unit that starts a tool when you log in. The workshop showed you the verbs. This chapter teaches you to write the nouns those verbs operate on.

The security argument: a service unit that runs as a dedicated low-privilege user, with restricted filesystem access, is the modern way to deploy almost anything. The defaults systemd provides are a real hardening layer most people never use because they never learn what is in a unit file.

---

## What a unit actually is

A systemd unit is a configuration file that describes something systemd manages. *A unit is a file that tells systemd what to manage and how.* Most units describe services (long-running processes), but there are other types: timers (scheduled triggers), targets (groupings), mounts (filesystem mounts), sockets (network listeners that activate services on demand), and others.

The file extension tells you the type. `nginx.service` is a service unit. `apt-daily.timer` is a timer unit. `multi-user.target` is a target unit.

### Where unit files live

There are three locations, in priority order:

1. `/etc/systemd/system/`: your local custom units. Highest priority. Anything here overrides anything below.
2. `/run/systemd/system/`: runtime-only units. Cleared on reboot.
3. `/usr/lib/systemd/system/`: units installed by packages. Lowest priority.

When you write a unit, put it in `/etc/systemd/system/`. When you read an existing unit installed by a package, look in `/usr/lib/systemd/system/`. Never edit files in `/usr/lib/systemd/system/` directly: a package update will overwrite your changes. If you need to modify a packaged unit, use override files (covered later in this chapter).

To find the actual file systemd is using:

```
systemctl cat ssh
```

That shows you the full content of the unit file plus any overrides. It is the single best command for understanding what is configured.

```
systemctl show ssh
```

That shows every property systemd has computed for the service, including defaults you did not set. Verbose, useful when debugging "why is the service behaving this way."

---

## Reading an existing unit

Run:

```
systemctl cat ssh
```

You see something close to:

```
[Unit]
Description=OpenBSD Secure Shell server
After=network.target auditd.service
ConditionPathExists=!/etc/ssh/sshd_not_to_be_run

[Service]
EnvironmentFile=-/etc/default/ssh
ExecStartPre=/usr/sbin/sshd -t
ExecStart=/usr/sbin/sshd -D $SSHD_OPTS
ExecReload=/usr/sbin/sshd -t
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartPreventExitStatus=255
Type=notify
RuntimeDirectory=sshd
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
Alias=sshd.service
```

Walk through the sections.

### [Unit] section

The Unit section describes the service in general terms and tells systemd how it relates to other units.

- **Description**: human-readable. Shows up in `systemctl status`.
- **After**: this unit should start after these others. Does not require them; it is ordering only.
- **Requires** (not shown here): hard dependency. If the listed unit fails, this one fails too.
- **Wants** (not shown here): soft dependency. Tries to start the listed unit but does not require it.
- **ConditionPathExists**: only run if a file does or does not exist (the `!` inverts).

The After/Requires/Wants distinction matters. After is just ordering. Requires and Wants are dependency relationships. A service that has `Requires=postgresql.service` will not start if postgresql is down; one with `Wants=postgresql.service` will try to start postgresql but does not fail if it cannot.

### [Service] section

The Service section is the meat. It tells systemd how to run the program.

- **Type**: how systemd knows the service is "ready." `simple` means "the main process is the service" (default). `forking` means the service forks and the parent exits. `oneshot` means the command runs to completion. `notify` means the service tells systemd when it is ready via the sd_notify protocol.
- **ExecStart**: the command to run.
- **ExecStartPre**: run this before ExecStart. Useful for sanity checks.
- **ExecReload**: what to run when someone calls `systemctl reload servicename`. Often a SIGHUP to the main process.
- **EnvironmentFile**: load environment variables from a file before starting. The `-` prefix means "do not fail if the file is missing."
- **Restart**: restart policy. `no`, `on-success`, `on-failure`, `on-abnormal`, `always`. `on-failure` is the right default for most services.
- **KillMode**: how to terminate. `control-group` (default) kills everything in the cgroup. `process` only kills the main process.

### [Install] section

This section says what happens when someone runs `systemctl enable`. It is not used during normal operation.

- **WantedBy**: when this target is enabled, also enable this unit. `multi-user.target` is the standard for "this should start when the system boots into multi-user mode," which is essentially always.
- **Alias**: alternative names for the service.

The pattern: a service file in `/etc/systemd/system/` plus a `WantedBy=multi-user.target` makes the service start automatically on every boot once you run `systemctl enable`.

---

## Writing your own service unit

The exercise: write a service unit that runs a simple long-running process as a non-root user.

### The script

Create the script first:

```
sudo tee /usr/local/bin/heartbeat.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
while true; do
    echo "$(date) heartbeat from $(whoami)" >> /var/log/heartbeat.log
    sleep 30
done
EOF
sudo chmod +x /usr/local/bin/heartbeat.sh
```

Set up the log file with the right ownership:

```
sudo touch /var/log/heartbeat.log
sudo chown webrunner:webops /var/log/heartbeat.log
sudo chmod 640 /var/log/heartbeat.log
```

(That uses the webrunner user from Chapter 3. If you do not have it, create it: `sudo useradd -r -s /usr/sbin/nologin -g webops webrunner`.)

### The unit

Create `/etc/systemd/system/heartbeat.service`:

```
sudo tee /etc/systemd/system/heartbeat.service > /dev/null <<'EOF'
[Unit]
Description=Periodic heartbeat to the log
After=network.target

[Service]
Type=simple
User=webrunner
Group=webops
ExecStart=/usr/local/bin/heartbeat.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Reload systemd so it sees the new unit:

```
sudo systemctl daemon-reload
```

Always run `daemon-reload` after creating or changing a unit file. Without it, systemctl will say "no such unit" or use the old version.

Enable and start:

```
sudo systemctl enable --now heartbeat
systemctl status heartbeat
```

Confirm it is running and writing to the log:

```
sudo tail -f /var/log/heartbeat.log
```

Stop the follow with Ctrl-C. Now see the journal:

```
sudo journalctl -u heartbeat -n 20
```

You have written a service. It runs as webrunner, restarts if it fails, starts at boot, and logs to the journal.

### What just happened, security-wise

Look at what this unit gave you, beyond just running the script:

- **User isolation**. The script runs as webrunner, not root. If the script gets compromised, the attacker's blast radius is what webrunner can touch.
- **Automatic restart**. If the script crashes, systemd restarts it after 5 seconds. Resilience as a configuration knob.
- **Boot integration**. The service starts automatically and predictably.
- **Logging integration**. journalctl works without you doing anything extra. Logs are durable across restarts.

These are the four things every service should have. Hand-rolled startup scripts that run as root, do not restart on failure, and dump output to `/dev/null` are the antipattern that systemd replaced.

---

## Hardening a service unit

The unit above is functional. There are sandboxing options that make it dramatically more restricted. Most distributions do not enable these by default for backward compatibility, but for new services you write, they should be on by default.

Add these to the [Service] section:

```
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/heartbeat.log
PrivateTmp=true
NoNewPrivileges=true
```

Walk through them:

- **ProtectSystem=strict** makes the entire filesystem read-only to the service, except `/dev`, `/proc`, `/sys`, and any explicitly granted paths. The service cannot modify any system files even if a vulnerability gives it the ability to try.
- **ProtectHome=true** makes `/home`, `/root`, and `/run/user` invisible to the service. The service cannot access user data.
- **ReadWritePaths=/var/log/heartbeat.log** adds back write access to one specific path. Without this, ProtectSystem=strict would prevent the script from writing to the log.
- **PrivateTmp=true** gives the service its own private `/tmp` and `/var/tmp`, isolated from the rest of the system. Prevents tempfile-based attacks between services.
- **NoNewPrivileges=true** prevents the service from gaining any new privileges via setuid binaries or similar. A compromised service cannot escalate by exploiting setuid programs.

Apply the change:

```
sudo systemctl daemon-reload
sudo systemctl restart heartbeat
sudo systemctl status heartbeat
```

If you want to see what systemd is actually enforcing now:

```
systemctl show heartbeat | grep -E '^(Protect|Private|NoNew|ReadWrite)'
```

These five lines turn a service from "runs as a user" into "runs as a user, in a restricted filesystem view, with no path to root." The cost is one minute of writing them. The payoff is real defense in depth.

This is what Security+ Domain 4.1 calls "secure baselines" and "hardening." It is also what most production deployments still skip. Writing it for your own services is the easy version of doing it everywhere.

---

## Override files: changing a unit you do not own

You will sometimes need to modify a unit installed by a package. Editing the file directly in `/usr/lib/systemd/system/` is wrong; the next package update overwrites your change. The right approach is an override file.

```
sudo systemctl edit nginx
```

That opens an editor on `/etc/systemd/system/nginx.service.d/override.conf`, an empty file with comments at the top explaining what to do. Add the directives you want to override.

For example, to add a stricter umask to nginx:

```
[Service]
UMask=0027
```

Save and exit. The override is now in effect, layered on top of the packaged unit. Confirm:

```
systemctl cat nginx
```

You see both the original unit and the override section, marked with the path of the override file. Both are in effect; later sections override earlier ones for the same directive.

To remove an override, delete the override.conf file (or use `systemctl edit --revert nginx`).

This pattern keeps customizations in `/etc/`, separate from package files in `/usr/`. It is the systemd version of the "do not touch the upstream config, override in /etc/" rule that comes up everywhere on Linux.

---

## Timers: scheduled execution the modern way

A timer is a unit that triggers another unit on a schedule. Two files: a `.timer` file with the schedule, and a `.service` file with the work.

The workshop covered the basics. Now we go deeper.

### Timer types

There are two kinds of triggers, which solve different problems:

**Calendar-based** with `OnCalendar=`. Runs at specific times. Good for "every day at 2 AM" kinds of schedules.

```
OnCalendar=*-*-* 02:00:00
OnCalendar=daily
OnCalendar=weekly
OnCalendar=Mon..Fri 09:00
OnCalendar=*-*-01 03:00:00
```

The first runs every day at 2 AM. The next three are shorthand. The fifth runs weekdays at 9 AM. The last runs on the first day of every month at 3 AM.

**Monotonic** with `OnBootSec=`, `OnStartupSec=`, `OnUnitActiveSec=`, `OnUnitInactiveSec=`. Runs relative to events.

```
OnBootSec=15min
OnStartupSec=1h
OnUnitActiveSec=30min
OnUnitInactiveSec=1h
```

The first runs 15 minutes after boot. The third runs every 30 minutes after the unit last became active (which is the right pattern for "every 30 minutes regardless of when the box rebooted").

### A practical timer

Build a daily backup timer that calls the heartbeat unit (treat the heartbeat as a stand-in for a backup script for now).

`/etc/systemd/system/heartbeat-daily.timer`:

```
[Unit]
Description=Run heartbeat once a day at 03:00

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
Unit=heartbeat.service

[Install]
WantedBy=timers.target
```

Two new directives:

- **Persistent=true**. If the system was off at 3 AM, run the timer the next time it is on. Without this, missed timers are simply skipped.
- **Unit=heartbeat.service**. The unit to trigger. If you omit this, systemd looks for a service with the same name as the timer (so `heartbeat-daily.timer` would trigger `heartbeat-daily.service`). Specifying it explicitly is clearer.

Activate:

```
sudo systemctl daemon-reload
sudo systemctl enable --now heartbeat-daily.timer
systemctl list-timers heartbeat-daily.timer
```

The list-timers output shows you the next scheduled run, the previous run, and a few other useful columns.

### Why systemd timers and not cron

cron still works. Most older systems still use it. Both will fire scheduled jobs. The reasons to prefer systemd timers for new work:

- **Logging**. Output goes to journalctl with the same filters everything else uses.
- **Dependency management**. A timer can require other units, wait for the network, run only after another service. cron has no concept of dependencies.
- **Resource control**. Timer-triggered services inherit all the systemd sandboxing options. cron jobs run with whatever environment cron gave them.
- **Status visibility**. `systemctl list-timers` shows you everything scheduled, when it last ran, when it runs next. cron has no equivalent.
- **Persistent missed runs**. The `Persistent=true` directive handles "the box was off, run it when it comes back." cron requires anacron to do this, which is a separate tool.

For new automation, write timers. For maintaining an older system that uses cron, leave it alone unless there is a reason to migrate.

---

## User units: things that should run as you, not as root

Sometimes you want a unit that belongs to your user, not the system. Examples: a personal sync tool, a developer environment helper, a desktop background application. These do not need root and should not have it.

User units live in `~/.config/systemd/user/` and are managed with `systemctl --user`:

```
mkdir -p ~/.config/systemd/user/
cat > ~/.config/systemd/user/note-sync.service <<'EOF'
[Unit]
Description=Sync my notes folder

[Service]
Type=oneshot
ExecStart=%h/bin/sync-notes.sh
EOF

systemctl --user daemon-reload
systemctl --user enable --now note-sync
systemctl --user status note-sync
```

The `%h` is a systemd specifier that expands to the user's home directory. Other useful specifiers: `%u` for the username, `%H` for the hostname. There are dozens; `man systemd.unit` lists them.

User units run when the user logs in. They stop when the user logs out, unless `lingering` is enabled:

```
sudo loginctl enable-linger $(whoami)
```

With lingering on, your user units keep running even when you are not logged in. Useful for personal services that should be persistent.

User units are how you avoid putting personal automation into the system layer. They are also how multi-user systems let users schedule their own work without granting root.

---

## list-timers and list-units: situational awareness

When you walk into a system you did not build, two commands give you a fast overview of what is running and what is scheduled.

```
systemctl list-units --type=service --state=running
```

Every running service. Compare against your mental model of what should be there. Anything unexpected is a question.

```
systemctl list-timers --all
```

Every timer, including disabled ones. Reading this output is auditing the scheduled-work landscape on the box. Anything you do not recognize is a question. We come back to this in Chapter 11 (Linux artifacts) where unexpected timers are a persistence pattern.

```
systemctl --failed
```

Anything that failed to start. A clean system has no entries here. A non-empty list is a starting point for "what is broken."

These three commands are 90% of the situational awareness you need on a new box. Five seconds, three answers.

---

## Try this

**1. Read the unit for cron.**

Run `systemctl cat cron`. Walk through every directive. For each one, write down what it does and why. The point is to confirm you can read a unit file written by someone else. If any directive is unclear, look it up in `man systemd.service` (which is the canonical reference for service unit options).

**2. Write a timer that runs every 5 minutes.**

Build a simple service that appends a timestamp to a log:

```
[Unit]
Description=Log a timestamp

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo "$(date) tick" >> /var/log/tick.log'
```

Plus a timer that runs every 5 minutes using `OnUnitActiveSec=`. Confirm with `journalctl -u <service> -f` that it is firing.

**3. Harden an existing service.**

Pick a service on your box (try `cron` if it is there, or another simple one). Use `systemctl edit` to add `ProtectSystem=strict` and `NoNewPrivileges=true`. Restart the service. Confirm it still works (cron should still run cron jobs). Confirm via `systemctl show <name> | grep Protect` that the directives took effect.

**4. Configure user lingering.**

Enable lingering for your user. Create a user unit that does anything (a one-shot timestamp logger like the one above, but writing to `/tmp/user-tick.log` and running as you, not root). Log out and log back in. Confirm via `systemctl --user list-units` that the unit is still active.

**5. Audit timers on the box.**

Run `systemctl list-timers --all`. For each timer that is not yours, identify what it does. Specifically:

- `apt-daily.timer` and `apt-daily-upgrade.timer` (from Chapter 4)
- `logrotate.timer` 
- `motd-news.timer` (Ubuntu's "message of the day" updater)
- `fwupd-refresh.timer` (firmware updates)
- `man-db.timer` (rebuilds the man page index)
- `update-notifier-download.timer` and `update-notifier-motd.timer`

Find any that are not on this list. Read their corresponding service units (`systemctl cat <name>.service`) and explain what they do.

---

## Common stumbling blocks

> **`systemctl enable` says "no such unit" after I created the file.**
> Run `sudo systemctl daemon-reload`. systemd caches the unit list; daemon-reload rebuilds it. This is the most common stumble after writing a new unit.

> **The service starts but immediately exits with status 0.**
> Type=simple means "the foreground process is the service." If your script or program forks into the background and exits, systemd thinks the service is done. Either change Type to `forking` (and provide a PIDFile) or change the program to stay in the foreground. For most scripts, Type=simple plus a foreground loop is correct.

> **The service runs as root even though I set User=.**
> The User= directive only takes effect for the main process started by ExecStart. If your script then runs `sudo something` or starts processes as another user, those run as configured. Also: User= on a forking service can behave unexpectedly; for forking services, prefer Type=simple where possible.

> **My timer is enabled but is not firing.**
> Three usual causes. First, you forgot `daemon-reload` after writing it. Second, you enabled the service file instead of the timer file (you must enable the .timer, not the .service). Third, the script the service runs is not executable or is misnamed; check `journalctl -u <service>` for the actual error.

> **`systemctl edit` opens an empty file. Where do I put the directives?**
> The empty file is the override file, layered on top of the packaged unit. You only need to put the [Section] header and the directives you want to change or add. You do not need to repeat the entire original unit. Save and exit, and the override merges with the original.

> **My user units stop working when I close my SSH session.**
> User units stop when the user session ends, unless lingering is enabled. Run `loginctl enable-linger $(whoami)` to keep user units running across logout.

---

## What this gets you

After this chapter:

- You can read any service unit on a system you did not build.
- You can write a unit that runs your code as a non-root user, with sandboxing, with restart on failure, with proper logging.
- You can write a calendar-based or monotonic timer.
- You can override a packaged unit without breaking the package.
- You can use user units for personal automation that does not need root.
- You can take inventory of what is running and what is scheduled in 30 seconds.

The hardening directives (ProtectSystem, ProtectHome, NoNewPrivileges, PrivateTmp) are the part of this chapter that pays off most. Most production deployments do not use them. Yours will.

---

## What's next

Chapter 6 is Logs, auditing, and what the system records. The chapter where journalctl gets serious. By the end you will be able to answer "what did this user do on this box," "who logged in successfully and from where," and "when did this thing fail," with concrete commands and the right filters.
