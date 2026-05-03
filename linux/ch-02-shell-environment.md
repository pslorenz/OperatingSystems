# Chapter 2: Shell Environment and Vi Survival

**You come in with:** ability to run commands. You have noticed the prompt looks a certain way and that some things you type seem to be remembered.
**You leave with:** a customized shell that fits how you work, an alias that survives logout, an understanding of where dotfiles get sourced, and enough vi to fix a config file on any Linux box where nano is missing.

**Time:** 60 to 90 minutes including the exercises and the vi practice. This is the longest chapter early in the unit; it is fine to do in two sittings.

**Security+ alignment:** No direct exam content. Foundation skill. The bash history settings introduced here matter for Chapter 11 (Linux artifacts) which aligns with Domain 2.4 (indicators of malicious activity) and Domain 4.9 (data sources for investigation).

---

## Why this chapter exists

Most beginners use a default shell forever. They never customize the prompt. Their aliases do not stick. They cannot explain why their `.bashrc` change had no effect when they SSHed back in. They use nano on every box because they were taught nano.

This chapter fixes all of that in about an hour. By the end, your shell works the way you work, you can read someone else's dotfiles without confusion, and you can edit a config file in vi without panicking, breaking it, or being unable to quit.

There is a vi survival segment in the middle of this chapter, around the dotfile section. Vi is in this chapter rather than as its own chapter because by the time you need to edit a dotfile, you need an editor. Vi is the editor that exists everywhere. Five to fifteen minutes of vi practice now saves you from the "I SSHed into a server and nano is not installed and now I cannot fix anything" moment that happens to every junior admin eventually.

---

## What the shell actually is

The terminal is the window. The shell is the program inside the terminal that reads what you type, interprets it, and runs commands. On Ubuntu, the default shell is bash. There are others (zsh, fish, dash, sh) and you may switch later, but bash is what every Ubuntu box ships with and what we are using.

The distinction matters because some things you do affect the terminal (window size, scroll buffer) and some affect the shell (prompt, aliases, environment). When something is not behaving the way you expect, knowing whether to look at the terminal or the shell saves time.

Run this:

```
echo $SHELL
echo $0
```

`$SHELL` is the shell defined for your user. `$0` is the program currently running. If they are both `/bin/bash` or `/usr/bin/bash`, you are in bash. The shell process started when you logged in (or opened a new terminal) and it ends when you log out.

---

## How the shell finds programs

When you type `ls`, the shell does not know what `ls` means as a built-in. It searches a list of directories, in order, looking for a file named `ls`. The list is the PATH variable.

```
echo $PATH
```

The output is a colon-separated list of directories. On a default Ubuntu install, it looks something like:

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

When you type `ls`, the shell tries `/usr/local/sbin/ls`, then `/usr/local/bin/ls`, then `/usr/sbin/ls`, then `/usr/bin/ls`, and stops at the first one it finds. That order matters. If you put a directory at the front of PATH and put a file named `ls` in it, your `ls` runs instead of the system one. That is sometimes useful, often a security concern (PATH manipulation is a real attack pattern), and always worth understanding.

To see which one the shell would actually run:

```
which ls
type ls
```

`which` searches PATH and shows you the file. `type` is a bash built-in that does the same thing but tells you more (whether it is an alias, a function, a built-in, or an external program). Use `type` when you are not sure.

### Adding to PATH

If you write a script and put it in `/home/student/bin`, typing the script's name will not find it, because that directory is not in PATH. You have two options.

The crude option is to type the full path every time:

```
/home/student/bin/myscript
```

The right option is to add `/home/student/bin` to PATH. We do that in the dotfile section below.

---

## Environment variables

PATH is one of many environment variables. *An environment variable is a named value the shell keeps in memory and passes to every program it starts. Programs read environment variables to decide how to behave.*

To see all of them:

```
env
```

A few you will see often:

- `HOME`: your home directory. The shell uses this to expand `~`.
- `USER`: your username.
- `SHELL`: your default shell.
- `PATH`: the program search list.
- `EDITOR`: your preferred editor. Programs that need to launch an editor (like `git commit`, `crontab -e`, `visudo`) check this.
- `LANG`: your locale. Affects how dates and numbers are formatted.
- `TERM`: the terminal type. Tells programs what features the terminal supports.

To set one for the current shell:

```
export EDITOR=nano
```

That works until you log out. To make it stick, you put it in a dotfile.

To read one:

```
echo $EDITOR
```

The dollar sign tells the shell "expand this variable." Without it, the shell would just print the literal string `EDITOR`.

---

## The dotfile chain

When bash starts, it reads a series of files to set up your environment. *A dotfile is any file whose name starts with a `.` (a literal period). They are hidden by default in `ls`. Bash reads several of them at startup to load aliases, environment variables, and customizations.* The order depends on whether the shell is a login shell, an interactive shell, or both. This is where most "my .bashrc change did nothing" confusion comes from.

The simplified rules for Ubuntu:

- **SSH login or terminal login**: bash reads `/etc/profile`, then it reads the first one of `~/.bash_profile`, `~/.bash_login`, `~/.profile` that exists. On Ubuntu, `~/.profile` is the standard one and it sources `~/.bashrc` itself.
- **New terminal window in a desktop session, or `bash` started inside an existing shell**: bash reads `/etc/bash.bashrc` then `~/.bashrc`.

The practical takeaway: put your customizations in `~/.bashrc`. Ubuntu's default `~/.profile` already sources `.bashrc` for login shells, so a setting in `.bashrc` works for both.

Look at your `~/.profile`:

```
cat ~/.profile
```

Toward the end you will see something like:

```
if [ -n "$BASH_VERSION" ]; then
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi
```

That block is the line that makes `.bashrc` work for login shells. Without it, your aliases would not work over SSH. With it, you only need to edit one file.

---

## A vi survival break

You are about to edit `~/.bashrc`. You can use nano if it is installed, which on Ubuntu desktop and server it usually is. But this is a good moment to learn vi instead, because vi is on every Linux box and nano is not. Take ten minutes here. Vi is genuinely uncomfortable the first time. After ten minutes you will not love it, but you will be able to edit a file without panicking.

### The mental model that makes vi make sense

Vi has modes. Most editors do not. That is the source of every beginner stumble.

In **command mode**, your keystrokes are commands. Pressing `i` does not type the letter i. It enters insert mode.

In **insert mode**, your keystrokes are text. Typing letters types letters. Pressing Esc returns you to command mode.

You start in command mode. Always. The very first thing to do in vi is press `i` so you can actually type.

That is the whole model. Two modes. You switch between them.

### Open vi on a real file

Use a scratch file, not anything important:

```
vi ~/scratch.txt
```

You are in command mode. Your keystrokes are commands. Do not type yet.

### The four commands that get you out alive

These are the four you absolutely need. Memorize them. Practice them. Everything else is a nice-to-have.

| Action | Keys |
|---|---|
| Enter insert mode (start typing) | `i` |
| Leave insert mode | `Esc` |
| Save and quit | `:wq` then Enter |
| Quit without saving | `:q!` then Enter |

`:wq` and `:q!` start with a colon. The colon enters command-line mode (a third mode, technically), which is where the multi-character commands live. After typing the command, press Enter to execute it.

The pattern: press Esc to make sure you are in command mode, then type your command. If you are not sure what mode you are in, press Esc. Esc is always safe.

### Try it

In your `vi ~/scratch.txt` session:

1. Press `i`. Look at the bottom of the screen. It says `-- INSERT --`.
2. Type a sentence. Any sentence.
3. Press `Esc`. The `-- INSERT --` indicator disappears.
4. Type `:wq` and press Enter.

You are back at the shell prompt. Confirm the file saved:

```
cat ~/scratch.txt
```

You just edited a file in vi. The first time is the hardest. It does not get worse from here.

### A few more commands worth knowing

Once the four basics are muscle memory, these expand what you can do without leaving command mode:

| Action | Keys |
|---|---|
| Delete the current line | `dd` |
| Undo the last change | `u` |
| Redo (undo the undo) | `Ctrl-r` |
| Search forward for a string | `/string` then Enter |
| Next match after search | `n` |
| Go to the start of the file | `gg` |
| Go to the end of the file | `G` (capital G) |
| Go to a specific line number | `:42` then Enter |

You do not need these for the chapter exercises. They are listed because by the time vi feels less awful, these are the first additions worth making.

### Why you do not need to like vi

A common reaction at this point is "this is terrible, why does anyone use this on purpose." Two honest answers.

First, you do not need to use vi on purpose. Use whichever editor is in front of you. On your daily work machine, that probably means VS Code or Sublime or whatever you actually like. Vi is an emergency editor for when you SSH into a stripped-down box and discover nano was not installed. The commands above are enough for that case.

Second, the people who use vi on purpose are using vim, which is a different tool that happens to share the modal model. Vim has dozens of features that make it powerful (text objects, macros, plugins). What you learned here is plain vi. Vim is a possible future. It does not have to be your future.

The rest of this chapter assumes you can edit `~/.bashrc` with either vi or nano. Use whichever you prefer. If you used vi and survived, give yourself credit. The rest is easy.

---

## Customizing your shell

Now we edit `~/.bashrc` to add things that make daily work less annoying.

Open it:

```
vi ~/.bashrc
```

(Or `nano ~/.bashrc`. Your call.)

Read what is there. Ubuntu's default `~/.bashrc` is well-commented and has examples of common customizations, mostly disabled. You can enable some by uncommenting lines. We are going to add a few things at the end of the file.

Move to the end. In vi: press `Esc` to make sure you are in command mode, then `G` to go to the end. Press `i` to start inserting. (In nano: just scroll down with arrow keys.)

### Your prompt

The prompt is controlled by `PS1`. Ubuntu's default looks something like:

```
student@labbox:~$
```

You can change what goes there. Add this line at the end of `~/.bashrc`:

```bash
PS1='\u@\h \[\e[36m\]\w\[\e[0m\] $ '
```

That sets a prompt with your username, hostname, and the working directory in cyan. The `\u`, `\h`, `\w` are escape codes bash understands. The `\[\e[36m\]` is an ANSI color code (cyan), and `\[\e[0m\]` resets the color.

You do not need to memorize the codes. Most people copy a PS1 they like from somewhere and tweak it. Common variables:

- `\u`: username
- `\h`: short hostname
- `\H`: full hostname
- `\w`: full working directory
- `\W`: just the basename of the working directory
- `\$`: `$` for regular users, `#` for root
- `\t`: current time in 24-hour format

### Useful aliases

Aliases are shortcuts. You type the alias, the shell expands it to the full command. Add these:

```bash
alias ll='ls -la'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
alias gr='grep --color=auto'
alias h='history'
```

`ll` is the one most admins have. Long-format `ls` with hidden files. You will use it constantly.

The `..` and `...` aliases let you go up directories without typing `cd`. Save a few keystrokes a hundred times a day, and the savings add up.

### A useful HISTSIZE and HISTTIMEFORMAT

Bash by default remembers the last 1000 commands. That is fine for casual use but small for real work. Also by default, the history does not include timestamps, which makes "what did I do yesterday at 2 PM" impossible to answer.

```bash
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTTIMEFORMAT='%F %T  '
export HISTCONTROL=ignoredups:erasedups
```

`HISTSIZE` is how many commands to keep in memory for this session. `HISTFILESIZE` is how many to keep in `~/.bash_history` on disk. `HISTTIMEFORMAT` adds timestamps. `HISTCONTROL=ignoredups:erasedups` means duplicate commands are not added, and existing duplicates are removed.

After this is set and you have run a few commands, look at your history:

```
history | tail -20
```

You will see timestamps before each command. That is suddenly a useful audit trail.

> **A note on history and security.** Bash history is one of the first things a security investigator looks at when triaging a Linux host. It is also one of the first things attackers try to clear. A larger history with timestamps is more useful for forensics. We come back to this in Chapter 11.

### Adding a personal bin to PATH

If you write scripts, you need somewhere to put them. The convention is `~/bin` or `~/.local/bin`. Ubuntu's default `~/.profile` already adds `~/bin` to PATH if it exists. To use it:

```
mkdir -p ~/bin
```

Then any executable you put in `~/bin` will be findable by name from anywhere. We come back to this in Chapter 9 (Shell scripting).

### Saving and reloading

Save the file and quit your editor. In vi: `Esc`, then `:wq`, then Enter. In nano: `Ctrl-O`, Enter, `Ctrl-X`.

The changes are not active yet. The shell only reads `.bashrc` when it starts. You have two options.

The fast option is to source the file in your current shell:

```
source ~/.bashrc
```

Or the shorthand:

```
. ~/.bashrc
```

(That is a single dot, then a space, then the path.) `source` runs the file as if you had typed each line.

The complete option is to log out and log back in. That confirms your changes work for fresh sessions, not just the current one. Try this:

```
exit
```

Then SSH back in. Your prompt should be the customized one. Run `ll`. It should work. Run `history`. You should see timestamps.

If something does not work, the most common cause is a typo in `~/.bashrc` that broke everything after it. Look for messages when you log in. If you see "syntax error near unexpected token," you typoed something. Open `~/.bashrc` and read the line numbers in the error message.

---

## Why your alias did not stick: a worked example

The single most common beginner question about shells is some version of "I added an alias but it did not work." Walk through the actual chain.

You typed an alias at the command line:

```
alias gs='git status'
```

It worked in that session. You used `gs` for an hour, it was great. You logged out. You SSHed back in. `gs` was gone.

What happened: typing an alias at the command line creates it for the current shell only. When the shell ends (you log out), everything in its memory is gone. To make an alias permanent, you have to put it in a file the shell reads when it starts.

The file is `~/.bashrc` (because of the chain we walked through above). If you put `alias gs='git status'` at the end of `~/.bashrc`, every new shell will define it.

But you have to get a new shell. Editing the file does not retroactively apply to running shells. You either source the file (`source ~/.bashrc`) or open a new shell.

If you put the alias in `~/.bashrc` and it still does not work after a new shell, two things to check:

1. Did you save the file? Editors lie sometimes. `cat ~/.bashrc | tail` and look.
2. Is the line before it broken? A syntax error in `~/.bashrc` causes the rest of the file to not be processed. The shell may not show an error if the broken part was reached after some valid lines. Read the bottom of the file carefully for missing quotes or unclosed brackets.

This is the kind of debugging chain that is annoying the first time and intuitive after the third time. Walk it now.

---

## A tour of useful bits you did not know were there

A few small wins that come up constantly.

### Tab completion

You can type the first few characters of a path or command and press Tab. The shell completes it if there is a unique match. If there is not, pressing Tab twice shows you the options.

```
cd /et<Tab>
```

Becomes `cd /etc/`. Try it on your home directory:

```
cd ~/<Tab><Tab>
```

You see your home directory contents. Useful when you forget what is in a directory.

### Command history navigation

The up and down arrow keys cycle through your history. Ctrl-R searches it. Try:

- Press `Ctrl-R`. The prompt changes to `(reverse-i-search)`.
- Type a few characters from a command you ran earlier.
- It shows the most recent matching command.
- Press `Ctrl-R` again to see older matches.
- Press Enter to run the displayed command, or arrow keys to edit it.

Once this becomes muscle memory, you stop retyping commands.

### Globbing

The shell expands certain characters before running a command:

- `*` matches any string of characters.
- `?` matches a single character.
- `{a,b,c}` expands to each option.

```
ls /etc/*.conf
ls /etc/?ostname
echo {one,two,three}
```

That is why `ls /etc/*.conf` lists every `.conf` file. The shell expands the `*` before `ls` even runs.

### Quoting

Single quotes mean "literal." Double quotes mean "almost literal, but expand variables." No quotes mean "split on spaces and expand globs and variables."

```
echo Hello $USER
echo "Hello $USER"
echo 'Hello $USER'
```

The first two print the same thing. The third prints `Hello $USER` literally, because single quotes prevent expansion.

This becomes important in scripts. Variables that might contain spaces need double quotes, or the shell will split them. Variables that contain special characters need protection. We cover this in Chapter 9.

---

## Try this

Three exercises. Use vi for at least one of them. Real practice is the only way the muscle memory builds.

**1. Make your alias survive logout.**

Add an alias to `~/.bashrc` that runs `df -h` when you type `disk`. Save the file. Open a new shell. Type `disk`. Confirm it shows your disk usage in human-readable form.

This is the simplest possible "make a change, confirm it persists" exercise. If it works, the dotfile chain makes sense to you.

**2. Customize your prompt.**

Edit `~/.bashrc` and change your `PS1` to show the time at the start of the prompt. Reload your shell. The prompt should now show the current time, like:

```
14:23  student@labbox ~ $
```

Hint: `\t` is 24-hour time. Look back at the prompt variables list above.

**3. Audit your PATH.**

Run `echo $PATH | tr ':' '\n'`. Each line is one directory the shell searches for commands.

For each directory in your PATH, in order, check if it exists with `ls -d`. If a directory in PATH does not exist, the shell harmlessly skips it, but the entry is still cluttering your environment.

Bonus: write a one-liner that prints which PATH directories actually exist on this box. Hint: `for dir in $(echo $PATH | tr ':' ' '); do [ -d "$dir" ] && echo "$dir"; done`.

That one-liner uses bash features we have not formally covered (the `for` loop, the test command). You can copy it as a black box for now, or stretch yourself and read each part.

---

## Common stumbling blocks

> **My alias works in the current shell but disappears when I log out.**
> The alias is only saved when you put it in `~/.bashrc` and save the file. Re-read the "Why your alias did not stick" section above. The fast diagnosis: run `cat ~/.bashrc | grep alias` and confirm your line is actually saved in the file.

> **I edited `~/.bashrc` and now nothing in my shell works right.**
> A syntax error in `~/.bashrc` (a missing quote, an unclosed bracket) causes bash to stop processing the file at the broken line, so everything after it silently fails. Open `~/.bashrc` and look for missing quotes, especially around your `PS1` line. Even better, comment out your recent additions one by one until your shell works again, then add them back one by one.

> **In vi I press the arrow keys and they type letters instead of moving the cursor.**
> You are in insert mode and the keystrokes are being interpreted as text. Press `Esc` to leave insert mode. The arrow keys then move normally. (Some old systems handle arrow keys in insert mode differently. Modern Ubuntu's vim-tiny does it correctly.)

> **I cannot quit vi.**
> Press `Esc` first to make sure you are out of any mode you are in. Then type `:q!` and press Enter. The exclamation point means "quit no matter what, abandoning changes." If `:q!` does not work either, you typoed something earlier. Press `Esc` once more, then `:q!` again. Esc plus `:q!` always works on a real vi.

> **I customized my prompt and now the wrap is broken (text overlaps when lines get long).**
> Color escape codes need to be wrapped in `\[` and `\]` so bash knows they are zero-width. Without those wrappers, bash counts the color escape characters as visible text and miscalculates where the cursor is. Compare your `PS1` line carefully to the example in this chapter; every `\e[...m` should be inside `\[...\]`.

> **I added a directory to PATH but commands in it still are not found.**
> Either you did not source `~/.bashrc` after the change (run `source ~/.bashrc` or open a new shell), or the `export PATH=...` line is in the wrong file (use `~/.bashrc`, not `~/.bash_profile` on Ubuntu), or the directory does not actually exist (try `ls -d /your/path` to confirm).

---

## What this gets you

After this chapter:

- Your shell looks and behaves the way you want.
- Your aliases survive logout.
- You can edit a config file on any Linux box, with or without nano.
- You understand why `~/.bashrc` exists, what reads it, and when.
- Your bash history is large, timestamped, and useful for retracing your steps.

The vi survival is the part that pays off years from now. The first time you SSH into a stripped-down container or a recovery shell and nano is missing, you will be glad you spent ten minutes on this.

---

## What's next

Chapter 3 is Permissions deep dive. The chapter where 777 stops being an option and you learn the patterns a real admin uses. Set yourself up for it: read `~/.bashrc` once more before you move on, and notice how the conventions you saw there (lowercase variable names, simple syntax, short comments) repeat throughout the system.
