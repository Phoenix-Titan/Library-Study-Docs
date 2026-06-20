# Docker & Docker Compose Complete Guide

A full, study-friendly reference for **Docker** and **Docker Compose** — concepts, commands, Dockerfiles, networking, volumes, multi-container apps, optimization, security, and real-world patterns for shipping web apps (Next.js, React, MERN/PERN) that clients can run anywhere.

> **Why a dev should learn Docker:** it packages your app + its exact environment into a portable **image** that runs identically on your machine, a teammate's, a client's server, or the cloud — killing "but it works on my machine." For freelance/client work it means clean handoffs, easy deploys, and one-command setups for full-stack apps (frontend + backend + database).

---

## Table of Contents
1. [Core Concepts & Vocabulary](#1-core-concepts--vocabulary)
2. [Installation & First Run](#2-installation--first-run)
3. [The Docker Mental Model](#3-the-docker-mental-model)
4. [Images vs Containers](#4-images-vs-containers)
5. [Essential Docker CLI Commands](#5-essential-docker-cli-commands)
6. [Dockerfile — Building Images](#6-dockerfile--building-images)
7. [Dockerfile Instructions Reference](#7-dockerfile-instructions-reference)
8. [Multi-Stage Builds (production gold)](#8-multi-stage-builds)
9. [.dockerignore](#9-dockerignore)
10. [Volumes & Data Persistence](#10-volumes--data-persistence)
11. [Networking](#11-networking)
12. [Environment Variables & Secrets](#12-environment-variables--secrets)
13. [Docker Compose — Multi-Container Apps](#13-docker-compose)
14. [docker-compose.yml Reference](#14-docker-composeyml-reference)
15. [Compose CLI Commands](#15-compose-cli-commands)
16. [Real-World Stacks (copy-paste)](#16-real-world-stacks)
17. [Image Optimization & Speed](#17-image-optimization--speed)
18. [Security Best Practices](#18-security-best-practices)
19. [Debugging & Troubleshooting](#19-debugging--troubleshooting)
20. [Registries & Deployment](#20-registries--deployment)
21. [Tips, Tricks & Gotchas](#21-tips-tricks--gotchas)
22. [Study Path](#22-study-path)

---

## 1. Core Concepts & Vocabulary

| Term | Meaning |
|---|---|
| **Image** | A read-only **blueprint/template** for a container (your app + OS libs + dependencies). Like a class. |
| **Container** | A running **instance** of an image — an isolated process with its own filesystem, network, and resources. Like an object. |
| **Dockerfile** | A text file of instructions that **builds** an image step by step. |
| **Layer** | Each Dockerfile instruction creates a cached filesystem layer; images are stacks of layers. |
| **Registry** | A store for images (Docker Hub, GitHub Container Registry, AWS ECR). |
| **Volume** | Persistent storage that lives **outside** a container (so data survives restarts). |
| **Network** | A virtual network letting containers talk to each other by name. |
| **Docker Engine** | The background daemon (`dockerd`) that builds/runs containers. |
| **Docker Compose** | A tool to define and run **multi-container** apps from one YAML file. |
| **Build context** | The folder sent to the daemon when building (where the Dockerfile + code live). |
| **Tag** | A label/version for an image (`myapp:1.0`, `node:20-alpine`). |

**The one-line summary:** *A Dockerfile builds an Image; an Image runs as a Container; Compose orchestrates many Containers.*

---

## 2. Installation & First Run

- **Windows / Mac:** install **Docker Desktop** (includes Engine, CLI, Compose, and a GUI).
- **Linux:** install Docker Engine + the Compose plugin.

```bash
docker --version           # check install
docker compose version     # check Compose (v2 is built into the CLI)
docker run hello-world      # pulls & runs a test image — confirms everything works
docker info                 # system-wide info
```

> **⚡ Note:** Modern Docker uses **`docker compose`** (space, a built-in plugin — Compose v2). The old standalone **`docker-compose`** (hyphen) is legacy. Use the space version.

---

## 3. The Docker Mental Model

Think of it as a 3-step lifecycle:

```
   Dockerfile  ──(docker build)──►  Image  ──(docker run)──►  Container
   (recipe)                         (frozen meal)             (meal being eaten)
```

- You **write** a Dockerfile describing how to assemble your app environment.
- You **build** it into an Image (immutable, shareable, versioned).
- You **run** the Image to create one or more Containers (live, disposable).
- Containers are **ephemeral** — treat them as cattle, not pets. State that must persist goes in **volumes** or a **database**, never in the container's own filesystem.

**Why containers ≠ virtual machines:** VMs virtualize whole hardware + a full OS (heavy, GBs, slow boot). Containers share the host OS kernel and isolate just the process (lightweight, MBs, boot in milliseconds). That's why you can run many containers on one machine.

---

## 4. Images vs Containers

| | Image | Container |
|---|---|---|
| Nature | Read-only template | Running (or stopped) instance |
| Analogy | Class / recipe / blueprint | Object / cooked dish |
| Count | One image | Many containers from it |
| Mutability | Immutable | Has a writable top layer (lost on removal) |
| Command | `docker build`, `docker pull` | `docker run`, `docker start/stop` |

One image → run it 5 times → 5 independent containers.

---

## 5. Essential Docker CLI Commands

### Images
```bash
docker pull node:20-alpine          # download an image from a registry
docker build -t myapp:1.0 .         # build image from Dockerfile in current dir
docker images                       # list local images
docker rmi myapp:1.0                # remove an image
docker tag myapp:1.0 user/myapp:1.0 # re-tag (for pushing)
docker history myapp:1.0            # see layers
docker image prune                  # remove dangling images
```

### Containers — run
```bash
docker run myapp                            # run (foreground)
docker run -d myapp                          # detached (background)
docker run -p 3000:3000 myapp                # map host:container port
docker run --name web myapp                  # name the container
docker run -e NODE_ENV=production myapp       # set env var
docker run -v $(pwd):/app myapp              # mount a volume (bind mount)
docker run -it ubuntu bash                   # interactive shell (-i interactive, -t tty)
docker run --rm myapp                         # auto-remove when it exits
docker run --restart unless-stopped myapp     # restart policy

# A typical dev run, all together:
docker run -d --name web -p 3000:3000 -e NODE_ENV=production --restart unless-stopped myapp:1.0
```

### Containers — manage
```bash
docker ps                  # list RUNNING containers
docker ps -a               # list ALL (incl. stopped)
docker stop web            # graceful stop
docker start web           # start a stopped container
docker restart web
docker rm web              # remove a stopped container
docker rm -f web           # force-remove a running one
docker logs web            # view logs
docker logs -f web         # follow logs live
docker exec -it web sh     # open a shell INSIDE a running container
docker stats               # live resource usage
docker inspect web         # full JSON details
docker cp web:/app/file .  # copy file out of a container
```

### Cleanup (free disk space)
```bash
docker system df            # see what's using space
docker container prune      # remove all stopped containers
docker image prune -a       # remove unused images
docker volume prune         # remove unused volumes
docker system prune -a       # nuke everything unused (careful!)
```

---

## 6. Dockerfile — Building Images

A Dockerfile is read top-to-bottom; each instruction adds a cached layer.

### Simple Node/Express example
```dockerfile
# 1. Base image
FROM node:20-alpine

# 2. Working directory inside the container
WORKDIR /app

# 3. Copy dependency manifests FIRST (better layer caching)
COPY package*.json ./

# 4. Install dependencies
RUN npm ci --omit=dev

# 5. Copy the rest of the source
COPY . .

# 6. Document the port the app listens on
EXPOSE 3000

# 7. Default command when the container starts
CMD ["node", "server.js"]
```

```bash
docker build -t my-api:1.0 .
docker run -d -p 3000:3000 my-api:1.0
```

**The #1 Dockerfile rule:** copy `package.json` and install deps *before* copying source code. Dependencies change rarely, source changes constantly — this ordering lets Docker reuse the cached `npm install` layer on every code change, making rebuilds seconds instead of minutes.

---

## 7. Dockerfile Instructions Reference

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image to start from (required first). | `FROM node:20-alpine` |
| `WORKDIR` | Set/create the working directory. | `WORKDIR /app` |
| `COPY` | Copy files from build context into image. | `COPY . .` |
| `ADD` | Like COPY but also handles URLs & auto-extracts tar. Prefer `COPY`. | `ADD app.tar.gz /app` |
| `RUN` | Execute a command at **build time** (creates a layer). | `RUN npm ci` |
| `CMD` | Default command at **run time** (only one; overridable). | `CMD ["node","server.js"]` |
| `ENTRYPOINT` | Fixed executable at run time; args append to it. | `ENTRYPOINT ["nginx"]` |
| `EXPOSE` | Document a port (doesn't publish it — `-p` does). | `EXPOSE 3000` |
| `ENV` | Set an environment variable (persists in image). | `ENV NODE_ENV=production` |
| `ARG` | Build-time variable (not in final image). | `ARG VERSION=1.0` |
| `VOLUME` | Declare a mount point for persistent data. | `VOLUME /data` |
| `USER` | Run as a non-root user (security). | `USER node` |
| `LABEL` | Metadata. | `LABEL maintainer="you@x.com"` |
| `HEALTHCHECK` | Command to test container health. | `HEALTHCHECK CMD curl -f http://localhost:3000 || exit 1` |

### CMD vs ENTRYPOINT (a common confusion)
- `CMD` → the **default** command, easily overridden: `docker run myapp <other command>`.
- `ENTRYPOINT` → the **fixed** executable; `CMD` (or run args) become its arguments.
- Common combo: `ENTRYPOINT ["node"]` + `CMD ["server.js"]` → runs `node server.js`, but `docker run myapp other.js` runs `node other.js`.

### RUN vs CMD
- `RUN` happens **while building** the image (install packages, build the app).
- `CMD`/`ENTRYPOINT` happen **when a container starts** (run the app).

---

## 8. Multi-Stage Builds

The single most important production technique. **Build in a big image, ship a tiny one.** You compile/build using all the dev tooling in one stage, then copy *only the final artifacts* into a clean, small runtime stage.

### Next.js production example (multi-stage)
```dockerfile
# ---- Stage 1: deps ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ---- Stage 2: builder ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build          # produces .next (use output:'standalone' in next.config)

# ---- Stage 3: runner (tiny, production) ----
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S nodejs && adduser -S nextjs -G nodejs   # non-root user
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```
> For Next.js, set `output: "standalone"` in `next.config.ts` so the build produces a minimal self-contained server — perfect for the runner stage. Final image goes from ~1GB to ~150MB.

### React/Vite static site (build → serve with nginx)
```dockerfile
# ---- Build ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # outputs to /app/dist

# ---- Serve with nginx ----
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# (optional) custom config for SPA routing:
# COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Why it matters:** smaller images = faster pulls/deploys, less attack surface, lower cloud cost. Multi-stage is how every serious production Dockerfile is written.

---

## 9. .dockerignore

Like `.gitignore` — keeps junk out of the build context (faster builds, smaller images, no secret leaks).

```gitignore
node_modules
npm-debug.log
.next
dist
build
.git
.gitignore
.env
.env.local
Dockerfile
docker-compose.yml
README.md
coverage
.vscode
*.md
```
> Always exclude `node_modules` (installed fresh inside the image) and `.env` (don't bake secrets into images).

---

## 10. Volumes & Data Persistence

Containers are ephemeral — when removed, their writable layer (and any data in it) is gone. To **persist** data (databases, uploads), use volumes.

### Three mount types
| Type | Syntax | Use |
|---|---|---|
| **Named volume** | `-v mydata:/var/lib/postgresql/data` | Docker-managed persistent storage (databases). **Preferred.** |
| **Bind mount** | `-v $(pwd):/app` | Map a host folder → container (live code in dev). |
| **tmpfs** | `--tmpfs /tmp` | In-memory, ephemeral (secrets/scratch). |

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata

# Run postgres with a persistent named volume
docker run -d --name db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

### Bind mount for live-reload dev
```bash
# Code changes on your host reflect instantly inside the container
docker run -d -p 3000:3000 \
  -v $(pwd):/app \
  -v /app/node_modules \      # anonymous volume to NOT overwrite container's node_modules
  my-dev-image
```
> **Key distinction:** **named volume = persist data** (DB). **bind mount = sync source code** (dev). The `-v /app/node_modules` trick prevents your host's (possibly empty/wrong-OS) node_modules from shadowing the container's.

---

## 11. Networking

Containers on the same Docker network reach each other **by container/service name** (Docker provides internal DNS).

### Network drivers
| Driver | Use |
|---|---|
| `bridge` | Default; isolated network on one host. Containers talk by name. |
| `host` | Container shares host's network directly (no isolation). |
| `none` | No networking. |
| `overlay` | Multi-host (Swarm/clusters). |

```bash
docker network create mynet
docker network ls
docker run -d --name db --network mynet postgres
docker run -d --name api --network mynet my-api   # api can reach db at hostname "db"
docker network inspect mynet
```

### Port publishing
```bash
docker run -p 8080:3000 myapp    # host:8080 → container:3000
docker run -p 127.0.0.1:8080:3000 myapp   # bind only to localhost
docker run -P myapp              # publish all EXPOSEd ports to random host ports
```
> **Inside a Compose network**, your API connects to the DB using the service name as hostname (e.g. `postgres://db:5432`). No IP addresses needed — this is the magic of Docker DNS.

---

## 12. Environment Variables & Secrets

```bash
docker run -e NODE_ENV=production -e PORT=3000 myapp   # individual
docker run --env-file .env myapp                        # from a file
```
```dockerfile
ENV NODE_ENV=production      # baked into the image (non-secret config only)
ARG BUILD_VERSION             # build-time only, not in final image
```

**Secret handling rules:**
- ❌ Never `COPY .env` into an image or hardcode passwords in a Dockerfile — they end up in image layers forever.
- ✅ Pass secrets at **runtime** via `--env-file`, Compose `env_file`, or your platform's secret store.
- ✅ Use Docker **build secrets** (`RUN --mount=type=secret`) for build-time credentials (e.g. private npm tokens) so they don't persist in layers.
- ✅ Add `.env` to both `.gitignore` and `.dockerignore`.

---

## 13. Docker Compose

Compose defines a **whole multi-container app** (frontend + backend + database + cache) in one `docker-compose.yml`, started with a single command. Essential for full-stack work.

### Why it's a freelancer superpower
A client (or new teammate) clones your repo and runs **`docker compose up`** — and the entire app (Next.js + Express + Postgres + Redis) boots, networked together, with zero manual setup. That's a killer handoff experience.

### Minimal example
```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```
```bash
docker compose up -d     # build + start everything in the background
```

---

## 14. docker-compose.yml Reference

```yaml
services:
  app:
    # ----- Build or use an image -----
    build:
      context: .              # build context folder
      dockerfile: Dockerfile  # custom dockerfile name
      target: runner          # which multi-stage target to build
      args:                   # build-time ARGs
        VERSION: "1.0"
    image: myapp:1.0          # name the built image (or pull this if no build)

    # ----- Runtime config -----
    container_name: myapp
    ports:
      - "3000:3000"           # host:container
    environment:
      - NODE_ENV=production
    env_file:
      - .env                  # load vars from file

    # ----- Dependencies & ordering -----
    depends_on:
      db:
        condition: service_healthy   # wait until db passes healthcheck

    # ----- Storage -----
    volumes:
      - ./src:/app/src        # bind mount (dev live-reload)
      - app_node_modules:/app/node_modules

    # ----- Networking -----
    networks:
      - backend

    # ----- Reliability -----
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

    # ----- Resource limits -----
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      retries: 5

# ----- Top-level declarations -----
volumes:
  pgdata:
  app_node_modules:

networks:
  backend:
    driver: bridge
```

### Key fields cheat-sheet
| Field | Purpose |
|---|---|
| `services` | Each container in your app. |
| `build` / `image` | Build from a Dockerfile, or pull a prebuilt image. |
| `ports` | Publish ports (`"host:container"`). |
| `environment` / `env_file` | Set env vars inline or from a file. |
| `depends_on` | Start order + (with `condition`) wait for health. |
| `volumes` | Persist data / mount code. |
| `networks` | Which networks the service joins. |
| `restart` | `no` / `always` / `unless-stopped` / `on-failure`. |
| `healthcheck` | How Docker tests if the service is healthy. |
| `command` | Override the image's default CMD. |
| `deploy.resources` | CPU/memory limits. |

> `depends_on` only controls **start order**, not readiness — a DB container can be "started" but not yet accepting connections. Use `condition: service_healthy` + a `healthcheck` to truly wait.

---

## 15. Compose CLI Commands

```bash
docker compose up                 # build (if needed) + start, attached (logs in terminal)
docker compose up -d              # detached (background)
docker compose up --build         # force rebuild images
docker compose down               # stop + remove containers/networks
docker compose down -v            # ALSO remove volumes (wipes DB data!)
docker compose ps                 # list services
docker compose logs               # all logs
docker compose logs -f web        # follow one service's logs
docker compose exec web sh        # shell into a running service
docker compose run web npm test   # one-off command in a new container
docker compose build              # build images only
docker compose stop / start       # stop/start without removing
docker compose restart web
docker compose pull               # pull latest images
docker compose config             # validate & view the resolved config
docker compose up -d --scale worker=3   # run 3 instances of a service
```

**Daily dev loop:** `docker compose up -d` → work → `docker compose logs -f` to watch → `docker compose down` when done. Use `--build` after changing a Dockerfile or dependencies.

---

## 16. Real-World Stacks

### Full-stack: Next.js + Postgres + Redis
```yaml
services:
  web:
    build:
      context: .
      target: runner
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/appdb
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### MERN stack (React + Express + MongoDB) for development
```yaml
services:
  client:
    build: ./client
    ports:
      - "5173:5173"
    volumes:
      - ./client:/app
      - /app/node_modules
    command: npm run dev

  server:
    build: ./server
    ports:
      - "5000:5000"
    environment:
      MONGO_URI: mongodb://mongo:27017/myapp
    volumes:
      - ./server:/app
      - /app/node_modules
    depends_on:
      - mongo
    command: npm run dev

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongodata:/data/db

volumes:
  mongodata:
```

### Dev Dockerfile with live reload (Node)
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "run", "dev"]   # nodemon/next dev — paired with bind mount for hot reload
```

---

## 17. Image Optimization & Speed

| Technique | Why |
|---|---|
| **Use small base images** (`alpine`, `slim`, `distroless`) | `node:20-alpine` ≈ 50MB vs `node:20` ≈ 1GB. |
| **Multi-stage builds** | Ship only runtime artifacts, drop build tools. |
| **Order layers by change frequency** | Copy `package.json` + install before copying source → cache reuse. |
| **`npm ci` not `npm install`** | Faster, reproducible from lockfile. |
| **Combine `RUN` commands** | Fewer layers (`RUN a && b && c`), clean caches in the same layer. |
| **`.dockerignore`** | Smaller, faster build context. |
| **`--omit=dev` / prune dev deps** | No dev dependencies in production image. |
| **BuildKit cache mounts** | `RUN --mount=type=cache,target=/root/.npm npm ci` reuses the npm cache across builds. |
| **Pin versions** | `node:20.11-alpine` not `node:latest` — reproducible builds. |

### Layer caching golden pattern
```dockerfile
COPY package*.json ./    # ① rarely changes
RUN npm ci               # ② cached unless package.json changed
COPY . .                 # ③ changes constantly — only THIS layer rebuilds on code edits
```

### Clean up in the same layer (or it doesn't shrink the image)
```dockerfile
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*    # cleanup in the SAME RUN
```

---

## 18. Security Best Practices

- ✅ **Run as non-root.** Add a user and `USER` it (containers default to root = risky).
  ```dockerfile
  RUN addgroup -S app && adduser -S app -G app
  USER app
  ```
- ✅ **Use minimal/pinned base images** (alpine/distroless, specific versions) — smaller attack surface.
- ✅ **Never bake secrets** into images (no `COPY .env`, no hardcoded passwords). Pass at runtime.
- ✅ **Scan images** for vulnerabilities: `docker scout cves myapp:1.0` (or Trivy/Snyk).
- ✅ **Keep base images updated** — rebuild regularly to pick up patches.
- ✅ **Drop capabilities / read-only filesystem** where possible (`--read-only`, `--cap-drop ALL`).
- ✅ **Limit resources** (`--memory`, `--cpus`) to prevent a runaway container.
- ✅ **Don't expose the Docker socket** (`/var/run/docker.sock`) to containers unless absolutely needed.
- ✅ **Use `HEALTHCHECK`** so orchestrators can restart unhealthy containers.
- ✅ **Use official images** from trusted publishers; verify tags.

---

## 19. Debugging & Troubleshooting

```bash
docker logs -f <container>          # what is the app printing?
docker exec -it <container> sh      # poke around inside
docker inspect <container>          # full config (env, mounts, network, IP)
docker stats                        # CPU/mem/network usage live
docker compose logs -f              # all services' logs together
docker events                       # real-time daemon events
docker top <container>              # processes in the container
```

| Symptom | Likely cause / fix |
|---|---|
| `port is already allocated` | Another process/container uses that host port. Change `-p` or stop the other. |
| Container exits immediately | The main process ended/crashed. Check `docker logs`. CMD must run a **foreground** process. |
| Code changes don't show | Missing bind mount, or you need `--build`. In dev mount the source; in prod rebuild. |
| `Cannot connect to the Docker daemon` | Docker Desktop/Engine isn't running. |
| App can't reach DB | Wrong hostname — use the **service name** (e.g. `db`), not `localhost`, inside the network. |
| DB data gone after `down` | You used `down -v`, or didn't use a named volume. |
| Huge image | No multi-stage, big base image, copied `node_modules`/`.git`. |
| Build is slow every time | Bad layer order — copy `package.json` + install before source. |
| Permission denied on mounted files | Non-root user vs host file ownership mismatch — align UID or fix perms. |

> **Golden debugging move:** `docker logs <name>` then `docker exec -it <name> sh` to inspect from inside. 90% of issues are visible there.

---

## 20. Registries & Deployment

### Push/pull images
```bash
docker login                                  # log in to Docker Hub
docker tag myapp:1.0 username/myapp:1.0       # tag for the registry
docker push username/myapp:1.0                # upload
docker pull username/myapp:1.0                # download elsewhere

# GitHub Container Registry
docker tag myapp:1.0 ghcr.io/username/myapp:1.0
docker push ghcr.io/username/myapp:1.0
```

### Deployment options for client work
| Where | How |
|---|---|
| **VPS** (DigitalOcean, Hetzner, Linode) | Install Docker, copy your `docker-compose.yml`, `docker compose up -d`. Cheapest, full control. |
| **Render / Railway / Fly.io** | Connect repo, they build the Dockerfile and host it. Easiest. |
| **AWS ECS / Google Cloud Run / Azure** | Push image to their registry, run as a managed service. Scalable. |
| **Behind a reverse proxy** | Use **nginx** or **Caddy/Traefik** container for HTTPS + routing to your app. |

### Typical small-prod deploy (VPS)
1. Build & push image (or build on the server).
2. Copy `docker-compose.yml` + `.env` to the server.
3. `docker compose pull && docker compose up -d`.
4. Put **Caddy/Traefik** in front for automatic HTTPS.
5. Set `restart: unless-stopped` so it survives reboots.

---

## 21. Tips, Tricks & Gotchas

**Do:**
- ✅ Start every project with a `.dockerignore`.
- ✅ Copy `package.json` + install **before** copying source (layer caching).
- ✅ Use **multi-stage builds** + **alpine** for tiny production images.
- ✅ Use **named volumes** for databases, **bind mounts** for dev code.
- ✅ Reference services by **name** across the Compose network, never `localhost`.
- ✅ Add **healthchecks** + `restart: unless-stopped` for reliability.
- ✅ Pin image versions (`node:20-alpine`, not `latest`).
- ✅ Run as a **non-root** user in production.

**Don't / common bugs:**
- ❌ Storing important data only in a container's filesystem (it vanishes on `rm`).
- ❌ `COPY .env` or hardcoding secrets into images.
- ❌ Using `latest` tag — non-reproducible, surprise breakage.
- ❌ Running a background process as CMD (container exits — the main process must stay in foreground).
- ❌ Forgetting `-d` and wondering why your terminal is "stuck" (it's attached to logs).
- ❌ `docker compose down -v` in production (deletes volumes = data loss).
- ❌ Bloating images by not cleaning package caches in the same `RUN`.
- ❌ Expecting `depends_on` to wait for the DB to be *ready* (it only waits for *start* — add a healthcheck).
- ❌ Mapping the wrong port order (`-p host:container`, not the reverse).

**Tricks:**
- 🔹 `docker compose up --build -d` after dependency/Dockerfile changes.
- 🔹 `docker compose run --rm app npm install <pkg>` to add a package using the container's env.
- 🔹 `docker system prune -a` to reclaim gigabytes of unused images/cache.
- 🔹 `docker compose config` to debug a YAML that won't behave.
- 🔹 BuildKit cache mounts (`RUN --mount=type=cache,...`) for lightning rebuilds.
- 🔹 Override files: `docker-compose.override.yml` auto-merges for local dev tweaks.
- 🔹 `next.config: output: "standalone"` → tiny Next.js Docker images.

---

## 22. Study Path

Learn in this order for fastest offline mastery:

1. **Concepts** — image vs container vs Dockerfile (§1, §3, §4).
2. **Run existing images** — `docker run`, `ps`, `logs`, `exec`, `stop`, `rm` (§5).
3. **Write a Dockerfile** for a simple Node/Express app (§6, §7).
4. **Volumes & networking** — persist a database, connect two containers (§10, §11).
5. **Docker Compose** — combine app + DB in one file, `docker compose up` (§13–§15).
6. **Multi-stage + optimization** — shrink a Next.js/React image (§8, §17).
7. **Security + deployment** — non-root, secrets, push to a registry, deploy to a VPS (§18, §20).

### Build to learn (do these 3 projects)
1. **Dockerize an Express API** → single Dockerfile, run it, hit an endpoint.
2. **Compose a full-stack app** → Next.js/React + Postgres (or Mongo) + Redis, one `docker compose up`.
3. **Production multi-stage build** → ship a Next.js app as a <200MB image, deploy to Render/Railway or a VPS.

> Those three cover ~90% of what clients ever need: containerizing an app, running a full stack locally, and deploying a lean production image. Keep `docker ps`, `docker logs`, and `docker compose up/down` in muscle memory — they're 80% of daily use.
