# Nginx — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have heard of Nginx" to "I can run it as a hardened static web server, a reverse proxy in front of a Node/Go app, a load balancer across many backends, an API gateway with rate limiting and auth, and a TLS-terminating edge with HTTP/2 and HTTP/3 — and debug all of it" — entirely offline. Every concept is explained in **prose first** (what it is, *why* it exists, when and how to use it, the key directives and their parameters, best practices, and **security** notes), then demonstrated with **heavily-commented, runnable configuration**. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Nginx 1.29+** — both the **stable** branch (even minor numbers, e.g. 1.28.x) and the **mainline** branch (odd minor numbers, e.g. 1.29.x), which is what you should usually run in production despite the name. Modern features used throughout:
> - **HTTP/3 over QUIC** is in mainline since **1.25.0** (compiled in with `--with-http_v3_module`) — TLS 1.3-only, runs over UDP/443.
> - **HTTP/2** is configured with the modern `http2 on;` directive (per-server) rather than the old `listen ... http2` parameter (deprecated in 1.25.1).
> - The `ssl_protocols`, `ssl_conf_command`, and OCSP-stapling story is mature; **TLS 1.3** is the default-recommended floor in 2026.
> - **Nginx the company is now part of F5** (acquired 2019); **Nginx Plus** is F5's paid commercial edition (adds active health checks, a live dashboard, the JavaScript module conveniences, dynamic upstream reconfiguration via API, session persistence, etc.). Everything in this guide works on **open-source Nginx** unless a feature is explicitly flagged **(Plus only)**.
> - **OpenResty** is a popular distribution that bundles Nginx + LuaJIT + a rich set of Lua modules (`lua-nginx-module`), turning Nginx into a programmable application server — great for complex API-gateway logic. **Nginx Unit** is a separate F5 project: a polyglot *application server* (runs PHP/Python/Go/Node/etc. processes directly) with a JSON REST control API — not the same product as Nginx the web server. Both are noted where relevant.
>
> Where behaviour is version-sensitive it is flagged **⚡ Version note**. The author is on **Windows 11**; Nginx *does* run on Windows but the Windows build is a compatibility port (single worker, no `sendfile`, lower performance) — for real work you run Nginx on **Linux**, most often inside **Docker** (cross-reference the **Docker guide**). Cross-references to the **Docker**, **Go/Gin & net/http**, **Node/Fastify**, and **FTP** guides appear where relevant. Always confirm exact directive syntax in the official docs (`nginx.org/en/docs/`) or via `man nginx`.

---

## Table of Contents

1. [What Nginx Is & Why — The Event-Driven Architecture](#1-what-nginx-is--why--the-event-driven-architecture) **[B]**
2. [Installation, First Run & Docker](#2-installation-first-run--docker) **[B]**
3. [The Configuration Model: Master/Worker, Contexts, Directives](#3-the-configuration-model-masterworker-contexts-directives) **[B/I]**
4. [Static Web Server: root, alias, index, try_files, MIME, logs](#4-static-web-server-root-alias-index-try_files-mime-logs) **[B/I]**
5. [`location` Matching In Depth](#5-location-matching-in-depth) **[I]**
6. [Reverse Proxy](#6-reverse-proxy) **[I]**
7. [Load Balancing](#7-load-balancing) **[I/A]**
8. [API Gateway Patterns](#8-api-gateway-patterns) **[A]**
9. [TLS / SSL, HTTP/2 & HTTP/3](#9-tls--ssl-http2--http3) **[I/A]**
10. [Caching & Compression](#10-caching--compression) **[I/A]**
11. [Security Hardening](#11-security-hardening) **[I/A]**
12. [Performance Tuning](#12-performance-tuning) **[A]**
13. [Real-World Configs: SPA + API, Full Edge, Docker, k8s, Observability](#13-real-world-configs-spa--api-full-edge-docker-k8s-observability) **[A]**
14. [Gotchas & Best Practices](#14-gotchas--best-practices) **[I/A]**
15. [Study Path & Build-to-Learn Projects](#15-study-path--build-to-learn-projects)

---

## 1. What Nginx Is & Why — The Event-Driven Architecture

### 1.1 What Nginx is **[B]**

**Nginx** (pronounced "engine-x") is a high-performance **web server** that has grown into the Swiss-army knife of the web edge. In one binary it is, depending on how you configure it:

- a **static web server** that serves HTML/CSS/JS/images directly off disk, extremely fast;
- a **reverse proxy** that sits in front of one or more application servers (your Node, Go, Python, Java, PHP apps) and forwards requests to them;
- a **load balancer** that spreads traffic across many identical backend instances;
- an **API gateway** that routes, rate-limits, authenticates, and reshapes API traffic;
- a **TLS termination point** that handles HTTPS, HTTP/2, and HTTP/3 so your backends don't have to;
- a **cache** that stores backend responses and serves them without re-hitting the backend;
- a **mail proxy** (IMAP/POP3/SMTP) and a generic **TCP/UDP proxy** (the `stream` module).

The mental model that unlocks everything: **Nginx is an event-driven traffic handler that sits at the edge of your system and decides, per request, what to do** — serve a file, proxy upstream, redirect, rate-limit, reject, or cache. Almost every production web system on Earth has *something* like Nginx at its front door (Nginx itself, or its cousins Envoy, HAProxy, Caddy, or cloud load balancers). Learning Nginx teaches you how the web edge actually works.

### 1.2 The "why" — the C10k problem and the architecture that solved it **[B]**

To understand *why* Nginx exists and why it scales, you need the historical problem it was built to solve: **C10k — handling ten thousand concurrent connections on one machine.**

The old guard (notably **Apache HTTP Server** in its classic `prefork`/`worker` modes) used a **process-per-connection** or **thread-per-connection** model. Every incoming connection got its own OS process or thread. This is simple to reason about — your request handler runs top-to-bottom, blocking on I/O (reading the socket, reading a file, waiting for a database) — but it does not scale, for two reasons:

1. **Memory.** Each process/thread carries its own stack and bookkeeping — often megabytes. Ten thousand connections × a few MB = tens of gigabytes of RAM just for connection state, most of it idle.
2. **Context-switching.** The OS scheduler must constantly switch the CPU between thousands of threads. Above a few thousand, the machine spends more time *switching* than *working*. Throughput collapses.

The killer detail: most web connections are **mostly idle**, waiting on the network (a slow mobile client uploading a form, a `keep-alive` connection sitting between requests). A thread-per-connection server *parks an entire expensive thread* on each of those idle waits. That is enormously wasteful.

**Nginx (first released 2004 by Igor Sysoev) attacks this with an event-driven, asynchronous, non-blocking architecture.** Instead of one thread per connection, Nginx runs a **small fixed number of worker processes** (typically one per CPU core), and **each worker handles thousands of connections at once** inside a single thread, using an **event loop**.

Here is the logic of an event loop, in plain prose. A worker keeps a big list of connections it is responsible for. It asks the OS kernel — via an efficient **event-notification mechanism** (`epoll` on Linux, `kqueue` on BSD/macOS, `IOCP` on Windows) — *"which of these thousands of sockets is ready to do work right now?"* The kernel hands back only the handful that are ready (data arrived, a socket became writable, a timer fired). The worker does the small ready chunk of work for each one — read the bytes that arrived, write the bytes that can be sent — **without ever blocking**. If a socket isn't ready, the worker simply doesn't touch it; it moves on. Then it asks the kernel again. Round and round, thousands of times a second.

```
   APACHE (classic prefork)                 NGINX (event-driven)
   ┌─────────────────────────┐              ┌──────────────────────────────────┐
   │ conn 1  → process/thread │              │  worker 1 (1 thread, 1 CPU core) │
   │ conn 2  → process/thread │              │   event loop ── epoll() ──┐      │
   │ conn 3  → process/thread │              │     handles conn 1,2,3…N   │      │
   │ …                        │              │     (thousands) all at once│      │
   │ conn N  → process/thread │              │  worker 2 (another core) … │      │
   └─────────────────────────┘              └──────────────────────────────────┘
   N connections = N heavy threads          N connections = a few light workers
   RAM + context-switch bound               CPU-bound, tiny RAM per connection
```

Because an idle connection in Nginx is just an entry in a list (a few kilobytes of state), not a parked thread, **one Nginx worker can hold tens of thousands of idle keep-alive connections at almost no cost.** This is why Nginx became the default front-end of the high-traffic web: it does *more* with *less* hardware, and its memory use is flat and predictable under load.

> **The trade-off you must respect:** because each worker is a single thread serving thousands of connections, **a worker must never block.** If one request handler does a slow, blocking operation (a synchronous disk read on a slow disk, a long CPU computation), it freezes *every other connection that worker is handling*. Nginx avoids this internally (it offloads blocking disk I/O to a thread pool via `aio threads`, and it proxies slow application logic to *separate* backend processes rather than running it in-worker). The corollary: **Nginx is brilliant at I/O-bound traffic shuffling and terrible as a place to run your application code.** Your business logic belongs in a backend app (Node, Go, Python…) that Nginx *proxies to* — see §6.

### 1.3 Where Nginx fits in a system **[B]**

A typical modern web system layers like this, and Nginx is the **edge** layer:

```
   Internet
      │  HTTPS (TLS 1.3, HTTP/2 / HTTP/3)
      ▼
   ┌──────────────────────────────────────────────┐
   │  NGINX (the edge / reverse proxy)             │
   │   • terminates TLS                            │
   │   • serves static assets directly             │
   │   • rate-limits, blocks bad IPs, adds headers │
   │   • load-balances across app instances        │
   │   • caches responses                          │
   └──────────────────────────────────────────────┘
      │ plain HTTP over the private network (fast, local)
      ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │ Node/Fastify │  │ Go/Gin app   │  │ Python app   │   ← your application servers
   └──────────────┘  └──────────────┘  └──────────────┘
      │
      ▼
   ┌──────────────┐  ┌──────────────┐
   │ PostgreSQL   │  │ Redis        │   ← data layer (see POSTGRESQL_GUIDE / REDIS_GUIDE)
   └──────────────┘  └──────────────┘
```

Putting Nginx in front buys you: one place to do TLS, one place to do rate limiting and security headers, the ability to run multiple app instances behind one address, the ability to serve static files without waking the app, and a buffer that absorbs slow clients so your app threads aren't tied up. (Cross-reference: the **Docker guide** shows this exact stack in `compose.yaml`; the **Go/Gin** and **Fastify** guides build the app servers that sit behind Nginx.)

### 1.4 Nginx vs the alternatives (one table) **[B]**

| Tool | What it is | When you'd pick it |
|---|---|---|
| **Nginx** | Event-driven web server + reverse proxy + LB + cache | The default. Config-file driven, huge ecosystem, does everything in this guide. |
| **Apache httpd** | Older, module-rich web server (process/thread model, plus `event` MPM) | Legacy apps, `.htaccess` per-directory config, deep PHP `mod_php` integration. |
| **Caddy** | Modern Go web server, **automatic HTTPS** out of the box | Quick HTTPS with zero cert config; simpler syntax; smaller scale. |
| **HAProxy** | Specialized, very fast L4/L7 **load balancer** (not a file server) | Pure load balancing / TCP proxying at extreme scale. |
| **Envoy** | Modern L7 proxy, dynamic config via API, the data-plane of service meshes | Kubernetes/service-mesh (Istio), gRPC-heavy, dynamic environments. |
| **Traefik** | Cloud-native reverse proxy with **auto-discovery** (Docker/k8s labels) | Dynamic container environments where config should follow deployments. |
| **OpenResty** | Nginx + LuaJIT — programmable Nginx | Complex gateway logic in Lua (custom auth, dynamic routing). |
| **Nginx Unit** | F5 polyglot **app server** with a JSON control API | Running app processes directly with live, API-driven reconfiguration. |

---

## 2. Installation, First Run & Docker

### 2.1 Installing on Linux **[B]**

On Linux you have two sources, and the choice matters:

- **Your distro's package** (`apt install nginx` on Debian/Ubuntu, `dnf install nginx` on Fedora/RHEL). Easy, integrates with `systemd`, but often a few versions behind and built against the distro's module choices.
- **The official Nginx repository** (`nginx.org`). Gives you the latest **stable** or **mainline** build with the modules Nginx ships. Recommended when you need HTTP/3, recent TLS, or a specific module.

```bash
# Debian / Ubuntu — install the distro package (quickest start)
sudo apt update
sudo apt install nginx

# Confirm it installed and check the version + how it was compiled (which modules are built in)
nginx -v        # prints e.g. nginx version: nginx/1.27.3
nginx -V        # prints the FULL build: version + compile flags + module list (note: capital V).
                # Look here to see if --with-http_v3_module (HTTP/3) or --with-http_ssl_module is present.

# systemd manages the service on most Linux distros:
sudo systemctl enable --now nginx   # start now AND start on boot
sudo systemctl status nginx         # is it running?
sudo systemctl reload nginx         # graceful reload after a config change (no dropped connections)
sudo systemctl restart nginx        # full restart (drops connections — prefer reload)
```

After install, browse to `http://localhost/` and you'll see the default "Welcome to nginx!" page — proof the binary, the service, and port 80 all work.

**Where things live** (Debian/Ubuntu layout — RHEL differs slightly):

| Path | What it is |
|---|---|
| `/etc/nginx/nginx.conf` | The **main** config file (the entry point). |
| `/etc/nginx/conf.d/*.conf` | Drop-in config fragments, auto-included by the main file. |
| `/etc/nginx/sites-available/` | A Debianism: one file per site (server block), enabled by symlinking… |
| `/etc/nginx/sites-enabled/` | …into here. (Official Nginx uses `conf.d/` instead; see §3.5.) |
| `/var/www/html/` | Default document root for static files. |
| `/var/log/nginx/access.log` & `error.log` | The two logs you will live in. |
| `/usr/share/nginx/html/` | Default root on the official package. |

### 2.2 The essential CLI **[B]**

You will use a tiny set of commands constantly. Memorize these — they are the difference between confident operation and guessing:

```bash
nginx -t          # TEST the config for syntax errors WITHOUT applying it. Run this BEFORE every reload.
                  #   "syntax is ok / test is successful" = safe to reload. ALWAYS do this first.
nginx -T          # Test AND dump the FULL, fully-resolved config (all includes expanded) to stdout.
                  #   Invaluable for debugging "where is this directive actually coming from?".
nginx -s reload   # Send the master a "reload" signal: re-read config, spin up new workers, drain old
                  #   ones gracefully. ZERO dropped connections. This is how you apply changes.
nginx -s quit     # Graceful shutdown: finish in-flight requests, then stop.
nginx -s stop     # Fast shutdown: stop immediately (drops in-flight requests).
nginx -s reopen   # Reopen log files — used by logrotate so Nginx writes to the freshly-rotated file.
nginx -g 'daemon off;'        # Run in the foreground (don't daemonize) — REQUIRED in Docker (§2.4).
nginx -c /path/to/nginx.conf  # Use a non-default config file.
```

> **Golden rule:** the workflow for *every* config change is **edit → `nginx -t` → `nginx -s reload`** (or `systemctl reload nginx`). Never `restart` in production if you can `reload` — reload is zero-downtime, restart drops connections.

### 2.3 The stable vs mainline branch **[B]**

Nginx ships two branches and the naming is counter-intuitive:

- **Mainline** (odd minor versions: 1.25, 1.27…) — the **active development** branch. Despite "mainline" sounding bleeding-edge, **Nginx recommends it for production**: it gets new features (HTTP/3 landed here), bug fixes, and security patches *first*, and it is well-tested. New features only appear in mainline.
- **Stable** (even minor versions: 1.24, 1.26…) — a frozen branch that receives **only critical bug and security fixes**, no new features. Good when you want maximum predictability and never want behaviour to change.

For most people in 2026: **run mainline** unless your org mandates the stable branch. (This mirrors the Node "Current vs LTS" idea but inverted in spirit — see the **Node guide**.)

### 2.4 Nginx in Docker **[B/I]**

Running Nginx in a container is the most common way to use it today, and it interacts with a few Nginx quirks you must understand. (Full Docker mechanics: see the **Docker guide**.)

The key Nginx-specific facts when containerizing:

1. **Run in the foreground.** Docker tracks the container's **PID 1**; if Nginx daemonizes (forks into the background), PID 1 exits and Docker thinks the container died. The official image already sets `CMD ["nginx", "-g", "daemon off;"]`, which keeps the master process in the foreground. If you write your own command, you must include `daemon off;`.
2. **Logs go to stdout/stderr.** The official image symlinks `access.log → /dev/stdout` and `error.log → /dev/stderr` so `docker logs` shows them. Don't fight this; it's the container-native way.
3. **Mount your config and content** via volumes/bind mounts; don't bake dev config into the image.

```dockerfile
# Dockerfile — a static site served by Nginx (a classic multi-stage pattern; see DOCKER_GUIDE §9)
# ── Stage 1: build the frontend (e.g. a React/Vite SPA) ───────────────────────
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                         # reproducible install from the lockfile
COPY . .
RUN npm run build                  # emits static assets into /app/dist

# ── Stage 2: serve the built assets with Nginx (tiny final image) ─────────────
FROM nginx:1.27-alpine
# Replace the default server config with ours (SPA fallback, caching headers, etc.)
COPY nginx.conf /etc/nginx/conf.d/default.conf
# Copy ONLY the built static files from stage 1 — no Node, no source in the final image
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
# The base image's CMD already runs `nginx -g 'daemon off;'` — nothing more to do.
```

```yaml
# compose.yaml — Nginx reverse-proxying an API container (cross-reference: DOCKER_GUIDE §17)
services:
  web:
    build: .                       # the Nginx image from the Dockerfile above
    ports:
      - "8080:80"                  # host 8080 → container 80
    depends_on:
      - api
    # In Compose, services reach each other BY SERVICE NAME via Docker's DNS.
    # So inside nginx.conf you proxy_pass to http://api:3000 — "api" resolves to the API container.
  api:
    image: my-node-api:latest      # your backend (see the Fastify / Node guide)
    expose:
      - "3000"                     # visible to other services on the Docker network, NOT to the host
```

> **⚡ Version note / Docker DNS gotcha:** Nginx resolves upstream hostnames in a `proxy_pass http://api:3000;` **once at startup/reload** by default. If the `api` container restarts and gets a *new* IP, Nginx keeps using the stale one and you get `502`s. Fixes: (a) use a Docker **named network** and a `resolver` directive pointing at Docker's internal DNS `127.0.0.11` with a short `valid=` time, plus a variable in `proxy_pass` to force per-request resolution (shown in §6.7); or (b) put the upstream in an `upstream {}` block, which is more forgiving. This bites everyone once.

### 2.5 Nginx on Windows **[B]**

The Windows build exists and is fine for *learning and local testing*, but know its limits: it runs a **single worker** (no `worker_processes auto` benefit), lacks `sendfile`/`aio`/`epoll`, and is generally slower. **Do not run production Nginx on Windows.** On Windows 11, the realistic path is to run Nginx in **Docker Desktop** (Linux containers via WSL 2 — see the **Docker guide**), which gives you the real Linux Nginx. To try the native Windows build: download the zip from `nginx.org`, unzip, run `nginx.exe` from the folder; control it with `nginx.exe -s reload` etc. from that directory.

---

## 3. The Configuration Model: Master/Worker, Contexts, Directives

### 3.1 The master/worker process model **[B]**

When Nginx starts you get **one master process** and **several worker processes**. They have completely different jobs, and understanding the split explains how reloads, permissions, and tuning work.

- **The master process** runs as **root** (so it can bind to privileged ports 80/443 and read TLS private keys). It does **no request handling**. Its jobs are: read and validate the config, bind the listening sockets, spawn and supervise the workers, and respond to control signals (`reload`, `reopen`, `quit`). Because it holds the sockets and the privileged work, the workers don't need root.
- **The worker processes** run as an **unprivileged user** (`www-data` / `nginx`) and do **all the actual work** — accept connections, parse requests, serve files, proxy upstream, run the event loops described in §1.2. You typically run **one worker per CPU core** (`worker_processes auto;`), because each worker is single-threaded and event-driven, so one per core saturates the CPU without context-switch waste.

```
            ┌──────────────────────────────────────────┐
            │  master process  (root)                   │
            │   • reads nginx.conf, binds :80/:443      │
            │   • spawns + supervises workers           │
            │   • handles -s reload / reopen / quit     │
            └───────────────┬──────────────────────────┘
                  spawns     │      (no request handling here)
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ┌──────────┐       ┌──────────┐       ┌──────────┐
   │ worker 0 │       │ worker 1 │       │ worker 2 │   (unprivileged: www-data/nginx)
   │ epoll    │       │ epoll    │       │ epoll    │   each = 1 thread, 1000s of connections
   └──────────┘       └──────────┘       └──────────┘
```

**Why this matters for reloads:** when you run `nginx -s reload`, the master re-reads the config, starts **new** workers with the new config, and tells the **old** workers to stop accepting new connections and finish their in-flight ones, then exit. For a moment both generations run side by side. That is the mechanism behind **zero-downtime config changes** — and it's why a broken config caught by `nginx -t` would only fail to spawn new workers while the old ones keep serving.

### 3.2 The config file structure: contexts (blocks) **[B]**

An Nginx config is a tree of **directives** organized into **contexts** (also called blocks). A directive is either *simple* (a name, some arguments, a semicolon: `worker_processes auto;`) or a *block* (a name and a `{ … }` containing more directives). **Indentation is cosmetic; the `{}` and `;` are what matter.** Forgetting a semicolon is the #1 syntax error.

The contexts, from outermost to innermost, are:

```
main context (the file itself — global settings: user, worker_processes, error_log, pid)
│
├── events { }      ← connection-processing settings (worker_connections, the event model)
│
└── http { }        ← EVERYTHING about HTTP: the umbrella for all web serving
     │
     ├── (http-level directives: log formats, gzip, include mime.types, proxy_cache_path…)
     │
     ├── upstream backend { }   ← named groups of backend servers for load balancing
     │
     └── server { }    ← a "virtual host": one site/domain, one or more listen ports
          │
          ├── (server-level directives: server_name, listen, ssl_certificate, root…)
          │
          └── location /path { }   ← rules for a subset of URLs within this server
               │
               └── (and locations can nest, and contain limit_except, if, etc.)
```

There is also a `stream { }` context (a sibling of `http`) for proxying raw **TCP/UDP** (databases, MQTT, game servers) and a `mail { }` context for mail proxying. This guide focuses on `http`, with a `stream` mention in §6.8.

### 3.3 A minimal complete `nginx.conf` annotated **[B]**

Here is a complete, runnable main config with every line explained. This is the skeleton everything else hangs on:

```nginx
# ── MAIN CONTEXT (the top level of the file) ──────────────────────────────────
user  www-data;            # the UNPRIVILEGED user workers run as (security: never run workers as root)
worker_processes  auto;    # spawn one worker per CPU core (auto detects core count). See §12.
pid        /run/nginx.pid; # where the master writes its PID (used by signals / systemd)

error_log  /var/log/nginx/error.log warn;   # error log + minimum level (debug<info<notice<warn<error<crit)
                                             # Set to 'debug' only temporarily — it is very verbose.

# ── EVENTS CONTEXT — how workers handle connections ───────────────────────────
events {
    worker_connections  1024;   # MAX simultaneous connections PER WORKER. Total ≈ workers × this.
                                 # Bump to 4096–65535 for high-traffic edges (§12). Also raise ulimit -n.
    # multi_accept on;          # accept as many new connections as are pending per event (high-load tuning)
    # use epoll;                # the event mechanism. Nginx auto-picks epoll on Linux; rarely set manually.
}

# ── HTTP CONTEXT — the umbrella for all web serving ───────────────────────────
http {
    include       /etc/nginx/mime.types;   # map file extensions → Content-Type (e.g. .css → text/css)
    default_type  application/octet-stream; # fallback Content-Type for unknown extensions

    # --- Logging: define a named log FORMAT, then USE it (see §4.7) ---
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;   # log every request in 'main' format

    # --- Performance basics (explained fully in §12) ---
    sendfile        on;     # let the kernel copy file → socket directly (zero-copy); fast static serving
    tcp_nopush      on;     # with sendfile, send response headers + file in full packets (less overhead)
    keepalive_timeout  65;  # how long to hold an idle client keep-alive connection open (seconds)

    gzip  on;               # compress text responses on the fly (saves bandwidth; see §10.7)

    # --- Pull in per-site server blocks from drop-in directories ---
    include /etc/nginx/conf.d/*.conf;        # official-Nginx convention
    include /etc/nginx/sites-enabled/*;      # Debian/Ubuntu convention (one of these, usually)
}
```

### 3.4 Directives and inheritance — the rule that explains everything **[B/I]**

The single most important *behavioural* concept: **most directives are inherited downward into nested contexts, but can be overridden at a more specific level — and inheritance happens by *replacement*, not merging.**

- If you set `gzip on;` in `http`, every `server` and `location` inherits it unless one sets `gzip off;`.
- If you set `root /var/www/site;` in a `server`, every `location` inside serves from there unless a location sets its own `root`.

The subtle, bug-causing part is the **"array directives reset, they don't append"** rule. Directives that take a *list* (like `proxy_set_header`, `add_header`, `proxy_pass_header`) are **inherited only if the child context defines *none* of them.** The moment a child context specifies *even one* `add_header`, it **replaces the entire inherited set** — the parent's headers vanish in that child. This trips up everyone:

```nginx
http {
    add_header X-From-Http "yes";      # set at http level
    server {
        add_header X-From-Server "yes"; # ← this REPLACES the http-level add_header entirely.
                                        #    Responses from this server will NOT have X-From-Http!
        location /api {
            add_header X-From-Loc "yes"; # ← and THIS replaces the server-level one too.
                                         #    /api responses have ONLY X-From-Loc.
        }
    }
}
```

> **Best practice:** because of replacement semantics, **re-declare all needed `add_header` / `proxy_set_header` lines in the most specific context where any of them appear**, or factor them into an `include`d snippet you pull into each `location`. Don't assume parent headers carry through. (More on `add_header` pitfalls in §11.2.)

### 3.5 Includes and the `sites-available`/`conf.d` patterns **[B/I]**

`include` literally splices another file's contents in at that point. It keeps configs modular: one file per site, shared snippets, etc.

Two conventions you'll meet:

- **`conf.d/`** (official Nginx, and what Docker images use): the main `http` block ends with `include /etc/nginx/conf.d/*.conf;`. You drop `mysite.conf` into `conf.d/` and reload. Simple.
- **`sites-available/` + `sites-enabled/`** (Debian/Ubuntu): you *write* site files in `sites-available/`, then "enable" one by creating a **symlink** to it in `sites-enabled/` (`ln -s ../sites-available/mysite /etc/nginx/sites-enabled/`). The main config includes `sites-enabled/*`. The benefit is you can disable a site by deleting the symlink without losing the file. It's purely a workflow convention — functionally identical to `conf.d/`.

A common, clean structure:

```nginx
# /etc/nginx/snippets/security-headers.conf  — a reusable snippet (see §11)
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# /etc/nginx/snippets/proxy-headers.conf — reusable proxy header set (see §6.3)
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

```nginx
# /etc/nginx/conf.d/example.conf — a site that REUSES the snippets above
server {
    listen 80;
    server_name example.com;
    root /var/www/example;

    include /etc/nginx/snippets/security-headers.conf;   # splice the headers in here

    location /api/ {
        include /etc/nginx/snippets/proxy-headers.conf;  # and the proxy headers here
        proxy_pass http://127.0.0.1:3000;
    }
}
```

### 3.6 How Nginx processes a request — the phase pipeline **[I]**

Knowing the *order* in which Nginx evaluates things stops a whole class of "why didn't my rule fire?" confusion. For each HTTP request, Nginx roughly:

1. **Accepts the connection** and reads the request line and headers.
2. **Selects the `server` block** by matching the request's `Host` header against `server_name` (and the `listen` ip:port). If nothing matches, the **default server** for that port handles it (the first `server` for that port, or one marked `default_server`).
3. **Selects the `location`** within that server using the matching algorithm in §5 (this is where most surprises live).
4. **Runs through internal phases** in a fixed order: rewrite (URL rewriting), access (allow/deny, `auth_request`, `limit_req`, `limit_conn`), then the **content phase** (serve a file, proxy upstream, return a response).
5. **Applies output filters** (gzip, `add_header`, `sub_filter`) on the way back out.
6. **Logs** the request (`access_log`) once the response is sent.

The practical upshots: `server_name` chooses the site *before* `location` runs; `try_files` and `proxy_pass` happen in the content phase *after* access checks like `limit_req`; and `add_header` runs as an *output filter* on the way out, which is why it only applies to certain status codes unless you add `always`.

---

## 4. Static Web Server: root, alias, index, try_files, MIME, logs

### 4.1 Serving files — the simplest useful server **[B]**

At its core, serving static files is: "map the request URI to a file on disk and send it." Nginx is exceptionally good at this because of `sendfile` (kernel zero-copy) and its event model. Here is a complete static site:

```nginx
server {
    listen 80;                       # listen on port 80 (HTTP) on all interfaces
    server_name static.example.com;  # respond when the Host header is this

    root  /var/www/static;           # the document root: URIs are resolved UNDER this directory
    index index.html index.htm;      # if a directory is requested, try these files in order

    # That's it. A request for /css/site.css serves /var/www/static/css/site.css.
    # A request for / serves /var/www/static/index.html (via the index directive).
}
```

### 4.2 `root` vs `alias` — the distinction that bites everyone **[B/I]**

Both `root` and `alias` map a URL to a filesystem path, but **they combine the path differently**, and mixing them up gives 404s.

- **`root`** sets a base directory, and Nginx **appends the entire request URI** to it. `root /var/www; location /images/ {}` → a request for `/images/cat.png` serves `/var/www/images/cat.png`. The location prefix stays in the path.
- **`alias`** **replaces the matched location prefix** with the alias path. `location /images/ { alias /data/pics/; }` → `/images/cat.png` serves `/data/pics/cat.png`. The `/images/` part is *substituted out*.

```nginx
# Use ROOT when the URL path mirrors the directory structure:
location /static/ {
    root /var/www;          # /static/app.js  →  /var/www/static/app.js   (URI appended to root)
}

# Use ALIAS when you want to map a URL prefix onto a DIFFERENT directory name:
location /downloads/ {
    alias /mnt/files/;      # /downloads/report.pdf  →  /mnt/files/report.pdf  (prefix replaced)
}
```

> **Gotcha — the trailing slash with `alias`:** if your `location` ends in `/`, the `alias` should end in `/` too, and vice versa. Mismatched trailing slashes produce wrong paths. **Also avoid `alias` inside a *regex* location** unless you capture and reference the rest of the path explicitly — it's a known foot-gun. When in doubt, prefer `root`.

### 4.3 `index` and directory requests **[B]**

When a request maps to a **directory** (ends in `/`, or is the bare site root), Nginx looks for an **index file** to serve instead of the directory itself. `index index.html index.php;` tries each name in order; the first that exists is served (internally redirected to). If none exist and `autoindex` is off, Nginx returns `403 Forbidden` (it won't list the directory). The `index` directive is inherited, so you usually set it once at `server` level.

### 4.4 `try_files` — the workhorse, explained deeply **[B/I]**

`try_files` is the most important static-serving directive and the engine behind SPA routing and graceful fallbacks. **It tries a list of file/path candidates in order and uses the first one that exists; the *last* argument is a fallback** that is either an internal redirect (to a URI or named location) or a status code.

The logic: Nginx checks each candidate as a *filesystem path* (relative to `root`). The special token `$uri` is the request URI; `$uri/` tests for a directory. The final argument is different — it is **not** tested for existence; it's the action taken when nothing matched.

```nginx
# Classic static fallback: serve the file if it exists, else the directory, else 404.
location / {
    try_files $uri $uri/ =404;
    #         │     │     └── final arg: if neither exists, return HTTP 404
    #         │     └──────── try $uri as a DIRECTORY (will use the index file)
    #         └────────────── try $uri as a literal file
}
```

The killer use case is the **Single-Page App (SPA) fallback**. A React/Vue/Angular app does client-side routing: the URL `/users/42` has no corresponding `users/42.html` on disk — the server should return `index.html` and let the JS router handle the path. `try_files` expresses exactly that:

```nginx
# SPA fallback: real files (JS, CSS, images) are served directly; everything else → index.html
location / {
    try_files $uri $uri/ /index.html;
    #         │     │     └── FALLBACK to /index.html for any unknown path (client router takes over)
    #         └─────┴──────── but serve real assets (/app.abc123.js, /logo.png) directly if they exist
}
```

This is the single line at the heart of "serve a SPA from Nginx" (full SPA + API example in §13.1). You can also fall back to a **named location** (handy for proxying when a static file is absent):

```nginx
location / {
    try_files $uri @backend;     # serve the static file if present, otherwise hand off to @backend
}
location @backend {              # a NAMED location (the @ prefix) — only reachable via try_files/error_page
    proxy_pass http://127.0.0.1:3000;
}
```

> **Why `try_files` and not `if`?** Nginx's `if` inside `location` is notoriously surprising ("if is evil" — see §14.3). `try_files` is the declarative, predictable way to express "use this, or that, or fall back." Reach for it first.

### 4.5 MIME types — getting `Content-Type` right **[B]**

The browser decides how to treat a response based on its `Content-Type` header (render HTML, run JS, show an image). Nginx sets it from the file extension using the `mime.types` map, pulled in with `include mime.types;`. If an extension isn't in the map, Nginx uses `default_type` (commonly `application/octet-stream`, which makes browsers *download* rather than render). **If your CSS or JS isn't applying, check the `Content-Type`** — a missing mapping or a wrong `default_type` is a frequent cause. You can add mappings via a `types { }` block, but editing `mime.types` (or relying on the shipped one, which is comprehensive) is usual.

### 4.6 Autoindex (directory listing) and custom error pages **[B/I]**

**Autoindex** generates an HTML directory listing when no index file is present — useful for a file-download server, *dangerous* if it exposes files you didn't mean to share (security note below).

```nginx
location /files/ {
    alias /srv/public-files/;
    autoindex on;             # generate a browsable listing of this directory
    autoindex_exact_size off; # show human-readable sizes (KB/MB) instead of bytes
    autoindex_localtime on;   # show timestamps in local time, not UTC
}
```

> **Security:** **leave `autoindex off` (the default) everywhere except deliberate public-download directories.** An accidental autoindex on your app root can leak source, backups, and secrets. (Compare the **FTP guide** for purpose-built file serving; for *public downloads* a locked-down autoindex location or an app endpoint is safer than FTP.)

**Custom error pages** replace Nginx's bland default error responses with your own branded pages:

```nginx
error_page 404 /errors/404.html;             # serve this file for 404s
error_page 500 502 503 504 /errors/50x.html; # one page for all server errors

location = /errors/404.html {                 # serve the error page itself from disk…
    root /var/www;
    internal;                                 # …but make it reachable ONLY via error_page (not directly)
}
```

The `internal` directive is important: it means the URI can only be reached through an internal redirect (like `error_page`), not by a client typing it — so users can't navigate straight to `/errors/404.html`.

### 4.7 Logging: access logs, error logs, custom formats **[B/I]**

Logs are your eyes. Nginx has two:

- **`access_log`** — one line per request (who, what, status, size, timing). You define **named formats** with `log_format` and reference them. The default `combined` format is the well-known Apache-style line.
- **`error_log`** — diagnostics: failed upstreams, permission errors, config issues. Set its level (`warn` is sane; `debug` only when actively debugging, and it's huge).

A **rich custom format** that includes timing (invaluable for spotting slow upstreams) and the real client IP behind a proxy:

```nginx
http {
    # Define a detailed format. $request_time = total time Nginx spent on the request (client side);
    # $upstream_response_time = time the BACKEND took (isolates "is it Nginx or my app that's slow?").
    log_format detailed '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" '
                        'rt=$request_time uct="$upstream_connect_time" '
                        'urt="$upstream_response_time" cache=$upstream_cache_status';

    access_log /var/log/nginx/access.log detailed;   # use it globally
    # access_log off;                                 # you can DISABLE logging for noisy paths:
    #   location = /health { access_log off; return 200; }   ← don't log health checks

    # Buffer log writes to reduce disk I/O on high-traffic sites (flush every 1s or when buffer fills):
    # access_log /var/log/nginx/access.log detailed buffer=32k flush=1s;
}
```

Key log variables you'll reach for: `$status`, `$request_time`, `$upstream_response_time`, `$upstream_cache_status` (HIT/MISS/BYPASS — §10), `$ssl_protocol`/`$ssl_cipher` (which TLS was negotiated), `$http_x_forwarded_for` (the client chain through proxies).

> **Log rotation:** Nginx holds log files open, so deleting/renaming them doesn't free space until Nginx reopens them. `logrotate` (standard on Linux) handles this and signals Nginx with `nginx -s reopen` after rotating. In Docker you instead log to stdout/stderr and let the platform handle retention (§2.4).

---

## 5. `location` Matching In Depth

### 5.1 Why this section is its own chapter **[I]**

`location` matching is the **single biggest source of Nginx bugs and confusion**, because the matching order is *not* "top to bottom" — it's a specific priority algorithm that mixes prefix matches and regex matches with non-obvious precedence. Internalize this section and a huge fraction of Nginx mysteries evaporate.

A `location` block defines a rule for a **subset of request URIs** within a `server`. The question Nginx must answer for every request is: *given all these `location` blocks, which ONE wins?* (Exactly one location handles a request, though it may internally redirect to others.)

### 5.2 The five kinds of location **[I]**

| Syntax | Name | Meaning |
|---|---|---|
| `location /path/ { }` | **Prefix** | Matches any URI that *starts with* `/path/`. The default kind. |
| `location = /exact { }` | **Exact** | Matches the URI **exactly and only**. Fastest; checked first. |
| `location ^~ /path/ { }` | **Prefix, stop-regex** | A prefix match that, if it's the longest prefix, **skips regex checking**. |
| `location ~ \.php$ { }` | **Regex (case-sensitive)** | PCRE regex match, case-sensitive. |
| `location ~* \.(jpg\|png)$ { }` | **Regex (case-insensitive)** | PCRE regex match, ignoring case. |

### 5.3 The matching algorithm — the exact priority order **[I]**

Here is the algorithm Nginx runs, in order. **Memorize this list — it is the whole game:**

1. **Exact match (`=`) wins immediately.** If a `location = /foo` matches the URI exactly, Nginx uses it and **stops** — no further checking. (Great for `/health`, `/favicon.ico`.)
2. **Find the longest matching *prefix* location.** Nginx scans all plain-prefix and `^~` locations and remembers the one with the **longest** matching prefix. (Longest, not first — order in the file doesn't matter for prefixes.)
3. **If that longest prefix is `^~`, stop and use it** — skip all regex.
4. **Otherwise, check regex locations *in file order*** and use the **first** regex that matches. (Here order *does* matter — first match wins, not longest.)
5. **If no regex matches, fall back to the longest prefix** remembered in step 2.

Two rules summarize the gotchas: **prefixes compete by length (longest wins), regexes compete by order (first wins), and a matching regex beats a plain prefix — unless the prefix is `^~` or `=`.**

```
   URI arrives
        │
        ▼
   ┌─ exact `= /x` matches? ──► YES ──► use it, DONE
   │        │ NO
   ▼        ▼
   find LONGEST matching prefix (plain or ^~)
        │
        ├─ is that longest prefix `^~`? ──► YES ──► use it, DONE (skip regex)
        │        │ NO
        ▼        ▼
   check regexes IN ORDER; first match wins ──► use it, DONE
        │ (no regex matched)
        ▼
   use the longest plain prefix from above ──► DONE (or 404 if none)
```

### 5.4 A worked example you should be able to predict **[I]**

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example;

    location = / {                       # (A) EXACT match for the homepage only
        return 200 "homepage\n";
    }
    location / {                         # (B) catch-all prefix (matches everything as a fallback)
        try_files $uri $uri/ =404;
    }
    location /images/ {                  # (C) prefix for the images tree
        try_files $uri =404;
    }
    location ^~ /assets/ {               # (D) prefix that STOPS regex checking for /assets/*
        expires 30d;                     #     (so a .php regex below won't hijack /assets/script.js)
    }
    location ~* \.(jpg|jpeg|png|gif|css|js)$ {   # (E) regex for static asset extensions
        expires 7d;
        access_log off;
    }
    location ~ \.php$ {                  # (F) regex for PHP (would proxy to PHP-FPM)
        # fastcgi_pass 127.0.0.1:9000; …
        return 200 "php handler\n";
    }
}
```

Now predict the winners:

- `/` → **(A)** exact match wins immediately.
- `/about` → no exact, longest prefix is **(B)** `/`, no regex matches `/about` → **(B)**.
- `/images/cat.png` → longest prefix is **(C)** `/images/`, but `.png` matches regex **(E)**, and a regex beats a plain prefix → **(E)** wins (so `/images/` is effectively bypassed for image extensions!).
- `/assets/app.js` → longest prefix is **(D)** `^~ /assets/`; because it's `^~`, regex checking is **skipped** → **(D)** wins (this is *why* you use `^~` — to protect a tree from regex hijacking).
- `/index.php` → no exact, longest prefix is **(B)** `/`, regexes checked in order: **(F)** `\.php$` matches → **(F)**.

> **Practical takeaways:** (1) Use `=` for hot exact paths like `/health` and `/favicon.ico` to short-circuit matching. (2) Use `^~` to shield a static asset tree from a greedy regex (the `/assets/` case). (3) Remember that a regex location can *steal* requests you thought a prefix owned — when an asset rule misbehaves, this is usually why. (4) Regex order matters; put more specific regexes first.

---

## 6. Reverse Proxy

### 6.1 What a reverse proxy is and why **[I]**

A **reverse proxy** accepts client requests at the edge and **forwards them to one or more backend servers**, then relays the backend's response back to the client. "Reverse" distinguishes it from a *forward* proxy (which sits next to the *client* and forwards the client's outbound requests). Here Nginx sits next to the *servers* and fronts them.

Why front your app with a reverse proxy instead of exposing the app directly:

- **TLS termination.** Nginx handles HTTPS/HTTP-2/HTTP-3 once; your app speaks plain HTTP locally. One place to manage certs and ciphers.
- **One public entry point** for many internal services (route `/api` to one app, `/auth` to another).
- **Serve static assets directly** without waking the app — Nginx is faster at it and frees app workers.
- **Buffering slow clients.** Nginx reads a slow client's full request/serves a slow client the full response while your app's connection completes quickly — so app worker threads aren't tied up by slow networks (huge for thread-per-request app servers).
- **Cross-cutting concerns at the edge:** rate limiting, security headers, caching, compression, blocking bad IPs — applied uniformly, regardless of which backend handles the request.
- **Load balancing and zero-downtime deploys** (§7).
- **Hiding your topology** — clients never see how many backends you run or where they are.

### 6.2 `proxy_pass` — the core directive **[I]**

`proxy_pass` says "forward this request to that URL." A complete reverse proxy in front of a Node/Go app:

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;   # forward everything to the app listening on localhost:3000
        # (this is your Node/Fastify or Go/Gin server — see the Fastify / Go guides)
    }
}
```

A **critical, subtle rule about the trailing slash on `proxy_pass`** (the URI part), which changes how the path is rewritten:

```nginx
# WITHOUT a trailing slash/path on proxy_pass: the FULL original URI is passed through.
location /api/ {
    proxy_pass http://127.0.0.1:3000;     # request /api/users → backend receives /api/users
}

# WITH a trailing slash (a "URI" on proxy_pass): the matched location prefix is REPLACED.
location /api/ {
    proxy_pass http://127.0.0.1:3000/;    # request /api/users → backend receives /users  (/api/ stripped)
}
```

This mirrors the `root` vs `alias` distinction (§4.2): a trailing-slash `proxy_pass` strips the location prefix; a bare one preserves the full path. Decide based on whether your backend expects the `/api` prefix or not.

> **⚡ Gotcha:** you **cannot** use a trailing-slash (path-rewriting) `proxy_pass` together with a **regex** location or inside an `if` — Nginx errors or behaves unexpectedly. For regex locations, rewrite the URI explicitly with `rewrite` then use a bare `proxy_pass`.

### 6.3 Passing the right headers — and *why* each one matters **[I]**

When Nginx proxies a request, the backend by default loses information about the *original* client, because as far as the backend can tell, the request came from **Nginx** (Nginx's IP, Nginx as the host). You must explicitly pass the original details with `proxy_set_header`. Each of these is load-bearing — here's what each does and why your app needs it:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;

    # --- The four headers EVERY reverse proxy should set, and WHY ---

    proxy_set_header Host $host;
    #   WHY: by default Nginx sends the BACKEND's address as Host. Apps use Host for routing,
    #   virtual hosts, generating absolute URLs, and cookies. $host = the original requested hostname.

    proxy_set_header X-Real-IP $remote_addr;
    #   WHY: the backend sees Nginx's IP as the client IP. $remote_addr is the ACTUAL client IP.
    #   Your app needs the real IP for logging, geo, rate-limiting, abuse detection.

    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #   WHY: the STANDARD way to convey the client IP through a CHAIN of proxies. This special
    #   variable appends $remote_addr to any existing X-Forwarded-For, building "client, proxy1, proxy2".

    proxy_set_header X-Forwarded-Proto $scheme;
    #   WHY: tells the backend whether the ORIGINAL request was http or https. Critical when Nginx
    #   terminates TLS and talks plain HTTP to the backend — otherwise the app thinks it's on http,
    #   generates http:// links, sets insecure cookies, or redirect-loops. $scheme is http or https.

    proxy_set_header X-Forwarded-Host  $host;   # original Host (some frameworks read this)
    proxy_set_header X-Forwarded-Port  $server_port; # original port
}
```

> **Security note on X-Forwarded-For:** these headers are **client-spoofable** if the request reaches Nginx directly. Trust them only from proxies you control. If Nginx is *behind* another proxy/CDN (Cloudflare, an AWS ALB), use the **`realip` module** (`set_real_ip_from <trusted-cidr>; real_ip_header X-Forwarded-For;`) to safely extract the true client IP from a *trusted* hop, and don't blindly trust `X-Forwarded-For` from arbitrary clients. Your *backend* should likewise only trust these headers when the request came from Nginx. (This matters for `express`/`fastify` `trust proxy` settings — see the **Fastify/Node guides**.)

### 6.4 Buffering, timeouts, and body size **[I]**

Three families of directives govern proxy behaviour and prevent a class of production incidents:

**Buffering** — Nginx reads the backend's response into buffers and can serve a slow client from those buffers, freeing the backend connection quickly. Usually leave it **on** (default). Turn it **off** for streaming/SSE/long-polling where you want bytes to flow through immediately.

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;

    proxy_buffering on;                 # (default) buffer the backend response; good for normal requests
    proxy_buffers 8 16k;                # 8 buffers of 16k each per connection for the response body
    proxy_buffer_size 16k;              # buffer for the response HEADER (raise if you get "upstream sent
                                        #   too big header" 502s with large cookies/JWTs)
    # For Server-Sent Events / streaming, DISABLE buffering on that location:
    # proxy_buffering off;
    # proxy_cache off;
}
```

**Timeouts** — how long Nginx waits at each stage before giving up (and returning 502/504). Tune to your app's real latency; too short causes spurious 504s, too long lets slow requests pile up.

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_connect_timeout 5s;   # max time to establish the TCP connection to the backend
    proxy_send_timeout   60s;   # max gap between successive writes TO the backend
    proxy_read_timeout   60s;   # max gap between successive reads FROM the backend (the big one for
                                #   slow endpoints — raise for long-running APIs; a too-low value = 504)
}
```

**Request body size** — Nginx caps client upload size with `client_max_body_size` (**default 1 MB**). If users upload bigger files and you forget this, they get a confusing **413 Request Entity Too Large**. Set it where uploads happen.

```nginx
location /upload {
    proxy_pass http://127.0.0.1:3000;
    client_max_body_size 50m;     # allow up to 50 MB uploads (match your app's own limit)
    proxy_request_buffering on;   # (default) buffer the upload before sending to backend; set 'off' to
                                  #   stream large uploads straight through (less disk, but ties backend)
}
```

(For real file-upload backends, see the **Go/Gin file-upload guide** and the **FTP guide** for alternatives.)

### 6.5 Upstream keepalive — don't re-handshake every request **[I]**

By default Nginx opens a **new TCP connection to the backend for every proxied request** and closes it after. Under load that's a lot of TCP setup/teardown overhead. **Keepalive** reuses idle connections to the backend across requests — a big latency and CPU win. It requires an `upstream {}` block plus three header tweaks:

```nginx
upstream app_backend {
    server 127.0.0.1:3000;
    keepalive 32;          # keep up to 32 idle connections per worker open to this upstream for reuse
}

server {
    location / {
        proxy_pass http://app_backend;
        # Keepalive to a backend REQUIRES HTTP/1.1 and clearing the Connection header:
        proxy_http_version 1.1;          # HTTP/1.0 (the proxy default) cannot keep-alive
        proxy_set_header Connection "";   # remove "Connection: close" so the socket stays open for reuse
    }
}
```

### 6.6 WebSocket (and SSE) proxying — the Upgrade dance **[I]**

WebSockets start as an HTTP request that asks to "Upgrade" the connection to the WebSocket protocol. Nginx will **not** forward the `Upgrade`/`Connection` headers automatically — you must pass them, or the handshake fails and the socket never opens. This is the canonical WebSocket proxy recipe (works for Socket.IO, the **Gorilla WebSockets** Go backend, etc.):

```nginx
# A map to set the Connection header correctly: "upgrade" for WS requests, "close"/"" otherwise.
map $http_upgrade $connection_upgrade {
    default upgrade;       # if the client sent an Upgrade header, respond with Connection: upgrade
    ''      close;         # otherwise use Connection: close (a normal request)
}

server {
    location /ws/ {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;                       # WebSockets require HTTP/1.1
        proxy_set_header Upgrade    $http_upgrade;    # forward the client's Upgrade: websocket header
        proxy_set_header Connection $connection_upgrade; # set Connection: upgrade (via the map above)

        proxy_set_header Host $host;                  # still pass the normal proxy headers
        proxy_read_timeout 3600s;   # WS connections are long-lived; don't let read_timeout kill an idle
        proxy_send_timeout 3600s;   #   but active socket. Raise both well above your heartbeat interval.
    }
}
```

(Cross-reference: the **Go Gorilla WebSockets guide** builds the backend this fronts.) The same "disable buffering, long timeouts" idea applies to **Server-Sent Events** — for SSE add `proxy_buffering off;` so events stream instead of being held in a buffer.

### 6.7 Dynamic upstream resolution (Docker/Kubernetes) **[I/A]**

As warned in §2.4, Nginx resolves a hostname in `proxy_pass http://api:3000;` **once** (at startup or reload). In Docker/k8s where container IPs change on restart, that produces stale `502`s. The fix is to make Nginx re-resolve via DNS at runtime, using a **`resolver`** plus a **variable** in `proxy_pass` (a variable forces per-request resolution):

```nginx
server {
    resolver 127.0.0.11 valid=10s;   # Docker's embedded DNS; re-resolve names every 10s
    #         ↑ on Kubernetes use the cluster DNS (e.g. kube-dns) instead

    location /api/ {
        # Putting the hostname in a VARIABLE forces Nginx to use the resolver at request time
        # instead of caching the IP at startup. Note: with a variable you usually must handle the
        # URI yourself (here we strip /api/ and pass the rest).
        set $upstream "http://api:3000";
        proxy_pass $upstream;
        proxy_set_header Host $host;
    }
}
```

### 6.8 Proxying raw TCP/UDP — the `stream` module **[A]**

Beyond HTTP, Nginx can proxy and load-balance arbitrary **TCP and UDP** via the top-level `stream {}` context — useful for fronting databases, MQTT, DNS, game servers, or doing **TLS termination for non-HTTP** protocols. It's a sibling of `http {}`, not inside it:

```nginx
stream {
    upstream postgres_pool {
        server 10.0.0.11:5432;
        server 10.0.0.12:5432;
    }
    server {
        listen 5432;                 # accept Postgres connections on the edge…
        proxy_pass postgres_pool;    # …and balance them across the DB replicas (L4 — no HTTP awareness)
        proxy_timeout 1h;            # DB connections are long-lived
    }
}
```

(See the **PostgreSQL** and **Redis** guides for what's on the other end. Note: load-balancing a single *writable* primary database is more about failover than spreading writes — be careful.)

---

## 7. Load Balancing

### 7.1 What and why **[I]**

**Load balancing** spreads incoming requests across **multiple identical backend instances**, so no single instance is a bottleneck or a single point of failure. It's the foundation of **horizontal scaling** (add more app instances to handle more traffic) and **high availability** (if one instance dies, the others carry on). In Nginx you declare a pool of backends in an **`upstream {}`** block and point `proxy_pass` at the pool name.

```nginx
upstream api_cluster {
    server 10.0.0.21:3000;     # three identical instances of your app…
    server 10.0.0.22:3000;
    server 10.0.0.23:3000;
}
server {
    location /api/ {
        proxy_pass http://api_cluster;   # Nginx picks one backend per request, per the algorithm below
    }
}
```

### 7.2 The balancing algorithms — and when to use each **[I/A]**

The algorithm decides *which* backend gets the next request. Choose based on your app's statefulness and load profile:

| Directive | Algorithm | How it picks | Use when |
|---|---|---|---|
| *(default)* | **Round-robin** | Each backend in turn, optionally weighted | Stateless apps; the sensible default. |
| `least_conn;` | **Least connections** | The backend with the fewest active connections | Requests have **variable duration** (some slow, some fast) — avoids piling onto a busy node. |
| `ip_hash;` | **IP hash** | Hash of client IP → always same backend | You need **session stickiness** but have no shared session store. |
| `hash $key consistent;` | **Generic / consistent hash** | Hash of any key (URL, header…) | Cache-affinity (same URL → same node); `consistent` minimizes reshuffling when nodes change. |
| `random two least_conn;` | **Random (power-of-two-choices)** | Pick 2 at random, take the less busy | Large clusters; cheap, near-`least_conn` behaviour without global state. |

```nginx
# Round-robin with WEIGHTS — send more traffic to a beefier server:
upstream weighted {
    server 10.0.0.21:3000 weight=3;   # gets ~3× the requests (e.g. it has more CPU)
    server 10.0.0.22:3000 weight=1;
}

# Least connections — best when request durations vary a lot:
upstream varied {
    least_conn;
    server 10.0.0.21:3000;
    server 10.0.0.22:3000;
}

# IP hash — sticky sessions by client IP (see the caveat in §7.4):
upstream sticky {
    ip_hash;
    server 10.0.0.21:3000;
    server 10.0.0.22:3000;
}

# Consistent hash on the request URI — same URL tends to hit the same cache node:
upstream cache_nodes {
    hash $request_uri consistent;
    server 10.0.0.31:8080;
    server 10.0.0.32:8080;
}
```

### 7.3 Health checks: passive (open-source) vs active (Plus) **[I/A]**

Nginx must avoid sending traffic to a dead backend. Open-source Nginx does **passive** health checks; Nginx **Plus** adds **active** ones.

- **Passive (built in):** Nginx watches *real* traffic. If a request to a backend fails (connection refused, timeout, or — if configured — an error status), Nginx counts it; after `max_fails` failures within `fail_timeout`, it marks the backend **down** and stops sending to it for `fail_timeout` seconds, then tentatively retries. No extra requests — it learns from actual ones.

```nginx
upstream api_cluster {
    server 10.0.0.21:3000 max_fails=3 fail_timeout=15s;
    #   ↑ after 3 failed attempts within 15s, take this server out for 15s, then probe again.
    server 10.0.0.22:3000 max_fails=3 fail_timeout=15s;
    server 10.0.0.23:3000 backup;   # BACKUP: only used when all the primary servers are down
}
server {
    location /api/ {
        proxy_pass http://api_cluster;
        # Decide WHICH failures should trigger a retry on the NEXT backend:
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;     # try at most 2 backends before giving up
        proxy_next_upstream_timeout 10s; # …within a 10s overall budget
    }
}
```

- **Active (Plus only):** Nginx Plus periodically sends a *synthetic* probe (`health_check uri=/health interval=5s;`) to each backend and marks it up/down *before* real traffic hits a broken node. On open-source Nginx you approximate this with an external checker, or just rely on passive checks plus a fast `/health` endpoint and orchestration (Docker/k8s) restarting unhealthy containers.

`backup` servers (above) only receive traffic when **all** non-backup servers are down — a built-in failover tier. `down` marks a server as permanently out (handy for draining one during maintenance).

### 7.4 Session persistence ("sticky sessions") **[I/A]**

If your app stores session state **in the instance's memory**, consecutive requests from one user must hit the **same** instance, or they get logged out randomly. Options:

- **`ip_hash`** — sticky by client IP. Simple, but breaks when many users share an IP (corporate NAT, mobile carriers) — they all pile onto one backend — and a user's IP changing (mobile roaming) loses their session.
- **Sticky cookies (Plus only):** `sticky cookie srv_id expires=1h;` — Nginx sets a cookie naming the chosen backend, far more reliable than IP hashing.
- **The right fix: make your app stateless.** Store sessions in **Redis** or the database (see the **Redis guide**) instead of instance memory. Then *any* instance can serve *any* request, you can use plain round-robin, and you can add/remove instances freely. Sticky sessions are a workaround; shared session storage is the architecture.

### 7.5 Zero-downtime backend deploys **[I/A]**

To deploy a new version of your app with **no dropped requests**, you drain old instances gracefully:

1. Start the **new** instances and add them to the upstream (or run them in parallel containers).
2. Mark the **old** instances `down` in the upstream and `nginx -s reload` — Nginx stops sending them new requests but lets in-flight ones finish (because reload drains old workers gracefully, §3.1).
3. Once old instances are idle, stop them.

This is the **blue-green / rolling deploy** pattern. In Kubernetes the **Ingress controller** (often Nginx itself, §13.4) automates this via readiness probes and rolling updates; in Docker Compose you scale up the new service, reload Nginx, scale down the old (see the **Docker guide**).

---

## 8. API Gateway Patterns

### 8.1 What an "API gateway" means here **[A]**

An **API gateway** is a reverse proxy specialized for APIs: it's the single front door for many backend services, and it handles **cross-cutting API concerns** so each service doesn't have to — **routing** (which service handles which path/host), **authentication/authorization**, **rate limiting**, **request/response transformation**, **CORS**, **request validation/size limits**, **versioning**, and **uniform error responses**. Nginx is a capable, fast API gateway for the common cases; for very dynamic or programmable logic, teams reach for **OpenResty** (Lua), **Kong** (built on OpenResty), or **Envoy**. This section shows the patterns in pure Nginx.

### 8.2 Routing to multiple services by path and host **[A]**

The fundamental gateway job: map URL paths (or hostnames) to different backend services.

```nginx
upstream users_svc   { server 10.0.0.41:3001; keepalive 16; }
upstream orders_svc  { server 10.0.0.42:3002; keepalive 16; }
upstream payments_svc{ server 10.0.0.43:3003; keepalive 16; }

server {
    listen 443 ssl;
    server_name api.example.com;
    # (TLS config from §9 omitted for brevity)

    # Route by PATH PREFIX to the right microservice. Strip the prefix so each service sees clean paths.
    location /users/    { proxy_pass http://users_svc/;    include snippets/proxy-headers.conf; }
    location /orders/   { proxy_pass http://orders_svc/;   include snippets/proxy-headers.conf; }
    location /payments/ { proxy_pass http://payments_svc/; include snippets/proxy-headers.conf; }

    # Anything else under the API host → a 404 JSON (see §8.8 for clean JSON errors)
    location / { return 404 '{"error":"not found"}'; default_type application/json; }
}
```

You can equally route by **host** (`api.example.com` vs `admin.example.com`) using separate `server` blocks with different `server_name`s.

### 8.3 Rate limiting — the leaky bucket, burst, and nodelay explained **[A]**

**Rate limiting** protects your services from abuse, accidental floods, and brute-force attacks by capping how fast a client can make requests. Nginx implements it with the **leaky-bucket algorithm**, and understanding the model is essential to configuring it correctly.

**The leaky bucket:** imagine each client has a bucket. Requests pour in at the top; the bucket **leaks** (processes requests) at a *constant configured rate* (say 10 req/s). If requests arrive faster than the leak rate, the bucket fills. Once full, extra requests **overflow** and are rejected (HTTP 503). The bucket's size is the **burst**.

Two directives: `limit_req_zone` (defined at `http` level — declares the rate and a shared memory zone keyed by something, usually client IP) and `limit_req` (applied in a `location`/`server` — references the zone and sets the burst).

```nginx
http {
    # Define a zone: key on the client IP, 10 MB of shared memory (~160k IPs), leak rate 10 req/s.
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    #              └ key (binary IP is  └ zone name+size      └ steady-state allowed rate
    #                compact)

    # A stricter zone for login (brute-force protection): 5 requests per MINUTE per IP.
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
}

server {
    location /api/ {
        # Apply the limit. burst = bucket size = how many requests can queue above the steady rate.
        limit_req zone=api_limit burst=20 nodelay;
        #                        │         └ nodelay: serve the burst IMMEDIATELY (don't space them out),
        #                        │           but still reject anything beyond rate+burst. Best for APIs.
        #                        └ allow short spikes of up to 20 queued requests above 10/s
        proxy_pass http://api_backend;
    }

    location /login {
        limit_req zone=login_limit burst=3 nodelay;   # 5/min steady, allow a small burst of 3
        limit_req_status 429;        # return 429 Too Many Requests instead of the default 503
        proxy_pass http://auth_backend;
    }
}
```

**`burst` vs `nodelay` — the key distinction:**
- `burst=20` **without** `nodelay`: excess requests are *queued* and released at the steady rate (smooths spikes but adds latency — they wait their turn).
- `burst=20` **with** `nodelay`: excess requests (up to the burst) are served *immediately*, and only requests beyond rate+burst are rejected. **This is what you usually want for APIs** — real traffic is bursty, and you don't want to add artificial delay, you just want to cap sustained abuse.

> **Why `$binary_remote_addr` not `$remote_addr`?** The binary form is a fixed 4 bytes (IPv4), so the shared-memory zone holds far more clients per MB. Use it for IP-keyed limits.

### 8.4 Connection limiting **[A]**

Separately from *request rate*, you can cap **concurrent connections** per client with `limit_conn` — defends against a single client opening hundreds of slow connections (a Slowloris-style attack):

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;   # track connections per IP
}
server {
    location / {
        limit_conn conn_per_ip 20;   # at most 20 simultaneous connections from one IP
    }
    location /download/ {
        limit_conn conn_per_ip 2;    # downloads: only 2 parallel per IP
        limit_rate 1m;               # and cap each connection's bandwidth to 1 MB/s (be a good neighbour)
    }
}
```

### 8.5 Auth gateway with `auth_request` (subrequest authentication) **[A]**

A powerful gateway pattern: **centralize authentication** by having Nginx make an internal **subrequest** to a dedicated auth service *before* proxying to the real backend. The auth service inspects the token/cookie and answers `200` (allow) or `401`/`403` (deny). The backend never sees unauthenticated traffic, and every service is protected uniformly without each one re-implementing auth.

The logic of `auth_request`: for each incoming request, Nginx fires a **subrequest** to the URL you name. If that subrequest returns **2xx**, the original request proceeds; if **401/403**, Nginx returns that to the client and the original request is **never proxied**. You can also copy headers the auth service returns (e.g. a resolved user ID) onto the upstream request.

```nginx
server {
    location /api/ {
        auth_request /_auth;                 # ← before proxying, ask the auth service (subrequest below)

        # Pull values the auth service returned (e.g. the authenticated user id) and forward them upstream:
        auth_request_set $user_id $upstream_http_x_user_id;
        proxy_set_header X-User-Id $user_id;

        proxy_pass http://api_backend;       # only reached if /_auth returned 2xx
    }

    # The internal auth subrequest target — not reachable by clients (note `internal`).
    location = /_auth {
        internal;
        proxy_pass http://auth_service/verify;   # your auth microservice validates the token/cookie
        proxy_pass_request_body off;             # the auth check doesn't need the body…
        proxy_set_header Content-Length "";      # …so don't forward it (faster, safer)
        proxy_set_header X-Original-URI $request_uri;  # tell the auth svc what was requested (for ACLs)
    }

    # Optional: turn a 401 from auth into a friendly redirect to a login page.
    error_page 401 = @login_redirect;
    location @login_redirect { return 302 https://example.com/login; }
}
```

(The auth service itself is a tiny app — see the **Go JWT/Argon2 guide** or a **Fastify/Node** service. This pattern is exactly how `oauth2-proxy` and many SSO gateways integrate with Nginx.)

### 8.6 CORS handling at the gateway **[A]**

**CORS** (Cross-Origin Resource Sharing) controls whether a browser lets JavaScript on `site-a.com` call your API on `api.site-b.com`. Handling it at the gateway keeps every backend free of CORS code. You must answer the browser's **preflight** `OPTIONS` request and add the right headers to real responses:

```nginx
location /api/ {
    # Reflect/allow the calling origin (use a strict allowlist in production, NOT "*" with credentials).
    add_header Access-Control-Allow-Origin "https://app.example.com" always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
    add_header Access-Control-Allow-Credentials "true" always;   # if you send cookies/Authorization
    add_header Access-Control-Max-Age 86400 always;              # cache the preflight for a day

    # The browser PREFLIGHT: an OPTIONS request that must get a 204 with the headers above, no body.
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "https://app.example.com" always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
        add_header Access-Control-Max-Age 86400 always;
        return 204;     # answer the preflight immediately; don't proxy it to the backend
    }

    proxy_pass http://api_backend;
}
```

> **Security:** **never** combine `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true` — browsers forbid it and it's a data-leak footgun. Use an explicit origin allowlist (a `map` of allowed origins → reflected value is the clean way for multiple origins).

### 8.7 Request size limits, versioning, header manipulation **[A]**

- **Request size limits:** `client_max_body_size` (§6.4) caps upload size per location — set small (e.g. `100k`) for JSON APIs to reject oversized payloads early; larger only on upload endpoints.
- **API versioning:** route `/v1/` and `/v2/` to different upstreams, or rewrite a version header to a path. Path-based is simplest:
  ```nginx
  location /v1/ { proxy_pass http://api_v1/; }
  location /v2/ { proxy_pass http://api_v2/; }
  ```
- **Header manipulation:** strip sensitive internal headers before they reach clients (`proxy_hide_header X-Powered-By;`), add request IDs for tracing (`proxy_set_header X-Request-Id $request_id;` — `$request_id` is a unique per-request ID), and pass them back (`add_header X-Request-Id $request_id always;`) so you can correlate client reports with logs.

### 8.8 Returning clean JSON errors **[A]**

A polished API returns **JSON** errors, not Nginx's HTML error pages, so clients can parse them uniformly. Use `error_page` to route errors to named locations that emit JSON:

```nginx
server {
    # Map Nginx-generated errors to JSON responses:
    error_page 401 = @err401;
    error_page 403 = @err403;
    error_page 404 = @err404;
    error_page 429 = @err429;
    error_page 500 502 503 504 = @err5xx;

    location @err401 { default_type application/json; return 401 '{"error":"unauthorized"}'; }
    location @err403 { default_type application/json; return 403 '{"error":"forbidden"}'; }
    location @err404 { default_type application/json; return 404 '{"error":"not found"}'; }
    location @err429 { default_type application/json; return 429 '{"error":"rate limited"}'; }
    location @err5xx { default_type application/json; return 503 '{"error":"service unavailable"}'; }
}
```

---

## 9. TLS / SSL, HTTP/2 & HTTP/3

### 9.1 What TLS termination is and why Nginx does it **[I]**

**TLS** (the protocol behind HTTPS; "SSL" is its obsolete predecessor name, still used colloquially) encrypts traffic between client and server, and authenticates the server via a **certificate**. **TLS termination** means Nginx is the endpoint that does the encryption/decryption: clients connect to Nginx over HTTPS, Nginx decrypts, and talks **plain HTTP** to your backends over the trusted private network. This centralizes cert management, offloads crypto from your apps, and lets you enforce one modern TLS policy everywhere.

(If the backend network is *not* trusted — e.g. across data centers — you **re-encrypt**: Nginx terminates the client's TLS and opens a *new* TLS connection to the backend with `proxy_pass https://...`. This is "TLS re-encryption" / end-to-end TLS.)

### 9.2 Getting a certificate — Let's Encrypt / certbot **[I]**

You need a **certificate** (proving you own the domain) and its **private key**. The standard free, automated source is **Let's Encrypt** via the **certbot** tool, which proves domain ownership (the ACME protocol), fetches a 90-day cert, installs it, and auto-renews:

```bash
# Install certbot + its Nginx plugin, then issue & auto-configure a cert for your domain:
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
#   certbot proves you control the domain (HTTP-01 challenge), gets the cert, and EDITS your Nginx
#   config to add the ssl_certificate lines + an HTTP→HTTPS redirect. It also installs a renewal timer.

sudo certbot renew --dry-run   # test that automatic renewal works (certs last 90 days; renew ~every 60)
```

For containers/automation, **`acme.sh`** or **Caddy** (auto-HTTPS) are popular alternatives; many setups also terminate TLS at a cloud load balancer or Cloudflare instead.

### 9.3 A modern, hardened HTTPS server block **[I/A]**

Here is a complete, production-grade TLS server with every line explained, plus the HTTP→HTTPS redirect:

```nginx
# ── Redirect ALL http to https (so users typing http:// land on the secure site) ──
server {
    listen 80;
    listen [::]:80;                       # also IPv6
    server_name example.com www.example.com;
    return 301 https://$host$request_uri; # permanent redirect, preserving host + path + query
}

# ── The real HTTPS server ──
server {
    listen 443 ssl;                       # HTTPS on IPv4
    listen [::]:443 ssl;                  # HTTPS on IPv6
    http2 on;                             # enable HTTP/2 (the MODERN directive; not "listen ... http2")
    server_name example.com www.example.com;

    # --- Certificate + private key (from certbot, or your CA) ---
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;  # cert + intermediate chain
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;    # the PRIVATE key (chmod 600!)

    # --- Protocol & cipher policy: modern, secure defaults ---
    ssl_protocols TLSv1.2 TLSv1.3;        # disable everything below 1.2 (1.0/1.1 are broken/deprecated)
    ssl_prefer_server_ciphers off;        # for TLS 1.3, let the client pick (its order is fine); 1.2 uses
                                          #   the strong suites below regardless
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
                                          #   ↑ only forward-secret (ECDHE) AEAD suites — no RC4/3DES/CBC

    # --- Session resumption (faster reconnects) ---
    ssl_session_cache shared:SSL:10m;     # shared cache of TLS sessions across workers (~40k sessions)
    ssl_session_timeout 1d;
    ssl_session_tickets off;              # disable tickets unless you rotate keys — they can weaken PFS

    # --- OCSP stapling: Nginx fetches the cert's revocation status and "staples" it to the handshake,
    #     so the CLIENT doesn't have to contact the CA (faster + more private). ---
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;  # chain to verify the OCSP reply
    resolver 1.1.1.1 8.8.8.8 valid=300s;  # Nginx needs DNS to reach the OCSP responder

    # --- HSTS: tell browsers "only ever use HTTPS for this domain" for 2 years. ---
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    #   CAUTION: HSTS is sticky. Only enable once you're sure ALL subdomains do HTTPS, or you'll lock
    #   users out of http-only subdomains. "preload" submits you to browsers' built-in HSTS list.

    root /var/www/example;
    location / { try_files $uri $uri/ =404; }
}
```

### 9.4 HTTP/2 — what it buys you **[I]**

**HTTP/2** is a binary, multiplexed evolution of HTTP/1.1 over the same TLS connection. The headline win is **multiplexing**: many requests/responses share ONE connection concurrently, eliminating HTTP/1.1's head-of-line blocking and the need for hacks like domain sharding. It also adds header compression (HPACK) and stream prioritization. Enabling it is one directive (`http2 on;` per server, as above). Browsers only use HTTP/2 over HTTPS, so it rides on your TLS config.

> **⚡ Version note:** since **1.25.1** you enable HTTP/2 with the standalone **`http2 on;`** directive in the `server` block. The old `listen 443 ssl http2;` parameter form still parses but is deprecated — use `http2 on;`.

### 9.5 HTTP/3 and QUIC **[A]**

**HTTP/3** runs over **QUIC**, a transport built on **UDP** (not TCP) with TLS 1.3 baked in. Why it matters: QUIC eliminates **TCP head-of-line blocking** (a lost packet in HTTP/2 stalls *all* multiplexed streams; in QUIC only the affected stream stalls), has **faster connection setup** (0-RTT/1-RTT handshakes), and **survives network changes** (connection migration — your phone switching Wi-Fi→cellular keeps the connection). It's especially good on lossy mobile networks.

```nginx
server {
    # HTTP/3 needs a build with --with-http_v3_module (check `nginx -V`). It listens on UDP/443.
    listen 443 ssl;                  # HTTP/1.1 + HTTP/2 over TCP
    listen 443 quic reuseport;       # HTTP/3 over QUIC (UDP). reuseport balances UDP across workers.
    listen [::]:443 ssl;
    listen [::]:443 quic reuseport;

    http2 on;                        # HTTP/2 over the TCP listener
    http3 on;                        # HTTP/3 over the QUIC listener
    ssl_protocols TLSv1.3;           # QUIC REQUIRES TLS 1.3 (1.2 won't work over QUIC)

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Advertise HTTP/3 to clients so browsers know to upgrade from H2 → H3 (the Alt-Svc header):
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    server_name example.com;
    root /var/www/example;
}
```

> **⚡ Firewall note:** HTTP/3 is **UDP/443** — you must open UDP 443 in your firewall/security group (people forget, then wonder why H3 never activates). Browsers fall back to H2 over TCP if QUIC is blocked, so it fails silently.

### 9.6 TLS hardening checklist **[A]**

| Setting | Recommendation | Why |
|---|---|---|
| `ssl_protocols` | `TLSv1.2 TLSv1.3` only | TLS 1.0/1.1 are deprecated and exploitable. |
| Ciphers | ECDHE + AEAD (GCM/ChaCha20) only | Forward secrecy + no padding-oracle/CBC weaknesses. |
| HSTS | Enable (carefully) | Forces HTTPS, blocks SSL-strip downgrade attacks. |
| OCSP stapling | On | Faster handshakes, client privacy, fewer CA round-trips. |
| Key file perms | `600`, owner root | The private key is the crown jewel — never world-readable. |
| `ssl_dhparam` | 2048-bit+ (if using DHE) | Weak DH params enable Logjam; ECDHE avoids this entirely. |
| Cert renewal | Automated (certbot timer) | 90-day certs WILL expire and take your site down if manual. |

Use the **Mozilla SSL Configuration Generator** mindset ("Intermediate" profile) as your baseline, and test with `openssl s_client -connect example.com:443` offline or SSL Labs online.

---

## 10. Caching & Compression

### 10.1 Why cache at the proxy **[I]**

Nginx can **store backend responses** and serve subsequent identical requests from its own cache **without hitting the backend at all**. This slashes backend load and latency: a response computed once serves thousands of clients. The trade-off is **staleness** — cached data can lag behind the source — so caching is about choosing *what* is safe to cache and *for how long*.

### 10.2 `proxy_cache` — zones, keys, validity **[I/A]**

You declare a **cache zone** on disk (with `proxy_cache_path` at `http` level) and enable it per location. The **cache key** determines what counts as "the same request."

```nginx
http {
    # Define a cache: store files under /var/cache/nginx, with a 10MB in-memory index of keys ("keys_zone"),
    # cap the on-disk cache at 1GB, and evict entries not accessed in 24h (inactive).
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=app_cache:10m
                     max_size=1g inactive=24h use_temp_path=off;
    #                levels=1:2 = directory nesting (avoids one huge dir); use_temp_path=off = write
    #                straight into the cache dir (faster).
}

server {
    location / {
        proxy_cache app_cache;                       # use the zone defined above
        proxy_cache_key "$scheme$request_method$host$request_uri";  # what makes a request "the same"

        # How long to cache, BY RESPONSE STATUS:
        proxy_cache_valid 200 301 302 10m;           # cache successful pages for 10 minutes
        proxy_cache_valid 404 1m;                     # cache 404s briefly (avoid hammering backend for them)

        # Serve STALE content while refreshing / when the backend is down (resilience + speed):
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;             # refresh expired entries in the background…
        proxy_cache_lock on;                          # …and let only ONE request refresh a key (stop a
                                                      #   "cache stampede" of simultaneous backend hits)

        add_header X-Cache-Status $upstream_cache_status;  # expose HIT/MISS/BYPASS/STALE for debugging
        proxy_pass http://app_backend;
    }
}
```

`$upstream_cache_status` in your logs/headers tells you what happened: **HIT** (served from cache), **MISS** (fetched from backend, now cached), **BYPASS** (cache deliberately skipped), **EXPIRED**, **STALE**, **UPDATING**. Watching your HIT ratio is how you tune caching.

### 10.3 Honoring (or ignoring) Cache-Control **[I/A]**

By default Nginx respects the backend's `Cache-Control`/`Expires` headers — if the backend says `no-cache`, Nginx won't cache. You can override:

```nginx
location / {
    proxy_cache app_cache;
    proxy_ignore_headers Cache-Control Expires Set-Cookie;  # force caching even if backend says no
    proxy_cache_valid 200 5m;
    # Don't cache requests/responses that carry a session cookie (per-user content must not be shared!):
    proxy_no_cache       $http_authorization $cookie_sessionid;
    proxy_cache_bypass   $http_authorization $cookie_sessionid;
}
```

> **Critical safety rule:** **never cache per-user/authenticated content under a shared key**, or you'll serve User A's private page to User B. Bypass the cache when a session cookie or `Authorization` header is present (as above), or include the user identity in the cache key.

### 10.4 Microcaching — caching dynamic content for *seconds* **[A]**

A powerful trick for high-traffic dynamic sites: cache even "uncacheable" dynamic pages for a **very short time** (1–10 seconds). At 1000 req/s, caching for just **1 second** means the backend handles ~1 request/s instead of 1000 — a 1000× reduction — while users see data at most 1 second stale. This is **microcaching**:

```nginx
location / {
    proxy_cache app_cache;
    proxy_cache_valid 200 1s;          # cache for ONE second — backend hit at most once per second
    proxy_cache_lock on;               # collapse the stampede so only one request refreshes
    proxy_cache_use_stale updating;    # serve the 1s-old copy while the refresh happens
    proxy_pass http://app_backend;
}
```

### 10.5 Cache purging **[A]**

Invalidating cache entries on demand (e.g. after a content update) is **`proxy_cache_purge` — Plus only** as a directive. On open-source Nginx the common approaches are: (a) the third-party **`ngx_cache_purge`** module (added at build time), (b) deleting cache files matching the key from disk, or (c) versioning your cache key (append a content version so a bump effectively invalidates). For static assets, the cleanest "purge" is **cache-busting filenames** (`app.abc123.js`) — change the hash, change the URL, the old cache entry is simply never requested again (§10.6).

### 10.6 Browser caching headers **[I]**

Separate from *proxy* caching is telling the **client's browser** how long to keep assets, so repeat visits don't re-download them. Use `expires` and `Cache-Control`:

```nginx
# Long-cache fingerprinted static assets (their filename changes when content changes → safe to cache forever)
location ~* \.(?:css|js|woff2|png|jpg|jpeg|gif|svg|ico)$ {
    expires 1y;                                   # set Expires + Cache-Control: max-age for 1 year
    add_header Cache-Control "public, immutable"; # 'immutable' = browser won't even revalidate
    access_log off;
}

# NEVER long-cache index.html (it references the fingerprinted assets and must always be fresh):
location = /index.html {
    add_header Cache-Control "no-cache";   # always revalidate so users get new asset references promptly
}
```

This is the modern SPA caching strategy: `index.html` is `no-cache` (tiny, always fresh, points at the latest hashed assets), while the hashed assets are `immutable` for a year.

### 10.7 Compression: gzip and brotli **[I]**

Compressing text responses (HTML/CSS/JS/JSON) before sending cuts bandwidth and speeds page loads dramatically. **gzip** is built into Nginx; **brotli** (better ratios, especially for static assets) needs the `ngx_brotli` module compiled in.

```nginx
http {
    gzip on;                       # enable gzip
    gzip_comp_level 5;             # 1 (fast/low) … 9 (slow/high). 4–6 is the sweet spot for CPU vs size.
    gzip_min_length 1024;          # don't bother compressing tiny responses (<1KB) — overhead > savings
    gzip_vary on;                  # send "Vary: Accept-Encoding" so caches store gzip + non-gzip variants
    gzip_proxied any;              # also compress responses that came from a proxied backend
    gzip_types text/plain text/css application/json application/javascript
               application/xml text/xml image/svg+xml font/woff2;
    #   ↑ only TEXT types — never gzip already-compressed formats (jpg/png/woff2-glyf/zip): wasted CPU.
    #   (text/html is ALWAYS compressed and need not be listed.)

    # If the brotli module is available, prefer it (better ratios) with gzip as fallback:
    # brotli on;
    # brotli_comp_level 5;
    # brotli_types text/css application/javascript application/json image/svg+xml;
}
```

> **Security (BREACH):** compressing responses that mix **secrets and attacker-controlled input** (e.g. a CSRF token in a page that also reflects a query param) can leak the secret via the **BREACH** attack. Mitigations: don't reflect user input into secret-bearing responses, keep CSRF tokens out of compressed bodies, or disable compression on sensitive endpoints. For most static/JSON APIs this isn't a concern, but know it exists.

### 10.8 `fastcgi_cache` (PHP and FastCGI backends) **[A]**

When Nginx talks to a **FastCGI** backend (classically **PHP-FPM**) instead of an HTTP upstream, the cache directives are the `fastcgi_*` siblings (`fastcgi_cache_path`, `fastcgi_cache`, `fastcgi_cache_valid`) — identical concepts to `proxy_cache`, just for the FastCGI protocol. A WordPress front-end cache is the textbook use:

```nginx
http {
    fastcgi_cache_path /var/cache/nginx/fcgi keys_zone=php_cache:10m inactive=60m;
}
server {
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;   # PHP-FPM over a Unix socket
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

        fastcgi_cache php_cache;
        fastcgi_cache_valid 200 10m;
        fastcgi_cache_bypass $cookie_logged_in;       # logged-in users skip the cache (see their own data)
        add_header X-FastCGI-Cache $upstream_cache_status;
    }
}
```

---

## 11. Security Hardening

This is a dedicated checklist-style section. Apply these by default on any internet-facing Nginx.

### 11.1 Hide the version and the server token **[I]**

By default Nginx advertises its exact version in the `Server: nginx/1.27.3` header and on error pages, helping attackers fingerprint known-vulnerable versions. Turn it off:

```nginx
http {
    server_tokens off;   # send just "Server: nginx" (no version) and drop the version from error pages
}
# (Fully removing the "Server" header requires the headers-more module: more_clear_headers Server;)
```

### 11.2 Security response headers **[I/A]**

These headers instruct the browser to enable protections. Add them globally (remember the `add_header` replacement rule from §3.4 — re-declare in any context that adds its own headers, and use `always` so they're sent on error responses too):

```nginx
# A reusable snippet (include it in each server/location that sets headers):
add_header X-Content-Type-Options "nosniff" always;
#   Stops the browser from MIME-sniffing a response into a different Content-Type (a classic XSS vector).

add_header X-Frame-Options "SAMEORIGIN" always;
#   Prevents your site being embedded in an <iframe> on another site → blocks CLICKJACKING.

add_header Referrer-Policy "strict-origin-when-cross-origin" always;
#   Controls how much of the URL is leaked in the Referer header to other sites (privacy).

add_header Content-Security-Policy "default-src 'self'; img-src 'self' data:; script-src 'self'" always;
#   THE big one: a CSP whitelists where scripts/styles/images may load from → strongest XSS defense.
#   Start strict, then loosen per real needs. Test in Report-Only mode first (Content-Security-Policy-Report-Only).

add_header Permissions-Policy "geolocation=(), camera=(), microphone=()" always;
#   Disable powerful browser features your site doesn't use.

add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;  # HTTPS only (see §9.3)
```

> **Note:** `X-XSS-Protection` is **obsolete** (modern browsers removed the auditor it controlled; a strong CSP replaces it). Don't rely on it.

### 11.3 Rate limiting and connection limits **[I/A]**

Already covered in §8.3–8.4 — applied for *security*, rate/connection limits blunt brute-force, scraping, and DoS. **Always** rate-limit auth endpoints (`/login`, `/reset-password`, token endpoints) aggressively.

### 11.4 Blocking by IP and geography **[I/A]**

```nginx
# Allow/deny by IP/CIDR (e.g. lock an admin panel to the office network):
location /admin/ {
    allow 203.0.113.0/24;     # office range
    allow 10.0.0.0/8;         # VPN
    deny  all;                # everyone else gets 403
    proxy_pass http://admin_backend;
}

# Geo-blocking with the geo module (block/allow by country needs the GeoIP2 module + a MaxMind DB):
# geo $blocked_country { default 0;  CN 1;  RU 1; }   # then: if ($blocked_country) { return 403; }
# Better: use the geoip2 module to map $remote_addr → country, then deny in a map. (Use sparingly — IP
# geolocation is imperfect and VPNs bypass it; it's a coarse filter, not real security.)
```

### 11.5 Request filtering and method limiting **[I/A]**

```nginx
# Only permit the HTTP methods your API actually uses; reject the rest with 405:
location /api/ {
    limit_except GET POST PUT DELETE {    # everything NOT in this list…
        deny all;                         # …is denied (e.g. TRACE, exotic methods used in attacks)
    }
    proxy_pass http://api_backend;
}

# Block requests for hidden/sensitive files (dotfiles, VCS, backups) anywhere on the site:
location ~ /\.(?!well-known) { deny all; }   # block .git, .env, .htpasswd… but allow /.well-known/
location ~* \.(bak|sql|conf|log|old)$ { deny all; }  # never serve backups/dumps/configs by accident

# Reject requests with no/empty Host header (often bots/scanners) via a default_server that 444s:
server {
    listen 80 default_server;
    server_name _;             # catch-all
    return 444;                # 444 = Nginx-specific "close connection without a response"
}
```

### 11.6 Run as non-root; least privilege **[I/A]**

As covered in §3.1, the **master runs as root** (to bind 80/443 and read keys) but **workers must run as an unprivileged user** (`user www-data;` / `user nginx;`). Never set `user root;`. In Docker, the official image's workers already drop to `nginx`; for fully rootless containers there's `nginxinc/nginx-unprivileged` (binds to 8080 instead of 80). Keep TLS private keys `chmod 600`, owned by root, readable only by the master. Don't give the worker user write access to the document root (a compromised app shouldn't be able to overwrite served files).

### 11.7 ModSecurity / WAF **[A]**

A **Web Application Firewall (WAF)** inspects requests for attack patterns (SQL injection, XSS, path traversal) and blocks them before they reach your app. **ModSecurity** (the `ModSecurity-nginx` connector) with the **OWASP Core Rule Set (CRS)** is the classic open-source WAF for Nginx; it's compiled in as a module. It adds latency and false-positives to tune, so it's an *additional* layer, not a substitute for secure app code. Cloud WAFs (Cloudflare, AWS WAF) are the managed alternative and often sit *in front of* Nginx.

### 11.8 Preventing common attacks — a summary table **[A]**

| Attack | Nginx mitigation |
|---|---|
| **Slowloris** (slow-drip connections) | `limit_conn`, `client_body_timeout`, `client_header_timeout`, `send_timeout` (short); Nginx's event model already resists this well. |
| **Brute force** (login) | Aggressive `limit_req` on auth endpoints; `auth_request` gateway; fail2ban on logs. |
| **DDoS (volumetric)** | `limit_req`/`limit_conn` help at L7, but true volumetric DDoS needs an upstream provider (Cloudflare, scrubbing). |
| **Clickjacking** | `X-Frame-Options`/CSP `frame-ancestors`. |
| **XSS** | Strong CSP, `X-Content-Type-Options: nosniff`. |
| **MIME sniffing** | `X-Content-Type-Options: nosniff`, correct `Content-Type`. |
| **Path traversal / dotfile leaks** | `location ~ /\.` deny rules; careful `alias`; `internal`. |
| **SSL stripping / downgrade** | HSTS, redirect HTTP→HTTPS, TLS 1.2+ only. |
| **Large-payload DoS** | `client_max_body_size`, `client_body_buffer_size`. |
| **Host header injection** | A `default_server` that `return 444;`s unknown hosts; validate `server_name`. |

---

## 12. Performance Tuning

### 12.1 Workers and connections **[A]**

```nginx
worker_processes auto;        # one worker per CPU core (auto = detect). The whole point of the event
                              #   model is N workers saturating N cores; more than cores just adds switching.
worker_rlimit_nofile 65535;   # raise the per-worker open-file-descriptor limit (each connection + each
                              #   open file/socket uses an FD). Must also raise the OS limit (ulimit -n /
                              #   systemd LimitNOFILE), or workers hit "too many open files" under load.

events {
    worker_connections 16384; # max simultaneous connections PER WORKER. Theoretical max clients ≈
                              #   worker_processes × worker_connections (halve it for proxying, since each
                              #   client connection also uses a connection to the backend).
    multi_accept on;          # let a worker accept all pending new connections at once (good under bursts)
}
```

**The capacity formula:** `max connections ≈ worker_processes × worker_connections`. For a **reverse proxy**, each client request also opens (or reuses, with keepalive) a backend connection, so effective client capacity is roughly **half** — budget accordingly. And `worker_connections` can't exceed `worker_rlimit_nofile`, which can't exceed the OS `ulimit -n`. Tune all three together.

### 12.2 Efficient file delivery **[A]**

```nginx
http {
    sendfile on;        # kernel copies file → socket directly, skipping a user-space buffer (zero-copy).
                        #   Big win for static files. (No-op on the Windows build.)
    tcp_nopush on;      # only meaningful WITH sendfile: send the response headers and the start of the
                        #   file in FULL packets (fills packets before sending) → fewer, fuller packets.
    tcp_nodelay on;     # disable Nagle's algorithm so small packets (the last chunk, WS frames, keepalive)
                        #   send IMMEDIATELY instead of waiting to coalesce. Nginx toggles these smartly:
                        #   nopush while sending the body, nodelay for the final packet/keepalive.

    # Cache open file descriptors + metadata so Nginx doesn't stat() every static file on every request:
    open_file_cache max=10000 inactive=60s;   # cache up to 10k file handles, drop ones unused for 60s
    open_file_cache_valid 60s;                # re-check cached metadata every 60s
    open_file_cache_min_uses 2;               # only cache files requested at least twice
    open_file_cache_errors on;                # also cache "not found" results (avoid repeated failed stats)
}
```

### 12.3 Keepalive and buffer sizing **[A]**

```nginx
http {
    keepalive_timeout 65;          # hold idle CLIENT keep-alive connections 65s (reuse beats re-handshake)
    keepalive_requests 1000;       # max requests per kept-alive connection before Nginx closes it

    # Client request buffers — size to your typical request so Nginx doesn't spill to temp files:
    client_body_buffer_size 16k;   # buffer for the request body in memory (bigger → fewer temp-file writes)
    client_header_buffer_size 1k;  # buffer for request headers
    large_client_header_buffers 4 8k;  # for big headers (large cookies/JWTs) — too small → 400/494 errors
    client_max_body_size 10m;      # global upload cap (override per-location for upload endpoints, §6.4)
}
```

> **Tuning philosophy:** don't cargo-cult huge numbers. Start with sane defaults, **measure** (look at `$request_time`/`$upstream_response_time` in logs, watch CPU/RAM/FD usage), and raise only the limits you actually hit. Most "Nginx is slow" turns out to be the *backend* being slow — `$upstream_response_time` vs `$request_time` tells you which.

---

## 13. Real-World Configs: SPA + API, Full Edge, Docker, k8s, Observability

### 13.1 Serving a SPA frontend + proxying its API **[A]**

The most common single-server setup: a built React/Vue SPA served statically, with `/api` proxied to a backend, all over HTTPS. This is the config you'll write most often:

```nginx
upstream app_api {
    server 127.0.0.1:3000;
    keepalive 32;
}

server { listen 80; server_name app.example.com; return 301 https://$host$request_uri; }  # force HTTPS

server {
    listen 443 ssl;
    http2 on;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;

    root /var/www/app/dist;          # the built SPA assets (vite build output)
    index index.html;

    include /etc/nginx/snippets/security-headers.conf;   # §11.2

    # 1) API requests → backend. Defined BEFORE the SPA fallback so /api never falls back to index.html.
    location /api/ {
        proxy_pass http://app_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";              # enable upstream keepalive (§6.5)
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
        client_max_body_size 10m;
    }

    # 2) Hashed static assets → cache hard (filenames are fingerprinted, so this is safe forever).
    location ~* \.(?:css|js|woff2|png|jpg|jpeg|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    # 3) Everything else → index.html so the client-side router handles the route (§4.4).
    location / {
        try_files $uri $uri/ /index.html;
    }

    # index.html itself must always be fresh (points at the latest hashed assets):
    location = /index.html { add_header Cache-Control "no-cache"; }
}
```

### 13.2 A full edge: reverse proxy + load balancer + TLS + cache + rate limit **[A]**

Everything together — the kind of config that fronts a real production service:

```nginx
# ── http-level setup ──────────────────────────────────────────────────────────
proxy_cache_path /var/cache/nginx keys_zone=edge_cache:20m max_size=2g inactive=1h use_temp_path=off;
limit_req_zone  $binary_remote_addr zone=req_per_ip:10m  rate=20r/s;
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;

map $http_upgrade $connection_upgrade { default upgrade; '' close; }   # for WebSockets (§6.6)

upstream backend_pool {
    least_conn;                                   # spread variable-duration requests evenly (§7.2)
    server 10.0.0.21:3000 max_fails=3 fail_timeout=15s;
    server 10.0.0.22:3000 max_fails=3 fail_timeout=15s;
    server 10.0.0.23:3000 max_fails=3 fail_timeout=15s;
    server 10.0.0.29:3000 backup;                 # failover node (§7.3)
    keepalive 64;
}

# ── HTTP→HTTPS redirect + reject unknown hosts ────────────────────────────────
server { listen 80 default_server; server_name _; return 301 https://$host$request_uri; }

# ── The HTTPS edge ────────────────────────────────────────────────────────────
server {
    listen 443 ssl;
    listen 443 quic reuseport;                    # HTTP/3 (§9.5)
    http2 on;
    http3 on;
    server_name www.example.com;

    ssl_certificate     /etc/letsencrypt/live/www.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;
    ssl_stapling on; ssl_stapling_verify on;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
    include /etc/nginx/snippets/security-headers.conf;

    server_tokens off;
    client_max_body_size 25m;
    limit_conn conn_per_ip 50;                    # cap concurrent connections per client (§8.4)

    # Health check endpoint — exact match, never logged, never proxied:
    location = /health { access_log off; return 200 "ok\n"; }

    # WebSocket endpoint:
    location /ws/ {
        proxy_pass http://backend_pool;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
    }

    # Cacheable API GETs with a microcache + rate limiting + full proxy headers:
    location /api/ {
        limit_req zone=req_per_ip burst=40 nodelay;     # rate limit (§8.3)

        proxy_cache edge_cache;
        proxy_cache_valid 200 10s;                      # microcache GETs for 10s (§10.4)
        proxy_cache_use_stale error timeout updating;
        proxy_cache_lock on;
        proxy_no_cache     $http_authorization;         # never cache authenticated requests (§10.3)
        proxy_cache_bypass $http_authorization;
        add_header X-Cache $upstream_cache_status always;

        proxy_pass http://backend_pool;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-Id      $request_id;     # tracing id (§8.7)
        proxy_next_upstream error timeout http_502 http_503; # retry a dead backend (§7.3)
    }

    # Static assets straight off disk:
    location / {
        root /var/www/www.example.com;
        try_files $uri $uri/ /index.html;
    }
}
```

### 13.3 Nginx in Docker Compose fronting an app + DB **[A]**

This stitches the **Docker guide**'s patterns together: Nginx terminates the edge, proxies to an app service by name, which talks to Postgres/Redis. (See **DOCKER_GUIDE §17** for the full multi-service treatment.)

```yaml
# compose.yaml
services:
  nginx:
    image: nginx:1.27-alpine
    ports: ["80:80", "443:443", "443:443/udp"]   # note UDP 443 for HTTP/3 (§9.5)
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro      # your server blocks (read-only)
      - ./nginx/snippets:/etc/nginx/snippets:ro
      - ./certs:/etc/nginx/certs:ro              # TLS certs (or use a sidecar like certbot/Caddy)
    depends_on: [api]
    restart: unless-stopped

  api:                                            # your Node/Fastify or Go/Gin backend
    image: my-api:latest
    expose: ["3000"]                              # reachable by nginx via http://api:3000, not by host
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on: [db, cache]
    restart: unless-stopped

  db:    { image: postgres:17-alpine, environment: { POSTGRES_PASSWORD: secret }, volumes: ["pgdata:/var/lib/postgresql/data"] }
  cache: { image: redis:7-alpine }

volumes:
  pgdata:
```

```nginx
# nginx/conf.d/default.conf — note proxy_pass to the SERVICE NAME "api" (Docker DNS resolves it)
server {
    listen 80;
    location / {
        proxy_pass http://api:3000;     # "api" = the Compose service name (see §2.4 resolver caveat)
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 13.4 Nginx as a Kubernetes Ingress (mention) **[A]**

In **Kubernetes**, you rarely write `nginx.conf` by hand. The **Ingress NGINX Controller** runs Nginx as a pod and **generates** the Nginx config from declarative `Ingress` resources (and `ingressClassName`, annotations, and `ConfigMap`s). You write Kubernetes YAML describing "host `foo.com` path `/api` → service `api-svc:3000`," and the controller turns it into upstreams and server blocks under the hood, reloading Nginx on changes. Knowing raw Nginx (this whole guide) makes those annotations (`nginx.ingress.kubernetes.io/proxy-body-size`, `…/rate-limit`, `…/ssl-redirect`) immediately legible — they map directly to the directives you've learned. (Envoy-based ingresses like Contour/Istio are the alternative in heavily dynamic meshes.)

### 13.5 Observability and debugging **[A]**

- **`nginx -T`** dumps the *entire effective config* (all includes resolved) — your first move when "a directive isn't doing what I think" (find where it's actually set or overridden).
- **`error_log … debug;`** (temporarily) shows the request-processing decisions, including *which location matched* — gold for §5 mysteries. Revert it after; it's huge.
- **Access-log timing fields** (`$request_time` vs `$upstream_response_time`, §4.7) isolate Nginx-slow from backend-slow. **`$upstream_cache_status`** shows cache effectiveness.
- **The `stub_status` module** exposes live counters (active connections, accepts, handled, requests) for monitoring:
  ```nginx
  location = /nginx_status {
      stub_status;
      allow 127.0.0.1;     # expose ONLY to localhost / your monitoring host
      deny all;
  }
  ```
  Scrape it with the **Prometheus nginx exporter** (or use the structured-metrics modules) to graph traffic, error rates, and connection counts. (Nginx **Plus** ships a richer live dashboard + JSON status API.)
- **Log analysis offline:** tools like `goaccess` parse your access log into a live terminal/HTML dashboard (top URLs, status codes, bandwidth) without any external service — ideal for offline ops.

---

## 14. Gotchas & Best Practices

A consolidated list of the traps that cost people hours. Skim this periodically.

### 14.1 The reload/test workflow **[I]**
- **Always `nginx -t` before `nginx -s reload`.** A bad config that passes startup but you forgot to test can still bite on a future restart. Make `-t` a reflex.
- Prefer `reload` over `restart` (zero-downtime vs dropped connections).
- After editing, `nginx -T | grep -n something` to confirm your directive is where you think (in the *effective* config).

### 14.2 `root` vs `alias`, trailing slashes **[I]**
- `root` **appends** the URI; `alias` **replaces** the location prefix (§4.2). Mixing them up = 404s.
- Match trailing slashes between `location` and `alias`. Avoid `alias` in regex locations.
- On `proxy_pass`, a **trailing slash strips** the location prefix; no slash **preserves** the full path (§6.2). This silently changes the backend path.

### 14.3 "`if` is evil" **[I/A]**
- Inside a `location`, `if` has surprising semantics (it can break `try_files`, `add_header`, and more). The Nginx wiki literally titles a page "If Is Evil." **Prefer `try_files`, `map`, `return`, and `error_page`** over `if`. Safe uses of `if`: a bare `return`/`rewrite ... last` at the top of a server/location. Anything fancier — reach for a `map` instead.

### 14.4 `add_header` replacement **[I]**
- `add_header` in a child context **wipes** inherited `add_header`s (§3.4). Re-declare them, or use an `include`d snippet. Use **`always`** so headers appear on error responses (4xx/5xx), not just 2xx/3xx.

### 14.5 Location matching surprises **[I]**
- A **regex location can steal** requests from a prefix you thought owned them (§5.4). Use `^~` to protect a static tree from greedy regexes.
- `=` exact match short-circuits everything — use it for `/health`, `/favicon.ico`.
- Regexes match in **file order** (first wins); prefixes match by **length** (longest wins). Different rules!

### 14.6 Proxy & DNS **[I/A]**
- Forgetting `proxy_set_header Host`/`X-Forwarded-Proto` causes redirect loops, wrong absolute URLs, and "app thinks it's on HTTP" bugs (§6.3).
- Forgetting `proxy_http_version 1.1;` + `proxy_set_header Connection "";` disables upstream keepalive (§6.5).
- Forgetting the **WebSocket Upgrade headers** breaks WS silently (§6.6).
- Nginx resolves upstream **hostnames once**; in Docker/k8s use a `resolver` + variable `proxy_pass`, or `upstream` blocks (§2.4, §6.7). Stale IP = sudden 502s.

### 14.7 Sizes, timeouts, TLS **[I]**
- **`client_max_body_size` defaults to 1 MB** — uploads bigger than that get **413** until you raise it (§6.4).
- **502 "upstream sent too big header"** → raise `proxy_buffer_size` (large cookies/JWTs).
- **HSTS is sticky** — only enable when *all* subdomains are HTTPS, or you lock users out (§9.3).
- **HTTP/3 needs UDP/443 open** in the firewall, or it silently never activates (§9.5).
- **Let's Encrypt certs expire in 90 days** — ensure the renewal timer actually runs (`certbot renew --dry-run`).

### 14.8 Best-practice recap **[I/A]**
- Modular config: `snippets/` for reusable header/proxy blocks; one `server` per site.
- Hide the version (`server_tokens off`), set security headers, rate-limit auth endpoints, run workers non-root, lock down dotfiles/backups.
- Cache aggressively but **never cache authenticated content under a shared key**.
- Measure before tuning; `$upstream_response_time` usually reveals the real bottleneck is the backend.
- Keep your app *stateless* and let Nginx load-balance freely; store sessions in Redis (cross-ref **Redis guide**).

---

## 15. Study Path & Build-to-Learn Projects

### 15.1 A suggested learning order

1. **[B] Static fundamentals.** Install Nginx (or run the `nginx:alpine` Docker image). Serve a folder of HTML/CSS/JS. Master `root` vs `alias`, `index`, `try_files`, MIME types, custom error pages, and reading the access/error logs. Do the edit → `nginx -t` → `reload` loop until it's muscle memory.
2. **[B/I] The config model.** Internalize contexts (main/events/http/server/location), the master/worker split, directive inheritance, and the `add_header` replacement trap. Use `nginx -T` to inspect the effective config.
3. **[I] `location` matching.** Build the worked example from §5.4 and *predict* every match before testing. This single skill removes most future confusion.
4. **[I] Reverse proxy.** Stand up a tiny Node/Fastify or Go/Gin "hello" API (see those guides) and put Nginx in front. Get the proxy headers right, add upstream keepalive, then proxy a WebSocket echo server.
5. **[I/A] Load balancing.** Run 2–3 copies of the API and balance across them. Try each algorithm, kill a backend and watch passive health checks route around it, add a `backup`.
6. **[A] API gateway.** Add rate limiting (watch `limit_req` reject under `ab`/`wrk` load), `auth_request` to a tiny auth service, CORS, and JSON error pages.
7. **[I/A] TLS + HTTP/2/3.** Issue a real cert with certbot (or a self-signed one for local), harden the TLS config, enable HTTP/2 then HTTP/3, verify with `curl --http3` and `openssl s_client`.
8. **[I/A] Caching & compression.** Add `proxy_cache`, watch `$upstream_cache_status` go HIT, try microcaching under load, enable gzip/brotli, set browser cache headers for a SPA.
9. **[A] Hardening & tuning.** Apply the §11 checklist; tune §12; add `stub_status` + a Prometheus exporter and graph it; run `goaccess` on your logs.
10. **[A] Containerize & orchestrate.** Put the whole thing in Docker Compose (§13.3, **Docker guide**), then read how the Kubernetes Ingress NGINX Controller generates this same config from YAML (§13.4).

### 15.2 Build-to-learn projects

- **Project 1 — Hardened static host.** Serve a multi-page static site over HTTPS (certbot), with security headers, gzip/brotli, fingerprinted-asset caching, custom 404/50x pages, and HTTP→HTTPS redirect. Grade your TLS config offline with `nginx -T` + `openssl`, online with SSL Labs.
- **Project 2 — SPA + API reverse proxy.** Front a React/Vue SPA and a backend API (Node/Go) from one Nginx server: `try_files` SPA fallback, `/api` proxy with correct headers + keepalive, a proxied WebSocket, and `client_max_body_size` for an upload endpoint (§13.1). Cross-reference the **Fastify** / **Go/Gin** guides for the backend.
- **Project 3 — Load-balanced cluster.** Run three API replicas behind an `upstream`. Demonstrate round-robin → least_conn, weights, passive health checks (kill a node mid-load), a `backup` server, and a zero-downtime rolling deploy (drain old, reload, stop old).
- **Project 4 — API gateway.** Route three "microservices" by path, add per-IP rate limiting, an `auth_request` gateway to a JWT-validating auth service (**Go JWT/Argon2 guide**), CORS, request-id tracing, and uniform JSON errors.
- **Project 5 — Caching edge.** Put `proxy_cache` (with microcaching + cache-lock) in front of a slow backend; load-test with `wrk` and chart the HIT-ratio and backend-load reduction. Add stale-while-revalidate and verify it serves during a backend outage.
- **Project 6 — Dockerized full stack.** Compose Nginx + app + Postgres + Redis (§13.3, **Docker**, **PostgreSQL**, **Redis** guides), with Nginx as the only exposed service, TLS, and a `/health` endpoint. Then translate one route into a Kubernetes `Ingress` resource and run it on a local cluster (kind/minikube) to see the Ingress controller generate the equivalent Nginx config.
- **Project 7 — Observability.** Wire `stub_status` + the Prometheus nginx exporter + Grafana (or just `goaccess` offline), define a custom `log_format` with timing fields, and build a dashboard showing request rate, error rate, p95 `$request_time`, cache HIT ratio, and per-upstream response time.

---

*End of the Nginx reference. Cross-references: **DOCKER_GUIDE** (containerizing & Compose), **GO_GIN_REST_API_FILE_UPLOAD_GUIDE** / **GO_NET_HTTP_REST_API_GUIDE** (Go backends behind Nginx), **NODEJS_GUIDE** / **FASTIFY_GUIDE** (Node backends), **GO_JWT_ARGON2_GUIDE** (the auth service for `auth_request`), **GO_GORILLA_WEBSOCKETS_GUIDE** (the WebSocket backend), **POSTGRESQL_GUIDE** / **REDIS_GUIDE** (the data layer & shared sessions), and **FTP_SERVER_GO_AND_NODE_GUIDE** (file serving alternatives). Build, break, read the logs, and reload — that loop is how Nginx mastery is built.*
