# Servers & Hosting Models

A server is simply a computer that stays turned on and connected to the internet so it can respond to requests, and hosting is the service that provides and manages that computer for you.

## Table of Contents

- [What Is a Server, Physically?](#what-is-a-server-physically)
- [What Is Hosting?](#what-is-hosting)
- [Physical Server vs Virtual Server](#physical-server-vs-virtual-server)
- [The Main Hosting Models Explained](#the-main-hosting-models-explained)
- [Shared Hosting vs VPS vs Dedicated vs Cloud](#shared-hosting-vs-vps-vs-dedicated-vs-cloud)
- [What Is "The Cloud" Actually?](#what-is-the-cloud-actually)
- [Choosing a Model for Your Own Project](#choosing-a-model-for-your-own-project)
- [Popular Providers at a Glance](#popular-providers-at-a-glance)
- [Quick Revision](#quick-revision)

## What Is a Server, Physically?

- A **server** is just a computer — it has a CPU, RAM, storage, and a network connection, exactly like your laptop.
- The difference is *purpose*: a server is built and configured to stay on 24/7 and answer requests from other computers ("clients"), instead of being used interactively by one person sitting in front of it.
- Servers live in **data centers** — large buildings full of racks of these computers, with backup power, cooling, and high-speed internet connections.

```text
Your Laptop (client)  --- request --->  Server (data center, far away)
Your Laptop (client)  <--- response ---  Server
```

> 💡 **Tip:** When people say "deploy to the cloud," they really mean "put your code on someone else's server, which they let you rent by the hour or month."

## What Is Hosting?

- **Hosting** is a service where a company owns and maintains physical servers, and rents out computing power/storage/network on those servers to you.
- Instead of buying a $2,000 physical server, setting it up in your house, and paying for internet and electricity for it, you pay a hosting company a monthly fee and they handle the hardware.
- You are essentially renting: **compute** (CPU/RAM to run your app), **storage** (disk space), and **network** (bandwidth, a public IP address).

> ⚠️ **Warning:** "Hosting" and "domain" are two completely different purchases. Hosting is *where your app's code and data physically run*. A domain (like `yourapp.com`) is just a human-friendly name that *points to* your hosting's IP address. You can have hosting without a domain (accessed by raw IP) and you can own a domain with no hosting behind it yet.

## Physical Server vs Virtual Server

- **Bare metal server**: You rent (or own) an entire physical machine. Nobody else's code runs on it. Maximum performance and control, but expensive and you manage everything.
- **Virtual server (VPS = Virtual Private Server)**: One physical machine is split into several isolated virtual machines using software called a **hypervisor**. You get root access to your slice, but you're sharing physical hardware with other customers (invisibly to you).
- **Virtualization**: The technology that lets one physical computer pretend to be several separate computers, each with its own OS, isolated from the others.

```text
Physical Server (1 machine)
 ├── Virtual Server A (rented by Customer 1) — has its own OS, feels like a full computer
 ├── Virtual Server B (rented by Customer 2)
 └── Virtual Server C (rented by Customer 3)
```

> 💡 **Tip:** For 95% of learning projects and even many real startups, a VPS is more than powerful enough. Bare metal is usually only needed at large scale (heavy databases, high traffic, specific compliance needs).

## The Main Hosting Models Explained

- **Shared hosting**: Many customers' websites run on the *same* server, sharing the same resources and often the same software stack. Cheapest option, least control, and one customer's heavy traffic can slow down others.
- **VPS hosting**: You get your own isolated virtual server with root access. You install and configure everything yourself (OS, web server, runtime, database). Full control, more responsibility.
- **Dedicated hosting**: You rent an entire physical machine, used by nobody else. Most powerful and most expensive of the "you manage it" options.
- **PaaS (Platform as a Service)**: You give the platform your code (often via `git push`), and it handles the server, OS, and scaling for you. Examples: Railway, Render, Heroku.
- **Serverless**: You write individual functions; the platform runs them only when needed and you're billed per execution, not per server-hour. Examples: AWS Lambda, Vercel Functions.
- **Static hosting**: For sites with no backend server logic (just HTML/CSS/JS files) — the host just serves files. Examples: Netlify, GitHub Pages, Cloudflare Pages.

> ⚠️ **Warning:** A very common beginner trap is picking "shared hosting" (often marketed to build WordPress sites cheaply) for a custom Node.js/PHP backend app. Most cheap shared hosting plans do **not** let you run a persistent Node.js/Python process or install custom software — check this before buying.

## Shared Hosting vs VPS vs Dedicated vs Cloud

| Model | Control | Cost | Setup Difficulty | Who It's For |
|---|---|---|---|---|
| **Shared Hosting** | Very low | $ | Very easy | Simple WordPress/PHP sites |
| **VPS** | High | $$ | Medium–Hard | Custom apps, learning real deployment |
| **Dedicated Server** | Full | $$$$ | Hard | High-traffic, enterprise apps |
| **PaaS** | Medium | $$ | Easy | Fast shipping without infra headaches |
| **Serverless** | Low (per-function) | Pay-per-use | Easy–Medium | Spiky, event-driven workloads |
| **Static Hosting** | Low (no backend) | Often free | Very easy | Portfolios, landing pages, frontend-only apps |

## What Is "The Cloud" Actually?

- "The cloud" is not a magical concept — it just means **someone else's servers, accessed over the internet, that you can rent flexibly** (by the hour/minute, scaled up or down on demand).
- **Cloud provider**: A company that owns huge numbers of servers across many data centers and rents out compute/storage/networking as a service. Examples: AWS (Amazon), Google Cloud, Microsoft Azure, DigitalOcean.
- **Elasticity**: The ability to automatically add or remove server capacity based on real-time demand — something traditional single-server hosting can't do easily.
- **Region**: A specific geographic location where a cloud provider has a data center (e.g., `us-east-1`, `ap-southeast-1`). Choosing a region close to your users reduces latency.

> 💡 **Tip:** DigitalOcean, Linode, and Vultr are often recommended for beginners learning VPS deployment because their VPS products (called "Droplets" on DigitalOcean) are simpler and cheaper than AWS/GCP/Azure's equivalents, which have a much steeper learning curve.

## Choosing a Model for Your Own Project

- If your goal is **to learn how deployment actually works** (which is your stated goal): use a **VPS**. It forces you to understand Linux, web servers, processes, and security — all transferable knowledge.
- If your goal is **to ship something fast** without learning server internals: use a **PaaS** like Railway or Render.
- If your app is **just static frontend** (no backend server needed): use **static hosting** like Netlify or Vercel.
- If your traffic is **spiky/unpredictable** and cost-per-use matters: consider **serverless**.

> ⚠️ **Warning:** Don't pick a hosting model based only on price. A $4/month VPS that you don't know how to secure properly can be more costly in the long run (hacked servers, data loss) than a slightly pricier managed option. We'll cover security hardening later in this series.

## Popular Providers at a Glance

| Provider | Category | Notes |
|---|---|---|
| **DigitalOcean** | VPS | Beginner-friendly, simple pricing, great docs |
| **Linode (Akamai)** | VPS | Similar to DigitalOcean |
| **Vultr** | VPS | Similar, slightly more global locations |
| **AWS (EC2, etc.)** | Cloud (VPS + everything else) | Extremely powerful, steep learning curve |
| **Google Cloud Platform** | Cloud | Similar scope to AWS |
| **Railway** | PaaS | Very beginner-friendly, `git push` to deploy |
| **Render** | PaaS | Similar to Railway, free tier available |
| **Vercel** | PaaS/Serverless | Best for Next.js/frontend-heavy apps |
| **Netlify** | Static Hosting | Best for pure frontend/static sites |

## Quick Revision

- A server is just a computer built to stay on 24/7 and respond to requests; hosting is the service that rents you access to one.
- Hosting and domains are separate purchases — hosting is where your app runs, a domain is the name that points to it.
- A VPS is a virtual slice of a physical server, giving you root access without owning the whole machine — the sweet spot for learning real deployment.
- Shared hosting is cheap but restrictive; dedicated servers are powerful but expensive; PaaS and serverless trade control for convenience.
- "The cloud" simply means renting someone else's servers flexibly over the internet, with the ability to scale up or down on demand.
- For your stated goal of deeply understanding deployment, a VPS (e.g., DigitalOcean) is the recommended path for this series — full control forces full understanding.
- Always double-check that a hosting plan actually supports running your backend runtime (Node.js, PHP-FPM, etc.) before purchasing it.