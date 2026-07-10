# Process Management (systemd & PM2)

A process manager is a tool that keeps your application running continuously in the background, automatically restarting it if it crashes or the server reboots.

## Table of Contents

- [The Problem This Solves](#the-problem-this-solves)
- [What Is a Process?](#what-is-a-process)
- [Foreground vs Background Processes](#foreground-vs-background-processes)
- [What Is a Process Manager?](#what-is-a-process-manager)
- [Option 1: systemd (Built Into Linux)](#option-1-systemd-built-into-linux)
- [Writing a systemd Service File](#writing-a-systemd-service-file)
- [Managing a systemd Service](#managing-a-systemd-service)
- [Viewing Logs from systemd](#viewing-logs-from-systemd)
- [Option 2: PM2 (Popular for Node.js)](#option-2-pm2-popular-for-nodejs)
- [Installing and Using PM2](#installing-and-using-pm2)
- [Making PM2 Survive Server Reboots](#making-pm2-survive-server-reboots)
- [systemd vs PM2: Which to Use?](#systemd-vs-pm2-which-to-use)
- [Zero-Downtime Restarts with PM2](#zero-downtime-restarts-with-pm2)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## The Problem This Solves

- Back in deploy-01, we saw that running `npm start` directly over SSH has a fatal flaw: the process dies the moment you close your terminal or your SSH session disconnects.
- Additionally, if your app **crashes** due to a bug, an unhandled error, or the server running out of memory, nothing will restart it automatically — your site simply goes down and stays down until you notice and manually intervene.
- If the entire **server reboots** (e.g., after a provider maintenance update), your app won't start again on its own either, unless something is configured to launch it automatically.
- A **process manager** solves all three of these problems at once: it keeps your app running in the background, detached from any single terminal session, restarts it automatically on crash, and can be configured to start it automatically on server boot.

## What Is a Process?

- A **process** is a running instance of a program. When you run `node server.js`, the operating system creates a process for it, with its own memory space and a unique ID called a **PID (Process ID)**.
- You can view running processes with:

```bash
ps aux              # list all running processes
ps aux | grep node  # filter to just processes with "node" in the name
```

## Foreground vs Background Processes

- A **foreground process** is tied directly to your current terminal session — if you close the terminal or disconnect, the process is killed along with it.
- A **background process** keeps running independently of any specific terminal session.

```bash
# Foreground: dies when you disconnect
node server.js

# A crude way to background it manually (not recommended for production)
node server.js &
```

> ⚠️ **Warning:** Using `&` or tools like `nohup` to background a process manually is a common beginner workaround, but it doesn't solve automatic restarts on crash or reboot — you still need a real process manager for that. Treat this as a stepping stone to understanding the problem, not a production solution.

## What Is a Process Manager?

- A **process manager** is dedicated software whose entire job is supervising other processes: starting them, keeping them alive, restarting them on crash, managing logs, and optionally starting them automatically on boot.
- Two very common options for deploying apps on Linux:
  - **systemd** — built directly into most modern Linux distributions (including Ubuntu), works with any type of application, not just Node.js.
  - **PM2** — a process manager built specifically with Node.js in mind (though it can technically run other things too), very popular in the Node.js/JavaScript ecosystem for its simplicity.

## Option 1: systemd (Built Into Linux)

- **systemd** is the init system used by Ubuntu and most modern Linux distributions — it's the very first process that starts when the server boots, and it's responsible for starting and supervising all other system services (including Nginx, as you saw in deploy-04).
- You define your app as a **service** by creating a `.service` file, and then systemd manages it exactly like it manages any built-in system service.

> 💡 **Tip:** Since you already used `systemctl` to manage Nginx in deploy-04 (`systemctl restart nginx`), you're actually using systemd already — this note just teaches you to apply the same pattern to your *own* app.

## Writing a systemd Service File

Service files live in `/etc/systemd/system/`. Here's an example for a Node.js app:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js App
After=network.target

[Service]
ExecStart=/usr/bin/node /var/www/myapp/server.js
Restart=always
User=phyo
WorkingDirectory=/var/www/myapp
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```

- `[Unit]` section — general metadata. `Description` is a human-readable label; `After=network.target` tells systemd to wait until networking is up before starting this service.
- `[Service]` section — the actual behavior:
  - `ExecStart` — the exact command used to start your app.
  - `Restart=always` — the key line: automatically restart the app if it crashes or exits for any reason.
  - `User` — which Linux user the process should run as (avoid running your app as `root`, for security).
  - `WorkingDirectory` — the folder the command should run from.
  - `Environment` — sets environment variables the app needs (covered in more depth in deploy-10).
- `[Install]` section — `WantedBy=multi-user.target` tells systemd this service should start automatically during normal server boot.

> ⚠️ **Warning:** Use the **full absolute path** to your executable (e.g., `/usr/bin/node`, found via `which node`) in `ExecStart`. systemd services don't automatically have the same PATH environment your interactive terminal session has, so a bare `node` command can fail to be found.

## Managing a systemd Service

```bash
# After creating/editing a .service file, reload systemd's config
sudo systemctl daemon-reload

# Start the service now
sudo systemctl start myapp

# Enable it to auto-start on every server boot
sudo systemctl enable myapp

# Check its current status
sudo systemctl status myapp

# Restart after a deployment/update
sudo systemctl restart myapp

# Stop it
sudo systemctl stop myapp
```

- `daemon-reload` — required any time you create or modify a `.service` file, so systemd picks up the changes.
- `enable` vs `start` — these are different: `start` runs it right now; `enable` makes it start automatically on future reboots. You typically want both.

## Viewing Logs from systemd

```bash
# View recent logs for your service
sudo journalctl -u myapp

# Follow logs live, like "tailing" a log file
sudo journalctl -u myapp -f
```

- `journalctl` — the tool for reading systemd's centralized logging system, called the **journal**.
- `-u myapp` — filters to just your specific service's logs.
- `-f` — "follow" mode, streaming new log lines live as they happen, useful while debugging a fresh deployment.

## Option 2: PM2 (Popular for Node.js)

- **PM2** is a process manager built specifically for Node.js applications (with support for other languages via extensions), known for being much quicker to set up than writing raw systemd service files by hand.
- It adds convenient features on top of basic process supervision: a built-in log viewer, real-time monitoring dashboard, and easy zero-downtime reload support.

## Installing and Using PM2

```bash
# Install PM2 globally via npm
sudo npm install -g pm2

# Start your app under PM2's supervision
pm2 start server.js --name myapp

# List all apps PM2 is currently managing
pm2 list

# View logs
pm2 logs myapp

# Restart the app
pm2 restart myapp

# Stop the app
pm2 stop myapp
```

- `--name myapp` — gives your process a friendly name to reference in later commands, instead of a raw process ID.
- Once started, PM2 automatically restarts the app if it crashes — this happens by default, no extra config needed (unlike systemd, where you had to explicitly set `Restart=always`).

## Making PM2 Survive Server Reboots

By default, PM2 keeps your app running as long as the server is on, but won't automatically bring it back after a server reboot unless configured to:

```bash
# Generate a startup script for your system (systemd under the hood)
pm2 startup

# Follow the on-screen instructions — it will print a command to run with sudo

# Save the current list of running apps so PM2 knows what to restart
pm2 save
```

- `pm2 startup` — detects your OS and actually generates and registers a systemd service for PM2 itself, so PM2 (and everything it's managing) starts automatically on boot.
- `pm2 save` — snapshots the current list of running processes, so after a reboot, PM2 knows exactly which apps to bring back up.

> 💡 **Tip:** Interestingly, PM2 itself runs *on top of* systemd behind the scenes when you use `pm2 startup` — this is a nice example of how these tools aren't strictly competing, but can layer on each other.

## systemd vs PM2: Which to Use?

| Feature | systemd | PM2 |
|---|---|---|
| **Built into Linux** | Yes, no install needed | No, requires `npm install -g pm2` |
| **Works with any language** | Yes (Node, PHP, Python, Go, etc.) | Primarily Node.js-focused |
| **Setup complexity** | More manual (write a `.service` file) | Faster, simpler CLI commands |
| **Built-in monitoring dashboard** | No (use `journalctl` for logs) | Yes (`pm2 monit`) |
| **Zero-downtime reload** | Possible but more manual to configure | Built-in (`pm2 reload`) |
| **Industry usage** | Universal, used for all system services | Very popular specifically in Node.js shops |

> 💡 **Tip:** A common professional approach: use **PM2 for Node.js apps** specifically, because of its convenience and Node-specific tooling, while relying on **systemd for everything else** (Nginx, PostgreSQL, and other system-level services) since it's already there and language-agnostic. Both approaches are valid — this series will show PM2 for our Node.js backend in the final walkthrough.

## Zero-Downtime Restarts with PM2

```bash
pm2 reload myapp
```

- `pm2 reload` (as opposed to `pm2 restart`) starts new instances of your app before stopping the old ones, so there's no gap where the app is completely down — directly applying the zero-downtime concept introduced in deploy-01.
- This works especially well when running your app in **cluster mode**, where PM2 runs multiple instances of your app across your server's CPU cores simultaneously:

```bash
pm2 start server.js --name myapp -i max
```

- `-i max` — tells PM2 to automatically start one instance per available CPU core, both improving performance and making zero-downtime reloads smoother (since there's always at least one instance serving traffic while others restart).

## Common Mistakes Beginners Make

- Running an app manually with `node server.js` in production and being surprised when it dies after closing the SSH session.
- Forgetting `sudo systemctl daemon-reload` after editing a `.service` file, so changes don't take effect.
- Forgetting `pm2 save` after starting new apps, so a server reboot doesn't bring them back.
- Running the app as `root` inside a systemd service instead of a restricted regular user, for no real benefit and added risk.
- Using `pm2 restart` when `pm2 reload` (zero-downtime) was actually what was needed for a live production app.

## Quick Revision

- A process manager keeps your app running continuously, restarts it automatically on crash, and can relaunch it after a server reboot — solving the "dies when I close SSH" problem from deploy-01.
- systemd is Linux's built-in init system, already used to manage Nginx; you can define your own app as a `.service` file with `Restart=always` for automatic recovery.
- Key systemd commands: `daemon-reload` (after editing a service file), `start`/`enable` (run now / run on boot), `status`, and `journalctl -u yourservice -f` for live logs.
- PM2 is a Node.js-focused process manager that's faster to set up than raw systemd files and includes built-in monitoring and zero-downtime reloads.
- `pm2 startup` plus `pm2 save` are both required to make PM2-managed apps survive a full server reboot.
- A common real-world pattern: PM2 for Node.js apps specifically, systemd for everything else on the server.
- `pm2 reload` (not `restart`) achieves zero-downtime updates, especially when combined with cluster mode (`-i max`) across multiple CPU cores.