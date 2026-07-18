# Monitoring & Logging

Monitoring means watching your app's health in real time so you know something is wrong before your users tell you, and logging means recording what your app is doing so you can investigate what actually happened.

## Table of Contents

- [Why This Matters](#why-this-matters)
- [Monitoring vs Logging: The Difference](#monitoring-vs-logging-the-difference)
- [What Is a Log?](#what-is-a-log)
- [Log Levels Explained](#log-levels-explained)
- [Where Do Logs Go?](#where-do-logs-go)
- [Structured Logging vs Plain Text Logging](#structured-logging-vs-plain-text-logging)
- [A Simple Logging Example](#a-simple-logging-example)
- [Log Rotation: Stopping Logs From Filling Your Disk](#log-rotation-stopping-logs-from-filling-your-disk)
- [Centralized Logging Tools](#centralized-logging-tools)
- [What Is Monitoring, More Precisely?](#what-is-monitoring-more-precisely)
- [Key Metrics You Should Watch](#key-metrics-you-should-watch)
- [Health Checks Explained](#health-checks-explained)
- [Uptime Monitoring](#uptime-monitoring)
- [Alerting: Getting Notified Before Users Complain](#alerting-getting-notified-before-users-complain)
- [Error Tracking Tools](#error-tracking-tools)
- [Putting It All Together: A Realistic Beginner Setup](#putting-it-all-together-a-realistic-beginner-setup)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## Why This Matters

- Once your app is deployed and running on a server, you can't just watch it directly like you do on your laptop — it's running far away, all the time, with no one staring at the screen.
- Without monitoring and logging, the very first sign of a problem is often an angry user, or worse, silence — the app going down and nobody noticing for hours or days.
- **Senior engineers are defined partly by this skill**: not just building features, but knowing how to observe a running system, catch problems early, and figure out exactly what went wrong after something breaks. This topic is genuinely one of the biggest gaps between junior and senior-level thinking.

## Monitoring vs Logging: The Difference

These two words get used together so often that beginners sometimes think they're the same thing — they're related, but distinct.

| | Logging | Monitoring |
|---|---|---|
| **What it is** | A detailed written record of events that happened | A live, ongoing view of your system's current health |
| **Answers the question** | "What exactly happened, step by step?" | "Is everything okay right now?" |
| **Format** | Text entries, often with timestamps | Numbers, graphs, alerts |
| **Example** | "User 4521 failed to log in at 10:32am — wrong password" | "CPU usage is at 92% right now" |
| **Used for** | Debugging a specific incident after the fact | Noticing something is wrong, often before it fully breaks |

> 💡 **Tip:** A simple way to remember it: **logs tell a story**, **monitoring shows a dashboard**. You often use monitoring to notice something is wrong, then dig into the logs to understand *why*.

## What Is a Log?

- A **log** is a text-based record your application writes describing events as they happen — a request came in, a user logged in, an error occurred, a background job finished.
- Every serious application produces logs constantly, even if you never look at them until something goes wrong.
- Logs are typically written to a **log file**, or increasingly to a centralized logging system, rather than just printed to a terminal that nobody's watching.

## Log Levels Explained

Not all log messages are equally important. **Log levels** categorize messages by severity, so you can filter what you actually pay attention to.

| Level | Meaning | Example |
|---|---|---|
| **DEBUG** | Very detailed, only useful while actively debugging | "Cache lookup took 4ms" |
| **INFO** | Normal, expected events worth recording | "User 4521 logged in successfully" |
| **WARN** | Something unusual, not broken yet, but worth watching | "API response took 8 seconds (slow)" |
| **ERROR** | Something failed | "Failed to connect to database" |
| **FATAL / CRITICAL** | A severe failure, often causing the app to crash | "Out of memory, shutting down" |

- In production, it's common to only actively watch for `WARN` and above, since `DEBUG`/`INFO` logs are extremely high-volume and mostly useful for deep, specific investigation rather than everyday monitoring.

> 💡 **Tip:** A good habit while writing code is to genuinely think about which level fits each log message. Logging everything as `ERROR` (even harmless events) trains you — and your team — to start ignoring error alerts entirely, which defeats the whole purpose.

## Where Do Logs Go?

- **Console output**: The simplest form — your app prints messages using something like `console.log()`. When run under a process manager (PM2 or systemd, from deploy-06), this output is automatically captured.
- With **systemd**, console output from your app is automatically captured into the **journal**, viewable with `journalctl -u yourservice`, exactly as you learned in deploy-06.
- With **PM2**, output is automatically captured into log files, viewable with `pm2 logs myapp`, also from deploy-06.
- With **Docker**, container output is captured and viewable with `docker logs mycontainer`, as shown in deploy-09.

> 💡 **Tip:** Notice that this note isn't introducing a totally new concept — you've actually already been using basic logging this whole series, every time you ran `journalctl`, `pm2 logs`, or `docker logs`. This note goes deeper into doing it well, deliberately, and at scale.

## Structured Logging vs Plain Text Logging

- **Plain text logging**: A log message is just a human-readable sentence.

```text
2026-07-10 14:32:01 ERROR Failed to connect to database for user 4521
```

- **Structured logging**: A log message is written as structured data (commonly JSON), making it easy for tools and scripts to search, filter, and analyze automatically — not just for humans to read.

```json
{
  "timestamp": "2026-07-10T14:32:01Z",
  "level": "error",
  "message": "Failed to connect to database",
  "userId": 4521,
  "service": "backend-api"
}
```

- Structured logs are especially valuable once you're searching through thousands or millions of log lines — you can precisely filter (e.g., "show me every ERROR for `userId: 4521`") instead of manually scanning text.

> 💡 **Tip:** As a beginner, plain text logs (like simple `console.log` statements) are perfectly fine to start with. Structured logging becomes genuinely valuable once your app grows and you start using centralized logging tools (covered below) that can search and filter structured fields.

## A Simple Logging Example

Using a popular Node.js logging library called **Winston** as an example of doing this properly, instead of scattered `console.log` calls:

```bash
npm install winston
```

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
  ],
});

logger.info('Server started', { port: 3000 });
logger.error('Database connection failed', { error: err.message });
```

- `level: 'info'` — sets the minimum level this logger will actually record; `debug` messages would be ignored here, but `info`, `warn`, and `error` would all be recorded.
- `format: winston.format.json()` — outputs structured JSON logs, as described above.
- `transports` — defines *where* logs are sent: here, both the console (for immediate viewing) and a dedicated `error.log` file (capturing only error-level messages).

## Log Rotation: Stopping Logs From Filling Your Disk

- If logs are simply written to a growing file forever, that file will eventually consume all available disk space on your server — and a full disk can crash your entire app or even the whole server.
- **Log rotation** is the practice of automatically archiving or deleting old log data on a schedule, keeping log files at a manageable size.
- On Ubuntu, a built-in tool called **logrotate** handles this automatically for many system logs already, and can be configured for your own app's logs too.

```text
# /etc/logrotate.d/myapp
/var/www/myapp/logs/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
}
```

- `daily` — rotate (archive) the log file once a day.
- `rotate 14` — keep 14 days' worth of rotated archives, then delete older ones.
- `compress` — compress old rotated logs to save disk space.
- `missingok` — don't error out if the log file doesn't exist yet.
- `notifempty` — don't rotate if the log file is currently empty.

> ⚠️ **Warning:** A genuinely common real-world production incident is a server going down not because of a bug in the app itself, but because logs silently filled up the entire disk over weeks or months. Setting up log rotation early prevents this completely.

## Centralized Logging Tools

- For a single small app on one server, reading log files directly (`journalctl`, `pm2 logs`) is usually enough.
- Once you have multiple servers, multiple services, or containers, logs become scattered across many places — a **centralized logging tool** collects logs from everywhere into one searchable place.

| Tool | Notes |
|---|---|
| **ELK Stack (Elasticsearch, Logstash, Kibana)** | Powerful, widely used, but more complex to set up and run yourself |
| **Grafana Loki** | Lightweight, pairs well with Grafana dashboards (mentioned below) |
| **Datadog** | Popular managed/paid platform combining logging and monitoring together |
| **Better Stack / Papertrail** | Simpler, beginner-friendly managed logging services |

> 💡 **Tip:** You don't need any of these for a small personal project — `journalctl`/`pm2 logs`/`docker logs` will genuinely serve you well for a long time. Understanding *why* centralized logging exists (searching across many services/servers at once) is the important part for now.

## What Is Monitoring, More Precisely?

- **Monitoring** is the ongoing, automated collection of metrics — numeric measurements about your system's health — displayed in dashboards and used to trigger alerts.
- Unlike logs (which record discrete events), metrics are typically **continuous measurements over time** — e.g., "CPU usage every 10 seconds," graphed as a line over time.

## Key Metrics You Should Watch

| Metric | What It Tells You |
|---|---|
| **CPU usage** | Is the server's processor overloaded? |
| **Memory (RAM) usage** | Is the app using more memory than available, risking a crash? |
| **Disk usage** | Is the server running out of storage space (recall the log rotation warning above)? |
| **Response time** | How long does your app take to respond to requests? |
| **Request rate** | How much traffic is your app currently handling? |
| **Error rate** | What percentage of requests are failing? |
| **Uptime** | Is the app currently reachable at all? |

```bash
# Quick, built-in Linux commands to check some of these manually
top          # live view of CPU and memory usage by process
htop         # a nicer, interactive version of top (may need: sudo apt install htop)
df -h        # disk usage (introduced back in deploy-03)
free -h      # memory usage summary
```

> 💡 **Tip:** These simple built-in commands are a genuinely good starting point, and worth checking manually on your server occasionally even before you set up any dedicated monitoring tooling.

## Health Checks Explained

- A **health check** is a simple, dedicated endpoint in your app (e.g., `GET /health`) that returns a quick "yes, I'm alive and working" response, used by monitoring tools to check your app's status automatically and repeatedly.

```javascript
// A simple health check endpoint example
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

- A more thorough health check might also verify that critical dependencies (like the database) are reachable, not just that the app process itself is running:

```javascript
app.get('/health', async (req, res) => {
  try {
    await db.query('SELECT 1');
    res.status(200).json({ status: 'ok', database: 'connected' });
  } catch (err) {
    res.status(503).json({ status: 'error', database: 'unreachable' });
  }
});
```

- `res.status(503)` — deliberately returns an HTTP error status when something critical is broken, since monitoring tools typically check the status code to decide if your app is "healthy."

## Uptime Monitoring

- **Uptime monitoring** is a service that repeatedly checks whether your website/app is reachable (often by hitting your `/health` endpoint), from outside your own server, and alerts you immediately if it stops responding.
- This matters because it checks your app the same way a real user would — from the outside, over the internet — catching problems your own server might not even be aware of (e.g., a DNS misconfiguration, or the entire server being unreachable).

| Tool | Notes |
|---|---|
| **UptimeRobot** | Popular, has a genuinely useful free tier, beginner-friendly |
| **Better Stack (Uptime)** | Combines uptime monitoring with incident alerting |
| **Pingdom** | Established, more enterprise-focused |

> 💡 **Tip:** Setting up free uptime monitoring (like UptimeRobot) takes about five minutes and is one of the highest-value, lowest-effort things you can do for any deployed project — it turns "I found out my site was down because a friend told me" into "I got an alert the moment it happened."

## Alerting: Getting Notified Before Users Complain

- **Alerting** is the practice of automatically notifying a human (you) when monitoring detects a problem — via email, SMS, Slack, or another channel — instead of requiring you to constantly watch a dashboard yourself.
- Good alerts are set up around meaningful **thresholds** — e.g., "alert me if CPU usage stays above 90% for more than 5 minutes," not "alert me every single time CPU usage moves at all."

> ⚠️ **Warning:** Setting alert thresholds too sensitively creates **alert fatigue** — so many notifications that you start ignoring them entirely, including the ones that actually matter. This is a well-known, real problem in the industry, not just a beginner mistake — tuning alerts thoughtfully is a genuine skill.

## Error Tracking Tools

- Beyond general logs and metrics, **error tracking tools** specialize in capturing, grouping, and alerting on application errors specifically — often with much richer detail than a plain log line, like the full stack trace, the user affected, and how often that exact error has occurred.

```bash
npm install @sentry/node
```

```javascript
const Sentry = require('@sentry/node');
Sentry.init({ dsn: 'your-sentry-dsn-here' });

// Errors are now automatically captured and reported
```

- **Sentry** is the most widely used tool in this category, supporting virtually every major language/framework, with a genuinely useful free tier for small projects.

## Putting It All Together: A Realistic Beginner Setup

For a personal or early-stage project, here's a realistic, non-overwhelming monitoring/logging setup:

```text
1. Use your process manager's built-in logs (journalctl / pm2 logs / docker logs)
2. Set up log rotation so logs never fill your disk
3. Add a /health endpoint to your app
4. Set up free uptime monitoring (e.g., UptimeRobot) pointed at that /health endpoint
5. Add error tracking (e.g., Sentry free tier) to catch and alert on real app errors
6. Check `top`/`htop`/`df -h`/`free -h` manually now and then, or set basic alert thresholds
```

- This setup takes a few hours total to configure, and covers the overwhelming majority of what a small production app genuinely needs — the more advanced tools (ELK, Datadog, Grafana) become relevant as your app and traffic grow significantly.

## Common Mistakes Beginners Make

- Never checking logs at all until something breaks badly, instead of periodically glancing at them.
- Not setting up log rotation, eventually filling the server's disk and causing an outage.
- Logging everything as `ERROR` regardless of actual severity, training yourself to ignore alerts.
- Having no external uptime monitoring, so the app can be down for hours before anyone notices.
- Setting alert thresholds so sensitively that alert fatigue sets in and real problems get missed among the noise.

## Quick Revision

- Logging records what happened, in detail, after the fact; monitoring shows your system's live health right now — they work together, not instead of each other.
- Log levels (DEBUG, INFO, WARN, ERROR, FATAL) let you filter log messages by real severity; logging everything as ERROR trains you to ignore alerts.
- You've already been using basic logging tools all series — `journalctl`, `pm2 logs`, and `docker logs` — this note goes deeper into doing it deliberately and at scale.
- Log rotation (via `logrotate`) prevents logs from silently filling up your server's disk and causing an outage — a genuinely common real-world incident cause.
- Key metrics to watch include CPU, memory, disk usage, response time, and error rate — checkable manually with `top`, `htop`, `df -h`, and `free -h`.
- A `/health` endpoint gives monitoring tools a simple, standard way to check whether your app (and its critical dependencies) are actually working.
- Free uptime monitoring (like UptimeRobot) checks your app from the outside world and alerts you the moment it becomes unreachable — a high-value, low-effort setup step.
- Alert thresholds should be tuned thoughtfully to avoid alert fatigue, where too many notifications cause real problems to get missed.