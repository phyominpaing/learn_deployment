# Linux Server Basics & SSH

SSH is the secure remote-connection tool that lets you log into and control a Linux server from your own computer, and Linux is the operating system almost every server in the world runs on.

## Table of Contents

- [Why Linux for Servers?](#why-linux-for-servers)
- [What Is SSH?](#what-is-ssh)
- [Your First SSH Connection](#your-first-ssh-connection)
- [Understanding the Terminal Prompt](#understanding-the-terminal-prompt)
- [SSH Keys: The Secure Way to Connect](#ssh-keys-the-secure-way-to-connect)
- [Generating and Using SSH Keys](#generating-and-using-ssh-keys)
- [The Linux File System, Explained Simply](#the-linux-file-system-explained-simply)
- [Essential Linux Commands](#essential-linux-commands)
- [Users, Root, and Permissions](#users-root-and-permissions)
- [File Permissions Explained](#file-permissions-explained)
- [Installing Software with a Package Manager](#installing-software-with-a-package-manager)
- [Editing Files on the Server](#editing-files-on-the-server)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## Why Linux for Servers?

- **Linux** is a free, open-source operating system (like Windows or macOS, but built differently) that powers the vast majority of servers worldwide.
- It's preferred for servers because it's **lightweight** (no graphical interface needed, saving resources), **stable** (can run for months without rebooting), **secure**, and **free** to use.
- You don't need to install Linux yourself — when you rent a VPS, you simply choose "Ubuntu" (or another Linux distribution) from a menu, and the provider sets it up for you automatically.
- **Distribution (distro)**: A specific packaged version of Linux. **Ubuntu** is the most beginner-friendly and widely used distro for servers — this series will assume Ubuntu.

> 💡 **Tip:** You do not need to install Linux on your own laptop to learn this. You'll connect to a Linux server running remotely, and everything happens through commands you type, not through installing anything locally (aside from an SSH tool, which your computer likely already has).

## What Is SSH?

- **SSH (Secure Shell)** is a protocol (a set of rules) that lets you securely connect to and control a remote computer over the internet, by typing commands into a terminal.
- "Secure" means the connection is **encrypted** — nobody spying on the network traffic between you and the server can read what you're typing or what the server sends back.
- Once connected via SSH, your terminal essentially *becomes* the server's terminal — every command you type runs on the remote server, not on your own laptop.

```text
Your Laptop  ──── encrypted SSH connection ────►  Remote Server (VPS)
   (client)                                            (Linux)
```

> ⚠️ **Warning:** Never confuse "I'm connected via SSH" with "I'm on my own laptop." Once connected, commands like `rm` (delete) affect the *server*, not your local machine. This mix-up has caused real, painful mistakes for many developers.

## Your First SSH Connection

Assuming you already have a VPS with an IP address (we'll purchase and set one up for real in the final walkthrough), connecting looks like this:

```bash
ssh username@your-server-ip
# example:
ssh root@203.0.113.25
```

- `ssh` — the command itself.
- `username` — the account you're logging in as on the server (often `root` initially).
- `@your-server-ip` — the address of the server you're connecting to.
- The first time you connect, you'll see a message asking to confirm the server's "fingerprint" — type `yes` to continue. This is a security check confirming you're connecting to the right machine.
- You'll then be asked for a password (or, if set up, it'll use an SSH key automatically — covered below).

> 💡 **Tip:** Windows users can run `ssh` through **PowerShell**, **Windows Terminal**, or **WSL (Windows Subsystem for Linux)** — modern Windows includes an SSH client by default. Mac and Linux users already have `ssh` built into their normal Terminal app.

## Understanding the Terminal Prompt

Once connected, you'll see something like this:

```bash
root@myserver:~#
```

- `root` — the currently logged-in user.
- `myserver` — the server's hostname (its name).
- `~` — your current location in the file system (`~` is shorthand for the home directory).
- `#` — indicates you're logged in as `root` (the administrator). A regular, non-root user would see `$` instead.

## SSH Keys: The Secure Way to Connect

- A **password** can be guessed or brute-forced by attackers scanning the internet for open servers. **SSH keys** are a far more secure alternative.
- An SSH key comes as a **pair**: a **private key** (stays secret, only on your laptop, never shared) and a **public key** (safe to share, placed on the server).
- **How it works conceptually**: the server has a copy of your public key. When you connect, your laptop proves it holds the matching private key through cryptographic math — without ever sending the private key over the network. If the math checks out, you're let in, no password needed.

> ⚠️ **Warning:** Your **private key** must never be shared, emailed, committed to Git, or posted anywhere online. Anyone who has it can log into your server exactly as you. Treat it like a physical house key, not a password you can just reset.

## Generating and Using SSH Keys

```bash
# Run this on your own laptop, NOT on the server
ssh-keygen -t ed25519 -C "your_email@example.com"
```

- `-t ed25519` — specifies the key type (a modern, secure, and fast algorithm; `ed25519` is currently the recommended default).
- `-C "..."` — just a label/comment to help you identify the key later, usually your email.
- You'll be prompted for a file location (press Enter to accept the default) and optionally a **passphrase** (an extra password that protects the key file itself — recommended, but skippable while learning).
- This creates two files, typically:
  - `~/.ssh/id_ed25519` — your **private** key (keep secret).
  - `~/.ssh/id_ed25519.pub` — your **public** key (safe to share/upload).

```bash
# Copy your public key to the server so you can log in without a password
ssh-copy-id username@your-server-ip
```

- After this, running `ssh username@your-server-ip` will log you in automatically using the key, instead of asking for a password.

> 💡 **Tip:** Most VPS providers (like DigitalOcean) let you paste your public key directly into their website when creating a new server — meaning the server is set up with SSH key access from the very first boot, before you even connect once.

## The Linux File System, Explained Simply

Unlike Windows (`C:\`, `D:\`), Linux has a single unified structure starting from one root, `/`.

```text
/               ← the very top of the file system ("root", not to be confused with the root user)
├── home/       ← personal folders for each user (e.g., /home/phyo)
├── etc/        ← configuration files for installed software
├── var/        ← variable data — logs, databases, running app data
│   └── www/    ← common location for website files
├── usr/        ← installed programs and their supporting files
├── tmp/        ← temporary files, cleared periodically
└── root/       ← the home directory of the root user specifically
```

- **`/`** is called "root" as in "root of the tree" — different from the **root user**, which is confusingly also called "root." Context tells them apart.
- Your app's code will typically live somewhere like `/var/www/your-app` or `/home/youruser/your-app`.

## Essential Linux Commands

```bash
pwd                     # print working directory (shows where you currently are)
ls                      # list files in the current directory
ls -la                  # list files, including hidden ones, with details
cd folder-name          # change directory (move into a folder)
cd ..                   # move up one directory level
mkdir new-folder        # create a new folder
touch file.txt          # create a new empty file
cat file.txt            # print a file's contents to the screen
rm file.txt             # delete a file
rm -rf folder-name      # delete a folder and everything inside it (DANGEROUS)
cp source.txt dest.txt  # copy a file
mv old-name new-name    # move or rename a file/folder
whoami                  # show which user you're currently logged in as
df -h                   # show disk space usage
```

> ⚠️ **Warning:** `rm -rf` is one of the most dangerous commands in Linux — it deletes permanently, with **no trash bin, no undo, no confirmation prompt**. Running `rm -rf /` (or a mistyped path) can wipe an entire server. Always double-check the path before pressing Enter.

## Users, Root, and Permissions

- **root** is the superuser — it has unlimited power to do absolutely anything on the server, including breaking it irreversibly.
- Logging in and working as `root` for everyday tasks is considered bad practice, because one typo can cause catastrophic, unrecoverable damage.
- The standard practice is: create a regular user for daily work, and use `sudo` only when you specifically need elevated permissions for a single command.

```bash
# Create a new regular user
adduser phyo

# Grant that user permission to use sudo (run admin commands when needed)
usermod -aG sudo phyo

# Now log in as that user instead of root
ssh phyo@your-server-ip
```

- **`sudo`** ("superuser do") temporarily elevates a single command to root-level permissions, after confirming your password. It's a controlled, deliberate way to get admin power only when you need it.

```bash
sudo apt update    # this specific command runs with root privileges
```

> 💡 **Tip:** Think of working as a regular user with `sudo` as "keeping the safety on" — you have to deliberately pull the trigger (`sudo`) for dangerous actions, rather than walking around with the safety off (`root`) the whole time.

## File Permissions Explained

Every file and folder in Linux has permissions controlling who can read, write, or execute it.

```bash
ls -l
# example output:
# -rw-r--r-- 1 phyo phyo 220 Jul  7 10:00 file.txt
```

| Symbol | Meaning |
|---|---|
| `r` | **Read** — can view the file's contents |
| `w` | **Write** — can modify the file |
| `x` | **Execute** — can run the file as a program/script |
| `-` | Permission not granted |

- The permission string is split into three groups: **owner**, **group**, and **others** — e.g., `rw-r--r--` means the owner can read/write, everyone else can only read.

```bash
chmod +x script.sh     # make a file executable
chmod 644 file.txt      # set specific numeric permissions (owner rw, others r)
chown phyo:phyo file.txt # change who owns the file
```

> ⚠️ **Warning:** Never run `chmod 777` on files "just to make an error go away." It grants everyone full read/write/execute access, which is a serious security hole. Fix the actual ownership/permission issue instead of opening everything wide.

## Installing Software with a Package Manager

- A **package manager** is a tool that downloads, installs, updates, and removes software for you, handling dependencies automatically. On Ubuntu, this is `apt`.

```bash
sudo apt update          # refresh the list of available software versions
sudo apt upgrade         # install available updates for existing software
sudo apt install nginx   # install a specific piece of software (e.g., the Nginx web server)
sudo apt remove nginx    # uninstall software
```

> 💡 **Tip:** Always run `sudo apt update` before installing anything new — it ensures you're installing the latest known version, rather than an outdated one from a stale local list.

## Editing Files on the Server

Since servers have no graphical interface, you edit configuration files directly in the terminal using a text editor. The most common beginner-friendly one is **nano**.

```bash
nano filename.txt
```

- Type to edit normally.
- `Ctrl + O` then `Enter` — save the file ("O" stands for "Write Out").
- `Ctrl + X` — exit the editor.

> 💡 **Tip:** `vim` is another very common editor on servers, but it has a steep learning curve (even exiting it confuses beginners — the answer is typing `:q` then Enter). Stick with `nano` while learning; you can pick up `vim` later once you're comfortable.

## Common Mistakes Beginners Make

- Working as `root` for everything instead of creating a regular user with `sudo` — risky and not industry standard.
- Losing the private SSH key with no backup, permanently locking themselves out of the server.
- Running destructive commands (`rm -rf`, `chmod 777`) without fully understanding the path/scope first.
- Forgetting they're connected to the *server's* terminal and accidentally running commands meant for their local laptop (or vice versa).
- Not running `sudo apt update` before installing software, leading to outdated package errors.

## Quick Revision

- Linux is the standard operating system for servers because it's lightweight, stable, secure, and free; Ubuntu is the most beginner-friendly distro.
- SSH is the encrypted protocol used to remotely log into and control a server's terminal from your own computer.
- SSH keys (a private key kept secret + a public key placed on the server) are more secure than passwords and are the industry-standard login method.
- The Linux file system is a single tree starting at `/`, with important folders like `/home`, `/etc`, `/var`, and `/var/www` for website files.
- Core commands to memorize: `pwd`, `ls`, `cd`, `mkdir`, `rm`, `cp`, `mv`, `cat`, and the dangerous `rm -rf`.
- `root` is the all-powerful superuser account; best practice is creating a regular user and using `sudo` only when elevated permissions are truly needed.
- File permissions (`r`, `w`, `x` for owner/group/others) control who can read, write, or execute a file — never blanket-fix permission errors with `chmod 777`.
- `apt` is Ubuntu's package manager for installing software; always run `sudo apt update` first.
- `nano` is the easiest terminal text editor for beginners to edit server configuration files.