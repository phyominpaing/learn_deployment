# CI/CD Pipelines

CI/CD is the practice of automating the steps between writing code and having it safely running in production, so you don't have to manually repeat the same deployment steps every single time.

## Table of Contents

- [The Manual Deployment Problem](#the-manual-deployment-problem)
- [What Does CI/CD Actually Mean?](#what-does-cicd-actually-mean)
- [Continuous Integration (CI) Explained](#continuous-integration-ci-explained)
- [Continuous Delivery vs Continuous Deployment](#continuous-delivery-vs-continuous-deployment)
- [What a Pipeline Actually Looks Like](#what-a-pipeline-actually-looks-like)
- [Introducing GitHub Actions](#introducing-github-actions)
- [Anatomy of a GitHub Actions Workflow File](#anatomy-of-a-github-actions-workflow-file)
- [A Real CI Example: Running Tests Automatically](#a-real-ci-example-running-tests-automatically)
- [A Real CD Example: Auto-Deploying to Your Server](#a-real-cd-example-auto-deploying-to-your-server)
- [Secrets in CI/CD](#secrets-in-cicd)
- [Rollbacks in a CI/CD World](#rollbacks-in-a-cicd-world)
- [Other Popular CI/CD Tools](#other-popular-cicd-tools)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## The Manual Deployment Problem

- Think back to how we've deployed so far in this series: SSH into the server, `git pull` the latest code, install dependencies, run migrations, restart the process manager. That's a lot of manual steps, every single time you want to update your live app.
- Manual deployment has real problems: it's **slow**, it's **easy to forget a step** (like forgetting to run migrations), and it's **error-prone** (typos, wrong directory, wrong server).
- **CI/CD** exists to take all of these repetitive, error-prone manual steps and turn them into an automated, repeatable, reliable process that runs the same way every time.

> 💡 **Tip:** Even as a solo developer working on a personal project, CI/CD saves real time and prevents real mistakes. It's not just a "big company" practice — it's genuinely useful from day one.

## What Does CI/CD Actually Mean?

- **CI** stands for **Continuous Integration** — automatically testing and checking your code every time you push changes, to catch problems early.
- **CD** stands for **Continuous Delivery** or **Continuous Deployment** (two related but slightly different things, explained below) — automatically getting that code ready for, or actually onto, production.
- Together, "CI/CD" describes an automated **pipeline**: a defined sequence of steps that runs automatically whenever you push code, taking it from "just written" to "tested" to "deployed," without you manually doing each step by hand.

## Continuous Integration (CI) Explained

- **Continuous Integration** means: every time a developer pushes new code (e.g., to GitHub), an automated process immediately runs to check that the code is valid — running tests, checking code style, making sure the project still builds successfully.
- The word "integration" refers to the original problem CI was designed to solve: when multiple developers work on the same project and only combine ("integrate") their code occasionally, merging becomes painful and bugs pile up. Testing on every single push catches problems immediately, while they're small and easy to fix.
- Even for a solo project, CI is valuable because it catches your own mistakes automatically — e.g., "this change broke the build" — before you ever get to the deployment step.

```text
You push code
     │
     ▼
CI automatically runs:
   - Install dependencies
   - Run tests
   - Check code style/linting
   - Try building the project
     │
     ▼
✅ Pass → safe to continue
❌ Fail → you're notified immediately, before anything reaches production
```

## Continuous Delivery vs Continuous Deployment

These two terms are easy to confuse because they sound almost identical, but they mean different things:

| Term | What Happens After CI Passes |
|---|---|
| **Continuous Delivery** | The code is automatically prepared and ready to deploy, but a human still clicks a button to actually release it to production. |
| **Continuous Deployment** | The code is automatically deployed straight to production with no human step at all, as long as all checks pass. |

> 💡 **Tip:** As a beginner, **Continuous Delivery** (with a manual "deploy" button/approval) is usually the safer starting point — it gives you full automation up until the final release, while still keeping you in control of exactly when changes go live.

## What a Pipeline Actually Looks Like

A **pipeline** is just the defined sequence of automated steps, usually written in a configuration file that lives in your project's repository. A typical pipeline for a full-stack app might look like:

```text
1. Trigger: code is pushed to the "main" branch
2. Install dependencies
3. Run automated tests
4. Build the project (e.g., compile frontend assets)
5. Connect to the server via SSH
6. Pull the latest code on the server
7. Install dependencies on the server
8. Run database migrations
9. Restart the app (via PM2 or systemd, from deploy-06)
```

- Every one of these steps is something you've already learned to do manually in this series — CI/CD simply automates running them in order, consistently, every time.

## Introducing GitHub Actions

- **GitHub Actions** is GitHub's built-in CI/CD tool — since your code likely already lives on GitHub, this is often the simplest starting point, with no extra service to sign up for.
- You define a **workflow**: a YAML configuration file describing what should happen and when, stored inside your repository at `.github/workflows/`.
- Other popular alternatives exist (covered later in this note), but GitHub Actions is a great first tool to learn because of how tightly it's integrated with GitHub, which you're likely already using for version control.

## Anatomy of a GitHub Actions Workflow File

```yaml
# .github/workflows/deploy.yml
name: Deploy App

on:
  push:
    branches:
      - main

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test
```

- `name:` — a human-readable label for this workflow, shown in GitHub's interface.
- `on:` — the **trigger**, defining when this workflow should run. Here, it runs whenever code is pushed to the `main` branch.
- `jobs:` — a workflow can contain one or more jobs, each running independently (potentially in parallel).
- `runs-on: ubuntu-latest` — tells GitHub Actions which type of temporary virtual machine to run this job on; `ubuntu-latest` gives you a fresh, temporary Ubuntu Linux server for each run (this connects directly to what you learned about Linux in deploy-03).
- `steps:` — the ordered list of individual actions to perform within the job.
- `uses: actions/checkout@v4` — a pre-built, reusable "action" (hence the tool's name) that fetches your repository's code onto that temporary machine, since it starts out completely empty.
- `run: npm install` — runs a literal shell command, exactly like you'd type it yourself in a terminal.

> 💡 **Tip:** Every GitHub Actions run happens on a brand new, temporary, disposable virtual machine — it's created fresh for that run and destroyed afterward. This guarantees your build/test process isn't accidentally relying on some leftover state from a previous run.

## A Real CI Example: Running Tests Automatically

Expanding on the workflow above — this is a genuinely useful CI setup for a typical backend project:

```yaml
name: CI

on:
  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install
      - run: npm test
```

- Notice the trigger here is `pull_request`, not `push` — this means the tests run automatically whenever someone opens a **pull request** proposing to merge changes into `main`, letting you catch problems *before* the code is even merged, not after.
- This is one of the most common real-world CI patterns: run checks on every pull request, and only allow merging once they pass.

## A Real CD Example: Auto-Deploying to Your Server

Building on the CI job, here's a simplified deployment job that connects to your VPS over SSH (using concepts directly from deploy-03) and updates the running app:

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm install
            npx prisma migrate deploy
            pm2 reload myapp
```

- `appleboy/ssh-action@v1` — a popular, pre-built community action that handles connecting via SSH for you, so you don't have to write raw SSH connection logic yourself.
- `script:` — the actual commands run *on your server* once connected: pull the latest code, install dependencies, run migrations (from deploy-07), and reload the app with zero downtime using PM2 (from deploy-06).
- Notice this is quite literally the same manual steps you already know how to do by hand — CI/CD is just running them automatically and consistently.

## Secrets in CI/CD

- The workflow above references `${{ secrets.SERVER_IP }}`, `${{ secrets.SERVER_USER }}`, and `${{ secrets.SSH_PRIVATE_KEY }}` — these are **GitHub Secrets**, encrypted values stored securely in your repository's settings, not written directly in the workflow file.
- This follows the exact same principle from deploy-07 about never hardcoding database credentials directly into code — the same logic applies to server IPs, usernames, and private keys used in deployment.
- You add secrets through your GitHub repository's **Settings → Secrets and variables → Actions** page, and reference them in workflows using the `secrets.NAME` syntax.

> ⚠️ **Warning:** Never paste a real SSH private key, password, or API key directly into a workflow YAML file. Even if you later delete it, it often remains visible in your Git history forever. Always use GitHub Secrets (or your CI tool's equivalent) instead.

## Rollbacks in a CI/CD World

- Recall **rollback** from deploy-01: reverting to a previous, known-working version after a bad deployment.
- With CI/CD, since every deployment is triggered by a Git commit, rolling back is often as simple as reverting the problematic commit (`git revert`) and pushing again — the same automated pipeline redeploys the previous, working version automatically.
- Some teams also keep the previous release's build artifact ready to instantly swap back to, for even faster rollback without waiting for a full pipeline re-run — an advanced pattern you won't need immediately, but worth knowing exists.

## Other Popular CI/CD Tools

| Tool | Notes |
|---|---|
| **GitHub Actions** | Built directly into GitHub, easiest starting point if your code is already there |
| **GitLab CI/CD** | Built into GitLab, similar concept, YAML-based |
| **CircleCI** | Standalone CI/CD service, works with any Git provider |
| **Jenkins** | Older, highly customizable, self-hosted; common in larger/legacy enterprise setups |
| **Vercel/Netlify built-in CI** | Automatic build/deploy pipelines built into those specific hosting platforms |

> 💡 **Tip:** You don't need to learn all of these. GitHub Actions alone covers the vast majority of real-world personal and small-team projects, and the core *concepts* (triggers, jobs, steps, secrets) transfer directly to any other CI/CD tool you encounter later in your career.

## Common Mistakes Beginners Make

- Hardcoding secrets (SSH keys, passwords) directly into workflow files instead of using GitHub Secrets.
- Setting up Continuous Deployment (fully automatic, no approval) before trusting their test coverage enough to catch real problems.
- Forgetting that each CI job runs on a fresh, empty virtual machine — assuming files from a previous manual setup will "just be there."
- Not testing the deployment workflow itself on a low-stakes branch before wiring it up to auto-deploy from `main`.
- Confusing Continuous Delivery (ready to deploy, manual trigger) with Continuous Deployment (fully automatic) when describing their own setup.

## Quick Revision

- CI/CD automates the repetitive, error-prone manual steps of testing and deploying code, running them the same reliable way every time.
- Continuous Integration (CI) automatically tests and validates code on every push or pull request, catching problems early.
- Continuous Delivery keeps code ready to deploy with a manual final step; Continuous Deployment deploys fully automatically with no human step — Delivery is usually the safer starting point.
- A pipeline is just the ordered list of automated steps (install, test, build, deploy) — the same steps you've already learned to do manually throughout this series.
- GitHub Actions is a great first CI/CD tool since it's built directly into GitHub, using YAML workflow files stored in `.github/workflows/`.
- Deployment workflows commonly use a community SSH action to connect to your VPS and run the same `git pull`, install, migrate, and restart steps you already know from earlier notes.
- Secrets (SSH keys, server IPs, passwords) must be stored securely as GitHub Secrets, never hardcoded directly into workflow files.
- Rollbacks in a CI/CD setup are often as simple as reverting a bad commit and letting the pipeline redeploy the previous working version automatically.