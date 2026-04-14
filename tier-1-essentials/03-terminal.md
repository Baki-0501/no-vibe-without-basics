# Terminal Fundamentals

## What is it?

The terminal is a text-based interface to your computer. Instead of clicking icons and dragging windows, you type commands and the computer responds with text. Every command you run is a program — the terminal is just a shell that lets you invoke those programs by name, pass them arguments, and chain them together.

Think of it like a restaurant kitchen: you (the user) send orders (commands) to the kitchen (the shell), and the kitchen sends back plated dishes (output). The terminal itself doesn't do the work — it just relays your instructions to the operating system's program layer.

On Linux and macOS, the default shell is **Bash** (Bourne Again Shell). On Windows, the recommended approach is **WSL 2** (Windows Subsystem for Linux), which gives you a genuine Linux kernel running inside Windows with a Bash shell. The commands in this module are Bash commands unless noted otherwise.

## Why it matters for vibe coding

AI coding tools — Claude Code, Cursor, Copilot — frequently generate shell commands for you to run. Without terminal literacy, you face a critical failure mode: **you run commands you don't understand, and you can't diagnose the consequences when they fail or do something unexpected.**

Concrete examples:

- **AI generates a `find` command that deletes files**: Without understanding pipes and redirects, you won't notice that `find . -name "*.txt" | xargs rm -f` is about to wipe your entire project directory. You just see "clean up old files" and hit Enter.
- **AI generates a script that modifies `/etc/hosts`**: The command looks like a simple DNS fix. You don't realize it's touching a system file that could break your network if the AI gets the IP wrong. You run it, lose connectivity, and have no idea what happened.
- **AI generates a one-liner using `curl | sh`**: This pattern — download and execute in one step — is a classic infection vector. Without understanding pipes and what `sh` does, you can't recognize that you're handing a stranger the keys to your machine.
- **AI generates a remote server command via SSH**: You don't understand what flags like `-i` (identity file) or `>` (redirect) do, so you can't verify the AI's suggestion is appropriate before running it against a production server.
- **AI generates commands that set environment variables**: You paste `export OPENAI_API_KEY=sk-xxxx` into your terminal and it works, but you don't realize you've just set a variable in your current shell — close the terminal and it's gone. Meanwhile your secrets are still exposed in your shell history.

The thread connecting all these failures: **you cannot critically evaluate AI output if the output is in a language you don't understand.**

## The 20% you need to know

This section covers the concepts that cover 80% of real situations. Everything here is Bash on Linux/macOS (or WSL2 on Windows).

### Navigation and file operations

The filesystem is a tree. Every file and folder has a path. Your current location is called the **working directory**.

```bash
pwd                  # Print working directory — shows where you are
ls                   # List files in current directory
ls -la               # List all files including hidden ones, with details
cd /path/to/dir      # Change directory to absolute path
cd ./relative/path   # Change directory relative to current location
cd ..                # Go up one directory (parent)
cd ~                 # Go to your home directory
mkdir newdir          # Create a directory
touch file.txt        # Create an empty file
rm file.txt          # Remove a file (permanent — no recycle bin)
rm -rf directory     # Remove a directory and everything inside it (-r recursive, -f force)
cp source dest       # Copy a file
mv source dest       # Move or rename a file
```

The `-la` flags: `-l` gives detailed output, `-a` shows hidden files (those starting with `.`).

**Important:** `rm -rf` is the nuclear option. `rm -rf /` would delete everything on your system. AI tools sometimes generate commands like this — always read the full command before hitting Enter.

### Pipes and redirects

Pipes (`|`) connect the output of one command to the input of another. Redirects (`>` and `>>`) send output to a file instead of the terminal.

```bash
ls -la | grep ".md"       # List files, then filter to only lines containing ".md"
cat file.txt | wc -l       # Count lines in file.txt (wc = word count, -l = lines)
echo "hello" > out.txt     # Write "hello" to out.txt (overwrites existing content)
echo "world" >> out.txt     # Append "world" to out.txt (keeps existing content)
command > /dev/null         # Discard output entirely — send to the null device
```

The `/dev/null` trick is useful when a command produces noisy output you don't care about. But be careful: if you're redirecting errors with `2>/dev/null`, you're discarding diagnostic information that might tell you why the command failed.

### Environment variables

Environment variables are named values that persist in your shell session and get passed to every program you run. They're how you configure tools without hardcoding values.

```bash
echo $HOME                    # Print the value of HOME
export MY_VAR="hello"         # Set a variable (only persists in this shell session)
echo $MY_VAR                  # Print it
printenv                      # Print all environment variables
printenv PATH                 # Print just PATH
```

The critical pattern for vibe coding: **API keys and secrets should always be environment variables, never hardcoded**. When an AI tells you to run `export OPENAI_API_KEY=sk-xxxx`, understand that this variable lives in your shell history. A better pattern is storing secrets in a `.env` file and using a tool like `dotenv` or `direnv` to load them — or using your shell's per-project environment management.

Shell history is a security risk. Anyone with access to `~/.bash_history` can see every command you've typed, including exported secrets. Use `HISTCONTROL=ignoredups` or `HISTIGNORE` to suppress sensitive commands.

### SSH

SSH (Secure Shell) lets you connect to remote machines and run commands on them. This is the backbone of remote development, CI/CD, and server management.

```bash
ssh user@hostname                 # Connect to a remote server
ssh -i key.pem user@hostname      # Connect using a specific private key file
ssh -p 2222 user@hostname         # Connect on a non-default port (22 is default)
```

The `-i` flag specifies an identity file (private key). This is how you authenticate without a password. **Never share your private key.** Only ever share the public key (`.pub` file).

When vibe coding, you might ask an AI to generate an SSH command to deploy code or pull from a repo on a remote server. Before running it, verify: which user? which host? which port? is the key path correct? is the command idempotent (can it safely run twice)?

### Process management

Programs running on your machine are **processes**. You can inspect and control them.

```bash
ps aux                # List all running processes (a = all users, u = detailed, x = not attached to a terminal)
top                   # Show real-time process usage (press q to quit)
htop                  # Nicer real-time process viewer (if installed)
kill <pid>            # Send SIGTERM (graceful shutdown) to a process by PID
kill -9 <pid>         # Send SIGKILL (force kill) — use only when graceful shutdown fails
pkill -f "python"     # Kill all processes matching "python" by name
```

PIDs (Process IDs) are assigned by the OS. When a program hangs, `kill` is how you stop it. Start with plain `kill` (SIGTERM) — it gives the program a chance to clean up. Only use `kill -9` when the program is unresponsive and you need to force it dead.

### tmux

tmux (Terminal Multiplexer) lets you run multiple terminal sessions in one window. Sessions persist even when you disconnect — your remote work keeps running if your laptop sleeps or your SSH connection drops.

```bash
tmux new -s mysession       # Create a new named session
tmux attach -t mysession    # Attach to an existing session
```

tmux is essential for remote development because:

- You start a long-running process in one pane
- You close your laptop, drive home, reconnect, and it's still running
- You can split one window into multiple panes (vertical/horizontal) and run different commands side by side

Core tmux commands (prefixed by `Ctrl+b` by default):

- `Ctrl+b d` — Detach from session (leave it running)
- `Ctrl+b c` — Create a new window
- `Ctrl+b %` — Split pane horizontally
- `Ctrl+b "` — Split pane vertically

### Shell scripting basics

A shell script is a text file containing terminal commands that run sequentially. You can automate repetitive tasks.

```bash
#!/bin/bash
# This is a comment
echo "Hello, world"
```

To make a script executable:

```bash
chmod +x myscript.sh    # Add execute permission
./myscript.sh           # Run it
```

Common patterns in AI-generated scripts:

```bash
if [ -f "file.txt" ]; then
    echo "file exists"
fi

for file in *.txt; do
    echo "Processing $file"
done
```

**The `#!/bin/bash` shebang line at the top is not optional.** It tells the OS which interpreter to use to parse the script. Without it, the shell might not parse it correctly.

For vibe coding: when AI generates a shell script, read it fully before running it with `chmod +x` or executing it. Look for what the script does to your filesystem, what it downloads from the internet, and whether it requires any arguments.

### Package managers and installing tools

On Linux/macOS, you install tools via package managers. The main ones:

```bash
# macOS
brew install tmux          # Install tmux via Homebrew

# Ubuntu/Debian (including WSL2)
sudo apt update             # Update package lists
sudo apt install tmux       # Install tmux
```

The `sudo` prefix runs the command as the superuser. This is required for installing system packages. But be deliberate about when you use it — `sudo` grants elevated privileges, and AI-generated commands that include `sudo` without explanation deserve scrutiny.

## Hands-on exercise

**Goal:** Create a small script that finds all markdown files in a directory tree, counts their total lines, and writes the result to a report file.

**Time:** Under 15 minutes.

**Step 1: Create a test directory with some files**

```bash
mkdir -p ~/vibe-terminal-practice/subdir1/subdir2
echo "# Hello" > ~/vibe-terminal-practice/file1.md
echo "## Section\n\nSome text here." > ~/vibe-terminal-practice/subdir1/file2.md
echo "More content\nLine three" > ~/vibe-terminal-practice/subdir1/subdir2/file3.md
```

**Step 2: Navigate to your home directory and verify structure**

```bash
cd ~
ls vibe-terminal-practice/
ls -R vibe-terminal-practice/    # -R = recursive listing
```

**Step 3: Find all `.md` files and count their total lines**

```bash
find ~/vibe-terminal-practice -name "*.md" | xargs wc -l
```

You should see output like:
```
  1 /Users/you/vibe-terminal-practice/file1.md
  4 /Users/you/vibe-terminal-practice/subdir1/file2.md
  3 /Users/you/vibe-terminal-practice/subdir1/subdir2/file3.md
  8 total
```

**Step 4: Write the result to a report file**

```bash
echo "Markdown line count report - $(date)" > ~/vibe-terminal-practice/report.txt
find ~/vibe-terminal-practice -name "*.md" | xargs wc -l >> ~/vibe-terminal-practice/report.txt
cat ~/vibe-terminal-practice/report.txt
```

**Step 5: Make a reusable script**

```bash
cat << 'EOF' > ~/vibe-terminal-practice/count-md.sh
#!/bin/bash
# Count total lines in all markdown files under a directory

TARGET_DIR="${1:-.}"
TOTAL=$(find "$TARGET_DIR" -name "*.md" -exec wc -l {} + | tail -1 | awk '{print $1}')

echo "Total lines in .md files under $TARGET_DIR: $TOTAL"
EOF

chmod +x ~/vibe-terminal-practice/count-md.sh
~/vibe-terminal-practice/count-md.sh ~/vibe-terminal-practice
```

**Expected output:**
```
Total lines in .md files under /Users/you/vibe-terminal-practice: 8
```

**What just happened:** You used pipes (`|`) to chain `find` output into `wc -l` for counting, redirected output to a file with `>>`, created an executable script with a shebang, and passed a command-line argument (`$1`) to make the script reusable. This is the exact workflow you use when AI generates scripts for you — now you understand the pieces.

**Cleanup:**

```bash
rm -rf ~/vibe-terminal-practice
```

## Common mistakes

### Mistake 1: Running `rm -rf` without reading the full path

**What happens:** A command like `rm -rf ~/vibe-coding-project/node_modules` accidentally becomes `rm -rf /` if the variable is empty.

**Why it happens:** Bash doesn't ask for confirmation. The `-f` flag forces deletion without prompts, and `rm` is permanent — there's no recycle bin, no undo.

**How to avoid:** Always run `ls` on a path before deleting it. Use `ls -d /path/to/dir` to confirm the directory exists and is the one you mean. If the AI generates a `rm -rf` command, read the full path twice before running it.

### Mistake 2: Not understanding what a pipe does to your data

**What happens:** `cat largefile.txt | head -20` — this works because `head` reads only the first 20 lines. But `cat largefile.txt | grep "pattern"` loads the entire file into memory, then filters it. For very large files, this causes memory exhaustion.

**Why it happens:** Pipes connect program outputs to inputs. Each program in the chain handles data differently. Some stream (process line by line), some buffer (load all into memory first).

**How to avoid:** Use tools that stream when possible. `grep "pattern" largefile.txt` (direct file argument) is more memory-efficient than `cat largefile.txt | grep "pattern"`. Save the pipe for chaining built-in Unix tools where you know the data sizes.

### Mistake 3: Accidentally running commands in the wrong directory

**What happens:** You run `npm install` in your home directory instead of your project directory, installing packages globally and causing version conflicts.

**Why it happens:** The terminal's working directory is easy to lose track of, especially with nested shell sessions or tmux panes.

**How to avoid:** Run `pwd` before running commands that modify files or install packages. Use absolute paths (`/home/user/project/src`) when working in complex directory trees. Configure your shell prompt to show the current directory (most modern prompts do this by default).

### Mistake 4: Secrets in shell history

**What happens:** You run `export API_KEY=sk-xxxx` to set an API key, and it gets written to `~/.bash_history`. Anyone who accesses that file — or a shared dev machine — sees your credentials.

**Why it happens:** By default, every command is logged to the history file. Exporting a variable doesn't redact it from the log.

**How to avoid:** Use `.env` files for secrets and load them with `source .env` (and add `.env` to `.gitignore`). Configure `HISTIGNORE` to exclude commands containing `KEY`, `PASS`, `SECRET`, or `TOKEN`. For high-sensitivity environments, use a secrets manager (AWS Secrets Manager, HashiCorp Vault) instead of environment variables.

### Mistake 5: Using `curl | sh` to install software

**What happens:** You run `curl https://install.example.com/setup.sh | sh` and the script silently downloads and executes code with your root privileges. If the URL is compromised or the script contains malicious code, you're fully compromised.

**Why it happens:** The internet is not a trusted source. This install pattern is a favorite of open source projects, but it's also a classic infection vector.

**How to avoid:** Always inspect scripts before running them. Replace `curl | sh` with:

```bash
curl -o install.sh https://install.example.com/setup.sh
less install.sh    # Read it — 'q' to quit
sh install.sh      # Only run after inspection
```

For popular tools, use official package managers (`brew install`, `apt install`) instead.

## AI-specific pitfalls

### Pitfall 1: AI generates commands that modify system files without warning

AI tools commonly generate commands that touch `/etc/hosts`, `/etc/sudoers`, or other privileged system files, framing them as simple fixes ("just add this line to your hosts file"). The AI often doesn't ask you to confirm the full path or warn about the consequences of an incorrect entry.

**What to look for:** Commands containing `/etc/`, `/usr/`, `/var/`, or any path outside your home directory. Commands prefixed with `sudo`. Commands that edit system configuration files.

**How to respond:** Stop and verify before running. Ask the AI: what does this file do? what happens if the entry is wrong? can we undo this? For `/etc/hosts` specifically: an incorrect IP entry will make that domain unresolvable or point it to the wrong server.

### Pitfall 2: AI doesn't explain what flags in a command actually do

You ask AI to "speed up the build" and it gives you `find . -type f -name "*.pyc" -delete`. You run it, and later you can't import those modules. The AI generated a valid command but never explained that `-type f` matches files, `-name "*.pyc"` matches compiled Python cache, and `-delete` removes them — which breaks any running Python process that was using those cached files.

**What to look for:** Commands with flags (`-`, `--`) that don't appear in the AI's explanation. Any command that modifies, deletes, or moves data.

**How to respond:** Run `man <command>` (e.g., `man find`) to read the manual page. Use `command --help` as a quick flag reference. If the AI's explanation doesn't cover the flags, ask for clarification before running.

### Pitfall 3: AI hallucinating invalid flags or options

AI confidently generates `grep -z "pattern" file.txt` (the `-z` flag doesn't exist in GNU grep, and means something different in BSD grep). You run it and get `grep: invalid option`. This happens because AI training data includes many versions of Unix tools with varying flag sets, and it mixes up which options belong to which version.

**What to look for:** Flags you're not familiar with. Unusual flag combinations. Commands that fail immediately with "invalid option" errors.

**How to respond:** Verify with `man command` or `command --help`. If a flag doesn't work, remove it and check if the command still achieves your goal. When in doubt, build the command incrementally — start with the simplest form and add flags one at a time.

### Pitfall 4: AI generating SSH commands that connect to the wrong host

AI might suggest `ssh user@example.com` when you meant `ssh deploy@prod-server.internal`. You run the command and connect to a machine you didn't intend, exposing your credentials or the wrong environment. Or AI generates an SSH command that doesn't include the right identity file (`-i key.pem`), causing authentication failures on well-configured servers.

**What to look for:** SSH commands that lack `-i` when connecting to servers requiring key-based auth. SSH commands that use generic hostnames instead of fully-qualified domain names.

**How to respond:** Always verify the hostname, port, user, and identity file path in any SSH command before running it. Keep a known-hosts list so you notice when you connect to an unexpected server.

## Quick reference

### Navigation

| Command | What it does |
|---|---|
| `pwd` | Print working directory |
| `ls` | List files |
| `ls -la` | List all files with details |
| `cd <path>` | Change directory |
| `ls -R` | Recursive listing |

### File operations

| Command | What it does |
|---|---|
| `touch <file>` | Create empty file |
| `cp <src> <dest>` | Copy file |
| `mv <src> <dest>` | Move or rename |
| `rm <file>` | Delete file |
| `rm -rf <dir>` | Delete directory and contents |

### Pipes and redirects

| Syntax | What it does |
|---|---|
| `A \| B` | Pipe output of A into input of B |
| `> file` | Redirect output to file (overwrite) |
| `>> file` | Redirect output to file (append) |
| `2>/dev/null` | Discard error output |
| `< file` | Read input from file |

### Environment and SSH

| Command | What it does |
|---|---|
| `echo $VAR` | Print variable |
| `export VAR=value` | Set variable |
| `printenv` | Print all variables |
| `ssh user@host` | Connect via SSH |
| `ssh -i key.pem user@host` | Connect with private key |

### Process management

| Command | What it does |
|---|---|
| `ps aux` | List all processes |
| `kill <pid>` | Graceful stop |
| `kill -9 <pid>` | Force stop |
| `pkill -f <name>` | Stop by name |

### Decision tree: reading an AI-generated command

1. Does the command contain `sudo`? → Verify it's a legitimate system operation.
2. Does it touch a path outside `~/` or `./`? → Stop and verify the full path.
3. Does it use `rm -rf`? → Verify the path exists and is correct.
4. Does it use `curl | sh`? → Download and inspect the script first.
5. Does it set or export a variable? → Confirm it's not a secret.
6. Does it redirect output to a file? → Verify the file path is what you expect.

## Go deeper

- [Bash Guide (Greg Wooledge)](https://mywiki.wooledge.org/BashGuide) — Practical, opinionated Bash reference. Updated regularly. (verified 2025-11)
- [The Linux Command Line (LinuxTraining Academy)](https://linuxcommand.org/tlcl.php) — Free online book covering everything from basics to shell scripting. (verified 2025-11)
- [tmux Cheat Sheet](https://tmuxcheatsheet.com/) — Quick reference for tmux commands and keybindings. (verified 2025-11)
- [Shell Scripting Tutorial (Steve's Guide)](https://www.shellscript.sh/) — Beginner-friendly shell scripting tutorial with examples. (verified 2025-11)
- [ExplainShell.com](https://explainshell.com/) — Paste any command and get a visual breakdown of what each part does. Useful when AI output is unclear. (verified 2025-11)