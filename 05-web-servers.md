# Web Servers (Nginx/Apache) & Reverse Proxies

A web server is software that listens for incoming website requests and decides what to send back, and a reverse proxy is a web server placed in front of your app to manage and forward traffic to it.

## Table of Contents

- [What Is a Web Server, Really?](#what-is-a-web-server-really)
- [Why Can't Users Just Hit Your App Directly?](#why-cant-users-just-hit-your-app-directly)
- [What Is a Reverse Proxy?](#what-is-a-reverse-proxy)
- [Nginx vs Apache](#nginx-vs-apache)
- [Installing Nginx](#installing-nginx)
- [Understanding Ports Again](#understanding-ports-again)
- [Your First Nginx Config File](#your-first-nginx-config-file)
- [Reverse Proxy Config for a Node.js/Backend App](#reverse-proxy-config-for-a-nodejsbackend-app)
- [Serving Static Files Directly with Nginx](#serving-static-files-directly-with-nginx)
- [Managing the Nginx Service](#managing-the-nginx-service)
- [Multiple Sites on One Server](#multiple-sites-on-one-server)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## What Is a Web Server, Really?

- A **web server** is a piece of software (not the physical machine, despite the confusing shared name) that listens on a network port, receives HTTP requests from browsers/clients, and sends back responses — HTML pages, files, JSON data, etc.
- The two most common web server software programs are **Nginx** (pronounced "engine-x") and **Apache**.
- Your VPS is a "server" in the hardware sense; Nginx or Apache is the "server" in the software sense that actually handles web traffic on it. Both meanings of "server" coexist and you'll hear both used interchangeably — context clarifies which is meant.

> 💡 **Tip:** It helps to think of the VPS as the building, and Nginx as the receptionist inside it — the receptionist decides which office (your app, a static file, another service) each visitor should be sent to.

## Why Can't Users Just Hit Your App Directly?

- When you run `npm start` for a Node.js app, or `php artisan serve` for PHP, that process usually listens on a specific **internal** port, like `3000` or `8000`.
- You *could* technically expose that port directly to the internet, but this is almost never done in production, for several important reasons:
  - Your app's process (Node.js, PHP-FPM, etc.) is usually not built to safely and efficiently handle thousands of raw internet connections, HTTPS encryption, or malicious traffic on its own.
  - You often want to run **multiple** apps or services on one server (e.g., your API, an admin panel, a static frontend) — each on its own internal port — but present them all through the standard web ports (`80`/`443`) with clean domain-based routing.
  - HTTPS/SSL setup, compression, caching, and security filtering are usually configured once, centrally, in the web server — not duplicated inside every single app.
- This is exactly the problem a **reverse proxy** solves.

## What Is a Reverse Proxy?

- A **reverse proxy** is a server that sits between the internet and your actual application, receiving all incoming requests first and forwarding ("proxying") them to the correct internal app/port behind the scenes.
- The word "reverse" distinguishes it from a regular (forward) proxy, which sits in front of *clients* to hide their identity; a reverse proxy sits in front of *servers* to hide their internal details from clients.

```text
Internet Users
     │
     ▼
 Nginx (reverse proxy) — listens on port 80/443, the public web ports
     │
     ├── forwards /api/*     → your backend app on internal port 4000
     ├── forwards /admin/*   → your admin panel app on internal port 5000
     └── forwards /          → your frontend app on internal port 3000
```

- From the outside, a visitor just sees `https://yourapp.com` — they have no idea Nginx is quietly routing their request to one of several internal processes running behind it.

> 💡 **Tip:** This single concept — Nginx forwarding public traffic to internal app processes — is the backbone of almost every real-world production deployment you'll ever work on, regardless of tech stack.

## Nginx vs Apache

| Feature | Nginx | Apache |
|---|---|---|
| **Architecture** | Event-driven, handles many connections efficiently | Process/thread-based per connection |
| **Performance under load** | Excellent, especially for static files & high traffic | Good, but can use more memory under heavy load |
| **Configuration style** | Single config files, considered cleaner/simpler | `.htaccess` per-directory files, very flexible |
| **Common use today** | Reverse proxy, load balancer, static file serving | Traditional PHP hosting (especially shared hosting) |
| **Popularity trend** | Widely preferred for new modern deployments | Still very common, especially in legacy/PHP-heavy hosting |

> 💡 **Tip:** For this series, we'll use **Nginx**, since it's the current industry standard for reverse-proxying modern full-stack apps (Node.js, Next.js, custom APIs) and is what you'll most likely encounter in real jobs today.

## Installing Nginx

```bash
sudo apt update
sudo apt install nginx
```

- After installation, Nginx starts automatically and begins listening on port `80`.
- You can immediately test it by visiting your server's IP address in a browser — you should see the default "Welcome to nginx!" page, confirming it's running.

```bash
sudo systemctl status nginx
```

- This command checks whether the Nginx service is currently running. You should see `active (running)` in green.

## Understanding Ports Again

- **Port 80** — the standard port for unencrypted HTTP traffic. When you type `http://yourapp.com`, the browser connects to port 80 by default.
- **Port 443** — the standard port for encrypted HTTPS traffic. `https://yourapp.com` connects here by default. (HTTPS/SSL setup is covered in deploy-05.)
- Your actual application processes (Node.js, PHP-FPM, etc.) typically run on **non-standard, internal ports** (e.g., `3000`, `4000`, `8000`) that are *not* directly exposed to the public internet — only Nginx talks to them directly, on the same machine.

> ⚠️ **Warning:** If you leave your app's internal port open to the public internet (e.g., forgetting to configure your firewall), users could bypass Nginx entirely and hit your app directly and insecurely. Firewall configuration (covered later in the series) closes this gap.

## Your First Nginx Config File

Nginx configuration files live in `/etc/nginx/`. Site-specific configs typically go in `/etc/nginx/sites-available/` and are "enabled" by linking them into `/etc/nginx/sites-enabled/`.

```nginx
# /etc/nginx/sites-available/yourapp
server {
    listen 80;
    server_name yourapp.com www.yourapp.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- `server { ... }` — defines one website/config block. You can have multiple of these for multiple sites.
- `listen 80;` — tells Nginx to accept traffic on port 80 (standard HTTP) for this block.
- `server_name` — the domain name(s) this config block should respond to.
- `location / { ... }` — a rule for matching request paths. `/` matches everything (the root and all sub-paths, unless a more specific `location` block overrides it).
- `proxy_pass http://localhost:3000;` — the key reverse-proxy line: forward matching requests to your app running locally on port 3000.
- `proxy_set_header Host $host;` — passes along the original domain name the visitor used, so your app knows what host was requested.
- `proxy_set_header X-Real-IP $remote_addr;` — passes along the visitor's real IP address, since without this, your app would only see Nginx's internal IP as the "visitor."

```bash
# Enable the site by linking it into sites-enabled
sudo ln -s /etc/nginx/sites-available/yourapp /etc/nginx/sites-enabled/

# Test the config for syntax errors before applying
sudo nginx -t

# Reload Nginx to apply the new config
sudo systemctl reload nginx
```

> ⚠️ **Warning:** Always run `sudo nginx -t` before reloading. If your config has a syntax error and you reload/restart anyway, Nginx can fail to come back up, taking your entire site offline until you fix it.

## Reverse Proxy Config for a Node.js/Backend App

Expanding the example above with multiple routes — useful once you have a separate frontend, API, and admin panel:

```nginx
server {
    listen 80;
    server_name yourapp.com;

    location /api/ {
        proxy_pass http://localhost:4000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /admin/ {
        proxy_pass http://localhost:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- Nginx checks `location` blocks and routes to the most specific matching path — so `/api/...` requests go to port 4000, `/admin/...` requests go to port 5000, and everything else falls through to the frontend on port 3000.

## Serving Static Files Directly with Nginx

Nginx doesn't always need to proxy to another app — it can serve plain files (HTML, CSS, JS, images) directly, which is faster since no extra app process is involved.

```nginx
server {
    listen 80;
    server_name static.yourapp.com;
    root /var/www/yourapp/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

- `root` — the folder on disk where the static files live.
- `index` — the default file to serve when a directory is requested (e.g., visiting `/` serves `index.html`).
- `try_files $uri $uri/ /index.html;` — tries to find the exact file requested, then a matching folder, and finally falls back to `index.html` — this fallback is essential for single-page apps (React/Vue) that handle their own client-side routing.

## Managing the Nginx Service

```bash
sudo systemctl start nginx      # start Nginx
sudo systemctl stop nginx       # stop Nginx
sudo systemctl restart nginx    # fully restart (brief downtime)
sudo systemctl reload nginx     # reload config without dropping connections (preferred)
sudo systemctl enable nginx     # make Nginx start automatically on server reboot
```

> 💡 **Tip:** Prefer `reload` over `restart` whenever you're just changing configuration — it applies changes without interrupting requests currently in progress, giving you a smoother, closer-to-zero-downtime update.

## Multiple Sites on One Server

- Because each `server { }` block can have its own `server_name`, a single Nginx installation on a single VPS can host and route traffic for many different domains/subdomains simultaneously — each pointing to a different app or set of static files.
- This is how one relatively cheap VPS can serve a frontend, backend API, and admin panel, all as if they were entirely separate deployments, from the user's perspective.

## Common Mistakes Beginners Make

- Forgetting to run `sudo nginx -t` before reloading, and breaking the live site with a typo.
- Exposing the app's internal port directly to the internet instead of only allowing Nginx to reach it, weakening security.
- Mixing up `location /` (catches everything) with more specific paths like `location /api/`, causing requests to route to the wrong app.
- Using `restart` instead of `reload` for routine config changes, causing unnecessary brief downtime.
- Forgetting the trailing slash differences between `proxy_pass http://localhost:4000` and `proxy_pass http://localhost:4000/` — these behave subtly differently in how the path is forwarded.

## Quick Revision

- A web server is software (like Nginx or Apache) that listens for HTTP requests and decides what to respond with — distinct from the physical VPS "server."
- Apps aren't exposed directly to the internet; instead, a reverse proxy (Nginx) sits in front, listening on the public ports (80/443) and forwarding traffic to internal app ports.
- Nginx is the current industry standard for reverse-proxying modern full-stack apps, and is what this series will use going forward.
- Nginx configs live in `/etc/nginx/sites-available/` and are activated by linking into `/etc/nginx/sites-enabled/`.
- The key config directive for reverse proxying is `proxy_pass`, which forwards matching requests to an internal app port.
- One Nginx installation can route multiple domains/paths to multiple different apps (frontend, API, admin panel) on the same VPS.
- Always run `sudo nginx -t` to check for config errors before reloading, and prefer `reload` over `restart` for near-zero-downtime updates.
- Nginx can also serve static files directly without proxying to an app, which is faster and commonly used for frontend builds.