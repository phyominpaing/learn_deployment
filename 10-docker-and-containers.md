# Docker & Containers

Docker lets you package your app together with everything it needs to run — code, dependencies, and system settings — into a single portable unit called a container, so it runs identically no matter where you deploy it.

## Table of Contents

- [The Problem Docker Solves](#the-problem-docker-solves)
- [What Is a Container?](#what-is-a-container)
- [Containers vs Virtual Machines](#containers-vs-virtual-machines)
- [Key Docker Concepts](#key-docker-concepts)
- [Installing Docker](#installing-docker)
- [Your First Dockerfile](#your-first-dockerfile)
- [Building and Running a Docker Image](#building-and-running-a-docker-image)
- [Understanding Docker Ports](#understanding-docker-ports)
- [Environment Variables in Docker](#environment-variables-in-docker)
- [Docker Volumes: Keeping Data Safe](#docker-volumes-keeping-data-safe)
- [Docker Compose: Running Multiple Containers Together](#docker-compose-running-multiple-containers-together)
- [A Full-Stack Example with Docker Compose](#a-full-stack-example-with-docker-compose)
- [Deploying Docker Containers to a VPS](#deploying-docker-containers-to-a-vps)
- [Docker and Nginx Together](#docker-and-nginx-together)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## The Problem Docker Solves

- Remember the "works on my machine" problem from deploy-01? It usually happens because your laptop and the server have subtle differences — a different Node.js version, a missing system library, different OS settings.
- Traditionally, you'd try to make the server match your laptop exactly by manually installing the right versions of everything (which is exactly what we did in deploy-03 through deploy-07) — but this is fragile and easy to get slightly wrong.
- **Docker** solves this differently: instead of trying to make the server match your laptop, you package your app together with its *entire environment* — the exact Node.js version, exact dependencies, exact OS-level settings — into one self-contained unit. That unit then runs identically anywhere Docker is installed, whether that's your laptop, a teammate's laptop, or a production server.

> 💡 **Tip:** Everything you learned in deploy-03 through deploy-07 (Linux, Nginx, process managers, databases) still applies with Docker — Docker doesn't replace those concepts, it changes *how* you package and ship your app on top of them.

## What Is a Container?

- A **container** is a lightweight, isolated environment that packages your application's code together with everything it needs to run: the runtime (e.g., Node.js), libraries, dependencies, and configuration.
- Despite feeling like "a mini virtual computer," a container does **not** include a full separate operating system — it shares the host machine's OS kernel, which is exactly what makes containers so much lighter and faster to start than a full virtual machine.
- **Image**: A read-only template/blueprint for a container — think of it as a snapshot of everything the container needs, before it's actually running.
- **Container**: A running (or stopped) instance created *from* an image — the same relationship as a class and an object in programming, if that analogy helps.

```text
Dockerfile (instructions)
     │  docker build
     ▼
Image (a packaged blueprint)
     │  docker run
     ▼
Container (a running instance of that image)
```

## Containers vs Virtual Machines

This is one of the most important distinctions to understand clearly, since the two are often confused.

| | Virtual Machine | Container |
|---|---|---|
| **Includes a full OS?** | Yes, a complete separate operating system | No, shares the host's OS kernel |
| **Startup time** | Slow (minutes) | Fast (seconds, often less) |
| **Size** | Large (gigabytes) | Small (often tens to hundreds of megabytes) |
| **Isolation level** | Very strong (fully separate OS) | Strong, but slightly less than a full VM |
| **Resource usage** | Heavier | Much lighter |
| **Managed by** | A hypervisor (recall deploy-02) | The Docker Engine |

```text
Virtual Machine                     Container
┌─────────────────┐                ┌─────────────────┐
│   App + Deps     │                │   App + Deps     │
│  Guest OS (full)  │                ├─────────────────┤
├─────────────────┤                │   App + Deps     │
│   Hypervisor      │                ├─────────────────┤
├─────────────────┤                │   Docker Engine   │
│   Host OS         │                ├─────────────────┤
└─────────────────┘                │   Host OS         │
                                     └─────────────────┘
```

> 💡 **Tip:** Recall from deploy-02 that a VPS itself is often already a *virtual machine* running on shared physical hardware. Docker containers typically run *inside* that VPS — so it's common to have a container running inside a virtual machine, which is running on physical hardware. Layers within layers.

## Key Docker Concepts

- **Dockerfile**: A text file containing step-by-step instructions for building a Docker image — essentially a recipe describing how to assemble your app's environment.
- **Image**: The built, reusable, read-only package created from a Dockerfile.
- **Container**: A running instance of an image.
- **Docker Hub**: A public registry (like GitHub, but for Docker images) where you can find and share pre-built images — e.g., official images for Node.js, PostgreSQL, Nginx.
- **Registry**: Any storage/distribution service for Docker images; Docker Hub is the most common, but private registries also exist (e.g., on AWS, GitHub Container Registry).

## Installing Docker

```bash
# Update package lists
sudo apt update

# Install Docker using the official convenience script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verify it installed correctly
docker --version

# Allow your user to run docker without needing sudo every time
sudo usermod -aG docker $USER
```

> ⚠️ **Warning:** After running `usermod -aG docker $USER`, you need to log out and back in (or start a new SSH session) for the group change to take effect — it won't apply to your current session automatically.

## Your First Dockerfile

Here's a Dockerfile for a simple Node.js app:

```dockerfile
# Start from an official Node.js image as the base
FROM node:20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy package files first (for efficient caching, explained below)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy the rest of the app's code
COPY . .

# Document which port the app listens on
EXPOSE 3000

# The command that runs when the container starts
CMD ["node", "server.js"]
```

- `FROM node:20-alpine` — every Dockerfile starts from a **base image**. Here, we're starting from an official Node.js 20 image built on **Alpine Linux**, a minimal, lightweight Linux distribution popular for keeping container sizes small.
- `WORKDIR /app` — sets the working directory inside the container's filesystem; all following commands run relative to this folder.
- `COPY package*.json ./` — copies just the dependency manifest files first, **before** the rest of the code. This is a deliberate optimization: Docker caches each step, so if your dependencies haven't changed, Docker can skip re-running `npm install` on the next build, even if your app's code changed. This ordering trick is one of the most common real-world Dockerfile optimizations.
- `RUN npm install --production` — installs dependencies inside the container (not using your laptop's `node_modules` at all — this is a completely fresh install, inside the container's own isolated environment).
- `COPY . .` — copies the rest of your project's files into the container.
- `EXPOSE 3000` — documents that this container listens on port 3000 (this is informational; it doesn't actually publish the port — that happens with `docker run -p`, shown below).
- `CMD ["node", "server.js"]` — the default command executed when a container starts from this image.

> ⚠️ **Warning:** Never `COPY` your `node_modules` folder, `.env` file with real secrets, or `.git` folder into a Docker image. Use a `.dockerignore` file (works exactly like `.gitignore`) to exclude them — otherwise you risk bloated, slow-to-build images or leaked secrets baked directly into the image.

```text
# .dockerignore
node_modules
.env
.git
*.log
```

## Building and Running a Docker Image

```bash
# Build an image from the Dockerfile in the current directory, tagging it "myapp"
docker build -t myapp .

# Run a container from that image
docker run -d -p 3000:3000 --name myapp-container myapp
```

- `docker build -t myapp .` — `-t myapp` tags (names) the resulting image `myapp`; the `.` tells Docker to look for the Dockerfile in the current directory.
- `docker run` — creates and starts a new container from an image.
- `-d` — runs the container in **detached** mode, meaning it runs in the background instead of tying up your terminal (directly connects to the foreground/background concept from deploy-06).
- `-p 3000:3000` — **publishes** a port, mapping `host_port:container_port`. This means: traffic arriving at port 3000 on the actual server gets forwarded into port 3000 inside the container.
- `--name myapp-container` — gives the running container a friendly name to reference later, instead of a random auto-generated ID.

```bash
docker ps                       # list running containers
docker ps -a                    # list all containers, including stopped ones
docker logs myapp-container      # view a container's logs
docker logs -f myapp-container   # follow logs live
docker stop myapp-container      # stop a running container
docker start myapp-container     # start it again
docker rm myapp-container        # remove a stopped container
docker rmi myapp                 # remove an image
```

## Understanding Docker Ports

- Inside the container, your app is listening on some port (e.g., `3000`) — but by default, that port is completely isolated and unreachable from outside the container.
- The `-p host_port:container_port` flag creates a mapping, opening up that connection deliberately.
- This connects directly back to the reverse proxy concept from deploy-04: your Nginx reverse proxy (running on the actual server, outside Docker) forwards public traffic to `localhost:3000` — and Docker's port mapping is what makes `localhost:3000` actually reach the app running inside the container.

## Environment Variables in Docker

```bash
# Passing environment variables directly on the command line
docker run -d -p 3000:3000 -e NODE_ENV=production -e DATABASE_URL=postgresql://... myapp
```

- `-e NAME=value` — sets an environment variable inside the container, exactly like the `Environment=` lines you saw in the systemd service file in deploy-06.
- For multiple variables, it's more common and manageable to use an **env file** instead:

```bash
docker run -d -p 3000:3000 --env-file .env.production myapp
```

> ⚠️ **Warning:** Never `COPY` a `.env` file containing real secrets into your Docker image using the Dockerfile — that would bake the secrets permanently into the image itself, which could then be shared or leaked. Pass secrets at *runtime* using `-e` or `--env-file` instead (this is expanded further in deploy-10).

## Docker Volumes: Keeping Data Safe

- By default, any data written *inside* a container disappears permanently when that container is removed — containers are meant to be disposable and easily replaceable.
- This is a serious problem for anything that needs to persist, like a database's actual data files.
- A **volume** is a mechanism for storing data *outside* the container's own disposable filesystem, on the host machine (or a managed storage system), so the data survives even if the container is deleted and recreated.

```bash
# Create a named volume
docker volume create myapp-db-data

# Run a PostgreSQL container using that volume to persist its data
docker run -d \
  -e POSTGRES_PASSWORD=mypassword \
  -v myapp-db-data:/var/lib/postgresql/data \
  --name myapp-db \
  postgres:16
```

- `-v myapp-db-data:/var/lib/postgresql/data` — mounts the named volume `myapp-db-data` to the path inside the container where PostgreSQL actually stores its data files. Even if this container is deleted, the volume (and the real data inside it) survives independently.

> ⚠️ **Warning:** Running a database inside a container *without* a volume is a common and serious beginner mistake — deleting or recreating that container (which happens routinely during updates) would permanently wipe all the data inside it.

## Docker Compose: Running Multiple Containers Together

- A real full-stack app usually needs multiple containers working together: your backend app, your database, maybe an admin panel, maybe a cache. Running each with individual, long `docker run` commands quickly becomes unmanageable.
- **Docker Compose** is a tool for defining and running multi-container applications using a single YAML configuration file, describing all your containers ("services") and how they relate to each other, at once.

## A Full-Stack Example with Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgresql://myapp_user:mypassword@db:5432/myapp_production
    depends_on:
      - db

  admin:
    build: ./admin-panel
    ports:
      - "5000:5000"
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=myapp_user
      - POSTGRES_PASSWORD=mypassword
      - POSTGRES_DB=myapp_production
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

- `services:` — each entry defines one container that's part of your overall app.
- `build: ./backend` — tells Compose to build an image from the Dockerfile located in the `./backend` folder, rather than pulling a pre-built one.
- `image: postgres:16` — for the database, instead of building our own image, we simply pull the official pre-built PostgreSQL image directly from Docker Hub.
- `depends_on: - db` — tells Compose to start the `db` service before starting `backend`/`admin`, since they need the database to be available.
- Notice the database URL uses `db` as the hostname (`@db:5432`) instead of `localhost` — Compose automatically creates an internal network where each service can reach the others by their service name, acting like a mini private DNS system between containers.
- `volumes:` at the bottom declares the named volume used for persisting the database's data, exactly as covered above.

```bash
# Start everything defined in docker-compose.yml
docker compose up -d

# View logs from all services
docker compose logs -f

# Stop everything
docker compose down

# Rebuild images after changing a Dockerfile, then restart
docker compose up -d --build
```

> 💡 **Tip:** `docker compose up -d` genuinely replaces a whole sequence of individual `docker run` commands, network setup, and volume creation with a single command — this is why Compose is the standard way real teams manage multi-container apps during development and often in production too.

## Deploying Docker Containers to a VPS

- The overall shape doesn't change much from what you already know: SSH into your VPS (deploy-03), install Docker, pull/build your images, and run them — often via `docker compose up -d`.
- A typical workflow: your CI/CD pipeline (deploy-08) builds the Docker image, pushes it to a registry (like Docker Hub or GitHub Container Registry), then connects to your server via SSH and pulls + runs the new image.

```yaml
# Example CD step, building on deploy-08's GitHub Actions workflow
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            docker compose pull
            docker compose up -d
```

## Docker and Nginx Together

- Nginx (from deploy-04) still plays exactly the same role even when your app runs in Docker — it sits on the host server, listening on ports 80/443, and reverse-proxies traffic to `localhost:PORT`, where `PORT` is whatever port you published from your container using `-p`.
- You can also run **Nginx itself as a container** as part of your Docker Compose setup, though for beginners it's often simpler to keep Nginx installed directly on the host VPS (as covered in deploy-04) and have it proxy to your Dockerized app's published ports.

## Common Mistakes Beginners Make

- Running a database in a container without a volume, then losing all data the first time the container is recreated.
- Copying `node_modules`, `.git`, or `.env` files into the image because a `.dockerignore` file wasn't set up.
- Confusing `EXPOSE` in a Dockerfile (just documentation) with `-p` in `docker run` (which actually publishes the port).
- Baking secrets directly into a Dockerfile with `COPY .env .` instead of passing them at runtime.
- Forgetting `depends_on` doesn't actually wait for a service to be *fully ready* (e.g., PostgreSQL accepting connections) — just that its container has *started* — which can cause race conditions on first startup that require additional handling in real production setups.

## Quick Revision

- Docker packages your app together with its entire runtime environment into a portable image, solving the "works on my machine" problem for good.
- A container is a lightweight, isolated running instance of an image; unlike a virtual machine, it shares the host's OS kernel, making it much faster and smaller.
- A Dockerfile is the recipe for building an image; ordering `COPY package*.json` before `COPY . .` lets Docker cache the dependency install step efficiently.
- `docker run -p host:container` publishes a container's internal port so it can be reached from outside — this is what your Nginx reverse proxy connects to.
- Volumes store data outside a container's disposable filesystem, which is essential for anything that needs to persist, especially databases.
- Docker Compose defines and runs multiple related containers (backend, admin panel, database) together from a single YAML file, with automatic internal networking between them.
- Deploying Docker to a VPS follows the same SSH-based workflow from earlier notes, typically automated through a CI/CD pipeline that builds, pushes, and pulls images.
- Nginx's role doesn't change with Docker — it still reverse-proxies from ports 80/443 to whatever port your container publishes.