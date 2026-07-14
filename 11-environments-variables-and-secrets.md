# Environment Variables & Secrets Management

Environment variables are values your app reads from its surrounding environment rather than from hardcoded code, and secrets management is the practice of keeping sensitive values like passwords and API keys safe throughout that process.

## Table of Contents

- [Why This Topic Matters So Much](#why-this-topic-matters-so-much)
- [What Is an Environment Variable?](#what-is-an-environment-variable)
- [Why Not Just Hardcode Values?](#why-not-just-hardcode-values)
- [Setting Environment Variables Different Ways](#setting-environment-variables-different-ways)
- [.env Files Explained](#env-files-explained)
- [Reading Environment Variables in Code](#reading-environment-variables-in-code)
- [.gitignore: Keeping .env Out of Git](#gitignore-keeping-env-out-of-git)
- [What Counts as a Secret?](#what-counts-as-a-secret)
- [Environment Variables Across Dev, Staging, and Production](#environment-variables-across-dev-staging-and-production)
- [Secrets in systemd, PM2, and Docker (Recap and Tie-Together)](#secrets-in-systemd-pm2-and-docker-recap-and-tie-together)
- [Secrets in CI/CD (Recap and Tie-Together)](#secrets-in-cicd-recap-and-tie-together)
- [Dedicated Secrets Managers (Beyond .env)](#dedicated-secrets-managers-beyond-env)
- [What to Do If a Secret Leaks](#what-to-do-if-a-secret-leaks)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## Why This Topic Matters So Much

- Throughout this series, we've mentioned "store this in an environment variable, don't hardcode it" repeatedly — database URLs in deploy-07, SSH keys and server details in deploy-08, secrets in Docker in deploy-09. This note finally explains that idea properly, in full depth.
- Secrets management is one of the most common sources of real, serious security incidents in the software industry — leaked API keys and database passwords have caused major, publicized breaches at real companies. Understanding this well, early, is genuinely one of the highest-value things you can learn in this whole series.

> ⚠️ **Warning:** This is not an exaggeration for beginner motivation — leaked credentials in public GitHub repositories are scanned for and exploited by automated bots within *minutes* of being pushed, in the real world, constantly. This topic deserves your full attention.

## What Is an Environment Variable?

- An **environment variable** is a named value that exists outside your application's code, in the surrounding operating system environment, that your program can read at runtime.
- They're a way of **configuring** your app's behavior without changing its source code — the same code can behave differently depending on what environment variables are present when it runs.

```bash
# Setting an environment variable directly in a terminal session (temporary, this session only)
export NODE_ENV=production
export PORT=4000

# Viewing it
echo $NODE_ENV
```

- `export` — makes the variable available to the current shell session and any programs started from it.
- This is the exact same mechanism you already saw being used inside a systemd service file (`Environment=NODE_ENV=production` in deploy-06) and in `docker run -e` (deploy-09) — both are just different ways of setting environment variables for your app to read.

## Why Not Just Hardcode Values?

Imagine your database connection string is written directly into your code:

```javascript
// ❌ Hardcoded — bad practice
const dbUrl = "postgresql://myapp_user:realPassword123@localhost:5432/myapp_production";
```

This causes several real problems:

- **Security risk**: If this code is pushed to GitHub (even a private repo can leak, get misconfigured, or be seen by more people than intended), your real database password is now exposed to anyone with access to that code.
- **Inflexibility**: Your local development database, staging database, and production database almost certainly have different connection details — hardcoding forces you to manually edit the code every time you switch environments, which is slow and error-prone.
- **No separation of code and config**: The same application code should be deployable to any environment; only the configuration around it should change.

```javascript
// ✅ Read from an environment variable instead
const dbUrl = process.env.DATABASE_URL;
```

- Now, the exact same, unchanged code can run in development, staging, or production — each environment simply provides a different value for `DATABASE_URL` from the outside.

> 💡 **Tip:** A well-known and widely respected set of production app principles, called **The Twelve-Factor App**, dedicates an entire principle specifically to this: "store config in the environment." It's considered a foundational best practice across the entire industry, not just a personal preference.

## Setting Environment Variables Different Ways

| Method | When It's Used |
|---|---|
| `export VAR=value` in the terminal | Temporary, current session only |
| `.env` file + a loader library | Local development, simple deployments |
| Systemd `Environment=` lines | Production apps run via systemd (deploy-06) |
| `docker run -e` / `--env-file` | Docker containers (deploy-09) |
| GitHub Secrets | CI/CD pipelines (deploy-08) |
| Cloud provider's secrets manager | Larger production systems (covered later in this note) |

## .env Files Explained

- A **`.env` file** is a plain text file, usually placed in your project's root folder, listing environment variables in a simple `KEY=VALUE` format — a common and convenient way to manage variables for local development.

```bash
# .env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://myapp_user:localpassword@localhost:5432/myapp_dev
JWT_SECRET=some-long-random-string-here
```

- Node.js doesn't read `.env` files automatically on its own — you typically use a small library like **dotenv** to load them:

```bash
npm install dotenv
```

```javascript
// At the very top of your app's entry file
require('dotenv').config();

// Now process.env contains everything from .env
console.log(process.env.DATABASE_URL);
```

- Many frameworks (Next.js, Laravel, etc.) have `.env` file support built in already, without needing a separate library.

> 💡 **Tip:** It's common practice to keep a `.env.example` file (with the same variable *names*, but placeholder/fake values) committed to Git, so other developers (or future you) know exactly which variables need to be set up, without ever exposing the real secret values.

```bash
# .env.example — safe to commit
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
JWT_SECRET=your-secret-here
```

## Reading Environment Variables in Code

```javascript
// Node.js / JavaScript
const port = process.env.PORT || 3000;
```

```php
// PHP
$dbUrl = getenv('DATABASE_URL');
// or, in Laravel specifically:
$dbUrl = env('DATABASE_URL');
```

```python
# Python
import os
db_url = os.environ.get('DATABASE_URL')
```

- Notice the JavaScript example includes `|| 3000` — a common, safe pattern providing a sensible **default value** if the environment variable isn't set at all, preventing the app from crashing outright over a missing non-critical variable.

> ⚠️ **Warning:** Don't apply this "default value" pattern to genuinely sensitive secrets like database passwords or API keys — an app should fail loudly and immediately if a required secret is missing, rather than silently falling back to some default that could be insecure or simply wrong.

## .gitignore: Keeping .env Out of Git

- A **`.gitignore`** file tells Git which files/folders to never track or commit — and your real `.env` file (containing actual secrets) should always be listed here.

```text
# .gitignore
.env
.env.local
.env.production
node_modules
```

- This means your `.env` file exists on your laptop and on each server, individually, containing that environment's specific real values — but it never travels through Git at all.

> ⚠️ **Warning:** If you accidentally commit a `.env` file with real secrets *before* adding it to `.gitignore`, simply adding it to `.gitignore` afterward does **not** remove it from your Git history — the secret remains recoverable in old commits. You would need to both rotate (change) the leaked secret immediately, and separately clean the Git history if truly necessary. Prevention is far easier than cleanup here.

## What Counts as a Secret?

Not every environment variable is sensitive — but many are, and it's important to correctly tell them apart.

| Not Secret (usually safe to see) | Secret (must be protected) |
|---|---|
| `PORT=3000` | `DATABASE_URL` (contains a password) |
| `NODE_ENV=production` | `JWT_SECRET` (used to sign auth tokens) |
| `APP_NAME=MyApp` | `STRIPE_SECRET_KEY` (payment provider key) |
| Public API base URLs | `AWS_SECRET_ACCESS_KEY` |
| Feature flags | `SSH_PRIVATE_KEY` |
| | Any third-party API key with write/paid access |

> 💡 **Tip:** A useful rule of thumb: if a value could let someone impersonate your app, access your data, spend your money, or log in as you, it's a secret. If it's just configuration that doesn't grant access to anything, it's usually safe.

## Environment Variables Across Dev, Staging, and Production

Recall the three environments from deploy-01 — each should have its **own separate** set of environment variable values, never shared or reused:

```text
Development:  DATABASE_URL=postgresql://.../myapp_dev
Staging:      DATABASE_URL=postgresql://.../myapp_staging
Production:   DATABASE_URL=postgresql://.../myapp_production
```

> ⚠️ **Warning:** Never let your local development environment accidentally point at your real production database "just to test something quickly." A local bug or a destructive test command could permanently damage real user data. Keep the environments genuinely separate, always.

## Secrets in systemd, PM2, and Docker (Recap and Tie-Together)

This note ties together how every deployment method from earlier in the series actually handles secrets in practice:

```ini
# systemd (deploy-06) — variables set directly in the service file
[Service]
Environment=DATABASE_URL=postgresql://...
Environment=JWT_SECRET=...
```

```bash
# PM2 (deploy-06) — using an ecosystem file
pm2 start server.js --name myapp --env production
```

```bash
# Docker (deploy-09) — using an env file, not baked into the image
docker run --env-file .env.production myapp
```

> ⚠️ **Warning:** Putting real secrets directly inside a systemd `.service` file is common in practice, but that file itself should have restricted permissions (readable only by root) since it's not protected by `.gitignore` the way `.env` is — it lives directly on the server's filesystem.

## Secrets in CI/CD (Recap and Tie-Together)

- As covered in deploy-08, GitHub Actions workflows reference secrets via `${{ secrets.NAME }}`, pulling from encrypted values stored in your repository's settings — never written directly in the workflow YAML file.
- During a deployment pipeline run, these secrets typically get passed onto the server as environment variables, completing the chain from "safely stored in GitHub" to "safely available to your running app in production."

## Dedicated Secrets Managers (Beyond .env)

- For small personal projects and early-stage apps, `.env` files and GitHub Secrets are genuinely sufficient and considered normal, professional practice.
- As applications and teams grow, dedicated **secrets managers** become common — specialized services designed specifically for storing, rotating, and controlling access to sensitive values, with extra features like access logging and automatic rotation.

| Tool | Notes |
|---|---|
| **AWS Secrets Manager** | Managed secrets storage integrated with AWS services |
| **HashiCorp Vault** | Powerful, widely used, self-hostable or managed |
| **Doppler** | Developer-friendly secrets management, syncs across environments/tools |
| **1Password / Bitwarden (for teams)** | Sometimes used for sharing secrets securely among a small team |

> 💡 **Tip:** You don't need any of these dedicated tools for a personal project or early-stage app — understanding *why* they exist (centralized control, rotation, access auditing) is more important right now than actually using one. `.env` files plus GitHub Secrets will serve you well for a long time.

## What to Do If a Secret Leaks

If you ever discover a real secret was accidentally committed to Git, exposed in a log, or shared somewhere it shouldn't have been:

```text
1. Rotate it immediately — generate a brand new value and update it everywhere it's used
   (e.g., change the database password, regenerate the API key)
2. Revoke/invalidate the old leaked value at the source (e.g., in the provider's dashboard)
3. Check logs/billing for any signs of unauthorized use during the exposure window
4. Only afterward, worry about cleaning Git history if truly necessary
```

> ⚠️ **Warning:** The single most important, time-sensitive step is **rotating the secret** — simply deleting the leaked commit or making the repository private again does *not* undo the exposure; the value may already have been seen or scraped by automated bots. Assume it's compromised the moment it's exposed, and act accordingly.

## Common Mistakes Beginners Make

- Committing a real `.env` file to Git because it wasn't added to `.gitignore` before the first commit.
- Reusing the exact same secrets (e.g., the same database password) across development, staging, and production.
- Treating "adding it to `.gitignore` now" as sufficient cleanup after a secret was already committed in a previous commit.
- Applying default fallback values to genuinely sensitive secrets, causing silent, confusing failures instead of clear errors.
- Sharing `.env` files insecurely (e.g., over email, chat, or public messaging apps) instead of using a secure method appropriate for the team's size.

## Quick Revision

- Environment variables let your app read configuration from its surrounding environment instead of hardcoding values into the source code.
- Hardcoding secrets creates real security risks and makes it painful to support different environments (dev/staging/production) with the same codebase.
- `.env` files are the standard local-development approach, loaded via a library like `dotenv`; a `.env.example` file with placeholder values is commonly committed instead.
- `.env` files (and any file containing real secrets) must always be listed in `.gitignore`, and adding it after-the-fact does not remove already-committed secrets from Git history.
- Not every environment variable is sensitive — a secret is any value that could let someone access data, impersonate your app, or spend money on your behalf.
- Each environment (development, staging, production) should have its own separate, non-shared set of secret values.
- Every deployment method covered so far (systemd, PM2, Docker, CI/CD) has its own way of safely injecting secrets at runtime, rather than baking them into code or images.
- If a secret ever leaks, the most urgent step is rotating (replacing) it immediately — assume it's compromised the instant it's exposed.