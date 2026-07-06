# Servers & Hosting Models

A server is simply a computer that stays turned on and connected to the internet so it can respond to requests, and "hosting" is the business of renting you access to one.

## Table of Contents

- [What Is a Server, Physically?](#what-is-a-server-physically)
- [What Does "Hosting" Actually Mean?](#what-does-hosting-actually-mean)
- [Physical Server vs Virtual Server](#physical-server-vs-virtual-server)
- [The Hosting Spectrum](#the-hosting-spectrum)
- [Shared Hosting](#shared-hosting)
- [VPS (Virtual Private Server)](#vps-virtual-private-server)
- [Dedicated Server](#dedicated-server)
- [PaaS (Platform as a Service)](#paas-platform-as-a-service)
- [Serverless](#serverless)
- [Static Hosting](#static-hosting)
- [Comparison Table](#comparison-table)
- [Which One Should You Actually Use?](#which-one-should-you-actually-use)
- [Popular Providers Cheat Sheet](#popular-providers-cheat-sheet)
- [Quick Revision](#quick-revision)

## What Is a Server, Physically?

- At the most basic physical level, a **server** is just a computer — it has a CPU, RAM, storage (disk), and a network connection, exactly like your laptop.
- The difference is *purpose*: your laptop is built for you to use directly (screen, keyboard, battery). A server is built to stay on permanently in a data center, with no screen attached, and to be accessed remotely over the network.
- Companies that own huge buildings full of these computers are called **data centers**. Examples: Amazon (AWS), Google (GCP), Microsoft (Azure), DigitalOcean.

> 💡 **Tip:** You never physically touch the server you deploy to. Everything is done remotely, through the internet, using a terminal. This will feel strange at first — you're "typing commands into a computer somewhere else in the world."

## What Does "Hosting" Actually Mean?

- **Hosting** means renting time and resources on someone else's computer (server) so your app can run there instead of on your personal laptop.
- You pay a **hosting provider** (e.g., DigitalOcean, AWS, Railway) a monthly fee, and in exchange you get access to a server (or a slice of one) to run your application on.
- This solves the core problem from deploy-01: your laptop can't stay on 24/7 with a stable public IP address, but a hosting provider's servers can.

## Physical Server vs Virtual Server

This is one of the most important distinctions to understand early.

- **Physical server (bare metal)**: One entire physical machine, dedicated to you. Nobody else's code runs on it.
- **Virtual server (VPS)**: A hosting company takes one big physical machine and uses software called a **hypervisor** to split it into multiple independent "virtual" computers. Each virtual computer thinks it's its own machine, but it's actually sharing physical hardware with other customers.

```text
Physical Machine (1 real computer)
│
├── Virtual Server 1 (yours)   ← isolated, thinks it's alone
├── Virtual Server 2 (another customer)
└── Virtual Server 3 (another customer)
```

- **Hypervisor**: The software layer that creates and manages these virtual servers, keeping them isolated from each other so one customer can't see or affect another's data.
- This is why VPS hosting is much cheaper than renting a whole physical server — you're sharing the underlying hardware cost with other customers, while still getting your own private, isolated environment.

> ⚠️ **Warning:** "Virtual" does not mean "less real" or "less capable." A VPS behaves exactly like a full Linux computer that you can install anything on — it's just sharing physical hardware behind the scenes. Beginners sometimes assume VPS is a lesser/fake server; it isn't.

## The Hosting Spectrum

Hosting options exist on a spectrum from "you manage everything" to "the provider manages everything." Each step up the ladder trades control for convenience.

```text
More control, more responsibility        More convenience, less control
◄─────────────────────────────────────────────────────────────────────►
Bare Metal  →  VPS  →  Managed VPS  →  PaaS  →  Serverless  →  Static Hosting
```

## Shared Hosting

- **Shared hosting**: Many websites (often hundreds) run on the same physical server, sharing its resources. Very cheap, but you have almost no control and performance can suffer if another site on the same server gets a traffic spike.
- Typically sold for simple websites — WordPress blogs, small brochure sites. Common providers: Bluehost, HostGator, Namecheap shared plans.
- You usually can't run custom Node.js/backend processes on shared hosting — it's mainly built for PHP + MySQL stacks accessed through a control panel like cPanel.

> ⚠️ **Warning:** Shared hosting is generally **not suitable** for modern full-stack apps (Node.js backends, custom APIs, Docker, etc.). It's mentioned here mainly so you recognize the term and know to avoid it for this kind of project.

## VPS (Virtual Private Server)

- A **VPS** gives you your own isolated slice of a physical server, with root/administrator access, a fixed amount of CPU/RAM/disk, and complete freedom to install anything.
- You are responsible for everything: installing the operating system's updates, configuring the web server, setting up security, keeping the app running, etc.
- This is the classic "learn how servers actually work" path, and it's what we'll deep-dive into for the rest of this series (SSH, Linux, Nginx, process managers, etc.).

```bash
# Example: what "having a VPS" lets you do — connect directly to your own machine
ssh root@203.0.113.25
```

- That single command is the entire concept of a VPS in action: a remote computer, at a specific address, that you can log into and control directly.

> 💡 **Tip:** Learning to deploy on a raw VPS is the single most valuable exercise for understanding "how the internet actually works" as a developer. Even senior engineers who use managed platforms daily benefit from having done this manually at least once.

## Dedicated Server

- A **dedicated server** is an entire physical machine rented just for you — no hypervisor splitting it with other customers.
- Used for large-scale, high-performance, or highly regulated applications (banks, large e-commerce) where sharing hardware with strangers is unacceptable for performance or compliance reasons.
- Significantly more expensive than a VPS. Not something you'll need for learning or early-stage projects.

## PaaS (Platform as a Service)

- **PaaS**: You give the platform your code (usually via a Git push), and it automatically handles building, deploying, running, and often scaling it — no server configuration required from you.
- You don't manage the operating system, security patches, or web server configuration at all — the platform abstracts all of that away.
- Examples: Railway, Render, Heroku, Fly.io.

```bash
# Example: a typical PaaS deployment is often this simple
git push origin main
# the platform detects the push, builds your app, and deploys it automatically
```

> 💡 **Tip:** PaaS is excellent for shipping quickly and for production apps where you don't want to spend engineering time on server maintenance. Many real companies deploy this way permanently — it's not just a "beginner" option.

## Serverless

- **Serverless**: You write individual functions (small units of code) that the provider runs on-demand, for the duration of a single request, then shuts down. There is no persistent server running at all — even though physical servers are obviously still involved behind the scenes (hence the name being a bit of a misnomer).
- You are billed per-execution (e.g., per function call), not for a server sitting idle 24/7.
- Examples: AWS Lambda, Vercel Functions, Cloudflare Workers.
- Ideal for: APIs with unpredictable/spiky traffic, background jobs, image processing triggers.
- Less ideal for: apps needing a constantly-running process (e.g., WebSocket servers, long background workers).

## Static Hosting

- **Static hosting**: Hosting that only serves pre-built files — HTML, CSS, JavaScript, images — with no backend code execution at all.
- Perfect for frontend-only apps (a React build with no server-side logic) or a documentation site.
- Examples: Netlify, GitHub Pages, Cloudflare Pages, Vercel (for static/frontend output).
- If your app needs a database or backend logic, static hosting alone isn't enough — you'd combine it with a separate backend hosted elsewhere.

## Comparison Table

| Hosting Type | You Manage | Best For | Typical Cost | Example Providers |
|---|---|---|---|---|
| **Shared Hosting** | Almost nothing | Simple WordPress/PHP sites | $ (very cheap) | Bluehost, Namecheap |
| **VPS** | OS, security, web server, app, everything | Learning, full control, custom stacks | $$ | DigitalOcean, Linode, Vultr |
| **Dedicated Server** | Everything, on your own hardware | High-scale, compliance-heavy apps | $$$$ | OVH, Hetzner dedicated |
| **PaaS** | Just app code and env config | Fast shipping, small-to-mid teams | $$ | Railway, Render, Heroku |
| **Serverless** | Just individual functions | Spiky APIs, event-driven tasks | Pay-per-use | AWS Lambda, Vercel Functions |
| **Static Hosting** | Just static files | Frontend-only sites | Often free | Netlify, GitHub Pages |

## Which One Should You Actually Use?

- For **this series**, we're going to deploy on a **VPS**, because it forces you to understand every layer (OS, web server, process manager, security, HTTPS) — this is the foundation that makes every other option (PaaS, serverless) make sense later.
- Once you understand VPS deployment deeply, switching to a PaaS in a real job will feel trivial — you'll understand what the platform is doing for you under the hood.
- For a full-stack app (frontend + backend + admin panel + database), you'll typically end up with a **combination**: e.g., frontend on static hosting, backend + admin panel on a VPS or PaaS, and database on a managed database service. We'll design this exact combination in the final real-world walkthrough at the end of this series.

## Popular Providers Cheat Sheet

| Provider | Type | Notes |
|---|---|---|
| **DigitalOcean** | VPS ("Droplets") | Beginner-friendly, clear pricing, great docs |
| **Linode (Akamai)** | VPS | Similar to DigitalOcean, reliable |
| **Vultr** | VPS | Similar, slightly cheaper in some regions |
| **Hetzner** | VPS / Dedicated | Very cheap, popular in Europe |
| **AWS (Amazon)** | VPS (EC2), Serverless (Lambda), and more | Industry standard, but complex for beginners |
| **Railway** | PaaS | Very beginner-friendly, git-push deploys |
| **Render** | PaaS | Similar to Railway, good free tier |
| **Vercel** | Serverless / Static | Best for frontend frameworks like Next.js |
| **Netlify** | Static | Great for static frontend sites |

> ⚠️ **Warning:** AWS is the industry standard and worth learning eventually, but it has a steep learning curve and its pricing/interface can be confusing for a first deployment. We'll start with a simpler VPS provider (like DigitalOcean) so the *concepts* aren't drowned out by interface complexity.

## Quick Revision

- A server is just a computer that stays on 24/7 and is accessed remotely; hosting is renting access to one.
- A VPS is a virtual slice of a physical machine, created by a hypervisor, that behaves exactly like a full independent Linux computer — it's not "less real," just shared hardware.
- Hosting exists on a spectrum: shared hosting → VPS → dedicated server → PaaS → serverless → static hosting, trading control for convenience as you move along it.
- Shared hosting is cheap but unsuitable for modern custom full-stack apps; it's mentioned mainly so you recognize and avoid it here.
- VPS gives full control and is the best way to learn how servers truly work — this series will deploy on a VPS for that reason.
- PaaS (Railway, Render, Heroku) automates deployment via git push and is genuinely used in production by real companies, not just beginners.
- Serverless runs individual functions on-demand with no persistent server, billed per-execution — great for spiky APIs, not ideal for always-on processes.
- Real-world full-stack apps often combine multiple hosting types: static hosting for frontend, VPS/PaaS for backend + admin panel, managed service for database — exactly what we'll build in the final walkthrough.