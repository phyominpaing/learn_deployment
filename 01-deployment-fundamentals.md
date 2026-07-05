# Deployment Fundamentals

Deployment is the process of taking code that works on your local machine and making it run reliably on a server so real users can access it over the internet.

## Table of Contents

- [What Is Deployment, Really?](#what-is-deployment-really)
- [Local vs Server: What Changes](#local-vs-server-what-changes)
- [The Three Environments](#the-three-environments)
- [The General Deployment Pipeline](#the-general-deployment-pipeline)
- [Deployment Models Overview](#deployment-models-overview)
- [Key Terminology You'll Hear Constantly](#key-terminology-youll-hear-constantly)
- [Downtime vs Zero-Downtime Deploys](#downtime-vs-zero-downtime-deploys)
- [A Minimal Real-World Example](#a-minimal-real-world-example)
- [Quick Revision](#quick-revision)

## What Is Deployment, Really?

- **Deployment** is the act of moving your application from your local dev environment onto a machine (server) that is reachable by other people, 24/7.
- On your laptop, only *you* can visit `localhost:3000`. Deployment turns that into `https://yourapp.com`, reachable by anyone.
- Deployment is not a single action — it's a **process** made of several steps: getting code onto the server, installing dependencies, configuring it to run continuously, and pointing a domain at it.

> 💡 **Tip:** Think of your laptop as a private workshop and the server as the storefront. Deployment is the process of moving the finished product from the workshop to the storefront, and keeping the lights on there permanently.

## Local vs Server: What Changes

When you run code locally, a lot of things are invisibly convenient. On a server, you have to handle each one explicitly.

| Concern | On Your Laptop | On a Server |
|---|---|---|
| **Uptime** | Only running while you have it open | Must run 24/7, even after reboots |
| **Access** | Only you, via `localhost` | Public internet, via a domain name |
| **Crashes** | You see the error immediately | Nobody notices until you check — needs monitoring |
| **Restart** | You manually re-run `npm start` | Needs a process manager to auto-restart |
| **HTTPS** | Usually skipped | Required for real users (browsers warn otherwise) |
| **Environment variables** | Stored in a local `.env` file | Must be set securely on the server |
| **Database** | Local SQLite/MySQL instance | Often a separate managed database service |
| **Performance** | Single user (you) | Many concurrent users |

> ⚠️ **Warning:** Code that "works on my machine" can fail on a server for reasons that have nothing to do with your code — wrong Node/PHP version, missing environment variables, file permission issues, or a firewall blocking the port. This is common enough that "works on my machine" is a running joke in the industry.

## The Three Environments

Professional teams almost never deploy straight from a laptop to real users. Code passes through **environments** first.

- **Development (dev)**: Your local machine, or a shared dev server. Code changes constantly and breaking things is expected.
- **Staging**: A near-identical copy of production, used to test changes safely before they reach real users. Same server setup, same database structure, but with fake or copied data.
- **Production (prod)**: The live environment real users interact with. Changes here should be tested and deliberate.

> 💡 **Tip:** A common beginner mistake is testing new features directly on production. Even solo developers benefit from a staging environment — it catches "it worked locally but broke on the server" issues before real users see them.

## The General Deployment Pipeline

Regardless of the tech stack (PHP, Node.js, Next.js, Python, etc.), almost every deployment follows the same general shape:

```text
1. Write code locally
2. Push code to a version control system (e.g., Git/GitHub)
3. Build the application (install dependencies, compile/transpile assets)
4. Transfer the build to a server
5. Configure the server (env vars, database connection, web server)
6. Start the app as a persistent process
7. Point a domain name at the server
8. Secure it with HTTPS
9. Monitor it to make sure it keeps running
```

- **Build**: The step that turns your source code into something runnable/servable — e.g., `npm run build` compiles React into static JS/HTML, or `composer install` pulls in PHP dependencies.
- **Artifact**: The output of a build step — a folder of compiled files, a Docker image, or a `.zip` — that actually gets deployed.
- **Release**: A specific version of your artifact that has been deployed and is currently (or was previously) live.

> ⚠️ **Warning:** Never deploy your raw source folder including `node_modules`, `.env` files with real secrets, or `.git` history to a public server without thinking about it — these can leak credentials or bloat your deployment unnecessarily.

## Deployment Models Overview

There isn't one way to deploy — there's a spectrum of how much infrastructure you manage yourself versus how much a provider manages for you. We'll cover each of these in depth in later notes, but here's the map:

| Model | You Manage | Provider Manages | Example Providers |
|---|---|---|---|
| **Bare Metal / VPS** | OS, web server, runtime, app, security, everything | The physical hardware only | DigitalOcean, Linode, a home server |
| **PaaS (Platform as a Service)** | Just your app code and config | OS, servers, scaling, most infra | Heroku, Railway, Render |
| **Serverless** | Just individual functions | Servers, scaling, runtime entirely | Vercel Functions, AWS Lambda |
| **Containers (Docker/Kubernetes)** | App + its exact environment, packaged | Underlying host, often | AWS ECS, Google Cloud Run, Kubernetes |
| **Static Hosting** | Just static HTML/CSS/JS files | Everything else | Netlify, GitHub Pages, Cloudflare Pages |

- **VPS (Virtual Private Server)**: A slice of a physical server, rented to you, where you get root access and full control — like renting an empty apartment you have to furnish yourself.
- **PaaS**: You hand over your code and the platform figures out how to run it — like a serviced apartment where utilities are handled for you.
- **Serverless**: You write small functions that run on-demand; there's no persistent server you manage at all.

> 💡 **Tip:** As a beginner, starting with a PaaS (like Railway or Render) is usually the fastest way to get a real deployment experience without fighting Linux configuration. Learning full VPS deployment (which we'll do in this series) teaches you what's happening underneath — valuable even if you use PaaS later in your career.

## Key Terminology You'll Hear Constantly

- **Server**: A computer (physical or virtual) that stays on and responds to requests from other computers over a network.
- **Client**: The device/browser making requests to the server (e.g., your visitor's browser).
- **Port**: A numbered "door" on a server that a specific service listens on (e.g., port `80` for HTTP, `443` for HTTPS, `3000` often used for Node.js dev servers).
- **Process**: A running instance of your program on the server (e.g., your Node.js app running as a process).
- **Daemon**: A process that runs continuously in the background, without a user directly interacting with it.
- **Reverse proxy**: A server that sits in front of your app and forwards incoming requests to it (commonly Nginx) — covered in depth later.
- **Firewall**: A system that controls which network traffic is allowed in or out of a server.
- **Rollback**: Reverting to a previous, known-working release after a bad deployment.
- **CI/CD**: **Continuous Integration / Continuous Deployment** — automating the build, test, and deploy steps so they happen automatically when you push code.

> ⚠️ **Warning:** Confusing "port" with "domain" is a very common beginner mix-up. A domain (`yourapp.com`) points to a server's IP address; a port determines *which service* on that server handles the request. `https://yourapp.com` implicitly uses port `443`.

## Downtime vs Zero-Downtime Deploys

- **Downtime deploy**: The simplest approach — stop the old version, deploy the new one, start it again. Users see an error or "site unavailable" for a few seconds to minutes.
- **Zero-downtime deploy**: The new version is started *before* the old one is stopped, and traffic is switched over only once the new version is confirmed healthy. Users never notice.
- **Blue-green deployment**: Two identical production environments ("blue" and "green"). One is live, one is idle. You deploy to the idle one, test it, then switch traffic to it instantly.
- **Canary deployment**: The new version is rolled out to a small percentage of users first, and gradually increased if no errors appear.

> 💡 **Tip:** For personal projects and early-stage apps, a few seconds of downtime on deploy is completely normal and not worth over-engineering against. Zero-downtime strategies matter once you have real, paying users who'd notice.

## A Minimal Real-World Example

Here's what a barebones first deployment might conceptually look like for a Node.js app on a VPS, just to see the shape of it (we'll do this for real in later notes):

```bash
# 1. SSH into your server
ssh user@your-server-ip

# 2. Clone your code from GitHub
git clone https://github.com/you/your-app.git

# 3. Install dependencies
cd your-app
npm install

# 4. Start the app so it keeps running
npm start
```

- This is intentionally the *simplest possible version* — it has no process manager (the app dies if you close the terminal), no HTTPS, and no reverse proxy. Every later note in this series fixes one of these gaps.

> ⚠️ **Warning:** Running `npm start` directly over SSH and then closing your terminal will kill the app on most systems, because the process is tied to your terminal session. This is exactly why process managers (covered in deploy-06) exist.

## Quick Revision

- Deployment means moving your app from your local machine to a server that's reachable by real users, 24/7.
- Things that "just work" locally — uptime, HTTPS, restarts, environment variables — must all be handled explicitly on a server.
- Professional workflows use three environments: development, staging, and production, to avoid testing directly on live users.
- The general deployment pipeline is: write code → push to Git → build → transfer → configure → run as a process → attach domain → secure with HTTPS → monitor.
- Deployment models range from full-control VPS/bare metal, to PaaS, to serverless, to static hosting — each trading control for convenience.
- Key vocabulary to lock in: server, client, port, process, daemon, reverse proxy, firewall, rollback, CI/CD.
- Zero-downtime, blue-green, and canary deployments are strategies for updating live apps without users noticing — useful once you have real traffic.
- A deployed app with no process manager will die when your SSH session ends — this motivates the next few notes in the series.