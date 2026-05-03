# Chapter 4: Package Management

**You come in with:** workshop-level apt install and apt remove. You can install nginx and you know `sudo apt update` exists.
**You leave with:** the ability to add a third-party repository safely, hold a package version when you need to, decide between snap and apt for any given piece of software, and configure unattended-upgrades so it does what you want.

**Time:** 45 to 75 minutes including the exercises.

**Security+ alignment:** Domain 2.5 (mitigation techniques: patching). Domain 4.1 (hardening techniques: removal of unnecessary software, patching). Domain 4.3 (vulnerability response and remediation: patching, segmentation, validation of remediation). Domain 5.1 (procedures: change management). Patch management is a recurring theme on Security+ and this chapter is the practical version of it on Linux.

---

## Why this chapter matters

Patching is the single most effective security control most organizations implement. It is also where most of them fail, because patching at scale is operationally hard. Understanding what `apt update` actually does, what `apt upgrade` actually changes, and how to control the timing matters as much as understanding any individual security tool.

The other reason this chapter exists: a real working admin will need to install software that is not in Ubuntu's default repositories. The right way to do that has changed significantly in the last few years. The old guidance ("just `apt-key add` the signing key and you are done") is now actively wrong, and following it leaves systems with deprecated security configurations.

This is the chapter that fixes both gaps.

---

## What apt actually does

`apt` is a package manager. *A package manager is a system that tracks installed software, resolves dependencies between packages, and provides a single entry point for install, update, remove, and upgrade.* On Ubuntu, apt is the standard.

Behind the scenes, apt is a frontend to lower-level tools: `dpkg` for installing and removing individual packages, libraries that download and verify package archives, and configuration files that describe where to get packages from.

A package on Ubuntu is a `.deb` file. *A `.deb` file is a compressed archive containing program files, metadata about what the package does, and scripts that run during install and remove.* Each `.deb` is signed by the publisher. apt verifies these signatures before installing anything. That signature check is the security model.

### The four states of a package

A package on your system can be in one of four states:

| State | Meaning |
|---|---|
| Not installed | apt knows about it but it is not on your system |
| Installed | The files are present and the package is registered |
| Installed, holds applied | Installed and explicitly excluded from updates |
| Removed but configs remain | Files gone but `/etc/<package>` is still there |

The fourth state is the one that surprises people. `apt remove` deletes the program files but leaves the configuration in `/etc`. This is intentional: if you reinstall, your customizations come back. To remove configs too, use `apt purge`.

```
apt list --installed | wc -l
apt list --installed | head
```

That tells you roughly how many packages are on the box. A default Ubuntu Server install is around 600. A desktop install is around 2,500.

### apt update versus apt upgrade

These two commands do completely different things and the names are confusing.

`apt update` refreshes the **list of available packages** from your configured sources. It downloads metadata, not actual packages. It does not install or change anything.

`apt upgrade` installs any **available updates** to packages already on the system. It uses the metadata from the most recent `apt update` to decide what to install.

The pattern:

```
sudo apt update                # refresh the catalog
sudo apt upgrade -y            # install all available updates
```

`-y` answers yes to prompts. Use it for automation. Skip it interactively so you can see what is changing.

There is also `apt full-upgrade` (formerly `apt-get dist-upgrade`), which can remove packages if necessary to install upgrades. Regular `upgrade` will not remove anything. `full-upgrade` is the right command when an Ubuntu point release introduces dependency changes; otherwise stick to `upgrade`.

---

## Where packages come from: sources

apt downloads packages from servers it has been told about. The list of servers lives in two places:

```
/etc/apt/sources.list
/etc/apt/sources.list.d/
```

The main file holds Ubuntu's default repositories. The directory holds drop-in files for additional sources. Look at the main file:

```
cat /etc/apt/sources.list
```

On modern Ubuntu (24.04 onward), this file is mostly empty or absent. The actual sources have moved to a new format in `/etc/apt/sources.list.d/ubuntu.sources`. The format changed from the one-line legacy syntax to a multi-line "deb822" syntax that is easier to read and machine-parse.

Look at the new format:

```
cat /etc/apt/sources.list.d/ubuntu.sources
```

You see something like:

```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Reading this:

- **Types**: `deb` (binary packages) or `deb-src` (source packages). Almost always `deb`.
- **URIs**: where to download from.
- **Suites**: which release. `noble` is Ubuntu 24.04's codename. `noble-updates` and `noble-backports` are progressively newer than the release version.
- **Components**: `main` (officially supported), `restricted` (proprietary drivers), `universe` (community-supported), `multiverse` (non-free, restricted-license).
- **Signed-By**: the file containing the GPG key that signs packages from this source.

The Signed-By line is the critical security control. Without it (or with a wrong key), apt cannot verify that packages came from who they claim. This is the part of apt that catches a malicious mirror serving altered packages.

---

## Adding a third-party repository (the modern way)

Sooner or later you will need software that is not in Ubuntu's repositories. Docker, Microsoft's repositories for things like .NET or VS Code, GitHub's CLI, and many others publish their own apt repositories.

The old way of adding these used `apt-key add`. **Do not use `apt-key`. It is deprecated and removed in modern Ubuntu.** Tutorials older than about 2022 will tell you to use it. They are wrong now.

The modern way: drop the signing key into `/etc/apt/keyrings/` and reference it from the source file with `Signed-By`. Each source has its own key, isolated from every other source. The old `apt-key` model put every key into one global trust store, which meant any added key could sign packages for any repository. The new model fixes that.

### Walkthrough: adding the GitHub CLI repository

GitHub publishes its CLI as an apt repository. Here is how you add it correctly.

**Step 1.** Create the keyrings directory if it does not exist:

```
sudo mkdir -p /etc/apt/keyrings
sudo chmod 755 /etc/apt/keyrings
```

**Step 2.** Download the signing key and store it in the keyrings directory. GitHub publishes its key at a documented URL:

```
sudo curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    -o /etc/apt/keyrings/githubcli-archive-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/githubcli-archive-keyring.gpg
```

The `-fsSL` flags on curl: fail on errors (`f`), silent (`s`), show errors (`S`), follow redirects (`L`). That combination is the standard pattern for piping curl output to a file or another command.

**Step 3.** Create a source file that points to GitHub's repo and references the key:

```
sudo tee /etc/apt/sources.list.d/github-cli.sources > /dev/null <<'EOF'
Types: deb
URIs: https://cli.github.com/packages
Suites: stable
Components: main
Architectures: amd64 arm64
Signed-By: /etc/apt/keyrings/githubcli-archive-keyring.gpg
EOF
```

The `<<'EOF'` ... `EOF` block is a heredoc. *A heredoc is a way to feed multiple lines of text into a command without putting them in a separate file.* The single quotes around `'EOF'` prevent the shell from expanding any variables inside.

**Step 4.** Refresh the package list and install:

```
sudo apt update
sudo apt install gh
```

If everything is set up correctly, apt downloads the package list from GitHub, verifies it against the key you stored, and installs `gh` without errors. If the key is wrong or missing, apt refuses to use the repository. That refusal is the security control working.

### When you find an old tutorial

If you find instructions that say `curl ... | sudo apt-key add -`, mentally translate it to the modern pattern. The structure is always the same: download the key to `/etc/apt/keyrings/`, set 644 permissions, create a source file in `/etc/apt/sources.list.d/` that references the key with `Signed-By`. Old patterns may write the source as a one-line `.list` file instead of the new `.sources` format; both work, but `.sources` is the format to use for new additions.

### Cleaning up old `apt-key` artifacts

If you administer a system that was set up before the deprecation, it may have keys in the legacy `/etc/apt/trusted.gpg` or `/etc/apt/trusted.gpg.d/`. These still work but they grant any key the right to sign for any repository. Migrating them to per-source keyrings is a hardening task. Worth knowing it exists; worth doing on systems you control.

```
sudo apt-key list 2>/dev/null         # see legacy keys (deprecation warning is fine)
ls /etc/apt/trusted.gpg.d/             # see legacy keyring files
```

---

## snap versus apt

Ubuntu now ships with two parallel package systems: apt (the traditional one) and snap (Canonical's container-style packaging).

A snap package is a self-contained bundle that includes the program plus most of its dependencies, runs in a sandbox, and updates automatically. Snap packages are mounted as read-only filesystems under `/snap`.

Look at what snaps are installed:

```
snap list
```

On a fresh Ubuntu Server install, you see at minimum `core`, `core22` (or `core24`), and `snapd`. On a desktop install, you see Firefox and others. Some applications (Firefox on Ubuntu, for example) are now snap-only by default.

### When apt is the right choice

- The package is in Ubuntu's repositories.
- You want fast startup time.
- You want to control update timing.
- You are managing a server where automatic updates of applications are not desired.

### When snap is the right choice

- The application is officially packaged as a snap by its publisher.
- You want sandboxing for additional containment.
- You are on a desktop and the app is only available as a snap.
- You want automatic updates and do not need to control timing.

### When you have a choice and either works

Default to apt for servers. Default to whatever the distribution recommends for desktops. The reasons:

- snap startup time on a server is annoying for command-line tools.
- snap's sandboxing complicates things like using a snap-installed editor to edit `/etc` files (the sandbox prevents access).
- automatic updates on a server are often not what you want; production deployments usually want controlled change windows.

There are religious arguments about snap. They are mostly not your problem as a working admin. Use the right tool for the job and move on.

---

## Holding a package version

Sometimes you need to prevent a package from being upgraded. Examples: an application that depends on a specific kernel module, a database that has a known incompatibility with the next minor version, a pinned version required by an internal compliance baseline.

```
sudo apt-mark hold nginx
sudo apt-mark showhold
```

After holding, `apt upgrade` will skip nginx even if a newer version is available. The held state persists across reboots and survives further apt operations.

To release the hold:

```
sudo apt-mark unhold nginx
```

Holds are the right pattern for "we cannot upgrade this until we test the new version." Holding indefinitely without revisiting is a pattern auditors flag, because it leaves systems running known-vulnerable software. The discipline: every hold should have a ticket, and every ticket should have a planned expiration.

---

## Pinning: when holds are too coarse

A hold prevents all upgrades to a package. Pinning lets you express more nuanced rules: "use the version from this repository," "do not auto-upgrade past version 1.x but allow updates within 1.x," and so on.

Pinning is configured in `/etc/apt/preferences` or in drop-in files under `/etc/apt/preferences.d/`. The format is verbose. A typical example:

```
Package: nginx
Pin: version 1.24.*
Pin-Priority: 1001
```

That says "for nginx, prefer any version matching 1.24.*, with priority high enough that it overrides newer versions from the standard repos." Priority 1001 is "always prefer this even over newer versions." Priority 1 to 999 is normal preference rules.

Pinning is power-tool territory. Most working admins go years without writing a pin file. Worth knowing it exists; worth deferring to documentation when you actually need it.

---

## dpkg: the lower layer

When apt is not enough, dpkg is the tool underneath. apt calls dpkg internally for the actual install and remove operations. You use dpkg directly for two things: inspecting what is installed, and installing a `.deb` file you have on disk.

### Inspecting installed packages

```
dpkg -l                              # list every installed package
dpkg -l | grep nginx                 # check if nginx is installed
dpkg -L nginx                        # list every file the nginx package installed
dpkg -S /etc/nginx/nginx.conf        # which package installed this file?
```

`dpkg -S` is genuinely useful. When you find a strange config file on a system, it tells you which package put it there. When you find a file with no package owner, that itself is information: it was put there manually, by another tool, or by something suspicious.

### Installing a local .deb

```
sudo dpkg -i /path/to/package.deb
sudo apt --fix-broken install         # if dependencies are missing, this resolves them
```

The pattern is "use dpkg to install the file, then ask apt to fix any missing dependencies." This is acceptable when you have a `.deb` from a trusted source and there is no apt repository for it. It is a yellow flag in audits because it bypasses the repository signature check.

---

## unattended-upgrades: automatic patching

Ubuntu Server installs `unattended-upgrades` by default. *Unattended-upgrades is a service that automatically downloads and installs security updates without operator action.* It is configured to install only security updates by default, not feature updates.

This is mostly good. It means a server you set up six months ago is still receiving security patches even if you have not touched it. It is also occasionally surprising, because the default configuration includes "reboot if a kernel update requires it," and that reboot can happen at 3 AM during what you thought was stable production.

### Seeing what unattended-upgrades is doing

```
ls /var/log/unattended-upgrades/
sudo tail /var/log/unattended-upgrades/unattended-upgrades.log
```

Every run is logged. If the box rebooted unexpectedly, the log here is the first place to look.

### Configuration files

Two files control this:

`/etc/apt/apt.conf.d/50unattended-upgrades` is the main rules file. It defines what gets upgraded (which package origins are trusted), what gets blacklisted, and reboot behavior. The defaults are reasonable.

`/etc/apt/apt.conf.d/20auto-upgrades` is the schedule. It enables or disables the automatic running. By default it runs daily.

### The settings most worth changing

Open the main config:

```
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Look for these lines:

```
//Unattended-Upgrade::Automatic-Reboot "false";
//Unattended-Upgrade::Automatic-Reboot-Time "02:00";
```

The `//` at the start is a comment. Uncomment the first line and change to `true` if you want automatic reboots when needed (with a chosen time). Leave it commented out if you want to handle reboots manually.

```
//Unattended-Upgrade::Mail "your-email@example.com";
```

Uncomment to receive email when updates run. Requires a mail relay configured on the box.

```
//Unattended-Upgrade::Package-Blacklist {
//    "vim";
//    "libc6";
//};
```

Uncomment to specify packages that should never be auto-upgraded. Useful if you have software that breaks on minor updates.

After changing the config, dry-run to confirm:

```
sudo unattended-upgrade --dry-run --debug
```

That shows you what it would do without doing it. Look for "no candidate packages" (good, there is nothing to install) or a list of packages it would upgrade.

### Disabling unattended-upgrades

Some environments want full control of patch timing. To disable:

```
sudo systemctl disable --now unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

The reconfigure command lets you toggle the enabled state interactively. Choose "no."

This is a defensible choice for production servers with mature change-control processes. It is not a defensible choice for "I do not want to think about it." Disabled unattended-upgrades plus no replacement patching schedule equals "this server is running unpatched code in six months."

---

## Security and apt: the practitioner view

Tying this back to Security+ Domain 2.5 (mitigation techniques) and 4.3 (vulnerability response).

The patching pipeline on a Linux box, conceptually:

1. **Source list defines what is trusted.** Repositories with valid signatures are the trust boundary.
2. **apt update refreshes the catalog.** No installation happens; only metadata moves.
3. **apt upgrade installs available updates.** Either invoked by an admin or by unattended-upgrades.
4. **Holds and pins prevent specific updates** when there is a documented reason.
5. **Logs in /var/log/apt/ and /var/log/unattended-upgrades/** record what happened.

The audit version: a security review of a Linux box's patching posture answers these questions. Which repositories are configured. Are their keys current and isolated. Is unattended-upgrades enabled. What is on the holds list and why. What is in the apt logs for the last 30 days. When was the last successful upgrade.

You can answer all of these now.

---

## Try this

**1. Audit the repositories on your lab box.**

List every configured apt source:

```
ls /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/*.sources 2>/dev/null
cat /etc/apt/sources.list.d/*.list 2>/dev/null
cat /etc/apt/sources.list 2>/dev/null
```

For each one, identify: the URI, the suite, and which key signs it (the `Signed-By` line, or check `apt-key list` for legacy entries). Confirm each key file actually exists.

**2. Add a third-party repository correctly.**

Pick one of: Docker, GitHub CLI, or Microsoft's package repository for VS Code. Look up the official documentation (do not trust random tutorials), and add the repository using the modern keyring pattern.

After adding, run `sudo apt update` and confirm there are no errors. Then install at least one package from the new repository.

This exercise is real-world. You will do this on production servers. Now is when you build the right habit.

**3. Hold a package version.**

Choose a package you have installed (try `nginx` if it is still on the lab box from the workshop). Hold it:

```
sudo apt-mark hold nginx
```

Run `sudo apt upgrade --simulate` and confirm nginx does not appear in the upgrade list, even if there is a newer version available. Then unhold it.

**4. Use dpkg to inspect a package.**

Pick a package, any package. Run `dpkg -L <name>` and read the file list. Pick one of the config files it installed. Run `dpkg -S <path>` and confirm dpkg correctly identifies the package as the owner. This is a quick sanity check on the round-trip and a useful muscle for forensic work later.

**5. Configure unattended-upgrades the way you want it.**

Look at `/etc/apt/apt.conf.d/50unattended-upgrades` and make at least one deliberate change. Suggested choices:

- Enable email notification (and confirm whether your lab box has a mail setup).
- Enable automatic reboot at a specific time.
- Add one package to the blacklist.

Run `sudo unattended-upgrade --dry-run` and confirm your change took effect.

---

## Common stumbling blocks

> **`apt update` fails with "the following signatures couldn't be verified."**
> Either a key is missing, the wrong key is referenced, or an upstream key has rotated. Look at the line before the error: it tells you which repository is failing. Fix is to re-download the correct key into `/etc/apt/keyrings/` and confirm the source file references that file with `Signed-By`.

> **I added a repository but `apt update` does not see new packages.**
> Check the `Suites` field in your source file. The codename matters: `noble` for 24.04, `jammy` for 22.04. If the third-party repo has not yet published packages for your release, you will see no new packages. Some repos use `stable` or `main` as their suite name regardless of Ubuntu version; check the publisher's documentation.

> **`apt upgrade` says packages are kept back and does nothing for those.**
> Held packages are skipped. So are packages that would require removing other packages. Run `sudo apt list --upgradable` to see what is available. For packages held back without an explicit hold, `sudo apt full-upgrade` resolves it (with the warning that full-upgrade can remove packages).

> **Snap-installed Firefox cannot open files in `/etc`.**
> Snap sandboxing prevents access to most of the filesystem. This is by design. If you need to edit a config file with a graphical editor, use a non-snap editor or copy the file to your home directory first, edit, then copy back with sudo.

> **unattended-upgrades rebooted the box at 3 AM and I do not want that.**
> Open `/etc/apt/apt.conf.d/50unattended-upgrades` and set `Unattended-Upgrade::Automatic-Reboot "false"`. The package will now upgrade but never reboot. You will need to reboot manually for kernel updates to take effect; check `/var/run/reboot-required` to know when one is pending.

> **I installed something with `dpkg -i` and now apt complains about broken dependencies.**
> Run `sudo apt --fix-broken install`. apt will figure out what is missing and install it. If that fails, the `.deb` file requires versions of dependencies that are not in your configured repositories, in which case you have to either update the package or accept that it will not run.

---

## What this gets you

After this chapter:

- You can read the modern apt source format and explain every field.
- You can add a third-party apt repository the right way, with a per-source key.
- You know why `apt-key` is deprecated and what replaces it.
- You can choose between snap and apt deliberately rather than by default.
- You can hold or pin a package when you have a reason, and you know not to leave holds in place forever.
- You can read what unattended-upgrades is doing and configure it the way you want.
- You can use dpkg directly when apt is not enough, and you know when not to.

This chapter is the foundation for any conversation about patch management. Security+ tests it at the conceptual level. Working admins do it weekly. The cert and the job converge here.

---

## What's next

Chapter 5 is systemd units and timers. The chapter where you stop running services with the verbs you learned in the workshop and start writing your own. By the end, you will have written a service unit, a timer unit, and a user unit, and you will be able to read existing units on a system you did not build.
