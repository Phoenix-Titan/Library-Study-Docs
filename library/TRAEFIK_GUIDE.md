# Traefik — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone who has never heard of a reverse proxy and wants to end up able to put **Traefik** in front of a real production SaaS — routing traffic to many backend instances, terminating HTTPS automatically, load-balancing under heavy load, hiding an admin panel behind an IP allow-list, and surviving traffic spikes from **half a million users** — and *understand every line* of how it works. It assumes zero prior proxy/DevOps knowledge and builds to the level a real platform engineer operates at. It is deliberately **explain-first**: every concept and every config line is introduced with *what* it is, *why* it exists, *where* it goes, *when* to reach for it, and the *gotcha* that bites people — and only then shown, heavily commented. A reverse proxy is the single public front door to your whole system; a label you copied but did not understand is an outage or a security hole waiting to happen. So the "why" matters as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. The second half (§13 onward) is one **complete, banking-grade production project** — a live-score SaaS — built end to end.
>
> **Version note:** This guide targets **Traefik v3.x** (v3.3+ current in 2026 — note v3 renamed several v2 things, e.g. `ipWhiteList` → **`ipAllowList`**, and changed the router rule syntax; everywhere it matters is flagged **⚡**). The stack around it is **Docker Engine 28.x** with **Compose v2**, a **Go 1.25/1.26** backend (**Gin v1.10**, **Ent v0.14**, **goose v3.27**, **pgx v5.10 / pgxpool**, **golang-jwt/jwt v5**, **Argon2id**, **Air**, **godotenv**), a **Next.js 16 / React 19** frontend, **PostgreSQL 17** (with a read replica and application-level sharding), **Redis 7.x/8.x**, and **Let's Encrypt** for TLS. Everything is **offline-first** and current as of **2026**. The author is on **Windows 11**; client-side commands are shown for PowerShell and POSIX where they differ, but the server and containers are Linux throughout.
>
> **This guide's place in the library:** Traefik is the *edge* — the ingress/reverse-proxy/load-balancer layer. It is an alternative to the **Caddy**-based edge in [Production VPS Management](PRODUCTION_VPS_GUIDE.md) (read that guide for the surrounding host hardening, firewall, backups, and DNS basics — this guide focuses on the proxy and the app tier and does not repeat them) and to [Nginx](NGINX_GUIDE.md) (§12 compares all three). The backend it fronts is built in the [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md), [pgx](GO_PGX_GUIDE.md), [Ent](GO_ENT_ORM_GUIDE.md), [goose](GO_GOOSE_MIGRATIONS_GUIDE.md), [JWT + Argon2](GO_JWT_ARGON2_GUIDE.md), and [Server-Sent Events in Go](GO_SSE_GUIDE.md) guides; the frontend in [Next.js 16](NEXTJS_16_GUIDE.md); the data tier in [PostgreSQL](POSTGRESQL_GUIDE.md) and [Redis](REDIS_GUIDE.md); the container basics in [Docker](DOCKER_GUIDE.md); and the DNS/TLS fundamentals in [Networking](NETWORKING_GUIDE.md). This guide assumes you can build those pieces; it teaches you to **route, secure, and scale them with Traefik**.

---

## Table of Contents

1. [What Traefik Is and Why It Exists](#1-what-traefik-is-and-why-it-exists) **[B]**
2. [The Core Concepts](#2-the-core-concepts) **[B]**
3. [Installing and Running Traefik](#3-installing-and-running-traefik) **[B]**
4. [Static vs Dynamic Configuration](#4-static-vs-dynamic-configuration) **[B/I]**
5. [The Docker Provider — Configuration by Labels](#5-the-docker-provider--configuration-by-labels) **[B/I]**
6. [Routers in Depth](#6-routers-in-depth) **[I]**
7. [Services and Load Balancing](#7-services-and-load-balancing) **[I]**
8. [Middlewares — the Full Toolbox](#8-middlewares--the-full-toolbox) **[I/A]**
9. [TLS and Automatic HTTPS with ACME](#9-tls-and-automatic-https-with-acme) **[I]**
10. [The Dashboard and API](#10-the-dashboard-and-api) **[I]**
11. [Observability — Logs, Metrics, Tracing](#11-observability--logs-metrics-tracing) **[A]**
12. [Traefik vs Nginx vs Caddy](#12-traefik-vs-nginx-vs-caddy) **[I]**
13. [The Project — A Live-Score SaaS](#13-the-project--a-live-score-saas) **[I]**
14. [Hosting on a VPS and Server Hardening](#14-hosting-on-a-vps-and-server-hardening) **[I/A]**
15. [DNS, Subdomains and the Cloudflare Firewall](#15-dns-subdomains-and-the-cloudflare-firewall) **[I/A]**
16. [Project Architecture and the Monorepo](#16-project-architecture-and-the-monorepo) **[I/A]**
17. [The Backend — Gin, Ent, SSE and REST](#17-the-backend--gin-ent-sse-and-rest) **[I/A]**
18. [PostgreSQL 17 — Read Replicas and Sharding](#18-postgresql-17--read-replicas-and-sharding) **[A]**
19. [Redis — Cache, Backplane and Rate Limits](#19-redis--cache-backplane-and-rate-limits) **[A]**
20. [Running Many Backend Instances behind Traefik](#20-running-many-backend-instances-behind-traefik) **[A]**
21. [The Hidden Admin Plane](#21-the-hidden-admin-plane) **[A]**
22. [The Next.js Frontend — an Untrusted Client](#22-the-nextjs-frontend--an-untrusted-client) **[I/A]**
23. [Banking-Grade Security](#23-banking-grade-security) **[A]**
24. [Load Balancing and Handling 500k Users](#24-load-balancing-and-handling-500k-users) **[A]**
25. [Scaling, Traffic Spikes and Performance](#25-scaling-traffic-spikes-and-performance) **[A]**
26. [CI/CD — GitHub Actions Auto-Deploy to the VPS](#26-cicd--github-actions-auto-deploy-to-the-vps) **[A]**
27. [Building ScoreLive — Every File and the Full Stack](#27-building-scorelive--every-file-and-the-full-stack) **[I/A]**
28. [Operations for the SaaS](#28-operations-for-the-saas) **[A]**
29. [Gotchas and Best Practices](#29-gotchas-and-best-practices) **[A]**
30. [Study Path and Build-to-Learn Projects](#30-study-path-and-build-to-learn-projects)

---

## 1. What Traefik Is and Why It Exists

### 1.1 The reverse-proxy problem, restated **[B]**

Your production system is not one program — it's many: several copies of your Go API (for capacity and redundancy), maybe a separate admin service, a Next.js frontend, and behind them Postgres and Redis. But the internet can only reach your server at one place: an IP address, on ports 80 (HTTP) and 443 (HTTPS). Something has to sit at that one public entry point, look at each incoming request, and decide *which* of your many internal programs should handle it — then forward it there, get the answer, and relay it back. That something is a **reverse proxy**.

It's called "reverse" because it fronts your *servers* (as opposed to a *forward* proxy, which fronts *clients*). A reverse proxy gives you, in one place: **routing** (send `api.app.com` to the API, `app.com` to the frontend, `/admin` to the admin service), **load balancing** (spread requests across your many API copies), **TLS termination** (handle the HTTPS certificate so your apps don't have to), **security** (rate limiting, IP allow-lists, header hardening, hiding your backends), and **cross-cutting concerns** (compression, retries, circuit breaking). It is the front door, the traffic cop, and the first line of defense, all at once.

### 1.2 What makes Traefik different **[B]**

There are several excellent reverse proxies (Nginx, HAProxy, Caddy). Traefik's defining idea is **dynamic, automatic configuration through service discovery.** With a traditional proxy you write a static config file listing every backend and its address; when you add or remove a backend, you edit the file and reload. Traefik instead *watches your infrastructure* — your Docker daemon, your orchestrator — and **configures itself automatically** from what it sees. You attach a few **labels** to a container ("route `api.app.com` to me, on port 8080"), and the instant that container starts, Traefik discovers it, creates a route, and begins sending it traffic. Scale your API from two containers to ten, and Traefik automatically load-balances across all ten — no config edit, no reload, no downtime. Stop a container, and Traefik stops routing to it.

This is transformative in a containerized, autoscaling world. Your routing configuration lives *next to your services* (as labels in the same Compose file), it updates *automatically* as services come and go, and there is no separate proxy config to keep in sync. For a system that scales up and down with traffic spikes — exactly the SaaS this guide builds — that dynamism is the whole point. Traefik also ships **automatic HTTPS** (Let's Encrypt certificates provisioned and renewed with zero cron jobs, like Caddy), a **live dashboard** showing every route and its health, and first-class **metrics and tracing** — batteries included for production.

### 1.3 When Traefik is the right tool **[B/I]**

Reach for Traefik when your backends are **dynamic** — containers/services that come and go, scale horizontally, and deploy frequently — and you want routing that keeps up automatically. That describes most Docker/Kubernetes deployments and is exactly our SaaS. It's also excellent when you have **many services and subdomains** (a monorepo of microservices, each on its own host), because each service declares its own routing and Traefik assembles the whole map. Prefer a simpler proxy (Caddy — see the [VPS guide](PRODUCTION_VPS_GUIDE.md)) when you have one or two static backends and want the least possible config; prefer Nginx when you need its specific mature modules or have deep Nginx expertise (§12 is the honest comparison). For a scaling, container-native SaaS with a hidden admin plane and heavy traffic, Traefik is squarely the right choice, and this guide takes you from its first `docker run` to fronting 500k users.

### 1.4 The mental model to hold **[B]**

Before any config, fix this pipeline in your head — a request flows through Traefik in four stages, and every concept in §2 is one of these stages:

```text
   request                                                        backend
   ────────►  [ EntryPoint ] ──► [ Router ] ──► [ Middlewares ] ──► [ Service ] ──►  container(s)
              "which port?"      "which rule    "transform/guard    "load-balance
               :80 / :443         matches?"      the request"        across copies"
```

A request arrives at an **EntryPoint** (a port Traefik listens on). A **Router** decides, by matching a **rule** (host, path, headers), whether this request is for a given service. Before the request reaches the service, a chain of **Middlewares** can transform or guard it (rate-limit it, check its IP, add headers, strip a path prefix). Finally the **Service** load-balances the request across the actual backend containers. That's the whole model: *EntryPoint → Router → Middlewares → Service → your app.* Everything else is detail.

---

## 2. The Core Concepts

### 2.1 EntryPoints — the ports Traefik listens on **[B]**

An **EntryPoint** is a network port (and protocol) that Traefik accepts connections on. You typically define two: **`web`** on `:80` (plain HTTP, used to redirect to HTTPS and to answer ACME challenges) and **`websecure`** on `:443` (HTTPS, the real traffic). EntryPoints are defined in the **static** configuration (§4) because they're about *how Traefik itself starts up*. Think of them as the doors of the building; routers then decide which room (service) each visitor goes to.

```yaml
# (static config) entryPoints — the ports Traefik binds
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
    http3: {}              # enable HTTP/3 (QUIC) on 443/udp — faster for mobile clients
```

### 2.2 Routers — matching a request to a service **[B]**

A **Router** connects incoming requests to a service, based on a **rule**. The rule is a boolean expression over the request's properties — most commonly its **Host** and **path**:

```text
Host(`api.app.com`)                         → matches requests for that hostname
Host(`api.app.com`) && PathPrefix(`/v1`)    → that host AND a path starting with /v1
Host(`app.com`) || Host(`www.app.com`)      → either host
```

A router says: *"if a request matches this rule, hand it (via these middlewares) to this service."* It also declares which **EntryPoint(s)** it listens on and whether it uses **TLS**. In the Docker provider, you express all of this as labels on the backend container (§5). The rule language is Traefik's routing brain, and §6 covers it fully.

> **⚡ v3 rule syntax.** Traefik v3 changed the rule matchers from v2. Use backticks around values, `&&`/`||`/`!` for logic, and the matchers `Host`, `PathPrefix`, `Path`, `PathRegexp`, `HostRegexp`, `Header`, `HeaderRegexp`, `Method`, `Query`, `ClientIP`. The v2 `{name:regex}` capture syntax is **gone** — `HostRegexp` now takes a plain Go regular expression (e.g. `` HostRegexp(`^.+\.app\.com$`) ``).

### 2.3 Services — the backends and load balancing **[B]**

A **Service** is the actual destination: your application. A service is usually a **load balancer** over one or more **servers** (backend instances). With the Docker provider, Traefik discovers the servers automatically — every container matching the service becomes a server, and Traefik round-robins across them. You mainly tell it **which port** inside the container to send to:

```text
traefik.http.services.api.loadbalancer.server.port=8080   # the app listens on 8080 inside the container
```

When you run three copies of that container, the service has three servers and Traefik balances across all three. Health checks, sticky sessions, and weighting are service options (§7).

### 2.4 Middlewares — the request pipeline **[B]**

A **Middleware** sits between the router and the service and can inspect, transform, or reject a request (or the response). Traefik has a rich built-in set — **rate limiting**, **IP allow-listing**, **header manipulation**, **basic auth**, **forward auth**, **path stripping**, **redirects**, **compression**, **retries**, **circuit breakers**. You compose them into a **chain** applied in order. Middlewares are how you enforce security and cross-cutting policy *at the edge*, before a request ever reaches your app — the hidden-admin IP allow-list, the login rate limit, the security headers, all live here (§8, §21, §23).

### 2.5 Providers — where config comes from **[B]**

A **Provider** is a source Traefik reads *dynamic* configuration from. The **Docker provider** reads labels off your containers (our main one). The **file provider** reads YAML/TOML files you write (for config that isn't tied to a container — TLS options, shared middlewares). There are providers for Kubernetes, Consul, and more. Traefik can use several at once and merges them into one live routing table. The key insight: **providers are how Traefik configures itself automatically** — the Docker provider is *why* scaling a service just works.

### 2.6 Putting it together **[B]**

One sentence ties the six concepts together: *Traefik listens on **EntryPoints**, reads dynamic config from **Providers**, and for each request a **Router** matches it by rule, a chain of **Middlewares** guards and transforms it, and a **Service** load-balances it across your backend containers.* Hold that sentence; the rest of the guide is filling in each noun.

---

## 3. Installing and Running Traefik

### 3.1 Running Traefik as a container **[B]**

Traefik is a single Go binary, but in production it runs as a **container** in your Compose stack — so it can watch the Docker socket to discover your other containers. Here is the minimal working Traefik service; we'll harden and extend it throughout the guide:

```yaml
# compose.yaml — the Traefik edge service (minimal, for learning)
services:
  traefik:
    image: traefik:v3.3              # pin the major.minor version (never :latest in prod)
    command:
      - "--api.dashboard=true"       # enable the web dashboard (secured later — §10)
      - "--providers.docker=true"    # watch Docker for containers to route to
      - "--providers.docker.exposedbydefault=false"  # only route containers that OPT IN (label)
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      # Read-only access to the Docker socket so Traefik can discover containers.
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks: [edge]

networks:
  edge:
    driver: bridge
```

Two lines are load-bearing for security. **`exposedbydefault=false`** means Traefik ignores every container unless it explicitly sets `traefik.enable=true` — so a new container is *never* accidentally exposed to the internet; it must opt in. And mounting the Docker socket **`:ro`** (read-only) limits what a compromised Traefik could do — though note the socket is still a powerful, sensitive mount (§23 covers the safer socket-proxy pattern).

### 3.2 A first routed service **[B]**

To prove it works, route a simple `whoami` container (a tiny app that echoes request info). Add it to the same Compose file:

```yaml
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"                                   # opt in to Traefik
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"  # match this host
      - "traefik.http.routers.whoami.entrypoints=web"           # on :80
      - "traefik.http.services.whoami.loadbalancer.server.port=80"   # app's port inside the container
    networks: [edge]
```

```bash
docker compose up -d
curl -H "Host: whoami.localhost" http://localhost/    # → whoami echoes the request; Traefik routed it
docker compose up -d --scale whoami=3                 # run THREE copies...
curl -H "Host: whoami.localhost" http://localhost/    # ...repeat: the responding IP rotates (load balanced!)
```

That's the Traefik magic in ten lines: label a container, and it's routed; scale it, and it's load-balanced — **no proxy config edited, no reload.** Everything else in this guide builds on this loop.

### 3.3 Reading the logs while you learn **[B]**

```bash
docker compose logs -f traefik            # follow Traefik's own logs (startup, provider events, errors)
docker compose logs traefik | grep -i error
```

When a route "doesn't work," Traefik's logs almost always say why — a malformed rule, a container with no matching port, a certificate challenge that failed. Get in the habit of `logs -f traefik` in one terminal while you edit labels in another; the feedback loop is fast because Traefik reconfigures itself the instant a container changes.

---

## 4. Static vs Dynamic Configuration

### 4.1 The two kinds of config, and why they're separate **[B/I]**

Traefik splits its configuration into two categories, and understanding the split prevents most beginner confusion:

- **Static configuration** — settings Traefik needs *at startup* and cannot change without a restart: the **EntryPoints** (ports), which **providers** to enable, the **certificate resolvers** (ACME), logging, metrics. It's set via a `traefik.yml` file, CLI flags, or environment variables (pick one; don't mix for the same setting). This is "how Traefik itself is built."
- **Dynamic configuration** — the **routers, services, and middlewares** that Traefik reads *while running* and reloads *automatically* when they change: from Docker labels (the Docker provider) or from watched files (the file provider). This is "what Traefik routes right now," and it updates live.

The mental split: **static = the engine; dynamic = the map it follows,** and the map can be redrawn without stopping the engine. You define EntryPoints and ACME once (static); you add and remove routes constantly (dynamic) just by starting and stopping labeled containers.

### 4.2 Static config as a file **[B/I]**

For anything beyond a couple of flags, prefer a `traefik.yml` file over a wall of CLI flags — it's readable and version-controlled. Mount it into the container:

```yaml
# traefik/traefik.yml  — the STATIC configuration
global:
  checkNewVersion: false
  sendAnonymousUsage: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:                 # auto-redirect ALL http → https (do this globally)
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    http3: {}                       # HTTP/3 (QUIC) on udp/443

providers:
  docker:
    exposedByDefault: false         # containers must opt in with traefik.enable=true
    network: edge                   # the Docker network Traefik uses to reach services
  file:
    directory: /etc/traefik/dynamic # watch this dir for dynamic YAML (TLS opts, shared middlewares)
    watch: true

api:
  dashboard: true                   # the dashboard (secured in §10)

log:
  level: INFO                       # DEBUG while learning; INFO/WARN in prod
accessLog: {}                       # enable access logs (tuned in §11)

certificatesResolvers:              # ACME — automatic HTTPS (§9)
  le:
    acme:
      email: ops@app.com
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web
```

```yaml
# the Traefik service in compose.yaml now mounts the static file:
    command:
      - "--configFile=/etc/traefik/traefik.yml"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik/traefik.yml:/etc/traefik/traefik.yml:ro"
      - "./traefik/dynamic:/etc/traefik/dynamic:ro"      # dynamic file provider dir
      - "letsencrypt:/letsencrypt"                        # persist ACME certs (§9)
```

The global HTTP→HTTPS **redirection** in the `web` EntryPoint is a one-time setting that upgrades *every* site to HTTPS automatically — you never write a per-router redirect. That's the kind of thing static config is for: cross-cutting startup behavior.

### 4.3 The file provider — dynamic config not tied to a container **[I]**

Some dynamic config isn't naturally attached to a single container — a middleware shared by many routers, TLS options (min version, cipher suites), or a route to something *not* in Docker (a legacy VM). The **file provider** reads these from YAML files in a watched directory, and reloads them live when you edit them:

```yaml
# traefik/dynamic/security.yml  — shared dynamic config, hot-reloaded on save
http:
  middlewares:
    security-headers:               # a reusable security-headers middleware (§8, §22)
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true
        frameDeny: true
        referrerPolicy: strict-origin-when-cross-origin
  # TLS options applied to routers that opt in
tls:
  options:
    modern:
      minVersion: VersionTLS12
      sniStrict: true
```

You then reference `security-headers@file` from any router. The `@file` suffix is important: Traefik **namespaces** dynamic objects by their provider — `security-headers@file` (from a file), `api@docker` (from Docker labels). When a router in Docker references a file-defined middleware, you must qualify it: `traefik.http.routers.x.middlewares=security-headers@file`. Forgetting the `@provider` suffix is a top source of "middleware not found" errors.

---

## 5. The Docker Provider — Configuration by Labels

### 5.1 The label grammar **[B/I]**

The Docker provider turns container **labels** into routers, services, and middlewares. Every label follows a strict, predictable grammar — once you see the pattern, you can write any of them:

```text
traefik.http.<routers|services|middlewares>.<your-name>.<setting>=<value>
        │    │                               │           └ the specific option
        │    │                               └ a name YOU invent (ties related labels together)
        │    └ the object type you're configuring
        └ the protocol (http, or tcp/udp for non-HTTP)
```

So `traefik.http.routers.api.rule=...` configures a **router** you're calling `api`; `traefik.http.services.api.loadbalancer.server.port=8080` configures a **service** you're calling `api`. The `<your-name>` is arbitrary but must be **consistent** across the labels that belong together, and **unique** across the whole stack (two containers using router name `api` would collide — name them `api` and `admin`, or Traefik will merge them). By convention people reuse one name for a container's router and service.

### 5.2 A complete, production-shaped labelset **[B/I]**

Here is every label a real backend service needs, annotated. This is the pattern you'll copy for each service in the project:

```yaml
  api:
    image: registry.app.com/api:1.4.2
    labels:
      - "traefik.enable=true"
      # ── Router: match requests and attach middlewares ──
      - "traefik.http.routers.api.rule=Host(`api.app.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"          # HTTPS only
      - "traefik.http.routers.api.tls.certresolver=le"            # auto-cert via ACME (§9)
      - "traefik.http.routers.api.middlewares=security-headers@file,api-ratelimit"
      # ── Service: which container port, health checks, sticky sessions ──
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/healthz"
      - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"
      # ── A middleware DEFINED on this service (a rate limit) ──
      - "traefik.http.middlewares.api-ratelimit.ratelimit.average=100"
      - "traefik.http.middlewares.api-ratelimit.ratelimit.burst=50"
    networks: [edge]     # MUST share Traefik's provider network so Traefik can reach it
```

Notice a middleware can be **defined** on the same container that **uses** it (`api-ratelimit` here) or defined once in a file and shared (`security-headers@file`). Notice too the `networks: [edge]` — a service is only routable if it's on the network Traefik watches (the `providers.docker.network` from §4.2); a container on a different network is invisible to Traefik even with perfect labels. That network mismatch is the #1 "my labels look right but it 404s" cause.

### 5.3 The Docker socket and security **[I]**

The Docker provider works by reading the Docker socket (`/var/run/docker.sock`). That socket is **root-equivalent** — anything that can write to it can control the host. Traefik only needs to *read* it, so mount it `:ro`, but even read access leaks information. The production-grade pattern is a **Docker socket proxy** (e.g. `tecnativa/docker-socket-proxy`): a tiny container that exposes only the read-only Docker API endpoints Traefik needs, over a private network, so Traefik never touches the real socket. §23 sets this up; for now, know that `docker.sock:ro` on Traefik is a real (if common) risk you'll tighten before production.

---

## 6. Routers in Depth

### 6.1 The rule matchers **[I]**

A router's **rule** is a boolean expression that decides whether a request belongs to this router. The v3 matchers:

| Matcher | Matches on | Example |
|---|---|---|
| `Host(`h`)` | the `Host` header (the domain) | `` Host(`api.app.com`) `` |
| `HostRegexp(`re`)` | host by Go regexp | `` HostRegexp(`^.+\.app\.com$`) `` (any subdomain) |
| `PathPrefix(`/p`)` | path starts with | `` PathPrefix(`/api/v1`) `` |
| `Path(`/p`)` | exact path | `` Path(`/healthz`) `` |
| `PathRegexp(`re`)` | path by regexp | `` PathRegexp(`^/users/[0-9]+$`) `` |
| `Header(`k`,`v`)` | a header equals a value | `` Header(`X-Env`, `staging`) `` |
| `Method(`m`)` | HTTP method | `` Method(`GET`) `` |
| `Query(`k`,`v`)` | a query param | `` Query(`ref`, `promo`) `` |
| `ClientIP(`cidr`)` | the client's IP | `` ClientIP(`203.0.113.0/24`) `` |

Combine them with **`&&`** (and), **`||`** (or), **`!`** (not), and parentheses:

```text
Host(`api.app.com`) && (PathPrefix(`/v1`) || PathPrefix(`/v2`))
Host(`app.com`) && !PathPrefix(`/admin`)     # everything on app.com EXCEPT /admin
```

> **⚡ v3 change:** in v2 you wrote `` PathPrefix(`/a`, `/b`) `` with multiple values; in v3 each matcher takes **one** value — use `||` to combine (`` PathPrefix(`/a`) || PathPrefix(`/b`) ``). And `HostRegexp`/`PathRegexp` now take real regexps, not the old `{name:pattern}` syntax. Migrating a v2 config? These two are the usual breakages.

### 6.2 Router priority — resolving overlaps **[I]**

When two routers could both match a request, Traefik picks the one with the **longest rule** by default (more specific wins). When that heuristic isn't what you want, set an explicit **priority** (higher = evaluated first):

```yaml
      # A catch-all for the app, LOW priority...
      - "traefik.http.routers.app.rule=Host(`app.com`)"
      - "traefik.http.routers.app.priority=1"
      # ...and a specific /admin route, HIGH priority, so it wins over the catch-all.
      - "traefik.http.routers.admin.rule=Host(`app.com`) && PathPrefix(`/admin`)"
      - "traefik.http.routers.admin.priority=100"
```

Explicit priorities make routing *deterministic* and are essential once you have overlapping rules (a catch-all plus specific carve-outs, exactly the frontend-plus-admin case in the project). Never rely on the length heuristic for anything that matters — set the number.

### 6.3 TLS on a router **[I]**

`traefik.http.routers.<n>.tls=true` tells the router to serve over TLS; adding `traefik.http.routers.<n>.tls.certresolver=le` makes Traefik obtain the certificate automatically via ACME (§9). You can also pin TLS options (`traefik.http.routers.<n>.tls.options=modern@file`) and, for wildcard certs, request them via the resolver's `domains`. A router on the `websecure` entrypoint with a certresolver is the entire recipe for "this hostname is HTTPS with a real, auto-renewing certificate."

---

## 7. Services and Load Balancing

### 7.1 How Traefik load-balances **[I]**

A **service** with the Docker provider is automatically a load balancer over every container that shares its labels. When you scale a service to N replicas, Traefik sees N servers and distributes requests across them using **weighted round-robin (WRR)** — by default equal weights, so requests cycle evenly. This is dynamic: scale up and the new containers *immediately* join the rotation; scale down and they leave. No config change. This automatic, live load balancing across replicas is the feature that makes Traefik shine for a scaling SaaS.

### 7.2 Health checks — routing only to healthy backends **[I]**

Load-balancing to a *dead* instance returns errors to users. **Health checks** make Traefik poll each server and route only to healthy ones:

```yaml
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/healthz"
      - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"
      - "traefik.http.services.api.loadbalancer.healthcheck.timeout=3s"
```

Traefik hits `/healthz` on each backend every 10s; a server that fails is pulled from rotation until it recovers. Combined with your Go app's health endpoint (which should check that it can reach Postgres/Redis), this gives *self-healing routing* — a crashed or degraded instance simply stops receiving traffic, and the survivors carry the load. This is a different, complementary layer to Docker's own `healthcheck` (which restarts the container): Traefik's decides *routing*, Docker's decides *lifecycle*.

### 7.3 Sticky sessions **[I/A]**

By default any request can go to any instance — correct for a **stateless** app (which yours must be; see the SSE/VPS guides). But sometimes you need a client pinned to one instance (in-memory per-connection state). **Sticky sessions** do this with a cookie:

```yaml
      - "traefik.http.services.api.loadbalancer.sticky.cookie=true"
      - "traefik.http.services.api.loadbalancer.sticky.cookie.name=srv"
      - "traefik.http.services.api.loadbalancer.sticky.cookie.secure=true"
      - "traefik.http.services.api.loadbalancer.sticky.cookie.httponly=true"
```

Traefik sets a cookie naming the chosen server and routes that client back to it. **Use stickiness sparingly** — it undermines even load distribution and complicates deploys (a pinned instance can't be drained cleanly). For SSE specifically you generally *don't* need it: each SSE stream is one long request to one instance already, and shared state lives in Redis, so a reconnect landing on a different instance is fine. Prefer stateless + shared Redis over stickiness whenever you can.

### 7.4 Retries and circuit breaking **[I/A]**

Two resilience middlewares live at the service edge. **Retry** re-sends a failed request to another backend (helpful for transient network blips):

```yaml
      - "traefik.http.middlewares.api-retry.retry.attempts=3"
      - "traefik.http.middlewares.api-retry.retry.initialinterval=100ms"
```

A **circuit breaker** stops sending traffic to a service that's failing en masse, giving it room to recover instead of hammering it:

```yaml
      - "traefik.http.middlewares.api-cb.circuitbreaker.expression=NetworkErrorRatio() > 0.30"
```

When >30% of requests to the service error, the breaker "opens" and Traefik fast-fails (or serves a fallback) until the service recovers. Under a **traffic spike** (500k users, a big match kicking off), retries + circuit breaking prevent a partial failure from cascading into a total outage — they're core to the resilience story in §24.

---

## 8. Middlewares — the Full Toolbox

### 8.1 The middleware chain **[I/A]**

A router applies a comma-separated **chain** of middlewares, in order, before the request reaches the service. Order matters: put cheap rejections first (IP allow-list, rate limit) so expensive work never runs for a request you'll drop. You reference them on the router:

```yaml
      - "traefik.http.routers.admin.middlewares=admin-ipallow,admin-ratelimit,security-headers@file"
```

Below is the toolbox — the middlewares you'll actually use in production, grouped by job.

### 8.2 Security middlewares **[A]**

```yaml
# ── IP allow-list — the hidden admin plane's core (§21) ──
- "traefik.http.middlewares.admin-ipallow.ipallowlist.sourcerange=203.0.113.5/32,198.51.100.0/24"
#   ⚡ v3: 'ipallowlist' (v2 was 'ipwhitelist'). Behind Cloudflare/another proxy, add ipStrategy:
- "traefik.http.middlewares.admin-ipallow.ipallowlist.ipstrategy.depth=1"   # trust the Nth X-Forwarded-For from the right

# ── Rate limiting — throttle abusive clients at the edge ──
- "traefik.http.middlewares.login-rl.ratelimit.average=5"       # 5 req/s average per source...
- "traefik.http.middlewares.login-rl.ratelimit.burst=10"        # ...bursting to 10
- "traefik.http.middlewares.login-rl.ratelimit.period=1s"
- "traefik.http.middlewares.login-rl.ratelimit.sourcecriterion.ipstrategy.depth=1"

# ── In-flight request limit — cap concurrent requests (DoS/overload guard) ──
- "traefik.http.middlewares.api-inflight.inflightreq.amount=200"

# ── Basic auth — quick protection for internal tools (dashboard, §10) ──
- "traefik.http.middlewares.dash-auth.basicauth.users=admin:$$apr1$$....hashed...."
```

The **`ipAllowList`** is the star for the hidden admin plane — a request whose source IP isn't in `sourcerange` gets a `403` at the edge, so the admin login page *doesn't exist* for anyone else (§21). The **`ipStrategy.depth`** is critical when Traefik sits behind Cloudflare: without it, Traefik sees *Cloudflare's* IP as the client and your allow-list matches the wrong thing (§15.5). **Rate limiting** and **in-flight limits** throttle abuse before it reaches your app.

### 8.3 Request/response transformation **[I/A]**

```yaml
# ── Strip a path prefix before forwarding (e.g. /api/* → /* to the backend) ──
- "traefik.http.middlewares.strip-api.stripprefix.prefixes=/api"

# ── Add/replace request or response headers ──
- "traefik.http.middlewares.real-ip.headers.customrequestheaders.X-Real-IP="   # example placeholder

# ── Compress responses (gzip/brotli) — smaller payloads, faster clients ──
- "traefik.http.middlewares.compress.compress=true"

# ── Redirect (e.g. force a canonical host) ──
- "traefik.http.middlewares.to-www.redirectregex.regex=^https://app.com/(.*)"
- "traefik.http.middlewares.to-www.redirectregex.replacement=https://www.app.com/$${1}"
```

**`stripPrefix`** is common when a path prefix routes to a service but the service doesn't expect the prefix. **`compress`** cuts bandwidth (big for a JSON-heavy API and SSE). **`headers`** injects security headers or forwards client info. Note the doubled `$$` in labels — in a Compose file, `$` starts variable interpolation, so a literal `$` in a label (as in regexp replacements or bcrypt hashes) must be written `$$`. Forgetting to double `$` is a classic label bug.

### 8.4 The headers middleware for security **[A]**

The `headers` middleware sets the security headers a banking-grade edge needs — you'll define this once in the file provider (§4.3) and share it:

```yaml
# traefik/dynamic/security.yml (file provider)
http:
  middlewares:
    security-headers:
      headers:
        stsSeconds: 31536000            # HSTS: force HTTPS for a year...
        stsIncludeSubdomains: true      # ...on all subdomains...
        stsPreload: true                # ...and allow browser preload lists
        contentTypeNosniff: true        # X-Content-Type-Options: nosniff
        frameDeny: true                 # X-Frame-Options: DENY (anti-clickjacking)
        browserXssFilter: true
        referrerPolicy: strict-origin-when-cross-origin
        permissionsPolicy: "geolocation=(), microphone=(), camera=()"
        customResponseHeaders:
          Server: ""                    # strip the Server header (don't advertise Traefik)
```

Applying `security-headers@file` to every public router hardens every response consistently — the same headers the [JWT/Argon2 guide](GO_JWT_ARGON2_GUIDE.md) prescribes, enforced at the edge for *all* services at once, so a new backend can't forget them.

### 8.5 forwardAuth — delegating authentication **[A]**

**`forwardAuth`** makes Traefik call an external auth service on every request; if that service returns 2xx, the request proceeds (optionally with headers the auth service added, like the user id); otherwise it's rejected. This centralizes auth at the edge:

```yaml
- "traefik.http.middlewares.auth.forwardauth.address=http://auth:9000/verify"
- "traefik.http.middlewares.auth.forwardauth.authresponseheaders=X-User-Id,X-User-Role"
```

For our SaaS the JWT is validated *inside* the Go app (fast, no extra hop), so we don't use `forwardAuth` for the main API — but it's the right tool when you have many services in different languages and want one auth gate in front of all of them, or for protecting internal tools. Know it exists; reach for it when auth needs to be language-agnostic and central.

---

## 9. TLS and Automatic HTTPS with ACME

### 9.1 How Traefik gets certificates automatically **[I]**

Like Caddy, Traefik obtains and renews **Let's Encrypt** certificates automatically via the **ACME** protocol — no certbot, no cron. You define a **certificate resolver** in static config (the `le` resolver in §4.2), and any router with `tls.certresolver=le` triggers automatic provisioning on first request. Traefik proves domain control via a **challenge**, gets the cert, installs it, serves HTTPS, and renews ~30 days before expiry, forever.

The certificates and ACME account are stored in **`acme.json`**, which you *must* persist on a volume (§4.2's `letsencrypt:` volume) — lose it and Traefik re-requests certs, and Let's Encrypt has **rate limits** (~50 certs/domain/week, 5 duplicates/week) that repeated loss can trip, temporarily blocking issuance. The file must also be `chmod 600` (Traefik enforces this; a too-open `acme.json` is rejected).

### 9.2 The three challenge types **[I]**

| Challenge | Proves control by | Use when |
|---|---|---|
| **HTTP-01** (`httpChallenge`) | serving a token on port 80 | default single hostnames; needs port 80 reachable |
| **TLS-ALPN-01** (`tlsChallenge`) | a special TLS handshake on 443 | port 80 blocked but 443 open |
| **DNS-01** (`dnsChallenge`) | creating a TXT record via your DNS API | **wildcard** certs; works with ports closed |

```yaml
# static config — HTTP-01 (simplest) vs DNS-01 (for wildcards / behind Cloudflare)
certificatesResolvers:
  le:
    acme:
      email: ops@app.com
      storage: /letsencrypt/acme.json
      httpChallenge:                 # for normal subdomains
        entryPoint: web
  le-dns:
    acme:
      email: ops@app.com
      storage: /letsencrypt/acme.json
      dnsChallenge:                  # for *.app.com wildcard certs
        provider: cloudflare         # Traefik writes the TXT record via the CF API
        resolvers: ["1.1.1.1:53"]
```

For the project we use **DNS-01 with Cloudflare** to get a **wildcard** cert covering all subdomains (`api.`, `app.`, `admin.`, `sse.`), which is cleaner than per-subdomain HTTP-01 and required once Cloudflare's proxy fronts the origin (HTTP-01 through Cloudflare's proxy is fragile). The Cloudflare API token goes in via env/secret and should be **scoped to editing DNS on this one zone only** (least privilege).

### 9.3 Wildcard certificate for the whole SaaS **[A]**

```yaml
  traefik:
    environment:
      - "CF_DNS_API_TOKEN_FILE=/run/secrets/cf_token"   # scoped Cloudflare token
    secrets: [cf_token]
    labels:
      # Request a wildcard cert on the Traefik container itself (a "dummy" router pattern),
      # or per-router via tls.domains:
      - "traefik.http.routers.api.tls.certresolver=le-dns"
      - "traefik.http.routers.api.tls.domains[0].main=app.com"
      - "traefik.http.routers.api.tls.domains[0].sans=*.app.com"
```

One wildcard cert (`app.com` + `*.app.com`) then covers every current and future subdomain — add `status.app.com` later and it's already covered, no new certificate. This is the right TLS setup for a multi-subdomain SaaS.

---

## 10. The Dashboard and API

### 10.1 What the dashboard shows **[I]**

Traefik's **dashboard** is a live web UI of your entire routing state: every EntryPoint, router (and whether its rule/TLS is valid), service (and its healthy/unhealthy servers), and middleware. It's invaluable for answering "is my route registered? why is it unhealthy? which middlewares apply?" — the single best debugging tool when a request misbehaves. It reads from Traefik's **API**, which also powers automation.

### 10.2 Securing the dashboard — never expose it raw **[A]**

The dashboard reveals your whole topology and, via the API, can be a foothold — so it must **never** be publicly reachable. Secure it with a router that requires HTTPS, an **IP allow-list**, and **basic auth**, on an internal-only host:

```yaml
  traefik:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.internal.app.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=le-dns"
      - "traefik.http.routers.dashboard.service=api@internal"        # the built-in API service
      - "traefik.http.routers.dashboard.middlewares=dash-ipallow,dash-auth"
      # only these IPs, and only with a password:
      - "traefik.http.middlewares.dash-ipallow.ipallowlist.sourcerange=203.0.113.5/32"
      - "traefik.http.middlewares.dash-auth.basicauth.users=admin:$$2y$$05$$hashed..."
```

The dashboard now lives on `traefik.internal.app.com`, reachable only from your IP *and* only with a password — the same "hidden plane" pattern as the admin app (§21). In static config keep `api.dashboard=true` but **never** enable `api.insecure=true` (which exposes the dashboard unauthenticated on port 8080 — fine for a 5-minute local test, a disaster in production). Generate the basic-auth hash with `htpasswd -nB admin` and remember to double the `$` for Compose.

---

## 11. Observability — Logs, Metrics, Tracing

### 11.1 Access logs **[A]**

Traefik's **access log** records every request (method, path, status, latency, which router/service handled it, client IP). Enable and tune it in static config, and log as JSON so it's machine-parseable:

```yaml
# static config
accessLog:
  format: json
  filePath: /var/log/traefik/access.log
  bufferingSize: 100                 # buffer writes for throughput under load
  filters:
    statusCodes: ["400-499", "500-599"]   # log only errors (optional; reduces volume at 500k users)
  fields:
    headers:
      names:
        User-Agent: keep
        Authorization: drop          # NEVER log credentials
```

At 500k users the access log is a firehose; the `filters` (log only errors) and `bufferingSize` keep it manageable, and **`Authorization: drop`** ensures tokens never land in logs. Ship the log off-box (Loki/ELK) so it survives and is searchable across instances (§28).

### 11.2 Metrics with Prometheus **[A]**

Traefik exposes **Prometheus** metrics — request counts, latencies (histograms), error rates, and open connections, per router/service/entrypoint. This is how you *see* a traffic spike and whether you're keeping up:

```yaml
# static config
metrics:
  prometheus:
    entryPoint: metrics              # a dedicated internal entrypoint, NOT public
    addRoutersLabels: true
    buckets: [0.1, 0.3, 1.2, 5.0]
entryPoints:
  metrics:
    address: ":8082"                 # scraped by Prometheus over the private network only
```

A Prometheus container scrapes `:8082/metrics`; Grafana graphs it. The metrics that matter for the SaaS: **request rate** (are we spiking?), **p99 latency** (are we slow?), **5xx rate** (are we failing?), and **healthy-server count per service** (did instances die?). Alert on these (§28). Keep the metrics entrypoint **off the public internet** — it's internal-only.

### 11.3 Tracing **[A]**

For request-level debugging across services, Traefik supports **OpenTelemetry** tracing — it starts a trace at the edge and propagates the context to your backends, so you can see a single request's journey (edge → API → DB) and where the time went. Enable it (`--tracing.otlp=true`) and point it at a collector (Tempo/Jaeger). Tracing is the advanced tier of observability; for most operations, logs + metrics + health answer "is it up, is it slow, what broke," and tracing answers "*why* is this specific request slow" when you need to dig.

---

## 12. Traefik vs Nginx vs Caddy

An honest comparison, since this library covers all three (Caddy in the [VPS guide](PRODUCTION_VPS_GUIDE.md), Nginx in the [Nginx guide](NGINX_GUIDE.md)):

| | **Traefik** | **Caddy** | **Nginx** |
|---|---|---|---|
| Config style | **Dynamic** — auto-discovers containers via labels | Static Caddyfile (simple) | Static config (verbose) |
| Auto HTTPS | Yes (ACME) | Yes (ACME, simplest) | Manual (certbot) |
| Service discovery | **Built-in** (Docker/K8s) | No | No |
| Scales with containers | **Automatically** (label a container, it's routed) | Manual config edit | Manual config edit |
| Dashboard | Built-in live UI | No (API only) | No (commercial) |
| Middlewares | Rich, composable | Directives + plugins | Modules |
| Best for | **Dynamic, container-native, many services, autoscaling** | 1–few static backends, minimal config | Deep tuning, existing expertise, extreme static perf |
| Learning curve | Moderate (concepts + label grammar) | Gentle | Steep for TLS/routing |

**Choose Traefik** (as this project does) when your backends are containers that scale and deploy dynamically and you want routing that keeps up automatically — the SaaS with autoscaling API instances is the ideal case. **Choose Caddy** for the simplest possible HTTPS in front of one or two static services. **Choose Nginx** when you need its mature module ecosystem or run at a scale where its tuning knobs earn their complexity. The three overlap; the deciding factor is usually *how dynamic your backend is* — the more it scales and changes, the more Traefik's auto-discovery pays off.

---

## 13. The Project — A Live-Score SaaS

### 13.1 What we're building **[I]**

Everything from here on builds **one complete, production, banking-grade system**: **ScoreLive**, a SaaS that streams live match scores to hundreds of thousands of fans in real time. It has:

- A **public user dashboard** (`app.scorelive.com`) — anyone can sign up, log in, and watch live scores update in real time via **SSE**. This is the money-maker and must handle **500,000+ concurrent-ish users** with **traffic spikes** when big matches kick off.
- A **hidden admin/owner dashboard** (`admin.scorelive.com`) — where the owner enters scores, manages users, and sees analytics. It is **invisible to the public**: its login page returns nothing (a `403`) unless the request comes from an allow-listed owner IP. To everyone else, the admin plane *does not exist*.
- A **Go backend API** (`api.scorelive.com`) — Gin + Ent + goose + pgx/pgxpool + JWT/Argon2, serving both REST (CRUD, auth) and SSE (the live stream), run as **many identical instances** behind Traefik.
- A dedicated **SSE endpoint host** (`sse.scorelive.com`) — the same backend, but a separate hostname so we can scale and tune the streaming path independently.
- **PostgreSQL 17** with a **read replica** (reads scale out) and **application-level sharding** (data split across shards for write scale), **Redis** (cache + SSE pub/sub backplane + rate-limit counters), all in **Docker**.
- A **Next.js 16** frontend — treated as an **untrusted client** (it runs in the user's browser; the backend trusts nothing it says).
- All in a **monorepo**, fronted by **Traefik**, on a **hardened VPS** behind the **Cloudflare** firewall, deployed automatically by **GitHub Actions**.

### 13.2 The request map **[I]**

```text
                          Cloudflare (DNS + WAF + DDoS + CDN)
                                       │  (only Cloudflare IPs allowed to origin — §15.5)
                          ┌────────────▼─────────────┐
                          │   VPS firewall (ufw)     │  22 (SSH, IP-restricted), 80, 443
                          └────────────┬─────────────┘
                          ┌────────────▼─────────────┐
                          │         TRAEFIK          │  routers by host, TLS, middlewares, LB
                          └──┬────────┬────────┬──────┘
        app.scorelive.com ──┘        │        └── admin.scorelive.com
        (Next.js public)             │            (Next.js admin — IP-allow-listed, §21)
                          api. / sse.scorelive.com
                          ┌──────────┴──────────┐
                     ┌────▼───┐ ┌───▼───┐ ┌──────▼──┐   N identical Go API instances
                     │ api-1  │ │ api-2 │ │  api-N  │   (Traefik load-balances)
                     └────┬───┘ └───┬───┘ └────┬────┘
             ┌────────────┼─────────┼──────────┼────────────┐
        ┌────▼────┐  ┌────▼─────┐  ┌▼──────────▼┐   ┌────────▼────────┐
        │ pg-write│  │ pg-read  │  │   redis    │   │  pg shards      │
        │ primary │  │ replica  │  │ cache/bus  │   │  (write scale)  │
        └─────────┘  └──────────┘  └────────────┘   └─────────────────┘
```

Read it top-down: Cloudflare fronts everything (DNS, WAF, DDoS absorption, static CDN); only Cloudflare's IPs may reach the VPS (origin lockdown); the firewall allows just SSH/HTTP/HTTPS; **Traefik** routes by hostname to the right service and load-balances the API instances; the API talks to Postgres (writes to the primary, reads from the replica, sharded for write scale) and Redis. Each layer is covered in a section below.

### 13.3 The properties this must have **[I]**

Because it's a real SaaS handling money and 500k users, the non-negotiables are: **available** (survives instance/DB failures, deploys with zero downtime), **secure** (banking-grade — the admin plane hidden, everything hardened, the frontend never trusted), **scalable** (absorbs a kickoff spike without falling over), **efficient** (no wasted CPU/RAM — a 500k-user system with a sloppy hot path is an expensive outage), and **operable** (observable, backed up, reproducible). Every design decision below serves one of these; where they trade off, we call it out.

---

## 14. Hosting on a VPS and Server Hardening

### 14.1 The host, in brief **[I/A]**

ScoreLive runs on a **VPS** (or a few — §24). The full treatment of provisioning and hardening a production VPS is the [Production VPS Management guide](PRODUCTION_VPS_GUIDE.md); this section is the **condensed checklist** specific to hosting this Traefik stack, so the guide is self-contained. Provision **Ubuntu 24.04 LTS** with enough CPU/RAM for the stack (start at 4 vCPU / 8 GB, scale up — §24), SSD/NVMe storage, and enable the provider's private networking and snapshots.

### 14.2 The security baseline **[I/A]**

Before Traefik ever runs, harden the host. Each item maps to the VPS guide for the how:

```bash
# On the SERVER (as a fresh root, then switch to a deploy user):
# 1) Create a non-root deploy user with sudo; use it for everything.
adduser deploy && usermod -aG sudo deploy && usermod -aG docker deploy

# 2) SSH: keys only, no root, restricted. Edit /etc/ssh/sshd_config:
#    PasswordAuthentication no ; PermitRootLogin no ; AllowUsers deploy ; Port 2222
sudo systemctl restart ssh

# 3) Firewall: default-deny; ONLY SSH (from your IP), HTTP, HTTPS.
sudo ufw default deny incoming && sudo ufw default allow outgoing
sudo ufw allow from 203.0.113.5 to any port 2222 proto tcp comment 'SSH me only'
sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp
sudo ufw enable

# 4) fail2ban (ban brute-forcers) + unattended-upgrades (auto security patches).
sudo apt install -y fail2ban unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades

# 5) Docker (official), log rotation to stop disk-fill.
curl -fsSL https://get.docker.com | sudo sh
echo '{"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"}}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

The security baseline before you deploy: **non-root user**, **key-only SSH on a restricted port from your IP**, **default-deny firewall exposing only 22/80/443**, **fail2ban**, **automatic patches**, and **Docker log rotation** (unrotated logs filling the disk is a top outage cause at 500k-user log volumes). The VPS guide's §23 has the full 22-point checklist; run it down before ScoreLive goes live.

### 14.3 The Traefik-specific host concerns **[I/A]**

Three things matter specifically because Traefik fronts the stack:

- **Only Traefik binds public ports.** In the Compose stack, *only* the `traefik` service publishes `80/443`. The Go API, Postgres, Redis, and Next.js publish **no host ports** — they're reachable only on Traefik's private Docker network. Verify with `ss -tulpn`: you should see only Traefik on `0.0.0.0:80/443`. (Recall from the VPS guide that Docker can bypass ufw for published ports — so "publish nothing but Traefik" is the rule.)
- **Persist `acme.json`** on a named volume, `chmod 600` (§9.1) — losing certs trips Let's Encrypt rate limits.
- **The Docker socket** Traefik reads is root-equivalent; §23 replaces the raw mount with a read-only socket proxy for production.

---

## 15. DNS, Subdomains and the Cloudflare Firewall

### 15.1 The subdomain plan **[I/A]**

ScoreLive is a **monorepo with several subdomains**, each routed by Traefik to a different service. The DNS plan:

| Subdomain | Serves | Routed by Traefik to | Exposure |
|---|---|---|---|
| `app.scorelive.com` | public user dashboard | Next.js (public build) | public |
| `admin.scorelive.com` | owner/admin dashboard | Next.js (admin build) | **IP-allow-listed** (§21) |
| `api.scorelive.com` | REST API | Go API instances | public (authed) |
| `sse.scorelive.com` | live-score SSE stream | Go API instances (stream path) | public (authed) |
| `traefik.internal.scorelive.com` | Traefik dashboard | Traefik API | IP-allow-listed + auth (§10) |

Each is a DNS record pointing at the VPS (or at Cloudflare, which proxies to the VPS), and each is a Traefik router matching that `Host(...)`. Splitting `api.` and `sse.` onto separate hostnames — even though the same backend serves both — lets you route, rate-limit, and *scale* the streaming path independently of the REST path (SSE connections are long-lived and behave very differently from short REST calls under load, §24).

### 15.2 Creating the DNS records **[I/A]**

In Cloudflare's dashboard (Cloudflare is both registrar-or-DNS and the firewall/CDN), create:

```text
Type   Name      Content            Proxy      Notes
A      app       203.0.113.10       Proxied 🟠  public dashboard, through Cloudflare
A      admin     203.0.113.10       Proxied 🟠  admin (Traefik still IP-gates it)
A      api       203.0.113.10       Proxied 🟠  REST API
A      sse       203.0.113.10       DNS only ⚪  SSE: see 15.6 (proxy + streaming caveats)
A      traefik   203.0.113.10       DNS only ⚪  internal dashboard (locked by IP anyway)
```

Each subdomain is an **A record** to the VPS IP. The **Proxy** column (orange vs grey cloud) is Cloudflare-specific and consequential — orange routes traffic *through* Cloudflare (CDN, WAF, DDoS protection, hides your origin IP); grey is DNS-only (traffic goes straight to your server). §15.6 covers when to use which, especially for SSE.

### 15.3 Cloudflare as your firewall and shield **[I/A]**

Cloudflare in front of your origin is a second, powerful security layer *before traffic even reaches the VPS*. For a 500k-user SaaS that will get attacked and spiked, its features are worth using deliberately:

- **DDoS protection** — Cloudflare absorbs volumetric attacks at its edge; your origin never sees them. Essential for a public SaaS.
- **WAF (Web Application Firewall)** — managed rulesets block common exploits (SQLi, XSS probes, bad bots) before they hit your app. Enable the managed rules; add custom rules for your patterns.
- **Rate limiting** — Cloudflare can rate-limit at the edge (e.g. per-IP request caps on `/api/login`), complementing Traefik's rate limit — the earlier you throttle abuse, the cheaper.
- **Bot management & CAPTCHA** — challenge suspicious traffic during a spike/attack.
- **CDN & caching** — static assets (the Next.js `_next/static/*`, images) are cached at Cloudflare's edge worldwide, offloading a huge fraction of requests from your origin (§24).
- **Origin lockdown** — the big one (§15.4).

### 15.4 Locking the origin to Cloudflare only **[A]**

If your VPS accepts traffic from anywhere, attackers can find your origin IP and **bypass Cloudflare entirely** — defeating the WAF and DDoS protection. The fix: **only allow Cloudflare's IPs to reach ports 80/443** on the VPS. Cloudflare publishes its IP ranges; allow only those in the firewall:

```bash
# On the SERVER: allow 80/443 ONLY from Cloudflare's published IP ranges (fetch the current list).
# (Automate this — Cloudflare's ranges change occasionally; a small cron re-syncs them.)
for cidr in $(curl -s https://www.cloudflare.com/ips-v4); do
  sudo ufw allow from "$cidr" to any port 443 proto tcp comment 'cloudflare'
  sudo ufw allow from "$cidr" to any port 80  proto tcp comment 'cloudflare'
done
# Then REMOVE the open 80/443 rules so ONLY Cloudflare can reach the origin.
```

Alternatively (and better), use **Cloudflare Tunnel** (`cloudflared`) so the VPS makes an *outbound* connection to Cloudflare and has **no inbound public ports at all** — the origin IP is never exposed. Either way, the goal is the same: **your origin is reachable only through Cloudflare**, so the WAF and DDoS shield can't be bypassed. This is a defining move of a hardened public SaaS.

### 15.5 The real-client-IP problem behind Cloudflare **[A]**

Here's the subtlety that breaks the hidden admin plane if you miss it. With Cloudflare (or any proxy) in front, the connection to Traefik comes from **Cloudflare's** IP, not the user's. The real client IP is in the **`X-Forwarded-For`** (or Cloudflare's `CF-Connecting-IP`) header. So:

- Traefik's **`ipAllowList`** would match *Cloudflare's* IP, not the visitor's — making your admin allow-list useless (it'd allow everyone or no one). You **must** tell Traefik to read the real IP from the forwarded header via **`ipStrategy.depth`** (§8.2): with one trusted proxy (Cloudflare), `depth=1` takes the first IP from the right of `X-Forwarded-For`, which is the real client.
- Traefik must **trust** Cloudflare's IPs as forwarders, or it won't honor the header. Set the entrypoint's `forwardedHeaders.trustedIPs` to Cloudflare's ranges:

```yaml
# static config — trust Cloudflare to set X-Forwarded-For
entryPoints:
  websecure:
    address: ":443"
    forwardedHeaders:
      trustedIPs:                    # Cloudflare's ranges (keep in sync with 15.4)
        - "173.245.48.0/20"
        - "103.21.244.0/22"
        # ... the full Cloudflare list ...
```

Get this wrong and one of two failures follows: either the admin allow-list matches Cloudflare (so it "works" for everyone — a security hole), or your rate limits and logs all show Cloudflare's IP (so per-user limiting is broken). **Trust the proxy, read the real IP at the right depth** — then `ipAllowList` and rate limiting see the actual visitor. Your Go app needs the same treatment (Gin's `SetTrustedProxies`) so *it* also sees real IPs.

### 15.6 SSE and Cloudflare's proxy **[A]**

Cloudflare's proxy buffers and has timeouts that can interfere with **long-lived SSE streams**. Two workable setups: (1) keep `sse.scorelive.com` **DNS-only (grey)** so SSE connects straight to the origin (you lose Cloudflare's shield on that hostname — mitigate with tight Traefik/app rate limits and the firewall), or (2) keep it **proxied** but disable Cloudflare's buffering for that hostname and ensure your SSE sends periodic heartbeats/comments (which the [Go SSE guide](GO_SSE_GUIDE.md) already does) so the connection isn't idle-timed-out. For 500k concurrent streams, many teams put SSE on a separate proxied hostname with tuned timeouts, or terminate SSE at the origin behind the firewall. The key awareness: **SSE is a long-lived connection and every proxy in the path (Cloudflare, Traefik) must be configured not to buffer or prematurely close it** — the same lesson as the SSE guide's `X-Accel-Buffering`/`proxy_buffering off`, applied at each hop.

---

## 16. Project Architecture and the Monorepo

### 16.1 The monorepo layout **[I/A]**

ScoreLive lives in one **monorepo** — backend, both frontends, and the infra/Traefik config in a single Git repo, so a change that spans layers (a new API field the UI consumes) is one atomic commit and one CI run. The full tree, mapping each part to the section that builds it:

```text
scorelive/                              # the monorepo root
├── apps/
│   ├── api/                            # the Go backend (§17) — ONE codebase, run as N instances
│   │   ├── cmd/server/main.go          # entry: config → pgxpool → ent → gin → routes → serve
│   │   ├── internal/
│   │   │   ├── config/config.go         # godotenv + typed env config
│   │   │   ├── auth/                     # Argon2id + JWT (§17.4)
│   │   │   ├── http/                     # Gin handlers, middleware, router
│   │   │   ├── sse/                      # the live-score SSE hub (§17.3)
│   │   │   ├── store/                    # pgx/pgxpool: primary+replica, sharding (§18)
│   │   │   └── redis/                    # cache, pub/sub backplane, rate limits (§19)
│   │   ├── ent/                          # Ent schema + generated code
│   │   ├── migrations/                   # goose SQL migrations
│   │   ├── Dockerfile                    # multi-stage → tiny non-root image
│   │   └── .air.toml                     # hot-reload in dev
│   └── web/                             # the Next.js frontend (§22)
│       ├── app/(public)/                 # public user dashboard → app.scorelive.com
│       ├── app/(admin)/                  # admin dashboard → admin.scorelive.com
│       ├── lib/api.ts                    # typed API client (in-memory token, refresh)
│       ├── Dockerfile
│       └── next.config.ts
├── infra/
│   ├── traefik/
│   │   ├── traefik.yml                   # STATIC config (§4.2)
│   │   └── dynamic/security.yml          # shared middlewares, TLS options (§4.3, §8.4)
│   ├── postgres/                         # primary + replica + shard configs (§18)
│   ├── redis/redis.conf                  # §19
│   ├── compose.yaml                      # the whole stack (§27)
│   ├── compose.prod.yaml                 # prod overrides (replicas, resources)
│   └── secrets/                          # gitignored: jwt, pg, cf token
├── .github/workflows/deploy.yml          # CI/CD auto-deploy (§26)
└── README.md
```

The split is deliberate: **`apps/`** holds the deployable services (each with its own Dockerfile), **`infra/`** holds everything about *running* them (Traefik config, Compose, DB/Redis config). One repo, clear boundaries, atomic cross-cutting changes. The `apps/web` uses Next.js **route groups** `(public)` and `(admin)` to build two dashboards from one codebase, routed to different subdomains by Traefik (§22).

### 16.2 The network topology **[I/A]**

The Compose stack uses **two Docker networks** for isolation:

- **`edge`** — Traefik and the *public-facing* services (Next.js, the API). Traefik watches this network.
- **`backend`** — the API and the *data stores* (Postgres primary/replica/shards, Redis). Traefik is **not** on this network, so it cannot reach the databases even if misconfigured.

The API is on **both** networks (it receives traffic from Traefik on `edge`, and reaches the data on `backend`); Postgres and Redis are on `backend` **only**. This two-network split means the databases are unreachable from the edge by construction — a stronger guarantee than "no published ports" alone. It's the network-level expression of the trust boundary.

---

## 17. The Backend — Gin, Ent, SSE and REST

### 17.1 The stack and why each piece **[I/A]**

The API is the standard high-performance Go stack this library teaches, chosen for a reason at 500k users:

- **Gin** — the HTTP framework (routing, middleware, binding). Fast, minimal overhead.
- **pgx + pgxpool** — the PostgreSQL driver and pool. `pgxpool` is what lets one instance handle thousands of concurrent requests over a bounded set of DB connections (the [pgx guide](GO_PGX_GUIDE.md) §22 has the pool-sizing math — crucial here).
- **Ent** — the type-safe query layer (compile-checked queries, no SQL-injection surface).
- **goose** — versioned migrations, run as a gated deploy step (never auto-on-boot).
- **JWT (golang-jwt v5) + Argon2id** — stateless auth: Argon2id verifies passwords, a short-lived JWT authorizes requests, so *any* API instance can serve *any* request (no server-side session lookup on the hot path).
- **godotenv** — load `.env` in dev; real env/secrets in prod (12-factor).
- **Air** — hot-reload in development only.

The architectural rule that makes this scale: **the API is stateless.** No per-user state in an instance's memory — sessions, rate-limit counters, and the SSE fan-out backplane all live in **Redis**; durable data in **Postgres**. That's what lets Traefik add or remove instances freely (§20) and route any request anywhere.

### 17.2 REST and SSE on the same backend **[I/A]**

The backend serves two very different traffic shapes:

- **REST** (`api.scorelive.com`) — short request/response: sign up, log in, fetch a match list, subscribe to a competition. Standard Gin handlers, DB-backed, JWT-authed.
- **SSE** (`sse.scorelive.com`) — long-lived streams: a client opens `GET /stream` and holds it open while scores push down in real time. Built exactly as the [Go SSE guide](GO_SSE_GUIDE.md) describes: a broker/hub, per-client bounded channels, backpressure + slow-consumer eviction, heartbeats, `Last-Event-ID` resume, and a **Redis pub/sub backplane** so a score entered on `api-1` reaches subscribers connected to `api-7`.

They share the same binary and data layer but are routed as **separate Traefik services** (via the `api.`/`sse.` hosts), so you can give the SSE path its own middlewares (longer timeouts, different rate limits) and scale it independently — long-lived SSE connections consume a slot for minutes, so their scaling math differs entirely from REST's (§24).

### 17.3 The live-score flow, end to end **[I/A]**

```text
Owner enters a score (admin dashboard)
  → POST api.scorelive.com/admin/matches/42/score   (JWT: owner role; IP-allow-listed edge)
  → Go API validates, writes to Postgres PRIMARY (the shard owning match 42)
  → API PUBLISHES the update to Redis channel "match:42"
  → EVERY API instance is SUBSCRIBED to Redis → each pushes the event to its local SSE clients
  → 500k browsers on sse.scorelive.com receive "score" events instantly (each on whichever instance it's connected to)
```

This is the **persist-then-broadcast** pattern (write to Postgres first for durability + reconnect replay, then publish to Redis for live fan-out) from the SSE guide, scaled across many instances via the Redis backplane. The owner writes once; Redis fans it out; every instance delivers to its own connected fans. No instance needs to know about any other — Redis is the shared nervous system. This is the core of how one score update reaches half a million screens.

### 17.4 Auth: stateless JWT so any instance serves any request **[I/A]**

Login (`POST /auth/login`) verifies the password with **Argon2id** and issues a short-lived **access JWT** (+ a refresh token, rotated, in an HttpOnly cookie — the [JWT/Argon2 guide](GO_JWT_ARGON2_GUIDE.md) pattern). Every subsequent request carries the JWT; each API instance validates it **locally** (signature check, no DB/Redis hit on the happy path) and reads the user id + role from the claims. Because validation is local and stateless, **request N+1 can hit a different instance than request N** with zero coordination — exactly what Traefik's load balancing needs. Role (`user` vs `owner`) in the JWT claims gates the admin endpoints, *in addition to* the edge IP allow-list (defense in depth — §21). Revocation (logout, ban) uses a short Redis deny-list checked only for sensitive actions, keeping the hot path DB-free.

---

## 18. PostgreSQL 17 — Read Replicas and Sharding

### 18.1 The two scaling axes **[A]**

A database scales along two independent axes, and ScoreLive uses both:

- **Read scaling → replicas.** A **read replica** is a copy of the primary that stays in sync via **streaming replication** and serves *read-only* queries. Reads (fetching match lists, scores, standings — the vast majority of a live-score app's traffic) go to the replica(s); writes go to the primary. Add replicas to add read capacity.
- **Write scaling → sharding.** A single primary can only absorb so many writes. **Sharding** splits the data across multiple independent Postgres instances ("shards") by a **shard key**, so writes spread across them. This is heavier machinery — reach for it only when a single primary's write throughput is genuinely the ceiling.

For a live-score SaaS, reads dominate massively (500k fans reading, one owner writing per match), so **read replicas are the primary win**; sharding is the write-scale insurance for when the platform hosts thousands of concurrent matches.

### 18.2 Read replica setup **[A]**

Run a primary and a streaming replica as separate Compose services on the `backend` network:

```yaml
  pg-primary:
    image: postgres:17
    environment: { POSTGRES_DB: scorelive, POSTGRES_USER: app, POSTGRES_PASSWORD_FILE: /run/secrets/pg_pw }
    command: ["postgres", "-c", "wal_level=replica", "-c", "max_wal_senders=10", "-c", "hot_standby=on"]
    volumes: [pg_primary:/var/lib/postgresql/data]
    networks: [backend]
    secrets: [pg_pw]

  pg-replica:
    image: postgres:17
    # Initialized from the primary via pg_basebackup, then streams WAL. (Init script omitted;
    # the DB Server Admin guide covers replication setup in depth.)
    environment: { PGUSER: replicator, POSTGRES_PASSWORD_FILE: /run/secrets/pg_pw }
    command: ["postgres", "-c", "hot_standby=on"]
    volumes: [pg_replica:/var/lib/postgresql/data]
    networks: [backend]
    depends_on: [pg-primary]
    secrets: [pg_pw]
```

The app then holds **two pgx pools** — one to `pg-primary` (writes + read-your-writes), one to `pg-replica` (bulk reads) — and routes each query to the right one:

```go
// internal/store/store.go
type Store struct {
	write *pgxpool.Pool   // → pg-primary:5432
	read  *pgxpool.Pool   // → pg-replica:5432
}
// Reads that tolerate slight staleness go to the replica; writes and
// read-after-write go to the primary. This is a DELIBERATE per-query choice.
func (s *Store) ListMatches(ctx context.Context) ([]Match, error) { /* uses s.read */ }
func (s *Store) SetScore(ctx context.Context, id int64, sc Score) error { /* uses s.write */ }
```

The one gotcha is **replication lag**: the replica is milliseconds-to-seconds behind. So a user who just wrote data and immediately reads it might not see it if that read hits the replica — hence "read-your-writes" queries deliberately use the primary. Route reads to the replica *only* when slight staleness is acceptable (which, for public score lists refreshed by SSE anyway, it almost always is).

### 18.3 Application-level sharding **[A]**

When writes outgrow one primary, **shard** by a key — for ScoreLive, `competition_id` (or a hash of it) is natural: each competition's matches and events live entirely on one shard, so a score write touches exactly one shard. The app maps the shard key to a pool:

```go
// internal/store/shards.go
type ShardedStore struct {
	shards []*pgxpool.Pool   // e.g. 4 independent Postgres primaries
}
// Deterministically pick the shard for a competition. Same key → same shard, always.
func (s *ShardedStore) shardFor(competitionID int64) *pgxpool.Pool {
	return s.shards[competitionID%int64(len(s.shards))]
}
func (s *ShardedStore) SetScore(ctx context.Context, competitionID, matchID int64, sc Score) error {
	pool := s.shardFor(competitionID)          // route the write to the owning shard
	// ... write to `pool` ...
}
```

**Sharding trade-offs to respect:** a query that needs data from *all* shards (a global leaderboard) must **scatter-gather** (query every shard, merge results) — more complex and slower; cross-shard transactions are hard (avoid them by choosing a shard key that keeps related data together); and **rebalancing** (adding a 5th shard) requires migrating data. So shard *late*, choose the key so that virtually all queries stay within one shard, and keep cross-shard operations (leaderboards) rare and cached in Redis. For most of ScoreLive's life, **read replicas alone** carry the load; sharding is the write-scale escape hatch documented and ready, adopted only when a single primary's writes are measured to be the ceiling. (The [Database Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md) guides go deeper on replication and partitioning.)

---

## 19. Redis — Cache, Backplane and Rate Limits

### 19.1 Redis's three jobs here **[A]**

Redis is the shared, in-memory brain that makes the stateless multi-instance design work. It does three distinct jobs:

- **Cache** — hot reads (a match's current score, a competition's standings) are cached so 500k fans reading the same match don't generate 500k Postgres queries. The score is written to Postgres *and* Redis; reads hit Redis first. This is the difference between a database that's bored and one that's on fire.
- **Pub/Sub backplane** — the SSE fan-out across instances (§17.3): the API publishes score updates to a Redis channel; every instance subscribes and pushes to its local SSE clients. Without this, a score entered on one instance wouldn't reach fans connected to another.
- **Rate-limit + session state** — per-user rate-limit counters (token buckets) and the JWT refresh/deny-list live in Redis, so limits and revocation are consistent across all instances (a user throttled on `api-1` is throttled on `api-9` too).

### 19.2 Why Redis is what makes it scale **[A]**

The chain of reasoning is worth stating explicitly, because it's the crux of the whole architecture: the API must be **stateless** so Traefik can load-balance freely across many instances → therefore no shared state can live in an instance's memory → therefore cache, the SSE backplane, and rate-limit/session state must live in a **shared store** → that store is **Redis** (fast enough to be on the hot path, unlike Postgres). Remove Redis and either the app stops scaling horizontally (state trapped in one instance) or every read hammers Postgres (which then becomes the bottleneck). Redis is the component that lets "just add more API instances" actually work.

### 19.3 Running it and guarding memory **[A]**

Redis runs as one service on the `backend` network (a Redis Cluster/replica for HA at the largest scale — the [Redis guide](REDIS_GUIDE.md)), with production settings that matter under spike load:

```ini
# infra/redis/redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru      # under memory pressure, evict least-recently-used CACHE keys
appendonly yes                     # persist so a restart doesn't drop rate-limit/session state
requirepass ${REDIS_PASSWORD}      # AUTH even on the private network (defense in depth)
```

`maxmemory` + `allkeys-lru` prevent a growing cache from OOM-killing the box during a spike (it evicts cold cache entries instead) — a critical safety valve when 500k users flood in for a big match. The rate-limit and session keys are small and hot, so LRU keeps them; the evictable pressure is cache, which is fine to lose (it just re-reads from Postgres). Size `maxmemory` with headroom and **alert** on eviction rate and memory (§28).

---

## 20. Running Many Backend Instances behind Traefik

### 20.1 Scaling the API with Compose **[A]**

Because the API is stateless, running many copies is trivial — and Traefik load-balances across all of them automatically (§7.1). Two ways to declare replicas:

```yaml
  api:
    image: registry.scorelive.com/api:${APP_VERSION}
    deploy:
      replicas: 4                 # run 4 identical instances (Compose/Swarm)
      resources:
        limits: { cpus: "1.5", memory: 768M }
    environment:
      DATABASE_WRITE_URL: postgres://app@pg-primary:5432/scorelive
      DATABASE_READ_URL:  postgres://app@pg-replica:5432/scorelive
      REDIS_ADDR: redis:6379
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.scorelive.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=le-dns"
      - "traefik.http.routers.api.middlewares=security-headers@file,api-ratelimit"
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/healthz"
      # a SEPARATE router for the SSE host, same service, its own middlewares:
      - "traefik.http.routers.sse.rule=Host(`sse.scorelive.com`)"
      - "traefik.http.routers.sse.entrypoints=websecure"
      - "traefik.http.routers.sse.tls.certresolver=le-dns"
      - "traefik.http.routers.sse.middlewares=security-headers@file,sse-ratelimit"
      - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"
    networks: [edge, backend]     # edge (Traefik reaches it) + backend (reaches DB/Redis)
```

All four instances share the same labels, so Traefik sees **one service with four healthy servers** and round-robins across them. Scale to eight with `docker compose up -d --scale api=8` (or bump `replicas`), and the four new instances join the rotation the instant they pass their health check — **no Traefik config change, no reload, no dropped requests.** That is the entire payoff of Traefik's dynamic discovery for a scaling SaaS.

### 20.2 Zero-downtime deploys with Traefik **[A]**

Traefik's health checks make rolling deploys clean: when you replace an instance, Traefik pulls the old (now-unhealthy/stopping) one from rotation and routes to the survivors, then adds the new one once *it's* healthy. Combined with the deploy script (§26), you roll instances one at a time and at no point is the service down. For graceful draining, your Go app should handle `SIGTERM` by finishing in-flight requests (and closing SSE streams so clients reconnect elsewhere) before exiting — Traefik stops sending it new requests as soon as its health check fails, so the drain window is clean.

### 20.3 Sizing the fleet **[A]**

How many instances? Driven by two very different workloads: **REST** (CPU-bound, short requests — scale by request rate and p99 latency) and **SSE** (memory/connection-bound, long-lived — scale by *concurrent connections*, since each holds a goroutine + buffer for minutes). Because they scale on different axes, ScoreLive can run the **same image** but conceptually treat the `api.` and `sse.` traffic separately — even pinning some instances to mostly-SSE duty at the largest scale. The pgxpool math from the [pgx guide](GO_PGX_GUIDE.md) §22 is the hard ceiling to respect: `instances × pool_size` must stay under Postgres `max_connections` (per primary/replica), which is exactly why PgBouncer (§24) enters the picture as the fleet grows.

---

## 21. The Hidden Admin Plane

### 21.1 The requirement, precisely **[A]**

The owner's admin dashboard must be **invisible to the public**. Not just "password-protected" — *invisible*: a random visitor (or an attacker scanning for admin panels) hitting `admin.scorelive.com` should get **nothing** (a `403`, or ideally a connection that looks like the site doesn't exist), never a login form. The login page, the admin API, the very existence of the admin plane, should be undetectable unless you're coming from an allow-listed owner IP. This shrinks the admin attack surface to essentially zero: you can't attack a login page you can't reach.

### 21.2 The layered implementation **[A]**

We enforce this at **three layers**, so no single failure exposes it:

**Layer 1 — Cloudflare (edge).** A Cloudflare WAF/firewall rule blocks all traffic to `admin.scorelive.com` except from the owner's IP(s), before it ever reaches the origin. First line, absorbs scanning at Cloudflare's edge.

**Layer 2 — Traefik `ipAllowList` (proxy).** The admin routers carry an IP-allow-list middleware; a non-allow-listed request gets `403` at Traefik, never reaching the app:

```yaml
  web-admin:
    image: registry.scorelive.com/web:${APP_VERSION}   # the admin Next.js build
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.admin.rule=Host(`admin.scorelive.com`)"
      - "traefik.http.routers.admin.entrypoints=websecure"
      - "traefik.http.routers.admin.tls.certresolver=le-dns"
      # the gate: only owner IPs, read from the real client IP behind Cloudflare (§15.5)
      - "traefik.http.routers.admin.middlewares=admin-ipallow,security-headers@file"
      - "traefik.http.middlewares.admin-ipallow.ipallowlist.sourcerange=203.0.113.5/32,198.51.100.7/32"
      - "traefik.http.middlewares.admin-ipallow.ipallowlist.ipstrategy.depth=1"   # trust CF's XFF
      - "traefik.http.services.admin.loadbalancer.server.port=3000"
    networks: [edge]
```

The **same `admin-ipallow` middleware also guards the admin *API* routes** (`api.scorelive.com/admin/*`) via a high-priority router, so both the admin UI and the admin API are IP-gated:

```yaml
      # on the api service — a high-priority admin router, IP-gated:
      - "traefik.http.routers.api-admin.rule=Host(`api.scorelive.com`) && PathPrefix(`/admin`)"
      - "traefik.http.routers.api-admin.priority=100"
      - "traefik.http.routers.api-admin.middlewares=admin-ipallow,security-headers@file"
      - "traefik.http.routers.api-admin.entrypoints=websecure"
      - "traefik.http.routers.api-admin.tls.certresolver=le-dns"
```

**Layer 3 — JWT role check (app).** Even a request that somehow reaches the admin API must carry a JWT with the `owner` role, or the Go handler rejects it. This is the innermost check: IP allow-listing proves *where you are*, the role claim proves *who you are*, and admin actions require both.

### 21.3 Why `ipstrategy.depth=1` is the linchpin **[A]**

This one line is what makes it actually secure behind Cloudflare, and getting it wrong is a silent catastrophe. Without it, Traefik sees **Cloudflare's** IP as the source, so `sourcerange=203.0.113.5/32` (the owner's real IP) matches *nothing that arrives* — and depending on how you'd (mis)configure it, you'd either lock out the owner too, or, if you naively allow-listed Cloudflare's ranges to "make it work," you'd allow **the entire internet** (since everything arrives from Cloudflare). `ipstrategy.depth=1` tells Traefik: "the real client is the first IP from the right in `X-Forwarded-For`" (with exactly one trusted proxy, Cloudflare, in front). Now the allow-list matches the *actual visitor's* IP. Combined with `forwardedHeaders.trustedIPs` (§15.5) so Traefik trusts Cloudflare to set that header, the hidden admin plane works as intended. Test it ruthlessly: from an allow-listed IP you see the login; from anywhere else, `403` — verify from a phone on cellular (a different IP) that the admin plane truly doesn't exist for you.

### 21.4 Operational reality — dynamic owner IPs **[A]**

Owner IPs change (home ISP, travel). Options, best first: put the owner behind a **small VPN/bastion with a static IP** and allow-list *that* (the professional answer — the owner connects to the VPN, then reaches admin from its stable IP); or use **Cloudflare Access** (identity-based, not IP-based — the owner authenticates with Cloudflare/SSO and *then* the origin is reached, no IP list to maintain); or, lowest-effort, a quick script/endpoint the owner hits to add their current IP to the allow-list. Cloudflare Access is arguably the cleanest modern answer (zero-trust: prove identity, not location), and it composes with everything above. Whichever you choose, the principle holds: **the admin plane is reachable only after passing an edge gate that the public cannot pass.**

---

## 22. The Next.js Frontend — an Untrusted Client

### 22.1 The trust boundary **[I/A]**

The Next.js app runs in the **user's browser** — an environment you do not control and must never trust. Anything the client sends can be forged; anything you send to the client is visible to the user. This is the single most important security principle for the frontend: **the backend validates and authorizes everything, server-side, on every request; the frontend's checks are UX, not security.** Hiding an "admin" button in the UI does not protect the admin API — the IP allow-list and the JWT role check (§21) do. The frontend's job is to be a good client (fast, correct, pleasant); the backend's job is to assume the client is hostile.

### 22.2 Two dashboards from one codebase **[I/A]**

The Next.js app builds **two dashboards** using route groups, routed to different subdomains by Traefik:

- `app/(public)/*` → `app.scorelive.com` — the public user dashboard (signup, login, live scores via SSE).
- `app/(admin)/*` → `admin.scorelive.com` — the owner dashboard (score entry, user management, analytics).

You can ship them as **one image serving both** (Next.js routes by hostname) or **two builds** (a public image and an admin image, so the admin bundle — with its admin components and code — is never even sent to public users). For a hidden admin plane, **two builds is stronger**: the admin JavaScript never reaches a public browser, so there's nothing to reverse-engineer, and the admin image only runs behind the IP gate. The monorepo builds both from the same `apps/web` source with different entry configs.

### 22.3 Consuming REST and SSE from the client **[I/A]**

The frontend talks to the backend over two channels, both cross-subdomain (`app.` → `api.`/`sse.`), which means **CORS** and cookie settings matter:

- **REST** via a typed client (`lib/api.ts`): the **access JWT is held in memory** (never `localStorage` — XSS-safe), with a silent-refresh flow using the HttpOnly refresh cookie. Because `app.` and `api.` are different subdomains of the same site, the refresh cookie is set for `.scorelive.com` with `SameSite=Lax/Strict` and `Secure`; the API's CORS allows the `app.` origin with credentials (never `*` with credentials).
- **SSE** via `EventSource` to `sse.scorelive.com` — with `withCredentials` so the auth cookie/ticket rides along (SSE can't set headers; the [Go SSE guide](GO_SSE_GUIDE.md) §6 covers the cookie/ticket auth). Scores stream in and update the UI (feed them into TanStack Query's cache or React state; re-render only what changed).

### 22.4 Frontend performance — no wasted resource **[I/A]**

At 500k users, an inefficient frontend is expensive at both ends. The optimizations that matter:

- **Cache static assets at Cloudflare** (§15.3): Next.js `_next/static/*` is content-hashed and immutable — cache it forever at the edge, so the origin serves almost no static bytes.
- **Stream and render only deltas**: the SSE feed sends *score changes*, not full page reloads; the UI updates the one changed number. Never poll when you have SSE.
- **One `EventSource` per tab** over **HTTP/2** (§15.6) — the browser's ~6-connection-per-host limit means many streams starve other requests on HTTP/1.1.
- **Server Components + minimal client JS**: render as much as possible server-side, ship a small client bundle, hydrate only interactive parts (the [Next.js guide](NEXTJS_16_GUIDE.md) covers RSC).
- **Debounce/coalesce UI updates** during a goal flurry so a burst of score events doesn't cause a render storm.

Efficiency is a security and cost property here, not just UX: a frontend that opens redundant connections or re-fetches needlessly multiplies load by 500k, turning a manageable spike into an outage and a big cloud bill.

---

## 23. Banking-Grade Security

The complete security posture for ScoreLive, layered so an attacker must defeat *every* layer. Each maps to its section.

### 23.1 The layered model **[A]**

| Layer | Controls | Section |
|---|---|---|
| **Cloudflare edge** | WAF, DDoS absorption, bot mgmt, edge rate limiting, admin firewall rule | §15.3, §21.2 |
| **Origin lockdown** | VPS accepts 80/443 only from Cloudflare (or Cloudflare Tunnel) | §15.4 |
| **VPS host** | key-only SSH, default-deny firewall, fail2ban, auto-patches, log rotation | §14.2 |
| **Traefik edge** | TLS everywhere, security headers, per-route rate limits, in-flight caps, `ipAllowList` | §8, §9, §21 |
| **Network isolation** | two Docker networks; DBs unreachable from edge; no DB host ports | §16.2 |
| **App auth** | Argon2id, short-lived JWT + rotating refresh, role checks, Redis deny-list | §17.4 |
| **Data** | secrets in files/Docker secrets (not Git/images), least-priv DB users, TLS to DB | §14, §18 |
| **Frontend** | untrusted client, in-memory token, strict CORS, CSP, XSS-safe rendering | §22 |

### 23.2 The Traefik/edge specifics **[A]**

- **TLS 1.2+ everywhere**, HSTS with preload, modern cipher suites (the `modern@file` TLS option, §4.3) — enforced by Traefik for every router.
- **Security headers on every response** via the shared `security-headers@file` middleware (§8.4) — a new backend can't forget them.
- **Rate limiting at three levels**: Cloudflare (edge), Traefik (`ratelimit` middleware per route — stricter on `/auth/login`), and the app (Redis token bucket per user). The earlier the throttle, the cheaper.
- **In-flight request caps** (`inflightreq`, §8.2) so a flood can't exhaust the app's goroutines/connections.
- **The Docker socket proxy** (§5.3): replace Traefik's raw `docker.sock:ro` mount with `tecnativa/docker-socket-proxy` exposing only the read endpoints Traefik needs, on a private network:

```yaml
  dockerproxy:
    image: tecnativa/docker-socket-proxy:latest
    environment: { CONTAINERS: 1, SERVICES: 1, TASKS: 1, POST: 0 }   # read-only, minimal API surface
    volumes: ["/var/run/docker.sock:/var/run/docker.sock:ro"]
    networks: [socket]
  traefik:
    command:
      - "--providers.docker.endpoint=tcp://dockerproxy:2375"   # Traefik talks to the PROXY, not the socket
    networks: [edge, socket]
```

Now Traefik never touches the real socket — a compromised Traefik can *read* container metadata but not control the host. This is the difference between a proxy breach being "annoying" and "total host compromise."

### 23.3 The app and data specifics **[A]**

- **Argon2id** password hashing (memory-hard, tuned to ~200–500ms), constant-time verify, a dummy-verify on unknown users to avoid a timing oracle (§17.4, the [JWT/Argon2 guide](GO_JWT_ARGON2_GUIDE.md)).
- **JWT**: short-lived access token, rigorous algorithm pinning (reject `alg:none`/confusion), refresh rotation with reuse detection, a Redis deny-list for revocation checked on sensitive actions.
- **Ent/pgx parameterized queries** — SQL injection is structurally impossible; never build SQL by string concatenation.
- **Secrets** in Docker secrets / gitignored files (`chmod 600`), never in Git or images; **least-privilege DB users** (the app user can't `DROP`; the replica user is read-only); **least-privilege Cloudflare token** (DNS-edit on one zone).
- **Money/scores as exact types** (`numeric`/integers, never float) where correctness matters.
- **Audit logging** of every admin action (who changed which score when) — you need it for a system real money rides on.

Nothing here is optional at 500k users on a system with paying customers. Security is layered on purpose: each control is cheap, and the attacker must beat all of them.

---

## 24. Load Balancing and Handling 500k Users

### 24.1 What "500k users" actually demands **[A]**

"500,000 users" is not one number — decompose it. The design targets are roughly: a **baseline** of steady REST traffic (browsing, logins) plus a **large pool of concurrent SSE connections** (fans watching live), punctuated by **spikes** when a big match kicks off (a flood of new connections and logins in a few minutes). These three stress different resources — REST stresses CPU and DB reads, SSE stresses concurrent connections and memory, spikes stress *everything at once and suddenly* — so the plan addresses each.

### 24.2 The load-balancing layers **[A]**

Traffic is balanced at multiple points, each doing part of the job:

1. **Cloudflare** distributes across its global edge and absorbs/curbs spikes and attacks before they reach the origin — the first and biggest lever for a public SaaS.
2. **Traefik** load-balances across the N API instances (weighted round-robin, health-checked). Scale N up during a match, and Traefik spreads load instantly.
3. **The app → DB**: reads spread across **read replicas**, writes to the primary (or shards) — so 500k readers don't converge on one database (§18).
4. **Redis** absorbs the read hot path (cached scores) so most reads never touch Postgres at all.

The load balancer only helps if there's capacity to balance *onto* — so §24.3 is about having enough backends, and §18–§19 about the data tier not being the bottleneck the LB can't fix.

### 24.3 Absorbing the kickoff spike **[A]**

A spike is the hard case: 100k users connect in five minutes. Defenses, in order of leverage:

- **Cache the hot data** so the spike hits Redis, not Postgres (§19.1). One match's score read by 100k fans is *one* Redis key, refreshed on each write — not 100k DB queries.
- **The SSE backplane fan-out** (§17.3): a score update is written once and *published once* to Redis; each instance delivers to its local subscribers. Delivering to 100k fans is 100k cheap socket writes spread across N instances, not 100k DB reads.
- **Autoscale the fleet ahead of known events**: matches are scheduled, so scale the API instances *up before* kickoff (a cron or a manual `--scale api=12`), not reactively. Pre-provisioning for known spikes is far safer than scrambling during one.
- **Rate limit and shed load gracefully**: Cloudflare + Traefik + app limits ensure that if the spike exceeds capacity, you shed *excess* requests (429s, a "try again" page) rather than *collapsing entirely* — a degraded service beats a dead one.
- **Circuit breakers** (§7.4) stop a struggling dependency from cascading into total failure.
- **PgBouncer** in front of Postgres so thousands of app connections multiplex onto dozens of real DB connections — otherwise the connection count is the ceiling the fleet hits first.

### 24.4 The connection-count math for SSE **[A]**

SSE is the unusual load here: each of 500k fans holds a connection open for the whole match. At the OS level this means **file-descriptor and memory limits** matter more than CPU. Tune the host (`ulimit`/`sysctl` — raise `nofile`, tune `somaxconn` and ephemeral ports, per the [Node WebSockets guide](NODE_WEBSOCKETS_GUIDE.md)'s C10M section, which applies to any long-connection server), keep per-connection memory tiny (a small bounded channel + minimal goroutine state — the Go SSE hub is built for this), and spread the connections across enough instances and, past one machine, enough *machines*. 500k concurrent connections is very achievable for Go, but only if each connection is cheap and you've raised the OS limits that otherwise cap you at ~1024 FDs per process.

### 24.5 Horizontal scaling past one VPS **[A]**

One big VPS carries a lot, but 500k concurrent SSE eventually wants **multiple machines**. Because the app is stateless with shared Redis/Postgres, this is straightforward: run the API fleet across several VPSes; put Traefik (or Cloudflare Load Balancing) in front distributing across them; Redis and Postgres remain the shared services all app servers reach (give Postgres its own machine[s], and Redis its own, first). Traefik itself can run on each app node (each fronting local instances) with Cloudflare balancing across nodes, or as a dedicated edge tier. The stateless design means adding a machine is adding capacity, linearly — the whole point of everything built so far.

---

## 25. Scaling, Traffic Spikes and Performance

### 25.1 The efficiency mandate **[A]**

The user was explicit: **no wasted resources.** At 500k users, waste is multiplied by 500k — a needless allocation per request, a missing index, an over-eager re-render, or a redundant DB round-trip becomes a real outage and a real bill. Efficiency here is not premature optimization; it's the difference between fitting on modest hardware and needing ten times as much. The discipline: measure (§11 metrics), find the hot path, make it cheap, and don't do work you don't have to.

### 25.2 Backend efficiency **[A]**

- **Right-size the pgxpool** (the pgx guide's §22 math) — too small queues requests, too large exhausts Postgres. Measure and tune.
- **Cache the hot reads in Redis** so the DB is bored, not on fire (§19.1).
- **Read from replicas** so the primary is reserved for writes (§18.2).
- **Reuse buffers, avoid per-request allocations** on the hot path; stream JSON rather than building giant slices; use `sync.Pool` where profiling shows churn (the Go testing/pgx guides cover profiling).
- **SSE hub efficiency**: bounded per-client channels, non-blocking sends with slow-consumer eviction, one broadcast fan-out per event — so one score reaches 100k fans in O(connections) cheap writes, not O(connections) DB hits.
- **Set container resource limits** so one service can't starve others, and so the scheduler can pack instances efficiently.

### 25.3 Edge and frontend efficiency **[A]**

- **Compression** (`compress` middleware) shrinks JSON/SSE payloads.
- **HTTP/2 and HTTP/3** multiplex many requests over one connection (fewer handshakes, the connection-limit fix for SSE).
- **CDN caching** at Cloudflare offloads static assets and cacheable responses entirely (§15.3).
- **Frontend**: server components, minimal client JS, delta-only SSE updates, one stream per tab, coalesced renders (§22.4).

### 25.4 Measuring and proving it **[A]**

You cannot claim "efficient" without numbers. **Load-test** the system (k6, Gatling, or Vegeta) at target concurrency *before* the real spike — simulate 100k connecting in five minutes, watch Traefik's p99 latency and error rate, Postgres connection count and slow-query log, Redis eviction rate, and host CPU/memory/FDs. The load test tells you where it breaks *first* (usually the DB connection count or an unindexed query), you fix that, and repeat until it holds at target with headroom. A system that "should" handle 500k but was never tested at 500k will discover its bottleneck during the real event — the most expensive time to find it.

---

## 26. CI/CD — GitHub Actions Auto-Deploy to the VPS

### 26.1 The pipeline **[A]**

A push to `main` should **test, build, and deploy** automatically — no one SSHing in to run commands by hand (error-prone, unauditable). The pipeline: GitHub Actions runs the tests, builds the API and web images, pushes them to a registry (GitHub Container Registry `ghcr.io`), then connects to the VPS over SSH and runs a **rolling deploy** that pulls the new images and replaces instances one at a time (zero downtime, §20.2).

### 26.2 The workflow **[A]**

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.26" }
      - run: cd apps/api && go test ./...          # gate: don't ship failing code
      # (add web tests: cd apps/web && npm ci && npm test)

  build-and-push:
    needs: test                                    # only build if tests passed
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - name: Log in to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
      - name: Build & push API
        run: |
          docker build -t ghcr.io/scorelive/api:${{ github.sha }} apps/api
          docker push  ghcr.io/scorelive/api:${{ github.sha }}
      - name: Build & push web
        run: |
          docker build -t ghcr.io/scorelive/web:${{ github.sha }} apps/web
          docker push  ghcr.io/scorelive/web:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host:     ${{ secrets.DEPLOY_HOST }}
          username: deploy
          port:     ${{ secrets.SSH_PORT }}
          key:      ${{ secrets.DEPLOY_SSH_KEY }}   # a deploy-ONLY key, revocable
          script: |
            cd /home/deploy/scorelive/infra
            export APP_VERSION=${{ github.sha }}
            ./deploy.sh ${{ github.sha }}            # the rolling-deploy script (§26.3)
```

The structure — **test → build/push → deploy**, each gated on the previous — means broken code never reaches production and the image that ran your tests is byte-for-byte the image that runs in prod. The **VPS never builds** (no source, no compiler on the box — smaller attack surface); it only pulls tested images.

### 26.3 The rolling-deploy script on the VPS **[A]**

```bash
#!/usr/bin/env bash
# infra/deploy.sh — pull new images, migrate, roll API instances one at a time.
set -euo pipefail
NEW="$1"; export APP_VERSION="$NEW"
cd /home/deploy/scorelive/infra

docker compose pull api web                          # fetch new images (no downtime yet)
docker compose run --rm api /app migrate up          # gated migrations (never auto-on-boot)

# Roll each API replica: recreate, wait until Traefik-healthy, then the next.
for i in 1 2 3 4; do
  docker compose up -d --no-deps --scale api=4 api   # recreate; Traefik drains the old, adds the new
  sleep 8                                             # (a real script polls /healthz per instance)
done
docker compose up -d --no-deps web                   # roll the frontend
echo "Deployed $NEW."
```

Traefik's health checks make this safe: as each old instance stops, Traefik routes around it to the healthy ones; each new instance joins rotation only once healthy. `set -euo pipefail` aborts on any error rather than half-deploying.

### 26.4 CI/CD security **[A]**

Two rules: use a **dedicated deploy SSH key** (its own pair, in `deploy`'s `authorized_keys`, revocable without touching your personal key), and keep **all credentials in GitHub's encrypted secrets** (`DEPLOY_SSH_KEY`, `DEPLOY_HOST`, registry tokens) — never in the workflow file or the repo. Scope the registry token to push-only. Pin action versions (ideally by SHA) so a compromised action tag can't inject code into your deploy. Now `git push origin main` → tested, built, and rolled out to 500k users with zero downtime, fully audited in the Actions log. (The [CI/CD guide](GITHUB_ACTIONS_CICD_GUIDE.md) covers the pipeline, OIDC keyless auth, and hardening in depth.)

---

## 27. Building ScoreLive — Every File and the Full Stack

This is the hands-on section: the actual files you write to build ScoreLive, in the order you write them. It is deliberately concrete — a beginner can follow it top to bottom and end up with a running system. Where a file's *internals* are taught in depth elsewhere (Argon2 hashing, JWT signing, the SSE hub mechanics), we show the exact ScoreLive version and point to the sibling guide for the theory, so this stays buildable without re-teaching six other guides. Every code block is headed with its file path.

### 27.1 The build order — what to do, in sequence **[I/A]**

Follow these steps in order; each produces working, testable progress:

1. **Scaffold the monorepo** (§16.1 tree): create `apps/api`, `apps/web`, `infra/`.
2. **Backend config** (§27.2) → **Ent schema + migration** (§27.7) → **the store** (§27.4) → **SSE hub + backplane** (§27.5) → **handlers + auth** (§27.6) → **`main.go`** (§27.3). Run it locally with `air` against a local Postgres+Redis; hit `/healthz`.
3. **Dockerize** the API and web (§27.8).
4. **Traefik config**: `traefik.yml` (static) + `dynamic/security.yml` (§27.9), and `redis.conf` (§19.3).
5. **The data tier**: Postgres primary + **replica init** (§27.10) + Redis.
6. **The frontend** (§27.11): the API client + a live-scores page consuming SSE + the admin build.
7. **Compose it all** (§27.12) and `docker compose up -d` on your dev box; verify routing via the Traefik dashboard.
8. **Go to production**: point DNS at the VPS, harden the host (§14), lock the origin to Cloudflare (§15.4), wire CI/CD (§26). Load-test (§25.4).

You now build each file. Start a local Postgres and Redis (`docker run` or a dev compose) so you can run the API as you go.

### 27.2 Backend config — `internal/config/config.go` **[I/A]**

```go
// apps/api/internal/config/config.go
package config

import (
	"os"

	"github.com/joho/godotenv"
)

// Config is loaded once at startup from the environment (godotenv fills it in dev).
type Config struct {
	WriteDSN    string   // → pg-primary
	ReadDSN     string   // → pg-replica
	ShardDSNs   []string // optional write shards (§18.3); empty = single primary
	RedisAddr   string
	RedisPass   string
	JWTSecret   []byte
	Port        string
}

func Load() (*Config, error) {
	_ = godotenv.Load() // in dev, read .env; in prod, real env vars already exist (no error if absent)
	return &Config{
		WriteDSN:  mustEnv("DATABASE_WRITE_URL"),
		ReadDSN:   getEnv("DATABASE_READ_URL", mustEnv("DATABASE_WRITE_URL")), // fall back to primary if no replica
		RedisAddr: getEnv("REDIS_ADDR", "redis:6379"),
		RedisPass: readSecretFile(getEnv("REDIS_PASSWORD_FILE", "")),
		JWTSecret: []byte(readSecretFile(mustEnv("JWT_SECRET_FILE"))),
		Port:      getEnv("PORT", "8080"),
	}, nil
}

func mustEnv(k string) string {
	v := os.Getenv(k)
	if v == "" {
		panic("missing required env: " + k)
	}
	return v
}
func getEnv(k, def string) string {
	if v := os.Getenv(k); v != "" {
		return v
	}
	return def
}
// readSecretFile reads a Docker-secret file path (the *_FILE convention); "" → "".
func readSecretFile(path string) string {
	if path == "" {
		return ""
	}
	b, _ := os.ReadFile(path)
	return string(b)
}
```

Config is read **once**, into a typed struct, from the environment — the 12-factor rule. Secrets come from files (`*_FILE`, the Docker-secret convention from §14) so they never sit in env vars visible to `docker inspect`.

### 27.3 The entry point — `cmd/server/main.go` **[I/A]**

```go
// apps/api/cmd/server/main.go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"

	"github.com/scorelive/api/internal/config"
	httpx "github.com/scorelive/api/internal/http"
	"github.com/scorelive/api/internal/redis"
	"github.com/scorelive/api/internal/sse"
	"github.com/scorelive/api/internal/store"
)

func main() {
	log := slog.New(slog.NewJSONHandler(os.Stdout, nil)) // structured JSON logs (§11.1)
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer stop()

	cfg, err := config.Load()
	if err != nil {
		log.Error("config", "err", err)
		os.Exit(1)
	}

	// Data layer: read/write pgx pools (§27.4), Redis (cache + backplane + rate limits).
	st, err := store.Open(ctx, cfg)
	if err != nil {
		log.Error("store open", "err", err)
		os.Exit(1)
	}
	defer st.Close()

	rdb := redis.New(cfg)
	defer rdb.Close()

	// The SSE hub: fans score events out to connected clients; subscribes to Redis so a
	// score entered on ANY instance reaches this instance's clients (§17.3).
	hub := sse.NewHub(log)
	go hub.Run(ctx)
	go hub.SubscribeBackplane(ctx, rdb) // Redis pub/sub → local fan-out

	// Gin router with all REST + SSE routes and middleware (§27.6).
	gin.SetMode(gin.ReleaseMode)
	r := httpx.NewRouter(log, cfg, st, rdb, hub)

	srv := &http.Server{Addr: ":" + cfg.Port, Handler: r, ReadHeaderTimeout: 5 * time.Second}
	go func() {
		log.Info("listening", "port", cfg.Port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Error("serve", "err", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done() // SIGTERM (a deploy) → graceful shutdown so Traefik can drain us cleanly (§20.2)
	log.Info("shutting down")
	shutCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutCtx) // finishes in-flight requests; SSE handlers return on ctx cancel
}
```

`main.go` is pure **wiring**: load config → open the data layer → start the SSE hub and its Redis subscription → build the router → serve → graceful shutdown on `SIGTERM` (so a rolling deploy drains cleanly, §20.2). Everything else lives in `internal/`.

### 27.4 The store — read/write pools and sharding — `internal/store/store.go` **[I/A]**

```go
// apps/api/internal/store/store.go
package store

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/scorelive/api/internal/config"
)

// Store holds a WRITE pool (→ primary) and a READ pool (→ replica). Optionally shards.
type Store struct {
	write  *pgxpool.Pool   // pg-primary: writes + read-your-writes
	read   *pgxpool.Pool   // pg-replica: bulk reads that tolerate slight lag (§18.2)
	shards []*pgxpool.Pool // optional write shards (§18.3); nil = single primary
}

func Open(ctx context.Context, cfg *config.Config) (*Store, error) {
	// pgxpool: bounded connections shared across thousands of goroutines. Tune the pool
	// size so (instances × pool) stays under Postgres max_connections (pgx guide §22).
	w, err := pgxpool.New(ctx, cfg.WriteDSN)
	if err != nil {
		return nil, err
	}
	rd, err := pgxpool.New(ctx, cfg.ReadDSN)
	if err != nil {
		w.Close()
		return nil, err
	}
	s := &Store{write: w, read: rd}
	for _, dsn := range cfg.ShardDSNs { // open a pool per shard, if configured
		p, err := pgxpool.New(ctx, dsn)
		if err != nil {
			return nil, err
		}
		s.shards = append(s.shards, p)
	}
	return s, nil
}

func (s *Store) Close() {
	s.write.Close()
	s.read.Close()
	for _, p := range s.shards {
		p.Close()
	}
}

// writePool picks the shard that OWNS this competition (same key → same shard, always),
// or the single primary when not sharding. This is application-level sharding (§18.3).
func (s *Store) writePool(competitionID int64) *pgxpool.Pool {
	if len(s.shards) == 0 {
		return s.write
	}
	return s.shards[competitionID%int64(len(s.shards))]
}

// SetScore is a WRITE → goes to the owning shard/primary. Read-after-write reads use write too.
func (s *Store) SetScore(ctx context.Context, competitionID, matchID int64, home, away int) error {
	_, err := s.writePool(competitionID).Exec(ctx,
		`UPDATE matches SET home_score=$1, away_score=$2, updated_at=now() WHERE id=$3`,
		home, away, matchID)
	return err
}

// ListLiveMatches is a bulk READ → goes to the REPLICA (slight staleness is fine; SSE
// pushes the fresh number anyway). Parameterized query → injection-proof (§23.3).
func (s *Store) ListLiveMatches(ctx context.Context) ([]Match, error) {
	rows, err := s.read.Query(ctx,
		`SELECT id, competition_id, home, away, home_score, away_score, status
		 FROM matches WHERE status='live' ORDER BY started_at`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []Match
	for rows.Next() {
		var m Match
		if err := rows.Scan(&m.ID, &m.CompetitionID, &m.Home, &m.Away, &m.HomeScore, &m.AwayScore, &m.Status); err != nil {
			return nil, err
		}
		out = append(out, m)
	}
	return out, rows.Err()
}

type Match struct {
	ID, CompetitionID              int64
	Home, Away, Status             string
	HomeScore, AwayScore           int
}
```

This is the heart of the data-scaling design (§18): **writes** go to `writePool(competitionID)` — the owning shard or the single primary; **reads** go to the `read` replica pool. The `%len(shards)` mapping is deterministic so a competition always lands on the same shard. (For richer queries, drop in the Ent client instead of raw pgx — the [Ent guide](GO_ENT_ORM_GUIDE.md) — over the same pools.)

### 27.5 The SSE hub and Redis backplane — `internal/sse/hub.go` **[I/A]**

```go
// apps/api/internal/sse/hub.go
package sse

import (
	"context"
	"encoding/json"
	"log/slog"

	"github.com/redis/go-redis/v9"
)

// Event is one score update pushed to clients.
type Event struct {
	MatchID int64  `json:"matchId"`
	Home    int    `json:"home"`
	Away    int    `json:"away"`
	ID      string `json:"-"` // SSE id for Last-Event-ID resume
}

type client struct{ send chan Event }

// Hub: single-owner goroutine owns the client set (no locks, no races — Go SSE guide §5).
type Hub struct {
	log        *slog.Logger
	register   chan *client
	unregister chan *client
	broadcast  chan Event
	clients    map[*client]struct{}
}

func NewHub(log *slog.Logger) *Hub {
	return &Hub{
		log: log, register: make(chan *client), unregister: make(chan *client),
		broadcast: make(chan Event, 1024), clients: make(map[*client]struct{}),
	}
}

func (h *Hub) Run(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case c := <-h.register:
			h.clients[c] = struct{}{}
		case c := <-h.unregister:
			if _, ok := h.clients[c]; ok {
				delete(h.clients, c)
				close(c.send)
			}
		case ev := <-h.broadcast:
			for c := range h.clients {
				select { // NON-blocking: drop a slow client rather than stall the hub (backpressure)
				case c.send <- ev:
				default:
					delete(h.clients, c)
					close(c.send)
				}
			}
		}
	}
}

// Publish sends a score to Redis so EVERY instance (not just this one) fans it out.
// Call this from the score handler AFTER persisting (persist-then-broadcast, §17.3).
func (h *Hub) Publish(ctx context.Context, rdb *redis.Client, ev Event) error {
	b, _ := json.Marshal(ev)
	return rdb.Publish(ctx, "scores", b).Err()
}

// SubscribeBackplane runs for the process's life: everything on the Redis "scores"
// channel → this instance's local broadcast → its connected clients (§17.3).
func (h *Hub) SubscribeBackplane(ctx context.Context, rdb *redis.Client) {
	sub := rdb.Subscribe(ctx, "scores")
	defer sub.Close()
	for {
		select {
		case <-ctx.Done():
			return
		case msg := <-sub.Channel():
			var ev Event
			if json.Unmarshal([]byte(msg.Payload), &ev) == nil {
				select {
				case h.broadcast <- ev:
				default: // hub buffer full under extreme load — drop; clients resume via Last-Event-ID
				}
			}
		}
	}
}
```

This is the multi-instance fan-out: a score `Publish`ed to Redis on `api-1` is received by *every* instance's `SubscribeBackplane`, each of which broadcasts to its own local SSE clients. One write reaches 500k screens across N instances. The hub's non-blocking send is the backpressure guard (a slow client is dropped, not allowed to stall everyone) — the exact pattern from the [Go SSE guide](GO_SSE_GUIDE.md) §5, here wired to a Redis backplane for horizontal scale.

### 27.6 Handlers, auth and the router — `internal/http/` **[I/A]**

```go
// apps/api/internal/http/router.go
package http

import (
	"log/slog"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"

	"github.com/scorelive/api/internal/config"
	"github.com/scorelive/api/internal/sse"
	"github.com/scorelive/api/internal/store"
)

func NewRouter(log *slog.Logger, cfg *config.Config, st *store.Store, rdb *redis.Client, hub *sse.Hub) *gin.Engine {
	r := gin.New()
	r.Use(gin.Recovery())
	// Behind Cloudflare+Traefik: trust the proxy so c.ClientIP() returns the REAL client (§15.5).
	_ = r.SetTrustedProxies([]string{"0.0.0.0/0"}) // in prod: Traefik's docker network only

	h := &Handler{cfg: cfg, st: st, rdb: rdb, hub: hub, log: log}

	r.GET("/healthz", h.Health) // Traefik + Docker health checks (§7.2)

	// Public REST
	pub := r.Group("/")
	pub.POST("/auth/login", h.Login)         // Argon2id verify → JWT (JWT/Argon2 guide)
	pub.GET("/matches/live", h.ListLive)      // reads from the replica (cached in Redis)

	// The SSE live stream (its own host sse.scorelive.com, routed by Traefik).
	r.GET("/stream", h.RequireAuth(), h.Stream)

	// Admin — protected by JWT owner-role HERE, and by Traefik ipAllowList at the EDGE (§21).
	admin := r.Group("/admin", h.RequireAuth(), h.RequireOwner())
	admin.POST("/matches/:id/score", h.SetScore)

	return r
}
```

```go
// apps/api/internal/http/handlers.go  (the key handlers; auth internals per the JWT/Argon2 guide)
package http

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"

	"github.com/scorelive/api/internal/sse"
)

func (h *Handler) Health(c *gin.Context) {
	// Cheap check the app can reach its deps; returns 200 only if healthy (Traefik routes on this).
	if err := h.st.Ping(c.Request.Context()); err != nil {
		c.JSON(http.StatusServiceUnavailable, gin.H{"status": "degraded"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"status": "ok"})
}

// SetScore: owner-only (RequireOwner + edge IP allow-list). Persist THEN broadcast (§17.3).
func (h *Handler) SetScore(c *gin.Context) {
	matchID, _ := strconv.ParseInt(c.Param("id"), 10, 64)
	var body struct {
		CompetitionID int64 `json:"competitionId"`
		Home, Away    int   `json:"home"`
	}
	if err := c.ShouldBindJSON(&body); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid body"})
		return
	}
	// 1) PERSIST to the owning shard/primary (durable; reconnecting clients replay from here).
	if err := h.st.SetScore(c.Request.Context(), body.CompetitionID, matchID, body.Home, body.Away); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "write failed"})
		return
	}
	// 2) Invalidate/refresh the Redis cache for this match (so ListLive reads the new value).
	h.cacheScore(c.Request.Context(), matchID, body.Home, body.Away)
	// 3) BROADCAST via Redis → every instance → all subscribed fans, instantly.
	_ = h.hub.Publish(c.Request.Context(), h.rdb, sse.Event{MatchID: matchID, Home: body.Home, Away: body.Away})
	c.JSON(http.StatusOK, gin.H{"ok": true})
}

// Stream: the SSE endpoint. Registers with the hub and forwards events until the client leaves.
func (h *Handler) Stream(c *gin.Context) {
	c.Writer.Header().Set("Content-Type", "text/event-stream")
	c.Writer.Header().Set("Cache-Control", "no-cache")
	c.Writer.Header().Set("X-Accel-Buffering", "no") // don't let any proxy buffer the stream (§15.6)
	cl := h.hub.Add()                                 // register; returns the client's channel
	defer h.hub.Remove(cl)

	c.Stream(func(w io.Writer) bool {
		select {
		case <-c.Request.Context().Done():
			return false // client disconnected or server shutting down
		case ev, ok := <-cl.Send():
			if !ok {
				return false
			}
			c.SSEvent("score", ev) // frame: event: score\n data: {...}\n\n
			return true
		}
	})
}
```

The score flow is the whole SaaS in one handler: **persist → cache → broadcast**. Owner-only is enforced twice (JWT role here + Traefik IP allow-list at the edge, §21 — defense in depth). The `Stream` handler is the [Go SSE guide](GO_SSE_GUIDE.md)'s pattern; `Login`/`RequireAuth`/`RequireOwner` (Argon2id verify, JWT issue/validate, role check) are exactly the [JWT/Argon2 guide](GO_JWT_ARGON2_GUIDE.md)'s code — import that guide's `password` and `token` packages rather than re-deriving them.

### 27.7 Ent schema and goose migration **[I/A]**

```go
// apps/api/ent/schema/match.go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

type Match struct{ ent.Schema }

func (Match) Fields() []ent.Field {
	return []ent.Field{
		field.Int64("competition_id"),          // the SHARD KEY (§18.3) — keep related data together
		field.String("home"),
		field.String("away"),
		field.Int("home_score").Default(0),
		field.Int("away_score").Default(0),
		field.Enum("status").Values("scheduled", "live", "finished").Default("scheduled"),
		field.Time("started_at").Optional(),
		field.Time("updated_at").Default(nil),
	}
}
```

```sql
-- apps/api/migrations/0001_matches.sql  (goose — run as a gated deploy step, never on boot)
-- +goose Up
CREATE TABLE matches (
	id             BIGSERIAL PRIMARY KEY,
	competition_id BIGINT      NOT NULL,
	home           TEXT        NOT NULL,
	away           TEXT        NOT NULL,
	home_score     INT         NOT NULL DEFAULT 0,
	away_score     INT         NOT NULL DEFAULT 0,
	status         TEXT        NOT NULL DEFAULT 'scheduled',
	started_at     TIMESTAMPTZ,
	updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Index the hot query (live matches) and the shard key.
CREATE INDEX matches_status_idx ON matches (status) WHERE status = 'live';
CREATE INDEX matches_competition_idx ON matches (competition_id);

-- +goose Down
DROP TABLE matches;
```

goose owns the schema (versioned, reviewed, run as a deliberate step — the [goose guide](GO_GOOSE_MIGRATIONS_GUIDE.md)); Ent generates the type-safe query layer over it (the [Ent guide](GO_ENT_ORM_GUIDE.md)). The `competition_id` is the shard key, and it's indexed. Run `docker compose run --rm api /app migrate up` at deploy (§26.3) — the app never auto-migrates on boot.

### 27.8 The Dockerfiles **[I/A]**

```dockerfile
# apps/api/Dockerfile — multi-stage → tiny, non-root image (§10.4)
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

```dockerfile
# apps/web/Dockerfile — Next.js standalone output → small runtime image
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # produces .next/standalone with next.config output:"standalone"

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Both use multi-stage builds and run as **non-root** — a tiny attack surface, tiny images (fast pulls during a scale-up). The API's `distroless` base has no shell at all.

### 27.9 Traefik static + dynamic config **[I/A]**

```yaml
# infra/traefik/traefik.yml — STATIC config (the engine)
global: { checkNewVersion: false, sendAnonymousUsage: false }

entryPoints:
  web:
    address: ":80"
    http: { redirections: { entryPoint: { to: websecure, scheme: https } } }
  websecure:
    address: ":443"
    http3: {}
    forwardedHeaders:
      trustedIPs: ["173.245.48.0/20", "103.21.244.0/22"]   # Cloudflare ranges (full list) — §15.5
  metrics:
    address: ":8082"                                        # Prometheus scrape (internal only)

providers:
  docker:
    endpoint: "tcp://dockerproxy:2375"                      # via the read-only socket proxy (§23.2)
    exposedByDefault: false
    network: edge
  file:
    directory: /etc/traefik/dynamic
    watch: true

certificatesResolvers:
  le-dns:
    acme:
      email: ops@scorelive.com
      storage: /letsencrypt/acme.json
      dnsChallenge: { provider: cloudflare, resolvers: ["1.1.1.1:53"] }  # wildcard certs (§9.3)

api: { dashboard: true }
log: { level: INFO }
accessLog:
  format: json
  filters: { statusCodes: ["400-599"] }
  fields: { headers: { names: { Authorization: drop } } }   # never log tokens (§11.1)
metrics: { prometheus: { entryPoint: metrics, addRoutersLabels: true } }
```

```yaml
# infra/traefik/dynamic/security.yml — shared DYNAMIC config (hot-reloaded)
http:
  middlewares:
    security-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true
        frameDeny: true
        referrerPolicy: strict-origin-when-cross-origin
        customResponseHeaders: { Server: "" }
    dash-auth:
      basicAuth:
        users: ["admin:$2y$05$replace-with-htpasswd-bcrypt-hash"]
tls:
  options:
    modern: { minVersion: VersionTLS12, sniStrict: true }
```

The static file is the engine (entrypoints, providers, ACME, metrics, the Cloudflare `trustedIPs` that make `ipStrategy` work); the dynamic file holds shared middlewares and TLS options, hot-reloaded on save. Reference them from container labels as `security-headers@file`, `modern@file` (§4.3).



### 27.10 The Postgres primary and read replica **[I/A]**

The primary needs a replication user and settings; the replica clones the primary once (`pg_basebackup`) then streams changes. An init script wires this up:

```bash
# infra/postgres/setup-replica.sh — runs ONCE in the replica container to clone the primary.
#!/usr/bin/env bash
set -euo pipefail
# Wait for the primary, then base-backup into the replica's data dir and start streaming.
until pg_isready -h pg-primary -U replicator; do sleep 2; done
if [ ! -s "/var/lib/postgresql/data/PG_VERSION" ]; then     # only clone if empty
	PGPASSWORD="$(cat /run/secrets/pg_pw)" pg_basebackup \
		-h pg-primary -U replicator -D /var/lib/postgresql/data -Fp -Xs -P -R
	# -R writes standby.signal + primary_conninfo so this instance boots as a streaming replica.
fi
exec postgres -c hot_standby=on
```

```sql
-- infra/postgres/init-primary.sql — create the replication role on the primary (runs once).
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'from-secret';
-- pg_hba: allow the replica to connect for replication (on the private backend network only):
--   host replication replicator  <backend-subnet>  scram-sha-256
```

The primary's `wal_level=replica` + `max_wal_senders` (set via `command:` in Compose, §27.12) let it feed replicas; the replica's `pg_basebackup ... -R` makes it a streaming standby that stays in sync. Reads then go to the replica, writes to the primary (§27.4). For production HA (automatic failover), a tool like Patroni promotes the replica if the primary dies — the [Database Server Admin guide](DATABASE_SERVER_ADMIN_GUIDE.md) covers that; here we have the read-scaling replica, which is the immediate win.

### 27.11 The Next.js frontend **[I/A]**

The API client holds the access token **in memory** (XSS-safe) and consumes SSE. Two small but complete files:

```ts
// apps/web/lib/api.ts — the typed API client (in-memory token + credentialed fetch)
let accessToken: string | null = null;   // in MEMORY, never localStorage (XSS-safe, §22.3)

export async function login(email: string, password: string) {
	const res = await fetch("https://api.scorelive.com/auth/login", {
		method: "POST",
		headers: { "Content-Type": "application/json" },
		credentials: "include",             // receive the HttpOnly refresh cookie
		body: JSON.stringify({ email, password }),
	});
	if (!res.ok) throw new Error("login failed");
	accessToken = (await res.json()).accessToken;
}

export async function apiGet<T>(path: string): Promise<T> {
	const res = await fetch(`https://api.scorelive.com${path}`, {
		headers: accessToken ? { Authorization: `Bearer ${accessToken}` } : {},
		credentials: "include",
	});
	// on 401: silent-refresh via the cookie, then retry (omitted here for brevity)
	return res.json();
}
```

```tsx
// apps/web/app/(public)/live/page.tsx — the live-scores page, consuming SSE
"use client";
import { useEffect, useState } from "react";

type Score = { matchId: number; home: number; away: number };

export default function LivePage() {
	const [scores, setScores] = useState<Record<number, Score>>({});

	useEffect(() => {
		// EventSource to the SSE host; withCredentials sends the auth cookie (§22.3).
		const es = new EventSource("https://sse.scorelive.com/stream", { withCredentials: true });
		// Named "score" events (matches c.SSEvent("score", ...) on the backend, §27.6).
		es.addEventListener("score", (e) => {
			const s: Score = JSON.parse((e as MessageEvent).data);
			setScores((prev) => ({ ...prev, [s.matchId]: s }));   // update ONLY the changed match
		});
		es.onerror = () => {/* browser auto-reconnects; server replays via Last-Event-ID */};
		return () => es.close();   // CLEANUP on unmount — or you leak a stream (§22.4)
	}, []);

	return (
		<ul>
			{Object.values(scores).map((s) => (
				<li key={s.matchId}>{s.home} – {s.away}</li>
			))}
		</ul>
	);
}
```

The client never trusts itself: the token lives in memory, the backend authorizes every call, and the SSE stream updates *only the changed match* (no full re-render, no polling — §22.4). The **admin build** is the same codebase with the `(admin)` route group and `NEXT_PUBLIC_MODE=admin`, shipped as a separate image (`web-admin`) that only runs behind Traefik's IP allow-list (§21) — so the admin JavaScript never reaches a public browser.

### 27.12 The whole stack in one Compose file **[I/A]**

Here is the production stack assembled — Traefik + the socket proxy + the API fleet + both frontends + Postgres primary/replica + Redis, with the two-network isolation. Read the comments; every choice traces to a section above.

```yaml
# infra/compose.yaml  — the ScoreLive production stack
services:
  # ── Edge: Traefik + a read-only Docker socket proxy (§23.2) ──
  dockerproxy:
    image: tecnativa/docker-socket-proxy:latest
    environment: { CONTAINERS: 1, SERVICES: 1, TASKS: 1, NETWORKS: 1, POST: 0 }
    volumes: ["/var/run/docker.sock:/var/run/docker.sock:ro"]
    networks: [socket]
    restart: unless-stopped

  traefik:
    image: traefik:v3.3
    command: ["--configFile=/etc/traefik/traefik.yml"]
    ports: ["80:80", "443:443", "443:443/udp"]     # ONLY Traefik exposes host ports
    environment: { CF_DNS_API_TOKEN_FILE: /run/secrets/cf_token }
    volumes:
      - "./traefik/traefik.yml:/etc/traefik/traefik.yml:ro"
      - "./traefik/dynamic:/etc/traefik/dynamic:ro"
      - "letsencrypt:/letsencrypt"                 # persist acme.json (§9.1)
    secrets: [cf_token]
    networks: [edge, socket]
    labels:                                        # the internal dashboard (§10.2)
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.internal.scorelive.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=le-dns"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=admin-ipallow,dash-auth"
    restart: unless-stopped

  # ── The Go API fleet (§20) — REST + SSE, N instances, on edge + backend ──
  api:
    image: ghcr.io/scorelive/api:${APP_VERSION:-latest}
    deploy: { replicas: 4, resources: { limits: { cpus: "1.5", memory: 768M } } }
    environment:
      DATABASE_WRITE_URL: postgres://app@pg-primary:5432/scorelive?sslmode=disable
      DATABASE_READ_URL:  postgres://app@pg-replica:5432/scorelive?sslmode=disable
      REDIS_ADDR: redis:6379
      PGPASSWORD_FILE: /run/secrets/pg_pw
      JWT_SECRET_FILE: /run/secrets/jwt
    secrets: [pg_pw, jwt]
    networks: [edge, backend]
    depends_on:
      pg-primary: { condition: service_healthy }
      redis:      { condition: service_healthy }
    healthcheck: { test: ["CMD", "/app", "healthcheck"], interval: 10s, retries: 3 }
    labels:
      - "traefik.enable=true"
      # REST router
      - "traefik.http.routers.api.rule=Host(`api.scorelive.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=le-dns"
      - "traefik.http.routers.api.middlewares=security-headers@file,api-ratelimit"
      # admin API router (higher priority, IP-gated) — §21.2
      - "traefik.http.routers.apiadmin.rule=Host(`api.scorelive.com`) && PathPrefix(`/admin`)"
      - "traefik.http.routers.apiadmin.priority=100"
      - "traefik.http.routers.apiadmin.entrypoints=websecure"
      - "traefik.http.routers.apiadmin.tls.certresolver=le-dns"
      - "traefik.http.routers.apiadmin.middlewares=admin-ipallow,security-headers@file"
      # SSE router (own middlewares) — §17.2
      - "traefik.http.routers.sse.rule=Host(`sse.scorelive.com`)"
      - "traefik.http.routers.sse.entrypoints=websecure"
      - "traefik.http.routers.sse.tls.certresolver=le-dns"
      - "traefik.http.routers.sse.middlewares=security-headers@file,sse-ratelimit,compress"
      # the shared service + health check + rate-limit middleware definitions
      - "traefik.http.services.api.loadbalancer.server.port=8080"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/healthz"
      - "traefik.http.middlewares.api-ratelimit.ratelimit.average=100"
      - "traefik.http.middlewares.api-ratelimit.ratelimit.burst=50"
      - "traefik.http.middlewares.sse-ratelimit.ratelimit.average=20"
      - "traefik.http.middlewares.admin-ipallow.ipallowlist.sourcerange=203.0.113.5/32"
      - "traefik.http.middlewares.admin-ipallow.ipallowlist.ipstrategy.depth=1"
      - "traefik.http.middlewares.compress.compress=true"
    restart: unless-stopped

  # ── Frontends (§22) — public + admin builds ──
  web-public:
    image: ghcr.io/scorelive/web:${APP_VERSION:-latest}
    environment: { NEXT_PUBLIC_MODE: public }
    networks: [edge]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.scorelive.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=le-dns"
      - "traefik.http.routers.app.middlewares=security-headers@file"
      - "traefik.http.services.app.loadbalancer.server.port=3000"
    restart: unless-stopped

  web-admin:
    image: ghcr.io/scorelive/web-admin:${APP_VERSION:-latest}
    networks: [edge]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.admin.rule=Host(`admin.scorelive.com`)"
      - "traefik.http.routers.admin.entrypoints=websecure"
      - "traefik.http.routers.admin.tls.certresolver=le-dns"
      - "traefik.http.routers.admin.middlewares=admin-ipallow,security-headers@file"  # IP-gated!
      - "traefik.http.services.admin.loadbalancer.server.port=3000"
    restart: unless-stopped

  # ── Data tier (§18, §19) — backend network ONLY, no host ports, unreachable from edge ──
  pg-primary:
    image: postgres:17
    environment: { POSTGRES_DB: scorelive, POSTGRES_USER: app, POSTGRES_PASSWORD_FILE: /run/secrets/pg_pw }
    command: ["postgres", "-c", "wal_level=replica", "-c", "max_wal_senders=10"]
    volumes: [pg_primary:/var/lib/postgresql/data]
    secrets: [pg_pw]
    networks: [backend]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U app -d scorelive"], interval: 10s, retries: 5 }
    restart: unless-stopped

  pg-replica:
    image: postgres:17
    command: ["postgres", "-c", "hot_standby=on"]
    volumes: [pg_replica:/var/lib/postgresql/data]
    secrets: [pg_pw]
    networks: [backend]
    depends_on: [pg-primary]
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes: ["./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro", redisdata:/data]
    networks: [backend]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 10s, retries: 5 }
    restart: unless-stopped

volumes: { letsencrypt: {}, pg_primary: {}, pg_replica: {}, redisdata: {} }

networks:
  edge:    { driver: bridge }   # Traefik + public services
  backend: { driver: bridge }   # API + data stores (Traefik NOT here)
  socket:  { driver: bridge }   # Traefik ↔ docker-socket-proxy only

secrets:
  cf_token: { file: ./secrets/cf_token.txt }
  pg_pw:    { file: ./secrets/pg_pw.txt }
  jwt:      { file: ./secrets/jwt.txt }
```

### 27.13 The full file tree **[I/A]**

```text
scorelive/
├── apps/
│   ├── api/  {cmd/, internal/{config,auth,http,sse,store,redis}/, ent/, migrations/, Dockerfile, .air.toml}
│   └── web/  {app/(public)/, app/(admin)/, lib/api.ts, Dockerfile, next.config.ts}
├── infra/
│   ├── traefik/
│   │   ├── traefik.yml               # §4.2  static: entrypoints, providers, ACME, metrics, accesslog
│   │   └── dynamic/security.yml      # §4.3, §8.4  shared middlewares + TLS options
│   ├── postgres/                     # §18  primary/replica/shard configs + replication init
│   ├── redis/redis.conf              # §19
│   ├── compose.yaml                  # §27.1  the whole stack
│   ├── compose.prod.yaml             # prod overrides (replicas, resource limits)
│   ├── deploy.sh                     # §26.3  rolling deploy
│   └── secrets/                      # GITIGNORED: cf_token.txt, pg_pw.txt, jwt.txt (chmod 600)
├── .github/workflows/deploy.yml      # §26  CI/CD
└── .gitignore                        # secrets/, .env, *.dump
```

Everything except the gitignored `secrets/` is in Git — the whole system is reproducible from the repo (rebuild-from-scratch is `git clone` + restore secrets + restore DB + `docker compose up -d` + point DNS).

---

## 28. Operations for the SaaS

### 28.1 The daily and incident commands **[A]**

```bash
docker compose ps                          # is everything up? healthy?
docker compose logs -f traefik             # the edge — routing/cert/health events
docker compose logs -f api | grep -i error # app errors across instances
docker compose up -d --scale api=8         # scale the fleet up (e.g. before a big match)
curl -s localhost:8082/metrics | grep traefik_service_requests_total   # quick metric peek
docker stats                               # live CPU/mem per container
```

The incident loop mirrors the VPS guide: **is a resource exhausted?** (`df -h`, `free -h`, FDs), **is a service unhealthy?** (Traefik dashboard shows which servers are down), **what do the logs say?** Traefik's dashboard and access log localize edge problems fast (which router, which service, which status codes); the metrics show whether you're keeping up under load.

### 28.2 What to monitor and alert on **[A]**

Wire Prometheus + Grafana (§11.2) and alert on: **5xx rate** (are we failing?), **p99 latency** (are we slow?), **healthy-server count per service** (did instances die?), **request rate** (are we spiking?), **Postgres connections vs max** and **slow-query count**, **Redis memory + eviction rate**, **SSE open-connection count** and host **file descriptors**, **disk >80%**, and **certificate expiry** (Traefik auto-renews, but alert if it ever fails). An **external uptime monitor** hitting `app.scorelive.com` and `api.scorelive.com/healthz` catches a total-origin-down that on-box monitoring can't report. Route alerts somewhere you'll see them; tune so every page is real.

### 28.3 Backups and DR **[A]**

Nightly `pg_dump` of each shard/primary shipped off-box (object storage), the `letsencrypt` and `redisdata` volumes captured, provider snapshots as a coarse net, and — the rule that matters — a **quarterly test-restore** proving the backup works. Because the whole stack is in Git, disaster recovery is a runbook: fresh VPS → harden → restore secrets + latest DB dumps → `docker compose up -d` → repoint DNS. (The [VPS guide](PRODUCTION_VPS_GUIDE.md) §21 has the full backup scripts and DR drill.)

---

## 29. Gotchas and Best Practices

The mistakes that actually break Traefik deployments, distilled.

| Pitfall | Symptom | Fix |
|---|---|---|
| **Container not on Traefik's network** | perfect labels, still 404 | Put the service on `providers.docker.network` (§5.2). The #1 cause. |
| **`exposedByDefault` left true** | a new container is accidentally public | Set `exposedbydefault=false`; opt in with `traefik.enable=true` (§3.1). |
| **File-provider middleware without `@file`** | "middleware not found" | Qualify cross-provider refs: `security-headers@file` (§4.3). |
| **`$` not doubled in Compose labels** | broken bcrypt hash / regexp | Write `$$` for a literal `$` in labels (§8.3). |
| **v2 rule syntax on v3** | router won't load | v3: one value per matcher, use `or` to combine; `HostRegexp` takes a real regexp (§6.1). |
| **`ipAllowList` behind Cloudflare without `ipstrategy.depth`** | admin allow-list matches Cloudflare (everyone!) | Set `ipstrategy.depth=1` + trust CF's IPs (§15.5, §21.3). **Security-critical.** |
| **`api.insecure=true`** | dashboard exposed unauthenticated on :8080 | Never in prod; secure the dashboard via a router + IP-allow + auth (§10.2). |
| **`acme.json` not persisted / not `chmod 600`** | certs re-issued → Let's Encrypt rate limit; or Traefik rejects the file | Persist on a volume; Traefik enforces 600 (§9.1). |
| **Raw `docker.sock:ro` on Traefik** | proxy breach → host compromise | Use a read-only socket proxy (§23.2). |
| **DB reachable from edge** | database exposed | Two networks; DBs on `backend` only; no host ports (§16.2). |
| **Reading replica right after writing** | user doesn't see their own write | Route read-after-write to the primary; replicas lag (§18.2). |
| **SSE buffered by Cloudflare/Traefik** | live stream stalls | Don't buffer the SSE path; heartbeats; tune timeouts (§15.6). |
| **Sticky sessions by default** | uneven load, messy deploys | Stay stateless + shared Redis; avoid stickiness (§7.3). |
| **No health checks** | traffic routed to dead/starting instances | Health checks gate LB routing + deploys (§7.2, §20.2). |
| **Untested at scale** | discovers the bottleneck during the real spike | Load-test to target before the event (§25.4). |

**Best-practice summary:** `exposedByDefault=false` and opt in per service; keep every service on the right network; secure the dashboard and use a socket proxy; TLS + security headers + rate limits on every router; **`ipstrategy.depth` whenever behind Cloudflare** (the hidden-admin linchpin); persist `acme.json`; isolate the data tier on its own network with no host ports; keep the app stateless with shared Redis so Traefik can scale it freely; health-check everything; deploy by rolling instances; and **load-test to 500k before the match, not during it.**

---

## 30. Study Path and Build-to-Learn Projects

### 30.1 A staged path **[B→A]**

1. **Route one container (§1–§3).** Run Traefik + `whoami`, label it, `curl` it, then `--scale whoami=3` and watch the responding IP rotate. Feel the auto-discovery.
2. **Learn the four stages (§4–§8).** Add a static `traefik.yml`, a file-provider middleware, a second service on a second host, and a rate-limit + security-headers chain. Break a label on purpose and read the dashboard/logs to diagnose it.
3. **Get real HTTPS (§9).** Point a cheap domain at a VPS, add the ACME resolver, watch a real certificate appear. Then get a **wildcard** via DNS-01 + Cloudflare.
4. **Stand up the data tier (§18–§19).** Postgres primary + replica + Redis on a private network; prove the API reads from the replica and writes to the primary, and that Redis fan-out reaches clients on different instances.
5. **Build the SaaS (§13–§22).** Wire the Go API (REST + SSE), both Next.js dashboards, and the subdomain routing. Then **hide the admin plane** and test from a different IP that it truly 403s.
6. **Harden and lock the origin (§14, §15, §23).** VPS hardening, Cloudflare firewall + WAF, origin-only-from-Cloudflare, the socket proxy, `ipstrategy.depth`.
7. **Automate and scale (§24–§26).** GitHub Actions rolling deploy; then **load-test** to your target and fix whatever breaks first (usually DB connections) until it holds with headroom.

### 30.2 Build-to-learn projects **[A]**

- **The full ScoreLive.** Build it end to end and run a **synthetic spike**: script 50k SSE connections opening in two minutes (k6) while you watch Traefik's dashboard, p99 latency, and Postgres connections. Fix the first bottleneck; repeat. This is the capstone.
- **Prove the hidden admin plane.** From your allow-listed IP, log into admin; from a phone on cellular, confirm `admin.scorelive.com` returns 403 and the admin API rejects you — *before* any password. Then break `ipstrategy.depth` on purpose and watch the plane leak, so you internalize why that line matters.
- **Zero-downtime deploy proof.** `watch -n0.5 curl -s https://api.scorelive.com/version` while GitHub Actions rolls a new build — no failed request.
- **Chaos + scale drill.** `docker kill` one API instance mid-traffic and confirm Traefik routes around it; then `--scale api=12` before a "match" and watch new instances join the load balancer live.

### 30.3 Where to go next **[A]**

Deepen each layer with its guide: [Docker](DOCKER_GUIDE.md) and [Production VPS Management](PRODUCTION_VPS_GUIDE.md) (the host + Caddy alternative + full backup/DR), [PostgreSQL](POSTGRESQL_GUIDE.md) + [Database Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md) (replication, sharding, PITR), [Redis](REDIS_GUIDE.md) (clustering), the Go stack ([Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md), [pgx](GO_PGX_GUIDE.md), [SSE](GO_SSE_GUIDE.md), [JWT/Argon2](GO_JWT_ARGON2_GUIDE.md)), [Next.js 16](NEXTJS_16_GUIDE.md), [Node WebSockets](NODE_WEBSOCKETS_GUIDE.md) (the C10M connection-scaling math), and [CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md). You now have the complete picture: Traefik from its first routed container to fronting a hardened, Cloudflare-shielded, auto-deploying, horizontally-scaled SaaS that streams live scores to half a million fans — with an admin plane the public can't even find.
