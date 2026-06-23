# Docker & Docker Compose — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never run a container" to "I can write a tiny, secure, multi-stage Dockerfile, orchestrate a full web + db + cache stack with Compose, debug it, and ship it to a registry and a server" — entirely offline. Every concept is explained in prose **first** (what it is, *why* it exists, when and how to use it, the important flags/instructions, best practices, and security notes), then demonstrated with heavily-commented, runnable commands and files. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Docker Engine 27.x / Docker Desktop 4.3x** and the **Compose v2** plugin (current in 2026). Modern features you should know and that this guide uses throughout:
> - **BuildKit is the default builder** — it enables parallel stages, `RUN --mount=type=cache`, `RUN --mount=type=secret`, and faster, smarter caching. `docker build` uses it automatically; `docker buildx` is the modern multi-platform front-end.
> - **`docker compose`** (a space, the built-in Go plugin — Compose **v2**) has fully replaced the legacy Python **`docker-compose`** (a hyphen). Use the space version everywhere.
> - **The Compose Specification** merged the old "version 2/3" file formats — the top-level `version:` key is now obsolete and you should omit it.
> - **`docker init`** scaffolds a Dockerfile, `.dockerignore`, and `compose.yaml` for common stacks; **`docker scout`** is the built-in vulnerability scanner; **`compose.yaml`** is now the preferred filename (over `docker-compose.yml`, which still works).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11** (Docker Desktop with the WSL 2 backend), so cross-platform notes (path separators, `$(pwd)` vs `%cd%`, line endings, the Docker socket) are called out. Always confirm exact flags with `docker <command> --help`.

---

## Table of Contents

1. [What Containers Are & Why — The Mental Model](#1-what-containers-are--why--the-mental-model) **[B]**
2. [Installation & First Run](#2-installation--first-run) **[B]**
3. [Images vs Containers vs Registries](#3-images-vs-containers-vs-registries) **[B]**
4. [The Docker CLI — Running & Managing Containers](#4-the-docker-cli--running--managing-containers) **[B/I]**
5. [Working with Images — pull, build, tag, push, inspect](#5-working-with-images--pull-build-tag-push-inspect) **[B/I]**
6. [The Dockerfile — Building Images In Depth](#6-the-dockerfile--building-images-in-depth) **[I]**
7. [Dockerfile Instruction Reference (every instruction)](#7-dockerfile-instruction-reference-every-instruction) **[I]**
8. [Layer Caching & Build Ordering](#8-layer-caching--build-ordering) **[I]**
9. [Multi-Stage Builds](#9-multi-stage-builds) **[I/A]**
10. [.dockerignore](#10-dockerignore) **[B/I]**
11. [Volumes & Bind Mounts — Data Persistence](#11-volumes--bind-mounts--data-persistence) **[I]**
12. [Networking](#12-networking) **[I]**
13. [Environment Variables & Secrets](#13-environment-variables--secrets) **[I]**
14. [Docker Compose In Depth](#14-docker-compose-in-depth) **[I]**
15. [The compose.yaml Reference (every key)](#15-the-composeyaml-reference-every-key) **[I/A]**
16. [Compose CLI Commands](#16-compose-cli-commands) **[I]**
17. [A Real Multi-Service Stack: web + db + redis](#17-a-real-multi-service-stack-web--db--redis) **[I/A]**
18. [Image Size & Build Optimization](#18-image-size--build-optimization) **[I/A]**
19. [Security Recommendations](#19-security-recommendations) **[I/A]**
20. [Debugging Containers](#20-debugging-containers) **[I]**
21. [Registries, Deployment & Orchestration](#21-registries-deployment--orchestration) **[I/A]**
22. [Tips, Tricks & Gotchas](#22-tips-tricks--gotchas) **[I/A]**
23. [Study Path & Build-to-Learn Projects](#23-study-path--build-to-learn-projects)

---

## 1. What Containers Are & Why — The Mental Model

### 1.1 The problem Docker solves **[B]**

Before containers, deploying software meant reproducing an *environment*: the right operating system, the right language runtime version, the right system libraries, the right environment variables, the right file layout. Do any of those differ between your laptop and the server — a newer OpenSSL, a missing C library, Node 18 instead of Node 20 — and the program that ran perfectly on your machine fails in production. This is the legendary **"but it works on my machine"** problem, and it costs an enormous amount of time.

A **container** solves it by packaging your application *together with* everything it needs to run — the runtime, the libraries, the file layout, the config — into one portable unit called an **image**. That image runs **identically** on your laptop, a teammate's machine, a client's server, or a cloud platform, because it carries its own little world with it. The host machine only has to provide a Linux kernel; everything above the kernel comes from the image.

For real-world / freelance work this is transformative: clean handoffs (a client runs **one command** and your whole stack boots), reproducible deploys, and no environment drift. This is also why nearly all modern infrastructure tooling is built on containers.

### 1.2 Containers vs Virtual Machines — the kernel/namespace/cgroup model **[B]**

The most important concept to internalize is **how a container differs from a virtual machine**, because it explains everything about why containers are small, fast, and cheap.

**A virtual machine** virtualizes *hardware*. A program called a hypervisor (VMware, VirtualBox, Hyper-V, KVM) emulates a complete computer — virtual CPU, virtual disk, virtual network card — and on top of that virtual hardware you install a **complete guest operating system**, kernel and all. So a VM running a tiny web server still carries gigabytes of guest OS, takes tens of seconds to boot, and consumes a fixed slice of RAM whether it's busy or idle. You can run maybe a handful of VMs on a laptop.

**A container** does *not* virtualize hardware and does *not* ship its own kernel. Instead, every container is just an **ordinary process** running directly on the host's Linux kernel — but that process is given a carefully constructed *illusion* that it is alone on its own machine. Three Linux kernel features create that illusion:

- **Namespaces** — *isolation of what a process can see.* The kernel can give a process its own private view of various system resources. The PID namespace makes the container's main process believe it is PID 1 and that it cannot see any other processes on the host. The mount namespace gives it its own filesystem root (so it sees the image's files, not the host's). The network namespace gives it its own network interfaces, IP, and ports. UTS gives it its own hostname; user namespaces can remap its user IDs. Strip away the jargon and a namespace is simply: *"this process gets its own private copy of this resource, and can't see anyone else's."*
- **Control groups (cgroups)** — *limits on what a process can use.* cgroups cap and meter how much CPU, memory, disk I/O, and network bandwidth a process (and its children) may consume. This is how `--memory 512m` or `--cpus 0.5` works: the kernel simply refuses to let the container exceed its quota, so one runaway container can't starve the others.
- **A union/overlay filesystem** — *efficient layered storage.* The container's filesystem is assembled by stacking read-only image layers and adding one thin writable layer on top (copy-on-write). This is why images are deduplicated and containers start instantly — no copying gigabytes, just stacking existing layers.

Put those together and a container is: **a normal process, isolated by namespaces, limited by cgroups, with a layered filesystem.** Because it's just a process sharing the host kernel, it boots in milliseconds, weighs megabytes, and you can run dozens on one laptop.

```
   ┌─────────────── VIRTUAL MACHINES ───────────────┐   ┌──────────────── CONTAINERS ────────────────┐
   │  App A     App B     App C                       │   │  App A     App B     App C                  │
   │  Bins/Libs Bins/Libs Bins/Libs                   │   │  Bins/Libs Bins/Libs Bins/Libs             │
   │  Guest OS  Guest OS  Guest OS   ← full kernel ×3 │   │  ── Docker Engine (namespaces/cgroups) ──  │
   │  ──────── Hypervisor ────────                    │   │  ──────── Host OS / shared kernel ───────  │
   │  ──────── Host OS ───────────                    │   │  ──────── Hardware ──────────────────────  │
   │  ──────── Hardware ──────────                    │   └─────────────────────────────────────────────┘
   └──────────────────────────────────────────────────┘
   Heavy: GBs each, boots in ~30s                          Light: MBs each, boots in ~ms — kernel is shared
```

> **⚡ Version note (Windows/Mac):** The Linux kernel that containers share doesn't exist natively on Windows or macOS. Docker Desktop quietly runs a *single* lightweight Linux VM (on Windows via **WSL 2**) and runs all your Linux containers *inside* it. So on those OSes there is technically one VM — but you still get the container benefits (one shared kernel for all containers, instant start, tiny images). Windows *containers* (running Windows binaries) also exist but are niche; this guide covers Linux containers, which is 99% of web work.

### 1.3 The three-step lifecycle **[B]**

Everything in Docker reduces to a three-step pipeline. Hold this in your head and the rest of the tool falls into place:

```
   Dockerfile  ──(docker build)──►   Image   ──(docker run)──►   Container
   (the recipe)                      (the frozen, immutable      (a live, disposable
                                      shareable package)          running process)
```

- You **write** a `Dockerfile` — a plain-text recipe describing how to assemble your app's environment step by step.
- You **build** it into an **image** — an immutable, versioned, shareable artifact made of stacked layers.
- You **run** the image to create a **container** — a live process. One image can spawn many independent containers.

A crucial corollary: **containers are ephemeral.** Treat them as *cattle, not pets* — disposable and replaceable, not lovingly maintained. Any state that must survive a container being deleted (database files, user uploads) must live **outside** the container in a **volume** or an external database — never in the container's own writable layer, which is destroyed when the container is removed.

> **One-line summary:** *A Dockerfile builds an Image; an Image runs as a Container; Compose orchestrates many Containers.*

### 1.4 Core vocabulary **[B]**

| Term | Meaning |
|---|---|
| **Image** | A read-only **blueprint/template** (your app + its libraries + a minimal OS userland). Like a *class*. |
| **Container** | A running (or stopped) **instance** of an image — an isolated process with its own filesystem, network, and resource limits. Like an *object*. |
| **Dockerfile** | A text file of instructions that **builds** an image, step by step, top to bottom. |
| **Layer** | Each image-building instruction produces a cached, content-addressed filesystem **layer**; an image is a stack of layers. |
| **Registry** | A remote store for images (Docker Hub, GitHub Container Registry / GHCR, AWS ECR, Google Artifact Registry). |
| **Repository** | A named collection of related image versions in a registry (e.g. `library/postgres`). |
| **Tag** | A label/version on an image within a repo (`myapp:1.0`, `node:20-alpine`). |
| **Volume** | Docker-managed persistent storage that lives **outside** any container, so data survives container removal. |
| **Bind mount** | A host directory mapped straight into a container (used for live source code in dev). |
| **Network** | A virtual network that lets containers reach each other **by name** via Docker's built-in DNS. |
| **Docker Engine / `dockerd`** | The background **daemon** that actually builds and runs containers. The CLI talks to it. |
| **Docker Compose** | A tool to define and run **multi-container** applications from one YAML file. |
| **Build context** | The set of files sent to the builder when building (the folder containing the Dockerfile + code). |
| **BuildKit** | The modern build engine (default) enabling parallelism, cache mounts, and build secrets. |

---

## 2. Installation & First Run

### 2.1 Installing **[B]**

- **Windows / macOS:** install **Docker Desktop**. It bundles the Engine, CLI, Compose v2, BuildKit, a GUI dashboard, and (on Windows) sets up the WSL 2 backend automatically. Enable WSL 2 integration when prompted — it's dramatically faster than the legacy Hyper-V backend, especially for bind-mounted source code.
- **Linux:** install **Docker Engine** plus the **Compose plugin** from Docker's official apt/yum repository (not the often-outdated distro package). After install, add your user to the `docker` group (`sudo usermod -aG docker $USER`, then re-login) so you don't have to `sudo` every command — but be aware this grants root-equivalent power (see §19).

### 2.2 Verifying the install **[B]**

After installing, run these to confirm every piece works. Each command is annotated with *why* you'd run it:

```bash
docker --version            # prints the client version — confirms the CLI is on your PATH
docker compose version      # confirms the Compose v2 plugin is installed (note: a SPACE, not a hyphen)
docker info                 # system-wide info: storage driver, # of images/containers, the active builder
docker run hello-world      # pulls a tiny test image from Docker Hub and runs it — proves the whole
                            # pipeline works end-to-end (daemon reachable, registry reachable, run works)
```

`docker run hello-world` is the canonical smoke test: it downloads a minimal image, runs it, the container prints a success message and exits. If you see that message, your daemon is running and can reach a registry.

> **⚡ Version note:** The old standalone binary `docker-compose` (a **hyphen**, written in Python) is deprecated. Everything modern is `docker compose` (a **space**, the Go plugin). If a tutorial uses the hyphen, mentally translate it. They take nearly identical arguments.

### 2.3 What just happened? **[B]**

When you ran `docker run hello-world`, the daemon: (1) looked for the `hello-world` image locally, (2) didn't find it, so **pulled** it from Docker Hub, (3) created a **container** from it, (4) ran the container's single command (printing a message), and (5) the process exited, so the container stopped. Run `docker ps -a` and you'll see that stopped container still sitting there — containers are not auto-deleted unless you ask (`--rm`).

---

## 3. Images vs Containers vs Registries

### 3.1 The class/object analogy **[B]**

The single most common beginner confusion is *image vs container*. They are as different as a **class** and an **object** in programming, or a **recipe** and a **cooked meal**:

| | **Image** | **Container** |
|---|---|---|
| Nature | Read-only template | Running (or stopped) instance |
| Analogy | Class / recipe / blueprint | Object / cooked dish |
| Count | One image… | …runs as many independent containers |
| Mutability | Immutable | Has a thin writable layer (lost when the container is removed) |
| Created by | `docker build`, `docker pull` | `docker run`, `docker create` |
| Lives in | Local image store / a registry | The Docker daemon's runtime |

Run one image five times and you get **five independent containers**, each with its own writable layer, its own PID, its own network identity — but all sharing the same read-only image layers underneath (which is why this is cheap).

### 3.2 Image names, tags, and digests **[B/I]**

A full image reference looks like:

```
   registry-host[:port]  /  repository           :  tag
   ghcr.io               /  myorg/myapp          :  1.4.2
   docker.io (implicit)  /  library/postgres     :  16-alpine
```

- If you omit the **registry host**, Docker assumes **Docker Hub** (`docker.io`). `postgres:16` really means `docker.io/library/postgres:16`. Official images live under the hidden `library/` namespace.
- If you omit the **tag**, Docker assumes `:latest` — which is **not** "the newest version," it's just the default tag name. Relying on `latest` is a top source of "it broke and nothing changed" bugs (see §19). Always pin a real version.
- A **digest** (`postgres@sha256:abc123…`) pins an *exact, immutable* image by content hash. Tags can be re-pointed; digests never change. Use digests when you need bit-for-bit reproducibility.

### 3.3 Registries **[B/I]**

A **registry** is where images are stored and shared. The flow is symmetrical with git: you `pull` images down and `push` them up.

- **Docker Hub** (`docker.io`) — the default public registry; hosts official images (`node`, `postgres`, `nginx`). Free public repos; rate-limited anonymous pulls (log in to raise limits).
- **GitHub Container Registry / GHCR** (`ghcr.io`) — integrates with GitHub Actions; great for CI-built images tied to a repo.
- **Cloud registries** — AWS **ECR**, Google **Artifact Registry**, Azure **ACR** — used when deploying to those clouds.

Pushing/pulling is covered in §21.

---

## 4. The Docker CLI — Running & Managing Containers

The CLI is how you do everything. This section covers the container lifecycle commands; §5 covers the image commands. The pattern is `docker <command> [flags] [arguments]`, and almost every command accepts a container by **name** or by (a prefix of) its **ID**.

### 4.1 `docker run` — create and start a container **[B]**

`docker run` is the workhorse: it creates a fresh container from an image and starts it. Conceptually it's `docker create` + `docker start` in one step. The image name comes last; everything before it is flags that configure the container.

The flags you will use constantly, each explained:

- **`-d` / `--detach`** — run in the *background* and return your prompt immediately. Without it the container runs in the *foreground* and your terminal is attached to its output (this surprises beginners who think their terminal "froze" — it's just showing logs).
- **`-p host:container` / `--publish`** — *publish* a port: forward a port on the host to a port inside the container. Order matters: **host first, container second.** `-p 8080:3000` means "traffic to the host's 8080 goes to the container's 3000." Without this, the container's ports are unreachable from outside Docker.
- **`--name`** — give the container a stable, human-friendly name (otherwise Docker invents a random one like `nostalgic_curie`). Names must be unique among existing containers.
- **`-e KEY=value` / `--env`** — set an environment variable inside the container. Repeatable. Use `--env-file path` to load many from a file.
- **`-v` / `--mount`** — attach a volume or bind mount (see §11).
- **`-it`** — actually two flags: **`-i`** keeps STDIN open (interactive) and **`-t`** allocates a pseudo-TTY (a terminal). Combine them to get an interactive shell. `docker run -it ubuntu bash` drops you into a bash prompt inside a fresh Ubuntu container.
- **`--rm`** — automatically delete the container when it exits. Perfect for one-off / throwaway runs so you don't accumulate stopped containers.
- **`--restart`** — a restart *policy* for crash/reboot resilience: `no` (default), `on-failure[:max]`, `always`, or `unless-stopped` (the usual production choice — restart on crash and on host reboot, but stay down if you deliberately stopped it).
- **`--network`** — attach the container to a specific network (see §12).
- **`-u` / `--user`** — run as a specific user/UID instead of the image's default.
- **`-w` / `--workdir`** — set the working directory for the command.
- **`--memory` / `--cpus`** — cgroup resource limits (e.g. `--memory 512m --cpus 0.5`).

```bash
docker run nginx                              # foreground; terminal is now attached to nginx's logs
docker run -d nginx                           # background; prints the new container ID and returns
docker run -d -p 8080:80 nginx                # browse http://localhost:8080 → the container's port 80
docker run -d --name web -p 8080:80 nginx     # same, but named "web" for easy reference later
docker run -e NODE_ENV=production myapp        # set an env var
docker run --rm -it ubuntu bash               # interactive shell in a throwaway Ubuntu container
docker run --restart unless-stopped -d redis  # survives crashes & host reboots

# A realistic production-style run, all flags together:
docker run -d \
  --name web \                  # stable name
  -p 80:3000 \                  # host:80 → container:3000
  -e NODE_ENV=production \      # runtime config
  --restart unless-stopped \    # resilience
  --memory 512m --cpus 0.5 \   # resource caps so it can't hog the host
  myapp:1.0                      # the image (always last)
```

> **Gotcha — the port order trap:** `-p` is **host:container**. If your app listens on 3000 inside the container and you want it on 8080 on your machine, it's `-p 8080:3000`, not `-p 3000:8080`. Getting this backwards is the #1 "why can't I reach my app" bug.

> **Gotcha — `EXPOSE` does not publish.** A `Dockerfile`'s `EXPOSE 3000` only *documents* the port; it does nothing by itself. You still need `-p` (or Compose `ports:`) to actually make it reachable.

### 4.2 Listing & inspecting containers **[B]**

```bash
docker ps                 # list RUNNING containers (ID, image, command, status, ports, name)
docker ps -a              # list ALL containers including stopped/exited ones
docker ps -q              # quiet: just the IDs (handy for scripting: docker rm $(docker ps -aq))
docker ps --filter "status=exited"   # filter, e.g. only exited containers
docker inspect web        # FULL JSON: env vars, mounts, network, IP, restart policy, exit code…
docker stats              # live, top-like view of CPU / memory / network / disk I/O per container
docker top web            # the processes running INSIDE the container
docker port web           # show the port mappings for a container
```

`docker inspect` is your truth source when something's misconfigured — it shows exactly what env vars, mounts, and network a container actually got, which is often different from what you *think* you passed.

### 4.3 Stopping, starting, removing **[B]**

```bash
docker stop web           # graceful stop: sends SIGTERM, waits ~10s, then SIGKILL
docker stop -t 30 web     # give it 30s to shut down cleanly before the kill
docker start web          # start a previously-stopped container (keeps its config & data)
docker restart web        # stop then start
docker kill web           # immediate SIGKILL (no graceful shutdown) — last resort
docker rm web             # remove a STOPPED container
docker rm -f web          # force-remove a running container (stop + remove in one)
docker rm $(docker ps -aq)   # remove ALL containers (running ones need -f)
```

> **Why graceful stop matters:** `docker stop` sends **SIGTERM** so your app can flush buffers, close DB connections, and finish in-flight requests, *then* SIGKILL if it ignores you. Make sure your app's main process actually handles SIGTERM (Node does by default; some shells swallow it — see the `exec`/PID 1 note in §7).

### 4.4 Logs and getting inside a container **[B/I]**

```bash
docker logs web           # print everything the container has written to stdout/stderr
docker logs -f web        # FOLLOW (stream) logs live — like tail -f
docker logs --tail 100 web        # only the last 100 lines
docker logs --since 10m web       # only the last 10 minutes
docker logs -t web                # prepend timestamps

docker exec -it web sh    # open an interactive shell INSIDE a running container (use 'bash' if present)
docker exec web ls /app   # run a one-off command inside without an interactive shell
docker exec -it -u root web sh    # get in as root even if the container runs as a non-root user
```

`docker exec` runs a *new* process inside an *already-running* container — that's how you poke around a live container ("why is the config wrong? let me look"). It's different from `docker run`, which makes a *new* container. Alpine-based images have no `bash`, only `sh` — reach for `sh` first.

### 4.5 Copying files and one-off cleanup **[I]**

```bash
docker cp web:/app/output.log ./        # copy a file OUT of a container to the host
docker cp ./fix.conf web:/etc/app/      # copy a file INTO a container
docker diff web                          # show files changed in the container's writable layer
```

---

## 5. Working with Images — pull, build, tag, push, inspect

### 5.1 Getting and listing images **[B]**

```bash
docker pull node:20-alpine     # download a specific image+tag from a registry (Docker Hub by default)
docker pull postgres            # no tag → pulls :latest (avoid relying on this; pin a version)
docker images                   # list local images (repository, tag, image ID, age, size)
docker images -q                # just image IDs
docker history node:20-alpine   # show the layers that make up an image, and each layer's size
docker inspect node:20-alpine   # full metadata: env, entrypoint, exposed ports, layers, architecture
```

`docker history` is a fantastic learning and debugging tool: it shows you *every layer*, which Dockerfile instruction created it, and how many bytes it added — instantly revealing what's bloating your image.

### 5.2 Building images **[B/I]**

```bash
docker build -t myapp:1.0 .              # build from ./Dockerfile, tag the result myapp:1.0
                                          # the "." is the BUILD CONTEXT (the folder sent to the builder)
docker build -t myapp:1.0 -t myapp:latest .   # apply two tags at once
docker build -f docker/prod.Dockerfile -t myapp:prod .   # use a non-default Dockerfile name
docker build --target builder -t myapp:dev .   # build only up to a named multi-stage target (§9)
docker build --build-arg VERSION=1.2 -t myapp .  # pass a build-time ARG (§7)
docker build --no-cache -t myapp .       # ignore the layer cache and rebuild everything from scratch
docker build --progress=plain -t myapp . # verbose, plain build output (great for debugging a failing RUN)
```

The trailing `.` is the **build context**: Docker tars up that directory and sends it to the daemon, and `COPY`/`ADD` instructions can only see files inside it. A bloated context (e.g. including `node_modules` or `.git`) makes builds slow — which is exactly what `.dockerignore` prevents (§10).

### 5.3 Tagging, pushing, pulling **[I]**

```bash
docker tag myapp:1.0 yourname/myapp:1.0          # add a registry-qualified tag (Docker Hub)
docker tag myapp:1.0 ghcr.io/yourname/myapp:1.0  # tag for GitHub Container Registry
docker login                                      # authenticate to Docker Hub (or: docker login ghcr.io)
docker push yourname/myapp:1.0                    # upload the image to the registry
docker pull yourname/myapp:1.0                    # download it elsewhere (server, CI, teammate)
```

A "tag" here both names and versions the image *and* tells Docker which registry/repo to push to. Full deployment flow is in §21.

### 5.4 Cleaning up disk space **[B/I]**

Images, stopped containers, unused volumes, and the build cache silently accumulate and can eat tens of gigabytes. Know how to reclaim it:

```bash
docker system df            # see exactly what's using disk (images, containers, volumes, build cache)
docker image prune          # remove DANGLING images (untagged leftovers from rebuilds)
docker image prune -a       # remove ALL images not used by any container (aggressive)
docker container prune      # remove all stopped containers
docker volume prune         # remove volumes not attached to any container (CAREFUL: data loss)
docker builder prune        # clear the BuildKit build cache
docker system prune         # remove stopped containers, unused networks, dangling images, build cache
docker system prune -a --volumes   # nuke EVERYTHING unused incl. volumes — reclaims the most, riskiest
```

> **⚠️ Be careful with `prune --volumes`** — it deletes volumes (and therefore databases) that aren't currently attached to a running container. Don't run it casually on a machine hosting real data.

---

## 6. The Dockerfile — Building Images In Depth

### 6.1 What a Dockerfile is and how it's read **[I]**

A **Dockerfile** is a plain-text recipe that the builder reads **top to bottom**, executing each instruction in order. Each instruction that changes the filesystem produces a new, cached **layer** stacked on the previous one. The result of running all instructions is your **image**.

Two mental models you need:

1. **It's a sequence of filesystem snapshots.** `FROM` gives you a starting filesystem (a base image). Every `RUN`, `COPY`, and `ADD` modifies that filesystem and freezes the result as a new layer. The final stack of layers *is* the image.
2. **Build-time vs run-time is a hard line.** Some instructions (`RUN`, `COPY`, `ADD`) execute **while building** the image. Others (`CMD`, `ENTRYPOINT`) only describe **what happens when a container later starts**. Confusing the two is the most common Dockerfile mistake (e.g. expecting `RUN npm start` to "run the app" — it runs it *during the build*, which hangs the build).

### 6.2 A complete, annotated example **[I]**

Here's a production-shaped Dockerfile for a Node/Express API, with every line explained:

```dockerfile
# syntax=docker/dockerfile:1
# ^ Opt into the latest stable Dockerfile syntax & BuildKit features (cache/secret mounts). Keep it.

# 1) BASE IMAGE — every Dockerfile starts FROM something. "alpine" is a ~5MB minimal Linux,
#    so node:20-alpine is far smaller than node:20 (Debian-based, ~1GB). Pin the version for reproducibility.
FROM node:20-alpine

# 2) WORKDIR — sets (and creates if missing) the working directory for all following instructions.
#    Every relative path after this is relative to /app. Beats scattering "cd" in RUN commands.
WORKDIR /app

# 3) COPY DEPENDENCY MANIFESTS FIRST — this is the layer-caching trick (see §8). package.json and the
#    lockfile change rarely; copying them alone means the expensive install layer below is cached and
#    reused until your dependencies actually change.
COPY package*.json ./

# 4) RUN — executes a command AT BUILD TIME and freezes the result as a layer. "npm ci" installs exactly
#    what the lockfile pins (reproducible, faster than "npm install"); --omit=dev drops devDependencies.
RUN npm ci --omit=dev

# 5) NOW copy the rest of the source. Because this comes AFTER the install layer, editing your code
#    invalidates only this layer and below — the cached install layer is reused (rebuilds in seconds).
COPY . .

# 6) Create and switch to a non-root user (security: containers run as root by default — don't). The node
#    image already ships a "node" user, so we just switch to it. (See §19 for why this matters.)
USER node

# 7) EXPOSE — DOCUMENTS that the app listens on 3000. It does NOT publish the port; you still need -p / ports:.
EXPOSE 3000

# 8) CMD — the DEFAULT command run when a container starts. Exec form (JSON array) is preferred so signals
#    (SIGTERM from "docker stop") reach your process directly. This runs at RUN-TIME, not build-time.
CMD ["node", "server.js"]
```

```bash
docker build -t my-api:1.0 .          # build it
docker run -d -p 3000:3000 my-api:1.0 # run it; visit http://localhost:3000
```

The rest of this section's instructions are covered exhaustively in §7; §8 dives deep into *why the ordering above makes builds fast*.

---

## 7. Dockerfile Instruction Reference (every instruction)

This is the heart of writing images. Each instruction is explained in prose — *what it does, why/when to use it, the key options, and the gotchas* — then shown.

### 7.1 `FROM` — choose the base image **[I]**

`FROM` sets the starting filesystem your image builds upon, and it **must be the first instruction** (after optional `# syntax`/`ARG`). The base image determines your OS userland, what libraries and shells exist, and a large part of your final size and attack surface. Choosing well is half of writing a good Dockerfile.

- **Pin a specific tag**, never `latest` (`FROM node:20.11-alpine`), for reproducible builds.
- Prefer **small bases**: `-alpine` (musl libc, ~5MB) or `-slim` (Debian, trimmed) over the full image. For compiled languages, **`scratch`** (an *empty* image) or Google's **distroless** images give the smallest, most secure result (see §9, §19).
- `FROM <image> AS <name>` names a build *stage* for multi-stage builds (§9).

```dockerfile
FROM node:20.11-alpine          # pinned, tiny base
FROM golang:1.24 AS builder     # a named stage (multi-stage)
FROM scratch                    # the empty base — only for fully-static binaries
```

### 7.2 `RUN` — execute a command at build time **[I]**

`RUN` runs a shell command **while building** and commits the resulting filesystem as a layer. Use it to install packages, build your app, create users — anything that should be baked into the image.

- **Shell form** `RUN npm ci` runs via `/bin/sh -c` (so shell features like `&&`, `|`, `$VAR` work).
- **Exec form** `RUN ["npm", "ci"]` runs the binary directly (no shell — no variable expansion).
- **Chain related commands with `&&`** in one `RUN` and clean up in the *same* layer — because each `RUN` is a separate layer, deleting a file in a *later* `RUN` doesn't shrink the image; the bytes are still in the earlier layer (see §8/§18).
- BuildKit adds **cache mounts** (`RUN --mount=type=cache,...`) to persist package caches across builds, and **secret mounts** (`RUN --mount=type=secret,...`) for build-time credentials that never land in a layer (§13).

```dockerfile
# Debian: update, install, and CLEAN the apt cache in ONE layer (or the image won't shrink)
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# BuildKit cache mount: reuse the npm cache across builds without baking it into the image
RUN --mount=type=cache,target=/root/.npm npm ci
```

### 7.3 `COPY` vs `ADD` — get files into the image **[I]**

Both copy files from the build context into the image. **Prefer `COPY`** for almost everything — it's explicit and predictable.

- **`COPY src dest`** — copies files/dirs from the build context into the image. That's it. Use `--chown=user:group` to set ownership and `--from=stage` to copy from another build stage (§9).
- **`ADD`** does the same but with two extra "magic" behaviors: it can fetch a **URL**, and it **auto-extracts** local tar archives. These surprises make builds harder to reason about, so use `ADD` *only* when you specifically want tar auto-extraction; use `COPY` otherwise. (To download files, prefer an explicit `RUN curl` so caching and errors are visible.)

```dockerfile
COPY package*.json ./                 # copy just the manifests (caching)
COPY . .                              # copy the whole context
COPY --chown=node:node . .            # copy AND set ownership in one step (avoids a later chmod layer)
COPY --from=builder /app/dist ./dist  # copy an artifact from an earlier stage (multi-stage)
ADD project.tar.gz /opt/app/          # legit ADD use: auto-extract a local tarball
```

### 7.4 `WORKDIR` — set the working directory **[I]**

`WORKDIR /path` sets the directory that subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` instructions operate in, creating it if needed. Use it instead of `RUN cd /app` (which doesn't persist — each `RUN` is a fresh shell). Always use an **absolute path**.

```dockerfile
WORKDIR /app          # all relative paths below are now relative to /app, and /app is created
```

### 7.5 `ENV` and `ARG` — variables (the crucial difference) **[I]**

These two look similar but live in different *phases* and have different *lifetimes* — getting them straight prevents secret leaks and surprising behavior.

- **`ENV KEY=value`** sets an environment variable that is **baked into the image** and present **at run time** inside every container. Use it for non-secret runtime config (`ENV NODE_ENV=production`). It persists and is visible via `docker inspect` — so **never put secrets in `ENV`.**
- **`ARG name[=default]`** declares a **build-time-only** variable, supplied with `--build-arg`. It exists *only during the build* and is **not** present in the final running container's environment. Use it for build parameters (versions, flags). **Caution:** an `ARG`'s value can still appear in the image's build history/layers, so it's *not* a safe place for secrets either — use BuildKit secret mounts for those (§13).

```dockerfile
ARG NODE_VERSION=20                 # build-time variable with a default
FROM node:${NODE_VERSION}-alpine    # ARG before FROM can parameterize the base image
ARG APP_VERSION                     # must be re-declared after FROM to use it in a build stage
ENV NODE_ENV=production \
    PORT=3000                       # run-time config, present in the container, visible in inspect
RUN echo "building version ${APP_VERSION}"   # ARG usable during build
```
```bash
docker build --build-arg APP_VERSION=1.4.2 -t myapp .   # supply the ARG at build time
```

### 7.6 `EXPOSE` — document a port **[I]**

`EXPOSE 3000` is **documentation only**: it records which port the app listens on, for humans and for tools (`docker run -P` will auto-publish all exposed ports to random host ports). It does **not** open or publish anything by itself — actual publishing is done with `-p` / Compose `ports:`. Always include `EXPOSE` anyway; it's a useful contract.

```dockerfile
EXPOSE 3000          # "this app listens on 3000" — still need -p 3000:3000 to reach it
```

### 7.7 `CMD` vs `ENTRYPOINT` — what runs when the container starts **[I]**

This is the most-misunderstood pair. Both define the process that runs **when a container starts** (run time, *not* build time), but they combine in a specific way:

- **`CMD`** sets the **default** command/args. It is **easily overridden** — anything you append to `docker run myimage …` *replaces* the `CMD` entirely. There can be only one effective `CMD`.
- **`ENTRYPOINT`** sets a **fixed** executable that always runs; arguments from `CMD` (or from `docker run`) are **appended** to it rather than replacing it.
- **The idiomatic combo:** `ENTRYPOINT` is the program, `CMD` is its default arguments. `ENTRYPOINT ["node"]` + `CMD ["server.js"]` runs `node server.js`, but `docker run myimg other.js` runs `node other.js` (the CMD was overridden, the entrypoint stayed).

**Always use the exec form (JSON array)**, `CMD ["node","server.js"]`, not the shell form `CMD node server.js`. The shell form wraps your process in `/bin/sh -c`, which becomes PID 1 and often **does not forward SIGTERM** to your app — so `docker stop` hangs for 10 seconds then hard-kills it, skipping graceful shutdown. The exec form makes your app PID 1 and receives signals directly.

```dockerfile
# Most web apps: just a default command, overridable
CMD ["node", "server.js"]

# Fixed program + default args (a CLI tool pattern): runs `ping localhost` by default,
# but `docker run myimg 8.8.8.8` runs `ping 8.8.8.8`
ENTRYPOINT ["ping"]
CMD ["localhost"]

# DON'T (shell form): /bin/sh becomes PID 1 and may swallow SIGTERM → no graceful shutdown
# CMD node server.js
```

> **Tip:** when you need shell features *and* correct signals, use a minimal init like `tini` (`ENTRYPOINT ["tini","--"]`) or run via `exec` so your process replaces the shell. Docker's `--init` flag also injects a tiny init for signal handling/zombie reaping.

### 7.8 `USER` — drop root **[I]**

`USER name|uid` sets the user that subsequent `RUN` steps and the final container process run as. Containers default to **root**, which is a security risk: a process that escapes the container, or a vulnerability in your app, then has root-equivalent power. Create or reuse an unprivileged user and switch to it before `CMD`. Place `USER` after the steps that genuinely need root (like installing packages).

```dockerfile
# Alpine: create a system group + user, then drop to it
RUN addgroup -S app && adduser -S app -G app
USER app
# Debian/Ubuntu equivalent:
# RUN groupadd -r app && useradd -r -g app app
# USER app
```

### 7.9 `HEALTHCHECK` — tell Docker how to know the app is alive **[I]**

`HEALTHCHECK` defines a command Docker runs periodically to decide if a container is **healthy**, **unhealthy**, or **starting**. This lets orchestrators (Compose `depends_on: condition: service_healthy`, Swarm, k8s-style restarts) wait for real readiness and restart sick containers — not just "the process is running" but "the app actually responds."

Options: `--interval` (how often), `--timeout` (how long to wait for the check), `--retries` (consecutive failures before "unhealthy"), and `--start-period` (a grace window during boot where failures don't count). The check command must exit `0` for healthy, `1` for unhealthy.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=10s \
  CMD curl -fsS http://localhost:3000/health || exit 1
# curl -f makes HTTP errors return non-zero; "|| exit 1" guarantees a clean unhealthy signal.
# Note: the base image must actually CONTAIN curl/wget, or use a tiny static healthcheck binary.
```

### 7.10 `VOLUME` — declare a persistent mount point **[I]**

`VOLUME /path` declares that a directory should hold data managed *outside* the image's layered filesystem. When a container starts, Docker auto-creates an anonymous volume for that path (unless you mount your own there). It signals "data here is meant to persist and not be part of the image" — common in database images (`postgres` declares `/var/lib/postgresql/data`). In practice you usually override it with a **named volume** at run time (§11) so you control the name and lifecycle. Avoid declaring `VOLUME` for app code (it can shadow your `COPY`).

```dockerfile
VOLUME /var/lib/postgresql/data    # this dir is for persistent data, not image content
```

### 7.11 `LABEL`, `SHELL`, `STOPSIGNAL`, `ONBUILD` **[I/A]**

- **`LABEL key="value"`** — attach metadata (maintainer, source repo, version). Tools and registries read these. The OCI standard keys start with `org.opencontainers.image.*`.
- **`SHELL ["pwsh","-Command"]`** — change the shell used by shell-form `RUN`/`CMD` (mainly for Windows containers).
- **`STOPSIGNAL SIGINT`** — change which signal `docker stop` sends (default SIGTERM).
- **`ONBUILD`** — register an instruction that runs only when *another* image uses this one as its base. Niche; used by some "builder" base images.

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/you/myapp" \
      org.opencontainers.image.version="1.4.2"
STOPSIGNAL SIGTERM
```

### 7.12 Quick reference table **[I]**

| Instruction | Phase | Purpose | Key options / notes |
|---|---|---|---|
| `FROM` | — | Base image (required, first). | `:tag` pin; `AS name` for stages; `scratch`/distroless for size. |
| `RUN` | build | Run a command, commit a layer. | Chain with `&&`; clean in same layer; `--mount=type=cache/secret`. |
| `COPY` | build | Copy from context into image. | `--chown`, `--from=stage`. Prefer over `ADD`. |
| `ADD` | build | COPY + URL fetch + tar auto-extract. | Use only for tar extraction. |
| `WORKDIR` | build | Set/create working dir. | Absolute path; replaces `cd`. |
| `ENV` | build→run | Runtime env var (baked in). | Non-secrets only; visible in `inspect`. |
| `ARG` | build | Build-time-only variable. | `--build-arg`; not in final container env; not for secrets. |
| `EXPOSE` | doc | Document a listening port. | Doesn't publish; pair with `-p`/`ports:`. |
| `CMD` | run | Default command/args (overridable). | Exec form `["…"]`; one effective CMD. |
| `ENTRYPOINT` | run | Fixed executable; CMD appends args. | Exec form; combine with CMD for defaults. |
| `USER` | build→run | Drop to a non-root user. | After root-needing steps; security must-do. |
| `HEALTHCHECK` | run | Liveness/readiness probe. | `--interval/--timeout/--retries/--start-period`. |
| `VOLUME` | run | Declare persistent mount point. | Usually override with a named volume. |
| `LABEL` | meta | Metadata. | Use OCI `org.opencontainers.image.*` keys. |

---

## 8. Layer Caching & Build Ordering

### 8.1 How the cache works **[I]**

Every instruction produces a layer, and BuildKit **caches** each one. On a rebuild it walks the Dockerfile and, for each instruction, asks: *"have I built this exact instruction, with the same inputs, before?"* If yes, it reuses the cached layer instantly. The catch is the **cache-invalidation rule**:

> **Once an instruction's inputs change, that layer and *every layer after it* are rebuilt from scratch.**

For `RUN`, the "input" is the command text. For `COPY`/`ADD`, the input is the *contents* of the files being copied. So if you `COPY . .` early and then `RUN npm ci`, then **any** source edit changes the copied files, busts the cache from that point, and forces a full `npm ci` every single time — turning a 2-second rebuild into a 2-minute one.

### 8.2 The golden ordering rule **[I]**

**Order instructions from least-frequently-changed to most-frequently-changed.** Dependencies (your lockfile) change rarely; source code changes constantly. So copy and install dependencies *before* copying source:

```dockerfile
# ✅ FAST: deps installed in a layer that's reused until package.json changes
COPY package*.json ./     # ① changes rarely
RUN npm ci                # ② cached unless ① changed — the expensive step, now cached
COPY . .                  # ③ changes constantly — only THIS layer (and below) rebuilds on a code edit

# ❌ SLOW: any code edit busts the cache and re-runs npm ci every time
# COPY . .
# RUN npm ci
```

The same principle applies to OS packages, downloaded tools, anything expensive: install the stable stuff first, copy volatile source last.

### 8.3 Combine and clean in the same layer **[I]**

Because deleting a file in a *later* layer doesn't reclaim space in an *earlier* one, anything you create and then delete must happen in **one** `RUN`:

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && make \                                  # use the build tools…
 && apt-get purge -y build-essential \      # …then remove them…
 && rm -rf /var/lib/apt/lists/*             # …and the apt cache — all in ONE layer, so it actually shrinks
```

(The cleaner alternative — building in one stage and copying only artifacts into a fresh stage — is multi-stage builds, next.)

---

## 9. Multi-Stage Builds

### 9.1 Why multi-stage exists **[I/A]**

Building software needs heavy tooling — compilers, dev dependencies, build caches — but **running** it needs almost none of that. A naive single-stage image ships all the build junk to production: huge, slow to pull, and a big attack surface. **Multi-stage builds** fix this by letting one Dockerfile contain several `FROM` stages: you do the heavy building in an early "builder" stage, then **copy only the final artifacts** into a clean, minimal "runtime" stage. Only the *last* stage becomes your image; the builder is discarded.

The payoff is dramatic: a Go service can go from ~800MB to **~10MB**; a Next.js app from ~1GB to ~150MB. Smaller images mean faster deploys, lower registry/bandwidth cost, and far less to attack or patch.

### 9.2 Go example — a tiny static binary on `scratch` **[I/A]**

Go compiles to a single static binary, so the runtime stage can be the *empty* `scratch` image — nothing but your binary:

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: build (has the full Go toolchain, ~800MB — discarded at the end) ----
FROM golang:1.24-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                     # cache deps separately from source (layer caching)
COPY . .
# CGO_ENABLED=0 → a fully static binary with no libc dependency, so it runs on "scratch".
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server
#  -ldflags "-s -w" strips debug symbols → smaller binary.

# ---- Stage 2: runtime (the EMPTY scratch image — final image is just the binary) ----
FROM scratch
# Copy CA certs so outbound HTTPS works (scratch has none):
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
EXPOSE 8080
USER 65534:65534                        # run as "nobody" (no shell, no root) — scratch has no /etc/passwd
ENTRYPOINT ["/server"]
# Result: a ~10MB image containing ONLY your binary + CA certs. Minimal attack surface.
```

### 9.3 Node example — build then ship only what's needed **[I/A]**

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: install production deps only ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

# ---- Stage 2: build (needs devDependencies for the build) ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                              # full deps, including dev
COPY . .
RUN npm run build                        # produces ./dist (or .next, etc.)

# ---- Stage 3: runtime (tiny — only prod deps + built output) ----
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps    /app/node_modules ./node_modules   # prod-only deps
COPY --from=builder /app/dist         ./dist           # compiled app
COPY package*.json ./
USER node                                               # non-root
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### 9.4 Distroless — even smaller and safer **[A]**

Google's **distroless** images contain your language runtime and *nothing else* — no shell, no package manager, no busybox. That shrinks the image and removes whole classes of attacks (an attacker who gets in has no `sh` to run). The trade-off: you can't `docker exec … sh` to debug (use the `:debug` variants or `docker debug` when needed).

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

FROM gcr.io/distroless/nodejs20-debian12   # runtime only — no shell, no apt
WORKDIR /app
COPY --from=builder /app /app
USER 1000
EXPOSE 3000
CMD ["server.js"]                          # distroless nodejs entrypoint is already "node"
```

> **Targeting a stage:** `docker build --target builder -t myapp:dev .` builds only up to the `builder` stage — handy for a dev image or for running tests in CI against the build stage.

---

## 10. .dockerignore

A **`.dockerignore`** file (placed next to the Dockerfile) works like `.gitignore` but for the **build context** — the files Docker sends to the builder. Excluding files makes the context smaller (faster builds), prevents `COPY . .` from baking in junk (smaller images), and — critically — **stops secrets like `.env` from leaking into image layers.**

Always exclude `node_modules` (you install fresh inside the image and the host copy may be the wrong OS/arch), `.git` (huge and pointless in an image), build outputs, and any secret/config files.

```dockerignore
# Dependencies (installed fresh inside the image; host copy may be wrong-OS)
node_modules
vendor

# Build outputs
dist
build
.next
out
coverage

# Secrets & local config — NEVER ship these into an image
.env
.env.*
*.pem
*.key

# VCS & editor noise
.git
.gitignore
.vscode
.idea

# Docker & docs (not needed at build time)
Dockerfile*
compose*.yaml
docker-compose*.yml
*.md
README*
```

> **Security note:** a missing or sloppy `.dockerignore` is a leading cause of credential leaks — a `COPY . .` happily bakes your `.env` and `.pem` keys into a layer that anyone who pulls the image can extract. Treat `.dockerignore` as a security control, not just an optimization.

---

## 11. Volumes & Bind Mounts — Data Persistence

### 11.1 Why persistence needs a mechanism **[I]**

A container's writable layer is **destroyed when the container is removed.** So a database that writes to its container filesystem loses everything on `docker rm`. To keep data alive across container restarts, rebuilds, and removals, you must store it *outside* the container — that's what volumes and bind mounts do. They also let two containers share files, and let your host edit code a running container sees.

### 11.2 The three mount types — when to use each **[I]**

| Type | Syntax (`-v`) | Stored where | Use it for |
|---|---|---|---|
| **Named volume** | `-v pgdata:/var/lib/postgresql/data` | Docker-managed area on host | **Persistent app data: databases, uploads.** The default choice for data. |
| **Bind mount** | `-v "$(pwd)":/app` | An exact host path you choose | **Live source code in development** (edit on host, see it in container instantly). |
| **tmpfs** | `--tmpfs /tmp` | RAM (never hits disk) | Ephemeral scratch / sensitive temp data you don't want persisted. |

The key distinction beginners must learn: **named volume = persist data; bind mount = sync source code.** A named volume is Docker's own storage with a name you reference; a bind mount maps a literal folder on your machine.

### 11.3 Named volumes (databases) **[I]**

```bash
docker volume create pgdata        # explicitly create (also auto-created on first use)
docker volume ls                   # list volumes
docker volume inspect pgdata       # see where it lives on disk and what's attached
docker volume rm pgdata            # delete it (and its data — careful)

# Postgres with a persistent named volume — data survives `docker rm db` and rebuilds:
docker run -d --name db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \   # named volume → DB files persist
  postgres:16-alpine
```

Destroy and recreate the `db` container all you like; as long as you reattach the `pgdata` volume, the data is there.

### 11.4 Bind mounts (live dev) **[I]**

```bash
# Your host's current folder is mounted at /app; edit code on the host, the container sees it instantly.
docker run -d -p 3000:3000 \
  -v "$(pwd)":/app \                # bind mount: host folder → container /app  (Windows PowerShell: ${PWD})
  -v /app/node_modules \            # anonymous volume to PROTECT the container's node_modules…
  my-dev-image                       # …from being shadowed by the host's (empty/wrong-OS) node_modules
```

> **The `node_modules` shadowing trick:** when you bind-mount your source folder over `/app`, it *replaces* everything there — including the `node_modules` you installed during the build. Adding a second mount `-v /app/node_modules` (an anonymous volume) re-covers that subpath so the container keeps its own correctly-built modules. Without this, you'll see "module not found" errors in dev.

> **⚡ Version note (Windows):** in PowerShell use `${PWD}` (or an absolute path); `$(pwd)` is the Bash form. Bind-mount performance is best when your code lives **inside the WSL 2 filesystem** (`\\wsl$\...`) rather than under `C:\Users\...` — cross-filesystem mounts are noticeably slower. Mark read-only mounts with `:ro` (`-v "$(pwd)":/app:ro`) when the container shouldn't write back.

### 11.5 Backing up a volume **[I]**

Volumes aren't files you can just copy. Back one up by mounting it into a throwaway container and tarring it:

```bash
docker run --rm \
  -v pgdata:/data \                  # the volume to back up
  -v "$(pwd)":/backup \              # where to write the archive (host)
  alpine tar czf /backup/pgdata.tar.gz -C /data .
# Restore: docker run --rm -v pgdata:/data -v "$(pwd)":/backup alpine \
#            sh -c "cd /data && tar xzf /backup/pgdata.tar.gz"
```

---

## 12. Networking

### 12.1 The core idea — DNS by name **[I]**

Docker gives containers a virtual network so they can talk to each other and to the outside world. The headline feature: **containers on the same user-defined network can reach each other by name.** Docker runs an internal DNS server, so your API container connects to the database at hostname `db` (the container/service name), not at some IP that changes on every restart. This is the magic that makes Compose stacks "just work."

A vital rule: **`localhost` inside a container means *that container itself*, not the host and not another container.** Your API must reach the DB at `db:5432`, never `localhost:5432`. `localhost` referring to "my machine" is a host-perspective habit that breaks inside containers.

### 12.2 Network drivers **[I]**

| Driver | What it does | When to use |
|---|---|---|
| **`bridge`** | Default. A private virtual network on one host; containers get internal IPs and talk by name (on *user-defined* bridges). | The normal choice for single-host apps and Compose. |
| **`host`** | Container shares the host's network stack directly — no isolation, no port mapping needed (the app binds host ports directly). | Max network performance / when you need the host's exact networking. Linux only; less isolation. |
| **`none`** | No networking at all. | Fully isolated, network-free workloads. |
| **`overlay`** | A network spanning *multiple hosts* (Swarm/clusters). | Multi-host orchestration. |

> **⚡ Note:** the **default** `bridge` network does *not* give you name-based DNS — only **user-defined** bridge networks do. So always create your own network (or use Compose, which does it for you) rather than relying on the default bridge.

### 12.3 Hands-on **[I]**

```bash
docker network create appnet                 # create a user-defined bridge (enables name DNS)
docker network ls                            # list networks
docker network inspect appnet                # see attached containers and their IPs

docker run -d --name db   --network appnet postgres:16-alpine
docker run -d --name api  --network appnet -p 8080:3000 my-api
#                                              ▲ api can now reach the DB at hostname "db":
#                                                e.g. DATABASE_URL=postgres://user:pass@db:5432/appdb

docker network connect appnet existing_container    # attach a running container to a network
docker network disconnect appnet existing_container # detach it
```

### 12.4 Publishing ports **[I]**

Containers are isolated by default; to let the *outside world* reach a container you **publish** a port from the host to the container:

```bash
docker run -p 8080:3000 myapp          # host 8080 → container 3000 (reachable from your network)
docker run -p 127.0.0.1:8080:3000 myapp  # bind ONLY to localhost — not exposed to the LAN (more secure)
docker run -P myapp                      # publish ALL EXPOSEd ports to random high host ports
docker port myapp                        # show what got mapped
```

> **Security note:** `-p 8080:3000` binds to **all** host interfaces (`0.0.0.0`), so the port is reachable from your whole network — fine locally, risky on a server. Bind to `127.0.0.1` (and put a reverse proxy in front) for anything you don't want publicly exposed. Containers that only need to talk to *each other* (e.g. your DB) should **not** publish ports at all — they reach each other over the internal network.

---

## 13. Environment Variables & Secrets

### 13.1 Passing configuration **[I]**

The clean way to configure a container is **environment variables supplied at run time** — same image, different config per environment (dev/staging/prod). This follows the 12-factor principle of keeping config out of code.

```bash
docker run -e NODE_ENV=production -e PORT=3000 myapp   # individual vars (repeatable)
docker run --env-file .env myapp                        # load many vars from a file
```

In a Dockerfile, `ENV` bakes **non-secret** defaults into the image; `ARG` passes **build-time** parameters (recap from §7.5). The rule: config that differs per environment or is sensitive should come in at **run time**, not be baked into the image.

### 13.2 Why secrets must never go in images **[I/A]**

An image is a stack of layers that anyone with the image can unpack and read with `docker history` / by extracting layers. So **anything you `COPY` or `ENV` into an image is permanently visible** — even if a *later* instruction deletes it, the bytes remain in the earlier layer. Therefore:

- ❌ **Never** `COPY .env` into an image, hardcode a password in a Dockerfile, or put a secret in `ENV`/`ARG`.
- ✅ Add `.env`, `*.pem`, `*.key` to both `.gitignore` **and** `.dockerignore`.
- ✅ Provide secrets at **run time** via `--env-file`, Compose `secrets:`/`env_file`, or your platform's secret manager (AWS Secrets Manager, Docker/Kubernetes secrets, Vault).
- ✅ For **build-time** secrets (a private npm token, a deploy key), use **BuildKit secret mounts**, which expose the secret only to a single `RUN` and never write it to a layer.

```dockerfile
# syntax=docker/dockerfile:1
# Build-time secret mount: the file is available ONLY during this RUN and is NOT baked into any layer.
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```
```bash
# Supply the secret at build time from a local file (it never enters the image or its history):
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

### 13.3 Compose secrets **[I]**

Compose supports a first-class `secrets:` mechanism that mounts secret *files* into containers (at `/run/secrets/<name>`) instead of exposing them as easily-leaked env vars — preferred for sensitive values like DB passwords:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password   # postgres reads the password from a file
    secrets:
      - db_password
secrets:
  db_password:
    file: ./secrets/db_password.txt    # the file is mounted into the container, not put in an env var
```

---

## 14. Docker Compose In Depth

### 14.1 What Compose is and why it exists **[I]**

Running one container with a long `docker run …` line is fine. Running a *full stack* — web app + database + cache + worker, each with its own ports, env, volumes, networks, and start order — by hand is error-prone and unrepeatable. **Docker Compose** lets you declare that entire multi-container application in **one YAML file** and bring it up or down with a single command. The file is checked into your repo, so the whole environment is version-controlled and reproducible.

**Why it's a real-world superpower:** a client or new teammate clones your repo and runs **`docker compose up`** — and the entire app (frontend + backend + DB + cache), correctly networked and configured, boots with zero manual setup. That handoff experience is hard to beat.

### 14.2 The structure of a Compose file **[I]**

A Compose file describes:

- **`services:`** — each container in your app (a web server, a database, etc.). This is the bulk of the file.
- **`volumes:`** — top-level named volumes shared/persisted across services.
- **`networks:`** — custom networks (Compose makes a default one automatically).
- **`secrets:` / `configs:`** — files mounted into containers.

By default Compose creates **one network** for the project and attaches every service to it, so services reach each other **by service name** out of the box (no manual network wiring). It also prefixes resource names with the project name (the folder name by default).

```yaml
# compose.yaml  — the modern filename (docker-compose.yml still works). No "version:" key needed anymore.
services:
  web:
    build: .                    # build the image from ./Dockerfile (vs `image:` to pull a prebuilt one)
    ports:
      - "3000:3000"             # publish host:container
    environment:
      - NODE_ENV=production
    depends_on:
      - db                       # start db before web (start order only — see §15 for real readiness)

  db:
    image: postgres:16-alpine    # pull a prebuilt image
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data   # persist DB data in a named volume

volumes:
  pgdata:                        # declare the named volume used above
```

```bash
docker compose up -d    # build (if needed) + create network + start everything in the background
```

### 14.3 `build` vs `image` **[I]**

- **`build:`** tells Compose to *build* the image from a Dockerfile (your own app). Can be a string (`build: .`) or an object with `context`, `dockerfile`, `target`, and `args`.
- **`image:`** tells Compose to *use* an existing image (pull it) — for off-the-shelf services like Postgres, Redis, nginx.
- You can use **both**: `build:` to build *and* `image:` to name/tag the result.

### 14.4 `depends_on` and the readiness trap **[I]**

`depends_on` controls **start order** — Compose starts `db` before `web`. But "started" ≠ "ready": a Postgres container can be running while the database is still initializing and not yet accepting connections, so your app crashes on first connect. The fix is to combine `depends_on` with a **healthcheck** using `condition: service_healthy`, so Compose waits until the dependency is actually *healthy* before starting the dependent service. This is one of the most important Compose patterns and is shown fully in §17.

---

## 15. The compose.yaml Reference (every key)

A heavily-annotated, near-complete service definition. Treat this as a lookup — you'll never use all of it at once.

```yaml
services:
  app:
    # ───── Build from a Dockerfile, or use a prebuilt image ─────
    build:
      context: .                 # folder sent to the builder (where the Dockerfile + code live)
      dockerfile: Dockerfile     # custom Dockerfile name/path (optional)
      target: runner             # which multi-stage target to build (optional)
      args:                      # build-time ARGs passed to the Dockerfile
        APP_VERSION: "1.4.2"
    image: myapp:1.4.2           # name the built image (or, with no build:, the image to pull)

    # ───── Identity & lifecycle ─────
    container_name: myapp        # fixed container name (omit to let Compose auto-name; needed for scaling)
    restart: unless-stopped      # no | always | on-failure | unless-stopped  (resilience policy)
    command: ["node", "dist/server.js"]   # override the image's CMD
    entrypoint: ["/entrypoint.sh"]        # override the image's ENTRYPOINT
    user: "1000:1000"            # run as this UID:GID (security)
    init: true                   # inject a tiny init (tini) for signal handling / zombie reaping
    stop_grace_period: 30s       # how long to wait after SIGTERM before SIGKILL

    # ───── Ports ─────
    ports:
      - "3000:3000"              # host:container (published to all interfaces)
      - "127.0.0.1:9229:9229"    # bind to localhost only (e.g. a debugger port — not LAN-exposed)
    expose:
      - "3000"                   # documents the port; reachable to other services, NOT published to host

    # ───── Environment / secrets ─────
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://app:secret@db:5432/appdb   # note: hostname "db" = the db service
      REDIS_URL: redis://cache:6379
    env_file:
      - .env                     # load additional vars from a file
    secrets:
      - db_password              # mounts the secret at /run/secrets/db_password

    # ───── Dependencies & ordering ─────
    depends_on:
      db:
        condition: service_healthy   # WAIT until db's healthcheck passes (true readiness)
      cache:
        condition: service_started   # just wait for the cache to start

    # ───── Storage ─────
    volumes:
      - ./src:/app/src                 # bind mount: live code in dev
      - app_node_modules:/app/node_modules   # named volume so host doesn't shadow container's modules
      - appdata:/app/data:rw           # named volume, read-write (use :ro for read-only)

    # ───── Networking ─────
    networks:
      - frontend
      - backend

    # ───── Health ─────
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s          # grace window during boot where failures don't count

    # ───── Resource limits / logging / extras ─────
    deploy:
      resources:
        limits:                  # hard caps (cgroups)
          cpus: "0.50"
          memory: 512M
        reservations:            # soft guarantee
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"          # cap log file size so disk doesn't fill up
        max-file: "3"
    cap_drop: [ALL]              # drop all Linux capabilities (least privilege)
    cap_add: [NET_BIND_SERVICE]  # then add back only what's needed
    read_only: true              # make the container filesystem read-only (security)
    tmpfs: ["/tmp"]              # writable in-RAM temp dir for a read-only container
    profiles: ["dev"]           # only start this service when the "dev" profile is enabled

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]   # CMD-SHELL runs via the shell (allows piping/vars)
      interval: 10s
      timeout: 5s
      retries: 5

# ───── Top-level declarations ─────
volumes:
  pgdata:
  app_node_modules:
  appdata:

networks:
  frontend:
  backend:
    driver: bridge

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Key fields cheat-sheet

| Field | Purpose |
|---|---|
| `services` | Each container in the app. |
| `build` / `image` | Build from a Dockerfile, or pull a prebuilt image. |
| `ports` / `expose` | Publish to host (`host:container`) / make reachable to other services only. |
| `environment` / `env_file` / `secrets` | Inline vars / vars from a file / mounted secret files. |
| `depends_on` | Start order, plus (with `condition`) wait for health. |
| `volumes` | Named volumes (persist data) and bind mounts (dev code). |
| `networks` | Which networks the service joins. |
| `restart` | `no` / `always` / `unless-stopped` / `on-failure`. |
| `healthcheck` | How Docker decides the service is healthy/ready. |
| `command` / `entrypoint` | Override the image's default CMD / ENTRYPOINT. |
| `deploy.resources` | CPU/memory limits and reservations. |
| `profiles` | Conditionally include a service (e.g. dev-only tools). |
| `logging` | Log driver + rotation options (cap disk usage). |

> **Profiles** let one file serve multiple scenarios: tag dev-only tools (a db admin UI, a mailcatcher) with `profiles: ["dev"]`; they stay off during a normal `docker compose up` and only start with `docker compose --profile dev up`. Keeps one source of truth for dev and prod.

---

## 16. Compose CLI Commands

Every command operates on the whole project (all services) unless you name specific services.

```bash
docker compose up                 # build if needed + create network + start; ATTACHED (logs stream here)
docker compose up -d              # detached (background) — the usual dev start
docker compose up --build         # force a rebuild of images before starting (after Dockerfile/dep changes)
docker compose up -d db cache     # start only specific services

docker compose down               # stop & remove containers + the default network (volumes KEPT)
docker compose down -v            # ALSO remove named volumes — WIPES DB DATA (never in production!)
docker compose down --rmi local   # also remove images built by this project

docker compose ps                 # list this project's services and their status/ports
docker compose logs               # all services' logs, interleaved
docker compose logs -f web        # follow (stream) one service's logs
docker compose exec web sh        # open a shell in the RUNNING web service
docker compose run --rm web npm test   # one-off command in a NEW throwaway container (e.g. tests)
docker compose build              # build images only (don't start)
docker compose pull               # pull the latest of all `image:` services
docker compose restart web        # restart a service
docker compose stop / start       # stop/start without removing
docker compose config             # render & validate the fully-resolved config (great for debugging YAML)
docker compose up -d --scale worker=3   # run 3 replicas of the "worker" service
```

**The daily dev loop:** `docker compose up -d` → work → `docker compose logs -f` to watch → `docker compose down` when done. Run `up --build` whenever you change a Dockerfile or dependencies, and `docker compose config` whenever the YAML misbehaves (it shows you exactly what Compose resolved, including merged override files and interpolated variables).

> **`run` vs `exec`:** `exec` runs a command in an *already-running* container; `run` spins up a *new* one-off container (handy for tasks, migrations, tests). Add `--rm` to `run` so it cleans up after itself.

> **Override files:** `compose.override.yaml` is auto-merged on top of `compose.yaml`, letting you keep prod settings in the base file and local dev tweaks (bind mounts, debug ports) in the override — no flags needed.

---

## 17. A Real Multi-Service Stack: web + db + redis

This is the payoff: a complete, production-shaped stack — a Node/Next-style **web** service, a **Postgres** database, and a **Redis** cache — wired together with proper healthchecks, ordering, persistence, networking, and secrets. It demonstrates every concept from the preceding sections in one file.

```yaml
# compose.yaml — full stack: web (built from Dockerfile) + Postgres + Redis
services:
  web:
    build:
      context: .
      target: runner            # the small multi-stage runtime target from §9.3
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      # Reach other services BY NAME over the internal network — never localhost:
      DATABASE_URL: postgres://app:secret@db:5432/appdb
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy   # wait until Postgres actually accepts connections
      cache:
        condition: service_healthy   # wait until Redis answers PING
    restart: unless-stopped
    networks: [backend]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "0.75"
          memory: 512M

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret    # in real prod, use a secret file (see §13.3) instead of inline
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume → data survives `down`, rebuilds, restarts
    networks: [backend]
    restart: unless-stopped
    # Healthcheck so `depends_on: condition: service_healthy` works. pg_isready returns 0 when accepting conns.
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    # NOTE: no `ports:` — the DB is only reachable INSIDE the network (web reaches it at db:5432).
    # Publishing 5432 to the host would needlessly expose your database. Add it ONLY for local debugging.

  cache:
    image: redis:7-alpine
    command: ["redis-server", "--save", "60", "1", "--loglevel", "warning"]  # periodic snapshot persistence
    volumes:
      - redisdata:/data
    networks: [backend]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]   # returns PONG when ready
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
  redisdata:

networks:
  backend:
    driver: bridge
```

```bash
docker compose up -d        # boot the whole stack; web waits for db & cache to be HEALTHY first
docker compose logs -f web  # watch the app
docker compose ps           # confirm all three are "healthy"
docker compose down         # tear down (KEEPS the volumes → your data is safe)
```

What this stack demonstrates, mapped to earlier sections: multi-stage runtime image (§9), name-based service DNS (§12), `depends_on` + healthchecks for *true* readiness (§14, §15), named volumes for persistence (§11), not publishing the DB port for safety (§12/§19), resource limits and restart policies (§15), and one-command startup (§16).

---

## 18. Image Size & Build Optimization

Smaller, faster-building images cost less, deploy quicker, and present a smaller attack surface. Here's the full toolkit, with the reasoning behind each technique.

| Technique | Why it helps |
|---|---|
| **Small base image** (`-alpine`, `-slim`, distroless, `scratch`) | `node:20-alpine` ≈ 50MB vs `node:20` ≈ 1GB. Fewer packages = smaller + fewer CVEs. |
| **Multi-stage builds** | Ship only runtime artifacts; drop compilers, dev deps, and caches (§9). |
| **Order layers by change frequency** | Copy `package.json` + install *before* source so the install layer caches (§8). |
| **`npm ci` over `npm install`** | Faster, deterministic from the lockfile; CI-friendly. Add `--omit=dev` for prod. |
| **Combine `RUN`s and clean in the same layer** | A delete in a later layer doesn't reclaim space (§8.3). |
| **`.dockerignore`** | Smaller build context → faster builds, no accidental junk/secrets in the image (§10). |
| **BuildKit cache mounts** | `RUN --mount=type=cache,target=/root/.npm npm ci` reuses the package cache across builds. |
| **Pin versions** | `node:20.11-alpine`, not `node:latest` — reproducible builds, no surprise breakage. |
| **Strip/minify artifacts** | e.g. Go `-ldflags="-s -w"`, tree-shaken JS bundles, remove source maps in prod. |

### 18.1 Measuring and diagnosing **[I]**

```bash
docker images                    # see image sizes at a glance
docker history myapp:1.0         # see which LAYER (and thus which instruction) is bloating the image
docker scout quickview myapp:1.0 # quick size + vulnerability overview (built-in)
# `dive` (3rd-party) is the best interactive layer explorer for hunting wasted space.
```

### 18.2 The layer-caching golden pattern (recap) **[I]**

```dockerfile
COPY package*.json ./    # ① rarely changes
RUN npm ci --omit=dev    # ② cached unless ① changed (the expensive step)
COPY . .                 # ③ changes constantly — only this layer rebuilds on a code edit
```

### 18.3 BuildKit cache mounts for lightning rebuilds **[I/A]**

```dockerfile
# syntax=docker/dockerfile:1
# The npm cache directory persists ACROSS builds (it's a cache mount, not a layer), so even a cold
# `npm ci` after a lockfile change re-downloads far less. Same idea exists for pip, apt, go, cargo, etc.
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev
```

---

## 19. Security Recommendations

Containers are *not* a security boundary as strong as a VM (they share the host kernel), so defense-in-depth matters. Each recommendation below explains the threat it addresses.

### 19.1 Run as a non-root user **[I/A]**

By default a container's process runs as **root** (UID 0). If an attacker exploits your app, or a flaw lets them escape the container, they have root — and a container root can sometimes be leveraged toward host root. Create and switch to an unprivileged user, and your blast radius shrinks dramatically.

```dockerfile
RUN addgroup -S app && adduser -S app -G app   # alpine
USER app                                         # everything after runs unprivileged
```
Verify: `docker exec <c> whoami` should not say `root`.

### 19.2 Use minimal / pinned / trusted base images **[I/A]**

Every package in your base is potential attack surface and a potential CVE. Prefer `-alpine`, `-slim`, **distroless**, or `scratch` (§9). **Pin** versions (and ideally digests) for reproducibility, and pull only **official** or verified-publisher images. Rebuild regularly so you pick up upstream security patches — an image you built a year ago is frozen in time with year-old vulnerabilities.

### 19.3 Keep secrets out of images and layers **[I/A]**

As covered in §13: never `COPY .env`, hardcode passwords, or put secrets in `ENV`/`ARG` — they're recoverable from image layers forever. Use runtime env/secret files and BuildKit `--mount=type=secret` for build-time creds. Make sure `.env`/keys are in `.dockerignore`.

### 19.4 Scan images for vulnerabilities **[I]**

Scanning compares your image's packages against CVE databases so you fix known holes before shipping. Wire it into CI.

```bash
docker scout cves myapp:1.0        # list known CVEs in the image (Docker's built-in scanner)
docker scout quickview myapp:1.0   # summary + base-image upgrade recommendations
# Alternatives: `trivy image myapp:1.0`, Snyk, Grype.
```

### 19.5 Least privilege at run time **[I/A]**

Tighten what a running container can do:

```bash
docker run --read-only \              # filesystem is read-only (attacker can't write a payload)…
  --tmpfs /tmp \                       # …with a writable in-RAM /tmp for legit temp files
  --cap-drop ALL \                     # drop ALL Linux capabilities…
  --cap-add NET_BIND_SERVICE \         # …then add back only what's needed (e.g. bind port <1024)
  --security-opt no-new-privileges \   # process can never gain MORE privileges (blocks setuid escalation)
  --memory 512m --cpus 0.5 \           # resource caps so a runaway/DoS can't take down the host
  --pids-limit 200 \                   # cap process count (fork-bomb protection)
  myapp:1.0
```

(The Compose equivalents are `read_only:`, `tmpfs:`, `cap_drop`/`cap_add`, `security_opt`, and `deploy.resources` — see §15.)

### 19.6 Don't mount the Docker socket; isolate networks **[A]**

Mounting `/var/run/docker.sock` into a container effectively gives that container **root on the host** (it can create privileged containers). Avoid it unless absolutely required, and never on an internet-facing service. Also: don't publish ports you don't need (an internal DB needs no `-p`), bind admin/debug ports to `127.0.0.1`, and put a reverse proxy (nginx/Caddy/Traefik) in front for TLS.

### 19.7 Add a `HEALTHCHECK`; use trusted registries **[I]**

Healthchecks (§7.9) let orchestrators replace sick containers automatically. Prefer official/verified images, and consider enabling content trust / signature verification for supply-chain integrity.

### 19.8 Security checklist **[I/A]**

- ✅ Non-root `USER`; `--security-opt no-new-privileges`.
- ✅ Minimal, pinned, regularly-rebuilt base (alpine/distroless/scratch).
- ✅ No secrets in layers; secrets at runtime / via BuildKit secret mounts; `.dockerignore` covers `.env`/keys.
- ✅ Scan in CI (`docker scout cves` / Trivy) and fail the build on criticals.
- ✅ `--read-only` + `--tmpfs`, `--cap-drop ALL` (+ minimal `--cap-add`), resource & PID limits.
- ✅ Don't expose the Docker socket; don't publish unneeded ports; bind admin ports to localhost.
- ✅ `HEALTHCHECK` defined; reverse proxy for TLS; official/verified images only.

---

## 20. Debugging Containers

Most container problems are diagnosed with a tight loop of **logs → inspect → exec**. Master these and you'll solve 90% of issues fast.

### 20.1 The core debugging commands **[I]**

```bash
docker logs -f <container>          # FIRST move: what is the app actually printing/erroring?
docker logs --tail 200 <container>  # just the recent lines
docker exec -it <container> sh      # SECOND move: get inside and look around (env, files, connectivity)
docker inspect <container>          # full config truth: env vars, mounts, network, IP, restart, exit code
docker stats                        # live CPU/mem/net/io — is it OOM-killed or pegged?
docker compose logs -f              # all services together (correlate web errors with db logs)
docker events                       # real-time daemon events (creates, dies, OOM kills)
docker top <container>              # processes running inside
docker diff <container>             # what files changed in the writable layer
docker inspect <c> --format '{{.State.ExitCode}}'   # why did it exit?
```

> **Debugging a container that won't start / exits instantly:** you can't `exec` into it (it's not running). Override the entrypoint to get a shell instead: `docker run -it --entrypoint sh myimage` — now poke around the image's filesystem to see what's wrong. For distroless images (no shell), use `docker debug` or copy in a debug tool.

### 20.2 Common symptoms and fixes **[I]**

| Symptom | Likely cause & fix |
|---|---|
| `port is already allocated` | Another process/container uses that host port. Change the host side of `-p`, or stop the other. |
| Container exits immediately | The main process ended or crashed. Check `docker logs`. CMD must run a **foreground**, long-lived process — not a background daemon. |
| Code changes don't appear | Dev: missing/incorrect **bind mount**. Prod: you need to **rebuild** (`up --build`) — the code is baked in. |
| `Cannot connect to the Docker daemon` | Docker Desktop/Engine isn't running, or your user isn't in the `docker` group. |
| App can't reach the DB | Using `localhost` instead of the **service name** (`db`), or the DB wasn't *ready* (add a healthcheck + `condition: service_healthy`). |
| DB data vanished after `down` | You ran `down -v`, or never used a **named volume**. |
| Image is huge | No multi-stage; big base; copied `node_modules`/`.git` (fix `.dockerignore`). |
| Build is slow every time | Bad layer order — copy `package.json` + install **before** source (§8). |
| `permission denied` on a mounted file | Non-root container UID ≠ host file owner. Align UIDs, fix ownership, or `--chown`/`-u`. |
| Container killed with code 137 | **OOM-killed** — it hit the memory limit. Raise `--memory` or fix a leak. |
| `docker stop` takes 10s | App isn't handling SIGTERM (often shell-form CMD). Use exec-form CMD and/or `--init`. |

> **Golden move:** `docker logs <name>` then `docker exec -it <name> sh`. Inside, check env (`env`), test connectivity (`wget -qO- http://db:5432` style probes), and look at config files. The truth is almost always visible from inside the container.

---

## 21. Registries, Deployment & Orchestration

### 21.1 Pushing and pulling images **[I]**

```bash
docker login                                   # authenticate to Docker Hub (docker login ghcr.io for GHCR)
docker tag myapp:1.0 yourname/myapp:1.0        # tag for the destination repo
docker push yourname/myapp:1.0                 # upload
docker pull yourname/myapp:1.0                 # download on the server / CI / a teammate's machine

# GitHub Container Registry:
docker tag myapp:1.0 ghcr.io/yourname/myapp:1.0
docker push ghcr.io/yourname/myapp:1.0
```

### 21.2 Building & pushing in CI (GitHub Actions) **[I/A]**

CI should build your image on every push to main, scan it, and push it to a registry — so deploys just pull a known-good image. A minimal pipeline:

```yaml
# .github/workflows/docker.yml
name: build-and-push
on:
  push:
    branches: [main]
jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write            # allow pushing to GHCR
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3        # enable BuildKit/buildx (cache, multi-platform)
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}    # auto-provided; no manual secret needed for GHCR
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha                      # reuse build cache across CI runs (fast)
          cache-to: type=gha,mode=max
```

### 21.3 Deployment options **[I]**

| Target | How it works | Best for |
|---|---|---|
| **VPS** (DigitalOcean, Hetzner, Linode) | Install Docker, copy `compose.yaml` + `.env`, `docker compose pull && up -d`. | Cheapest, full control; great for small client apps. |
| **PaaS** (Render, Railway, Fly.io) | Connect the repo; they build your Dockerfile and host it. | Easiest; minimal ops. |
| **Managed containers** (AWS **ECS**/Fargate, Google **Cloud Run**, Azure Container Apps) | Push image to their registry; run as a managed service that scales. | Production scale without managing servers. |
| **Kubernetes** (EKS/GKE/AKS or self-hosted) | Orchestrate many containers across many nodes (see §21.5). | Large, multi-service, high-availability systems. |

### 21.4 A typical small-prod deploy on a VPS **[I]**

```bash
# On the server (once): install Docker + the Compose plugin, add a non-root user to the docker group.
# Then, per deploy:
scp compose.yaml .env user@server:/srv/app/    # ship the compose file + runtime env (NOT in the image)
ssh user@server '
  cd /srv/app &&
  docker compose pull &&        # fetch the new image you pushed from CI
  docker compose up -d &&       # recreate changed services with zero/near-zero downtime
  docker image prune -f         # clean up old images
'
# Put Caddy or Traefik in front for automatic HTTPS, and set restart: unless-stopped so it survives reboots.
```

### 21.5 Orchestration overview — when Compose isn't enough **[A]**

Compose runs containers on **one host**. When you need to run across **many machines**, with automatic scaling, rolling updates, self-healing, load balancing, and service discovery at scale, you graduate to an **orchestrator**:

- **Docker Swarm** — Docker's built-in clustering. A near-drop-in step up from Compose (it reuses much of the same file format via `docker stack deploy`). Simple, but largely superseded.
- **Kubernetes (k8s)** — the industry-standard orchestrator. It introduces its own objects — **Pods** (one or more containers scheduled together), **Deployments** (declarative desired-state for replicas + rolling updates), **Services** (stable networking/load-balancing), **Ingress** (HTTP routing), **ConfigMaps/Secrets** (config), and **PersistentVolumes** (storage). You describe the *desired state* in YAML and k8s continuously reconciles reality toward it (restarting crashed pods, rescheduling on node failure, scaling on load).

The good news: the *image* you build here is exactly what k8s runs — your Dockerfile skills transfer directly. Kubernetes is a large topic deserving its own guide; for now, know that the path is **single container → Compose (one host) → Kubernetes (many hosts)**, and pick the lowest tier that meets your needs. Most freelance/client apps never need more than a VPS + Compose, or a PaaS.

---

## 22. Tips, Tricks & Gotchas

**Do:**
- ✅ Start every project with a `.dockerignore` (security + speed).
- ✅ Copy `package.json` + install **before** copying source (layer caching, §8).
- ✅ Use **multi-stage builds** + a minimal base for tiny, secure production images (§9).
- ✅ Use **named volumes** for databases, **bind mounts** for dev source code (§11).
- ✅ Reference other services **by name** over the Compose network, never `localhost` (§12).
- ✅ Pair `depends_on` with **healthchecks** + `condition: service_healthy` for real readiness (§14).
- ✅ Add `restart: unless-stopped` and resource limits for production reliability (§15).
- ✅ Pin image versions (`node:20.11-alpine`), never `latest` (§3).
- ✅ Run as a **non-root** user; drop capabilities; scan images (§19).
- ✅ Use the **exec form** of `CMD`/`ENTRYPOINT` so signals reach your app (§7.7).

**Don't / common bugs:**
- ❌ Storing important data only in a container's writable layer — it vanishes on `rm` (§11).
- ❌ `COPY .env` or hardcoding secrets — they're recoverable from image layers forever (§13).
- ❌ Relying on `latest` — non-reproducible, surprise breakage (§3).
- ❌ Running a background daemon as `CMD` — the container exits when the foreground process ends (§20).
- ❌ Forgetting `-d` and thinking your terminal "froze" (it's attached to logs) (§4).
- ❌ `docker compose down -v` in production — it deletes volumes (data loss) (§16).
- ❌ Reversing the port order — it's `-p host:container` (§4).
- ❌ Expecting `depends_on` alone to wait for the DB to be *ready* (it only waits for *start*) (§14).
- ❌ Publishing your database port to the host when only other containers need it (§12/§19).
- ❌ Mounting `/var/run/docker.sock` casually — it's root-on-host (§19).

**Tricks:**
- 🔹 `docker compose up --build -d` after any Dockerfile/dependency change.
- 🔹 `docker compose run --rm app npm install <pkg>` to add a package using the container's exact environment.
- 🔹 `docker run -it --entrypoint sh <image>` to debug an image whose container won't start.
- 🔹 `docker system prune -a --volumes` to reclaim gigabytes (knowing it deletes unused volumes).
- 🔹 `docker compose config` to debug a misbehaving YAML (shows the fully-resolved, merged config).
- 🔹 `compose.override.yaml` auto-merges for local dev tweaks — keep prod in the base file.
- 🔹 `docker init` to scaffold a sensible Dockerfile + `.dockerignore` + `compose.yaml` for your stack.
- 🔹 BuildKit cache mounts (`RUN --mount=type=cache,...`) for near-instant dependency rebuilds.
- 🔹 `docker history <image>` / `dive` to hunt down what's bloating an image.

---

## 23. Study Path & Build-to-Learn Projects

Learn in this order for the fastest path to offline mastery:

1. **Concepts** — containers vs VMs (kernel/namespaces/cgroups), image vs container vs Dockerfile (§1, §3).
2. **Run existing images** — `docker run` and its flags, `ps`, `logs`, `exec`, `stop`, `rm` (§4, §5).
3. **Write a Dockerfile** — for a simple Node/Express app; learn every instruction (§6, §7).
4. **Make builds fast** — layer caching and ordering, `.dockerignore` (§8, §10).
5. **Persist & connect** — volumes for a database, a user-defined network so two containers talk by name (§11, §12).
6. **Compose a stack** — combine app + DB + cache in one file; healthchecks + readiness; one `docker compose up` (§14–§17).
7. **Shrink & secure** — multi-stage builds, distroless/scratch, non-root, scanning, least privilege (§9, §18, §19).
8. **Deploy** — push to a registry, build/push in CI, deploy to a VPS or PaaS; understand where k8s fits (§21).

### Build to learn (do these 3 projects)

1. **Dockerize an Express API** → one Dockerfile (non-root, pinned alpine base, exec-form CMD), build it, run it, hit an endpoint. Add a `.dockerignore` and a `HEALTHCHECK`.
2. **Compose a full-stack app** → web + Postgres + Redis in one `compose.yaml`, with healthchecks, `condition: service_healthy`, named volumes, and name-based DNS — booted with a single `docker compose up`. (You essentially built this in §17.)
3. **Production multi-stage build** → ship the app as a sub-200MB image (multi-stage + distroless/standalone), scan it with `docker scout cves`, push it to GHCR from CI, and deploy it to a VPS (Compose) or a PaaS (Render/Railway/Fly.io) behind HTTPS.

> Those three cover ~90% of what real projects ever need: containerizing an app, running a full stack locally, and deploying a lean, secure production image. Keep `docker ps`, `docker logs`, `docker exec`, and `docker compose up/down` in muscle memory — they're 80% of daily Docker use.
