# Databases in Production

A production database is where your application's real, persistent data lives permanently, and deploying one safely requires different practices than just running a database locally on your laptop.

## Table of Contents

- [Local Database vs Production Database](#local-database-vs-production-database)
- [Self-Hosted vs Managed Database](#self-hosted-vs-managed-database)
- [Installing a Database Directly on Your VPS](#installing-a-database-directly-on-your-vps)
- [Why Databases Usually Shouldn't Be Publicly Exposed](#why-databases-usually-shouldnt-be-publicly-exposed)
- [Connecting Your App to the Database Securely](#connecting-your-app-to-the-database-securely)
- [Connection Pooling Explained](#connection-pooling-explained)
- [Database Migrations](#database-migrations)
- [Backups: The Most Important Habit](#backups-the-most-important-habit)
- [Replication and Read Replicas (High-Level)](#replication-and-read-replicas-high-level)
- [SQL vs NoSQL: A Quick Recap for Deployment](#sql-vs-nosql-a-quick-recap-for-deployment)
- [Popular Managed Database Providers](#popular-managed-database-providers)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## Local Database vs Production Database

- On your laptop, you likely run a database directly (e.g., PostgreSQL or MySQL installed locally, or a lightweight file-based one like SQLite) with no password protection worth worrying about, since only you can access it.
- In production, your database holds **real user data** — accounts, orders, personal information — so it needs to be protected, backed up, and kept running reliably, since data loss or a breach has real consequences.
- The core question for production is: **where does the database actually run?** — on the same VPS as your app, or as a separate managed service. This decision matters more than it might seem at first.

## Self-Hosted vs Managed Database

- **Self-hosted database**: You install the database software (e.g., PostgreSQL, MySQL) yourself on a server (could be the same VPS as your app, or a separate one), and you're responsible for configuring, securing, backing up, and updating it.
- **Managed database**: A cloud provider runs and maintains the database server for you — you just get a connection string. The provider handles backups, security patches, scaling, and failover.

| Aspect | Self-Hosted | Managed |
|---|---|---|
| **Setup effort** | You install and configure everything | Usually a few clicks in a dashboard |
| **Backups** | You must set them up yourself | Usually automatic, included |
| **Security patching** | Your responsibility | Handled by the provider |
| **Cost** | Cheaper (just server cost) | Somewhat more expensive |
| **Control** | Full control over every setting | Some settings are restricted |
| **Good for** | Learning, tight budgets, full control needs | Real production apps, teams, less operational burden |

> 💡 **Tip:** For learning purposes, self-hosting a database on your VPS is valuable — it teaches you what a managed service is actually doing for you behind the scenes. For a real production app with real users, a managed database is generally the safer and more time-efficient choice, especially early in your career, because backup and security mistakes with a self-hosted database can be costly.

## Installing a Database Directly on Your VPS

If self-hosting, here's what it looks like conceptually for PostgreSQL on Ubuntu:

```bash
# Install PostgreSQL
sudo apt update
sudo apt install postgresql postgresql-contrib

# Check it's running
sudo systemctl status postgresql

# Switch to the default postgres system user to manage the database
sudo -i -u postgres

# Open the PostgreSQL interactive shell
psql
```

```sql
-- Inside psql: create a dedicated database and user for your app
CREATE DATABASE myapp_production;
CREATE USER myapp_user WITH ENCRYPTED PASSWORD 'a-strong-password-here';
GRANT ALL PRIVILEGES ON DATABASE myapp_production TO myapp_user;
```

- Notice this creates a **dedicated user** for your app, rather than using the database's default superuser — this limits what your app can do if it were ever compromised, following the same "don't run everything as root" principle from deploy-03.

> ⚠️ **Warning:** Never use simple, guessable, or default passwords for database users, even "temporarily while testing." Automated bots constantly scan the internet for exposed databases with weak credentials — this is one of the most common real-world breach causes.

## Why Databases Usually Shouldn't Be Publicly Exposed

- By default, a freshly installed database often only listens for connections from `localhost` (the same machine) — this is a safe default and generally should stay that way unless you have a specific, well-secured reason to change it.
- If your app and database run on the **same VPS**, your app can connect via `localhost`, and the database never needs to be reachable from the public internet at all — this is the simplest and most secure setup.
- If your database runs on a **separate server** (common with managed databases, or when scaling out), the connection between your app server and database server should be restricted using a firewall to allow only your app server's specific IP address, not the whole internet.

> ⚠️ **Warning:** Leaving a database's port (e.g., `5432` for PostgreSQL, `3306` for MySQL) open to all incoming internet traffic is one of the most common and damaging misconfigurations in real-world deployments — it has led to major, publicized data breaches. Always restrict database access with a firewall (covered in more depth in deploy-13).

## Connecting Your App to the Database Securely

Your app connects to the database using a **connection string** — a URL-like string containing the credentials and location.

```text
postgresql://myapp_user:a-strong-password-here@localhost:5432/myapp_production
```

- Breaking this down: `protocol://username:password@host:port/database_name`.
- This connection string should **never** be hardcoded directly into your source code — it should be loaded from an **environment variable** instead, so it isn't committed to Git or exposed in your codebase (covered in depth in deploy-10).

```bash
# Example: stored as an environment variable, not in code
DATABASE_URL=postgresql://myapp_user:a-strong-password-here@localhost:5432/myapp_production
```

## Connection Pooling Explained

- Every time your app talks to the database, it needs an open **connection** — and creating a brand-new connection for every single request is slow and resource-intensive.
- A **connection pool** is a set of database connections that are opened once, in advance, and reused across many requests, instead of opening and closing a new one every time.
- Most modern database libraries and ORMs (e.g., Prisma, Sequelize, TypeORM for Node.js) include connection pooling automatically, but it's worth understanding what's happening underneath, since misconfigured pool sizes are a common real-world performance issue.

```text
Without pooling:  Request → open connection → query → close connection  (slow, repeated overhead)
With pooling:     Request → borrow connection from pool → query → return connection to pool  (fast, reused)
```

> 💡 **Tip:** If your app suddenly starts throwing "too many connections" database errors under load, it's very often a connection pooling misconfiguration — either the pool size is too large for the database's limits, or connections aren't being properly released back to the pool after use.

## Database Migrations

- A **migration** is a version-controlled, incremental change to your database's structure (schema) — e.g., "add a `phone_number` column to the `users` table" — that can be applied consistently across development, staging, and production.
- Migrations solve a real deployment problem: your database schema needs to change over time as your app evolves, but you can't just manually edit tables differently on every environment and expect them to stay in sync.
- Most frameworks/ORMs include migration tooling: Laravel has built-in migrations, Node.js commonly uses tools like Prisma Migrate, Knex, or TypeORM migrations.

```bash
# Example: running migrations as part of a deployment (tool-specific syntax varies)
npx prisma migrate deploy
```

> ⚠️ **Warning:** Running an *unreviewed* migration directly against production data is risky — always test migrations against a staging database first (recall the staging environment concept from deploy-01), especially for changes that could delete or alter existing data (e.g., dropping a column).

## Backups: The Most Important Habit

- A **backup** is a saved copy of your database's data at a point in time, so it can be restored if something goes catastrophically wrong — accidental deletion, a bad migration, hardware failure, or a security breach.
- **Managed databases** typically include automatic daily backups, plus **point-in-time recovery** (restoring to any specific moment, not just the last daily snapshot) as a premium feature.
- **Self-hosted databases** need backups configured manually — a common simple approach is a scheduled script that dumps the database and stores the result somewhere safe, off the same server.

```bash
# Example: manually dumping a PostgreSQL database to a file
pg_dump -U myapp_user myapp_production > backup-2026-07-10.sql

# Restoring from that backup file later
psql -U myapp_user myapp_production < backup-2026-07-10.sql
```

> ⚠️ **Warning:** A backup stored on the *same server* as the database it's backing up is not a real safety net — if that server fails entirely or is compromised, you lose the backup along with the original data. Real backups belong on separate storage (e.g., cloud object storage like S3, or a managed backup service).

> 💡 **Tip:** A widely used rule of thumb is the **3-2-1 backup rule**: keep at least **3** copies of your data, on **2** different types of storage, with at least **1** copy stored off-site. You don't need to implement this fully for a small personal project, but it's the professional standard worth knowing.

## Replication and Read Replicas (High-Level)

- **Replication** means continuously copying data from one database server (the **primary**) to one or more additional servers (**replicas**), keeping them in sync in near real-time.
- A common use is a **read replica**: a copy of the database used only for read queries (e.g., displaying data), while all writes go to the primary — this spreads out traffic and improves performance for read-heavy apps.
- Replication also provides a form of redundancy — if the primary server fails, a replica can potentially be promoted to take over, reducing downtime.
- This is a more advanced, scale-driven concern — not something a small personal project typically needs on day one, but important vocabulary for understanding how larger production systems are architected.

## SQL vs NoSQL: A Quick Recap for Deployment

| | SQL (e.g., PostgreSQL, MySQL) | NoSQL (e.g., MongoDB) |
|---|---|---|
| **Structure** | Fixed schema, tables with rows/columns | Flexible schema, documents/collections |
| **Managed hosting examples** | Amazon RDS, DigitalOcean Managed DB, Supabase | MongoDB Atlas |
| **Migrations** | Standard, well-established tooling | Often less strict, but still versioned in real apps |
| **Common use case** | Relational data, financial/transactional apps | Rapidly evolving schemas, document-style data |

- Deployment concepts in this note (connection strings, pooling, backups, firewalling) apply to **both** SQL and NoSQL databases — the specific tools differ, but the underlying production practices are the same.

## Popular Managed Database Providers

| Provider | Notes |
|---|---|
| **Amazon RDS** | Supports PostgreSQL, MySQL, and more; part of AWS, very widely used in industry |
| **DigitalOcean Managed Databases** | Simple pricing, beginner-friendly, pairs naturally with a DigitalOcean VPS |
| **Supabase** | Managed PostgreSQL with extra built-in features (auth, real-time, storage); popular for fast-moving projects |
| **PlanetScale** | Managed MySQL-compatible database, known for branching/schema workflows |
| **MongoDB Atlas** | The standard managed hosting for MongoDB (NoSQL) |

## Common Mistakes Beginners Make

- Exposing the database port directly to the public internet instead of restricting it to `localhost` or a specific trusted IP.
- Hardcoding database credentials directly in source code instead of using environment variables.
- Never setting up backups until after data has already been lost once.
- Running migrations against production without testing them on staging first.
- Using the database's default superuser account for the app's everyday connection, instead of a scoped-down dedicated user.

## Quick Revision

- Production databases need protection, backups, and reliability that local development databases don't require.
- Self-hosting a database teaches you the fundamentals; managed databases (RDS, DigitalOcean Managed DB, Supabase) handle backups/security/patching for you and are generally recommended for real production apps.
- Databases should almost never be exposed directly to the public internet — restrict access to `localhost` or specific trusted server IPs via a firewall.
- Connection strings hold your database credentials and should always be loaded from environment variables, never hardcoded.
- Connection pooling reuses database connections across requests instead of opening a new one every time, which is critical for performance.
- Migrations are version-controlled schema changes that should be tested on staging before running against production.
- Backups must live on separate storage from the database itself to actually protect against server failure — the 3-2-1 rule is the professional standard.
- Replication and read replicas are more advanced, scale-driven concepts for spreading read traffic and improving redundancy, not needed for small early-stage projects.