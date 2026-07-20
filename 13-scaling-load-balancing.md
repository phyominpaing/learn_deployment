# Scaling & Load Balancing

Scaling means adjusting how much computing power your app has available to handle traffic, and load balancing is the technique of spreading that traffic evenly across multiple servers so no single one gets overwhelmed.

## Table of Contents

- [Why Scaling Becomes Necessary](#why-scaling-becomes-necessary)
- [Vertical Scaling vs Horizontal Scaling](#vertical-scaling-vs-horizontal-scaling)
- [The Limits of Vertical Scaling](#the-limits-of-vertical-scaling)
- [What Is Load Balancing?](#what-is-load-balancing)
- [How a Load Balancer Actually Works](#how-a-load-balancer-actually-works)
- [Load Balancing Algorithms](#load-balancing-algorithms)
- [Setting Up Nginx as a Simple Load Balancer](#setting-up-nginx-as-a-simple-load-balancer)
- [Statelessness: The Key Requirement for Horizontal Scaling](#statelessness-the-key-requirement-for-horizontal-scaling)
- [Session Management Across Multiple Servers](#session-management-across-multiple-servers)
- [Scaling the Database Separately](#scaling-the-database-separately)
- [Caching: Reducing Load Before It Happens](#caching-reducing-load-before-it-happens)
- [Auto-Scaling](#auto-scaling)
- [Managed Load Balancers](#managed-load-balancers)
- [Do You Need Any of This Right Now?](#do-you-need-any-of-this-right-now)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## Why Scaling Becomes Necessary

- Every server has limited resources: a fixed amount of CPU, RAM, and network capacity. As more users visit your app, it needs to handle more simultaneous requests, more database queries, and more data — and eventually, one server isn't enough.
- **Scaling** is the general term for increasing your app's capacity to handle more load (more users, more traffic, more data) without it slowing down or crashing.
- This becomes a real, practical concern once an app grows past a small number of users — for a personal project, you likely won't need most of this note yet, but understanding it well is genuinely part of thinking like a senior engineer.

## Vertical Scaling vs Horizontal Scaling

There are two fundamentally different ways to give your app more capacity.

- **Vertical scaling** ("scaling up"): Making your *existing* single server more powerful — more CPU cores, more RAM, faster storage.
- **Horizontal scaling** ("scaling out"): Adding *more* servers, and spreading the traffic/work across all of them together.

```text
Vertical Scaling                    Horizontal Scaling
┌─────────────┐                    ┌─────┐  ┌─────┐  ┌─────┐
│   Bigger      │                    │Server│  │Server│  │Server│
│   Single       │        vs          │  1   │  │  2   │  │  3   │
│   Server       │                    └─────┘  └─────┘  └─────┘
└─────────────┘                       (traffic spread across all three)
```

| | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| **How** | Upgrade the existing server's specs | Add more servers |
| **Complexity** | Simple — usually a few clicks with your VPS provider | More complex — requires load balancing, shared state handling |
| **Downtime during upgrade** | Often requires a restart/brief downtime | Can often be done with zero downtime |
| **Ceiling** | Limited — there's a maximum size a single server can be | Much higher — can theoretically keep adding servers |
| **Redundancy** | None — if that one server fails, everything goes down | Better — if one server fails, others keep serving traffic |

> 💡 **Tip:** Vertical scaling is almost always the right *first* move, because it's simple. Most personal projects and even many real small-to-mid-sized businesses run comfortably on a single, reasonably-sized VPS for a long time before horizontal scaling becomes genuinely necessary.

## The Limits of Vertical Scaling

- Every VPS provider has a maximum server size they offer — you can't scale a single machine infinitely, and extremely large single servers become disproportionately expensive.
- Vertical scaling also has zero redundancy: if that one, bigger server crashes, has hardware failure, or needs a security patch requiring a reboot, your entire app goes down completely, with nothing else to fall back on.
- This is exactly the gap horizontal scaling and load balancing are designed to close.

## What Is Load Balancing?

- A **load balancer** is a component that sits in front of multiple copies (instances) of your app running on different servers, and distributes incoming traffic across them.
- This directly builds on the reverse proxy concept from deploy-04 — a load balancer is essentially a reverse proxy that has *multiple possible destinations* to forward each request to, instead of just one.

```text
Internet Users
     │
     ▼
Load Balancer
     │
     ├──► App Server 1
     ├──► App Server 2
     └──► App Server 3
```

- If one of those app servers crashes or becomes unreachable, the load balancer detects this and simply stops sending it traffic, routing everything to the remaining healthy servers instead — this is a major source of the redundancy horizontal scaling provides.

## How a Load Balancer Actually Works

- The load balancer continuously performs **health checks** (recall this concept from deploy-11) against each backend server — typically hitting each server's `/health` endpoint on a regular interval.
- Servers that respond successfully are considered healthy and remain in the pool of servers eligible to receive traffic.
- Servers that fail health checks are automatically removed from that pool until they recover, at which point they're added back automatically.

## Load Balancing Algorithms

There are different strategies for *how* the load balancer decides which server should get each incoming request.

| Algorithm | How It Works |
|---|---|
| **Round robin** | Requests are distributed to each server in simple rotating order (1, 2, 3, 1, 2, 3...) |
| **Least connections** | Send the next request to whichever server currently has the fewest active connections |
| **IP hash** | Requests from the same client IP are consistently sent to the same server |
| **Weighted round robin** | Like round robin, but more powerful servers receive a proportionally larger share of requests |

> 💡 **Tip:** **Round robin** is the simplest and most commonly used default, and is genuinely sufficient for the majority of real-world use cases. You don't need to overthink which algorithm to pick starting out.

## Setting Up Nginx as a Simple Load Balancer

Nginx (already your reverse proxy from deploy-04) can also act as a basic load balancer, using an `upstream` block:

```nginx
upstream myapp_backend {
    server 10.0.0.1:4000;
    server 10.0.0.2:4000;
    server 10.0.0.3:4000;
}

server {
    listen 80;
    server_name yourapp.com;

    location / {
        proxy_pass http://myapp_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- `upstream myapp_backend { ... }` — defines a named group of backend servers, each running an identical copy of your app.
- `server 10.0.0.1:4000;` — each line is one server in the pool, specified by its internal IP address and port. By default, Nginx uses round robin across these.
- `proxy_pass http://myapp_backend;` — instead of pointing to a single `localhost:PORT` (as in deploy-04), this points to the whole named upstream group, and Nginx handles distributing requests across it.

```nginx
# Example with weighted round robin and least connections
upstream myapp_backend {
    least_conn;
    server 10.0.0.1:4000 weight=3;
    server 10.0.0.2:4000 weight=1;
}
```

- `least_conn;` — switches the algorithm to "least connections" instead of the default round robin.
- `weight=3` — this server receives roughly three times as much traffic as a server with `weight=1`, useful if your servers have different capacities.

> ⚠️ **Warning:** Using Nginx itself as the load balancer means Nginx becomes a **single point of failure** — if that one Nginx server goes down, all your backend servers become unreachable, even though they're individually fine. Real large-scale production setups often address this with a managed load balancer (covered below) or multiple redundant Nginx instances.

## Statelessness: The Key Requirement for Horizontal Scaling

- For horizontal scaling to work correctly, your app generally needs to be **stateless** — meaning each server instance doesn't rely on anything stored only in its own local memory or disk that other instances don't also have access to.
- If your app stores something important only in one server's local memory (e.g., a user's login session, an uploaded file saved directly to that server's disk), and the load balancer happens to route that same user's *next* request to a *different* server, that data won't be there — causing broken logins, missing files, or inconsistent behavior.

> ⚠️ **Warning:** This is one of the most common real-world mistakes when scaling horizontally for the first time: an app that worked perfectly on a single server suddenly behaves inconsistently once traffic is spread across multiple servers, because of hidden local state the developer didn't realize existed.

## Session Management Across Multiple Servers

- **Sessions** (keeping track of who's logged in) are a classic example of state that needs special handling for horizontal scaling.
- Instead of storing session data in one server's local memory, the standard solution is storing it in a **shared, external store** that every server instance can access equally — commonly **Redis**, a fast, in-memory data store built exactly for this kind of use case.

```text
Without shared sessions:              With shared sessions (Redis):
User logs in on Server 1              User logs in, session stored in Redis
Next request routed to Server 2       Next request routed to Server 2
Server 2 doesn't know the user   →    Server 2 checks Redis, finds the session
is logged in — broken!                Works correctly, regardless of which
                                       server handled the request
```

- Similarly, uploaded files should be stored in shared, external storage (e.g., AWS S3 or another cloud object storage service) rather than directly on any one server's local disk, for exactly the same reason.

## Scaling the Database Separately

- Notice that everything in this note so far has been about scaling your **app servers** — but your **database** is a separate concern with its own scaling strategies, briefly introduced back in deploy-07's mention of replication and read replicas.
- Multiple app server instances (horizontally scaled) typically still connect to the **same** database (or a properly replicated database setup) — the database itself needs its own dedicated scaling approach, since simply running multiple independent, unsynced database instances would cause serious data consistency problems.

## Caching: Reducing Load Before It Happens

- **Caching** means temporarily storing the result of an expensive operation (like a slow database query) so it can be reused quickly next time, instead of redoing the expensive work every single time.
- Caching is often a far more cost-effective first step than scaling — reducing how much work each request actually requires can eliminate the need for extra servers entirely, for a while.
- **Redis** (already mentioned above for sessions) is also extremely commonly used as a caching layer for exactly this purpose.

```javascript
// Simplified caching example with Redis
const cached = await redis.get(`user:${userId}`);
if (cached) {
  return JSON.parse(cached); // fast — no database query needed
}

const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', 3600); // cache for 1 hour
return user;
```

> 💡 **Tip:** As a general rule of thumb followed by many experienced engineers: **cache before you scale**, and **scale vertically before you scale horizontally**. Reach for the more complex solution only once the simpler ones genuinely aren't enough.

## Auto-Scaling

- **Auto-scaling** is a cloud feature that automatically adds or removes server instances based on real-time demand — e.g., automatically launching two extra servers during a traffic spike, then shutting them down again once traffic returns to normal.
- This requires cloud infrastructure designed for it (commonly available on AWS, Google Cloud, Azure), and typically works together with a managed load balancer to automatically register/deregister the new servers as they come online or shut down.
- This is a genuinely advanced, cost-driven optimization — relevant once you have real, variable-scale production traffic, not something to set up for a small early-stage project.

## Managed Load Balancers

Rather than running Nginx yourself as a load balancer (with its single-point-of-failure limitation noted above), most cloud providers offer a **managed load balancer** as a service.

| Provider | Service Name |
|---|---|
| **AWS** | Elastic Load Balancer (ELB) |
| **DigitalOcean** | Load Balancers |
| **Google Cloud** | Cloud Load Balancing |
| **Azure** | Azure Load Balancer |

- These are built, maintained, and made redundant by the provider itself — removing the single-point-of-failure problem, and typically integrating cleanly with that provider's auto-scaling features.

## Do You Need Any of This Right Now?

- Honestly, for a personal project or an early-stage app: **almost certainly not yet**. A single, reasonably-sized VPS, properly configured with everything from deploy-01 through deploy-11, can comfortably handle a meaningful amount of real traffic.
- The value of this note right now is **understanding the concepts and vocabulary** — statelessness, load balancing, horizontal vs vertical scaling — so that when you eventually do need to scale (or discuss it in a job interview, or work on a team that already has this set up), you genuinely understand what's happening and why.

> 💡 **Tip:** Many real companies run successfully on a single well-configured server for a surprisingly long time. Premature scaling (adding complexity you don't yet need) is a common and genuine mistake, even among experienced engineers — it adds real operational cost and complexity without real benefit until traffic actually demands it.

## Common Mistakes Beginners Make

- Assuming horizontal scaling is needed far earlier than it actually is, adding unnecessary complexity to a small project.
- Storing sessions or uploaded files only in one server's local memory/disk, then being confused when horizontal scaling breaks things.
- Not realizing the database needs its own separate scaling strategy, distinct from scaling the app servers.
- Using a single Nginx instance as a load balancer without realizing it's a new single point of failure.
- Reaching for auto-scaling or complex infrastructure before trying simpler, cheaper solutions like caching or vertical scaling first.

## Quick Revision

- Scaling means increasing your app's capacity to handle more traffic; vertical scaling makes one server bigger, horizontal scaling adds more servers.
- Vertical scaling is simpler and usually the right first move; horizontal scaling offers better redundancy and a much higher ceiling, but adds real complexity.
- A load balancer is a reverse proxy with multiple possible destinations, distributing traffic across several app server instances and removing unhealthy ones automatically.
- Nginx can act as a simple load balancer using an `upstream` block, but becomes a single point of failure itself unless made redundant.
- Horizontal scaling requires your app to be stateless — session data and uploaded files need to live in shared external storage (like Redis or S3), not on any one server's local disk/memory.
- The database needs its own separate scaling strategy (like replication, from deploy-07), distinct from scaling app servers.
- Caching (often with Redis) reduces load before it happens, and is frequently a more cost-effective first step than scaling out.
- Auto-scaling and managed load balancers are advanced, cloud-provider features for handling variable, large-scale production traffic — not something a small early-stage project needs yet.
- For most personal and early-stage projects, a single well-configured VPS is genuinely sufficient for a long time — this note is mainly about understanding the concepts for when you eventually do need them.