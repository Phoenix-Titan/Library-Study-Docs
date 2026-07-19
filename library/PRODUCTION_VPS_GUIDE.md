# Production VPS Management — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone who has *never* logged into a server and wants to reach the point where they can take a bare Linux VPS and turn it into a **hardened, production-grade host** running a real backend — two Go application instances behind a load-balancing reverse proxy, a PostgreSQL database, and a Redis cache, all in Docker, reachable at `api.yourcompany.com` over automatic HTTPS, defended by a firewall and intrusion prevention, backed up, monitored, and able to serve **hundreds of thousands of users**. This guide assumes zero prior sysadmin knowledge and builds to the level a real infrastructure/DevOps engineer operates at. It is deliberately **explain-first**: every command is introduced with *what* it does, *why* you run it, *where* it runs, *when* to reach for it, and the *gotcha* that bites people — and only then shown, heavily commented. A production server holds other people's data and money; a command you ran but did not understand is an outage (or a breach) waiting to happen. So the "why" matters as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Ubuntu Server 24.04 LTS ("Noble")** — the most common production Linux — with notes for Debian 12/13 where they differ (RHEL/Rocky/Alma differ mainly in the package manager, `dnf` vs `apt`, and the firewall, `firewalld` vs `ufw`; the concepts are identical). The stack is **Docker Engine 28.x** with the **Compose v2** plugin (`docker compose`, not the old `docker-compose`), **Caddy v2.8+** as the reverse proxy (automatic HTTPS via Let's Encrypt/ZeroSSL), **PostgreSQL 17** and **Redis 7.x/8.x** as official images, and **Go 1.25/1.26** for the application. Security tooling: **OpenSSH**, **ufw** (a friendly front-end to `nftables`), **fail2ban**, and unattended-upgrades. Everything is **offline-first** and current as of **2026**; fast-moving details are flagged **⚡**. The author is on **Windows 11**, so client-side commands (SSH, `scp`) are shown for both PowerShell and POSIX where they differ — on the server itself, everything is Linux.
>
> **This guide's place in the library:** This is the *integration* guide that ties the operations stack together. It leans on several siblings you'll want open: [Linux Server Admin](LINUX_SERVER_ADMIN_GUIDE.md) is the deep reference for systemd, users, and networking that §9 summarizes; [Docker](DOCKER_GUIDE.md) covers images/Compose in more depth than §10–§11; [Nginx](NGINX_GUIDE.md) is the *alternative* reverse proxy (this guide uses **Caddy** and §15.10 compares them); [PostgreSQL](POSTGRESQL_GUIDE.md) and [Redis](REDIS_GUIDE.md) are the data stores we run; [Networking](NETWORKING_GUIDE.md) explains the DNS/TLS/HTTP concepts §16–§17 apply; [CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md) automates the deploy in §19; and the Go application itself is built in the [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md), [pgx](GO_PGX_GUIDE.md), and [JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) guides. This guide assumes you can build the app; it teaches you how to **run it in production**.

---

## Table of Contents

1. [What a VPS Is and How Production Hosting Works](#1-what-a-vps-is-and-how-production-hosting-works) **[B]**
2. [Provisioning Your VPS](#2-provisioning-your-vps) **[B]**
3. [First Contact — SSH and the Initial Login](#3-first-contact--ssh-and-the-initial-login) **[B]**
4. [Users, sudo and Least Privilege](#4-users-sudo-and-least-privilege) **[B]**
5. [Hardening SSH](#5-hardening-ssh) **[B/I]**
6. [The Firewall with ufw and nftables](#6-the-firewall-with-ufw-and-nftables) **[B/I]**
7. [Fail2ban and Intrusion Prevention](#7-fail2ban-and-intrusion-prevention) **[I]**
8. [Automatic Security Updates](#8-automatic-security-updates) **[I]**
9. [Linux Command Mastery for Operations](#9-linux-command-mastery-for-operations) **[B]**
10. [Docker From Zero](#10-docker-from-zero) **[B/I]**
11. [Docker Compose for the Stack](#11-docker-compose-for-the-stack) **[I]**
12. [The Production Backend — Two Go Instances, Postgres and Redis](#12-the-production-backend--two-go-instances-postgres-and-redis) **[I/A]**
13. [PostgreSQL in Production](#13-postgresql-in-production) **[I/A]**
14. [Redis in Production](#14-redis-in-production) **[I/A]**
15. [Caddy — The Reverse Proxy From Zero to Hero](#15-caddy--the-reverse-proxy-from-zero-to-hero) **[B→A]**
16. [DNS, Domains and Subdomains](#16-dns-domains-and-subdomains) **[B/I]**
17. [TLS and Automatic HTTPS](#17-tls-and-automatic-https) **[I]**
18. [Secrets and Configuration Management](#18-secrets-and-configuration-management) **[I/A]**
19. [Zero-Downtime Deploys and CI/CD](#19-zero-downtime-deploys-and-cicd) **[A]**
20. [Observability — Logs, Metrics and Health](#20-observability--logs-metrics-and-health) **[A]**
21. [Backups and Disaster Recovery](#21-backups-and-disaster-recovery) **[A]**
22. [Scaling to 100k Users](#22-scaling-to-100k-users) **[A]**
23. [The Production Security Checklist](#23-the-production-security-checklist) **[A]**
24. [The Complete Project Layout](#24-the-complete-project-layout) **[I/A]**
25. [Gotchas and Best Practices](#25-gotchas-and-best-practices) **[A]**
26. [Study Path and Build-to-Learn Projects](#26-study-path-and-build-to-learn-projects)

---

## 1. What a VPS Is and How Production Hosting Works

### 1.1 What "a server" actually is **[B]**

A **server** is just a computer that runs all the time and answers requests over the network. There is nothing magic about it — it has a CPU, RAM, a disk, and a network connection, exactly like your laptop. The differences are that it has no monitor or keyboard (you reach it *over the network*, by typing commands), it runs a server operating system (Linux, almost always), and it is expected to stay up 24/7 with a stable public **IP address** so clients can always find it.

A **VPS** — Virtual Private Server — is a *slice* of a bigger physical machine. A hosting company (DigitalOcean, Hetzner, Linode/Akamai, Vultr, AWS Lightsail, and the big clouds AWS/GCP/Azure) runs powerful physical servers and uses **virtualization** to divide each one into many isolated virtual machines, renting each to a different customer. Your VPS *feels* like a whole computer — its own OS, its own root access, its own IP — but it shares the underlying hardware with other tenants. This is why it's cheap (you pay for a slice, not a whole machine) and instant (the provider clones a virtual machine in seconds). For the vast majority of production backends — including one serving hundreds of thousands of users — a well-configured VPS (or a few of them) is entirely sufficient; you do not need a Kubernetes cluster to serve 100k users, and pretending you do is how startups burn their runway on complexity.

### 1.2 The shape of a production deployment **[B]**

Here is the whole system this guide builds, end to end, so you have the map before we start laying bricks:

```text
                          the internet
                               │
                    (DNS: api.company.com → 203.0.113.10)
                               │
                        ┌──────▼───────┐
                        │   FIREWALL   │   only ports 22, 80, 443 open
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │    CADDY     │   reverse proxy on the host:
                        │  :80 :443    │   terminates HTTPS, load-balances
                        └──────┬───────┘
                   ┌───────────┴───────────┐
                   │  Docker bridge network │   private, not exposed
              ┌────▼────┐             ┌─────▼────┐
              │  go-1   │             │   go-2   │   two identical Go app
              │ :8080   │             │  :8080   │   instances (redundancy)
              └────┬────┘             └────┬─────┘
                   └───────────┬───────────┘
                    ┌──────────┴──────────┐
              ┌─────▼─────┐         ┌──────▼──────┐
              │ postgres  │         │    redis    │   data stores, private,
              │  :5432    │         │   :6379     │   reachable ONLY by the
              └───────────┘         └─────────────┘   apps over the Docker net
```

Read the flow: a user's browser asks DNS "where is `api.company.com`?", gets your VPS's IP, and connects to port 443. The **firewall** allows that. **Caddy** — the only thing listening on the public ports — terminates TLS (handles the HTTPS certificate) and forwards the request to one of your two **Go instances** over a *private* Docker network. The Go app talks to **Postgres** and **Redis**, which are *not* exposed to the internet at all — they're reachable only from inside the Docker network. Everything except Caddy is invisible from outside. That is the core security idea of the whole design: **one narrow, hardened front door, everything else private.**

### 1.3 Why two Go instances **[B/I]**

Running *two* copies of your Go application, rather than one, buys you two things that matter enormously in production:

- **Redundancy.** If one instance crashes (a bug, an out-of-memory kill, a bad deploy), the other keeps serving. Caddy notices the dead one and routes all traffic to the survivor — users see nothing. With a single instance, any crash is a full outage.
- **Zero-downtime deploys.** To ship new code, you update one instance at a time: take `go-1` out of rotation, replace it, put it back, then do `go-2`. At every moment at least one instance is serving the old-or-new code, so there is no downtime window (§19).

Two is the minimum for high availability; the same pattern extends to three, four, or more, and later across multiple VPSes (§22). The reason it works at all is that your Go app must be **stateless** — it must keep no important data in its own memory, so any instance can handle any request. All shared state lives in Postgres (durable data) and Redis (sessions, cache, rate-limit counters). This "stateless app, stateful backing services" split is the single most important architectural rule for a scalable backend, and §12 makes it concrete.

### 1.4 What "production-grade" means **[B/I]**

Throughout this guide, "production-grade" is not a vibe — it's a concrete checklist that separates a hobby server from one you'd put a company on:

| Property | What it means | Where in this guide |
|---|---|---|
| **Secure** | Hardened SSH, firewall, no exposed databases, least privilege, patched. | §5–§8, §23 |
| **Available** | Survives a crash; deploys without downtime; auto-restarts. | §1.3, §12, §19 |
| **Recoverable** | Backups exist and *have been test-restored*. | §21 |
| **Observable** | You can see logs, metrics, and health at a glance. | §20 |
| **Reproducible** | The whole server can be rebuilt from files in Git, not memory. | §11, §18, §24 |
| **Scalable** | A clear path from 1k to 100k+ users without a rewrite. | §22 |

Keep this table in mind. Every section below is buying you one or more of these properties, and by the end you'll have all six.

---

## 2. Provisioning Your VPS

### 2.1 Picking a provider and a size **[B]**

For a production backend, choose a provider with a good network, hourly-or-monthly billing, snapshots, and a private network feature. **Hetzner** (best price/performance in 2026), **DigitalOcean** (best docs/UX), and **Vultr**/**Linode** are all excellent for this scale; the big clouds (AWS EC2, GCP Compute Engine) are fine but pricier and more complex. The concepts in this guide are identical everywhere.

Sizing for a backend serving up to ~100k users depends on traffic, but a sensible *starting* point and its upgrade path:

| Stage | vCPU / RAM | Notes |
|---|---|---|
| Launch / small | **2 vCPU / 4 GB** | Comfortable for the whole stack (2 Go + PG + Redis) at low-moderate traffic. |
| Growing | **4 vCPU / 8 GB** | Headroom; Postgres likes RAM for its cache. |
| Busy / 100k users | **8 vCPU / 16–32 GB**, or **split** the DB onto its own VPS | Past a point, give Postgres its own machine (§22). |

Always pick a plan with **SSD/NVMe storage** (databases are I/O-bound) and note the **bandwidth allowance**. Start smaller than you think — you can resize a VPS up in minutes, and paying for idle capacity is waste. The one thing you can't easily change later is the provider, so pick one you trust.

### 2.2 Creating the VPS **[B]**

When you create the server in the provider's dashboard, you'll make a few choices that matter:

- **OS image:** choose **Ubuntu 24.04 LTS**. "LTS" (Long-Term Support) means 5 years of security updates — essential for production. Never run a non-LTS or end-of-life OS in production.
- **Region:** pick the datacenter closest to your users (lower latency). If your users are global, this is where a CDN (§22) later helps.
- **SSH key:** **add your SSH public key here, now.** This is the single most important security choice at creation time — it means the server is born with key-only login for `root` and never has a password to brute-force. §3.1 shows how to generate the key if you don't have one. *Do this instead of choosing a root password.*
- **Hostname:** a name like `prod-api-1`. Cosmetic, but good hygiene.
- **Backups / snapshots:** enable the provider's automated snapshots if offered (cheap insurance; complements your own backups in §21).
- **Private networking / VPC:** enable it. When you later add a second server (a separate database host, say), they'll talk over a private network instead of the public internet.

Click create, and in under a minute you have a running Linux server with a **public IP address** (e.g. `203.0.113.10`). Write that IP down; it's how you'll reach the machine until DNS is set up (§16).

### 2.3 The mental model of "where am I typing?" **[B]**

Beginners get confused about *which computer a command runs on*. There are only two:

- **Your local machine** (your laptop) — where you run `ssh`, `scp`, `git push`, and edit code. Its prompt might look like `you@laptop:~$`.
- **The remote VPS** — where you run everything else (installing packages, Docker, etc.) *after* you've SSH'd in. Its prompt looks like `root@prod-api-1:~#` or `deploy@prod-api-1:~$`.

Throughout this guide, a command block is **on the server** unless it's explicitly a local/client command (SSH, `scp`, DNS). The prompt in the examples tells you: `#` means you're root, `$` means a normal user, and a comment will say "on your laptop" when it's local. Getting this wrong — running a server command locally or vice versa — is the most common beginner stumble, so always know which of the two machines you're talking to.

---

## 3. First Contact — SSH and the Initial Login

### 3.1 What SSH is and generating your key **[B]**

**SSH** (Secure Shell) is the encrypted protocol you use to log into and control a remote server's command line. Everything you do to the server, you do over SSH. It authenticates you in one of two ways: a **password**, or — far better — a **key pair**. A key pair is two matching files: a **private key** (stays on your laptop, secret, never leaves) and a **public key** (you copy it to the server). The server encrypts a challenge with your public key that only your private key can answer, proving your identity without any password crossing the wire. Key auth is both more convenient (no typing passwords) and dramatically more secure (a 4096-bit key is effectively impossible to brute-force, unlike a password).

Generate a key pair on **your laptop** (not the server) if you don't already have one:

```bash
# On YOUR LAPTOP. ed25519 is the modern, fast, secure key type — prefer it over RSA.
# -C is just a comment/label so you recognize the key later.
# Press Enter to accept the default path (~/.ssh/id_ed25519); set a passphrase when asked.
ssh-keygen -t ed25519 -C "you@company.com"
```

This writes two files in `~/.ssh/` (on Windows: `C:\Users\You\.ssh\`): `id_ed25519` (the **private** key — guard it like a password) and `id_ed25519.pub` (the **public** key — safe to share). The **passphrase** it asks for encrypts the private key on disk, so a stolen laptop doesn't hand over your servers; combined with an agent (below) you only type it once per session.

> **⚡ Never share or commit a private key.** The `.pub` file is public; the file *without* `.pub` is the secret. If a private key ever leaks, it's compromised forever — generate a new pair and replace the public key everywhere. Committing a private key to Git is a classic, catastrophic mistake.

### 3.2 Logging in for the first time **[B]**

If you added your public key at creation time (§2.2), you can log in as `root` immediately from your laptop:

```bash
# On YOUR LAPTOP. Replace with your server's real IP. The first time, SSH asks you
# to confirm the server's fingerprint (type 'yes') — this pins the server's identity
# so a later man-in-the-middle swap is detected.
ssh root@203.0.113.10
```

You're now at a prompt like `root@prod-api-1:~#`. The `#` at the end signals you are **root** — the all-powerful superuser who can do *anything*, including destroy the system. That power is exactly why §4's first job is to stop using root for daily work. If instead you're asked for a password (because you chose a root password rather than a key), you can still add your key now:

```bash
# On YOUR LAPTOP: copy your public key to the server so future logins are key-based.
ssh-copy-id root@203.0.113.10          # POSIX; prompts for the password once
# On Windows PowerShell (no ssh-copy-id), pipe the key in manually:
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@203.0.113.10 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 3.3 The SSH client config — stop typing IPs **[B/I]**

Typing `ssh root@203.0.113.10` every time is tedious and error-prone. The SSH **client config** on your laptop lets you define a short alias with all the connection details. Create or edit `~/.ssh/config` (on Windows: `C:\Users\You\.ssh\config`):

```text
# ~/.ssh/config  (on YOUR LAPTOP)
Host prod                      # a short alias you invent
    HostName 203.0.113.10      # the server's IP (or later, its domain)
    User deploy                # the user you log in as (we create 'deploy' in §4)
    Port 22                    # SSH port (we may change this in §5)
    IdentityFile ~/.ssh/id_ed25519   # which private key to use
    IdentitiesOnly yes         # only offer THIS key, not every key you own
```

Now `ssh prod` connects with all those settings. This also makes `scp` (copy files) and tools that read SSH config "just work" with the `prod` alias. As you accumulate servers, this file becomes your address book. It's the first quality-of-life upgrade every engineer makes.

### 3.4 Copying files with scp and rsync **[B/I]**

You'll frequently need to move files between your laptop and the server — a config file, a backup, a binary. Two tools:

```bash
# scp — secure copy, like cp but over SSH. On YOUR LAPTOP:
scp ./Caddyfile prod:/home/deploy/          # laptop → server
scp prod:/var/log/app.log ./                # server → laptop  (note the direction)
scp -r ./config prod:/home/deploy/app/      # -r copies a directory recursively

# rsync — smarter: copies only changed parts, resumable, great for deploys/backups.
rsync -avz --delete ./dist/ prod:/home/deploy/app/dist/
#      │││        └ make the destination MATCH the source (delete extra files) — careful!
#      ││└ z: compress in transit
#      │└ v: verbose (show what's happening)
#      └ a: archive mode (preserve permissions, timestamps, recurse)
```

Use `scp` for one-off single files; use `rsync` for directories, repeated syncs, and anything large — it only transfers the differences, which is why it's the backbone of many deploy and backup scripts (§19, §21). The `--delete` flag is powerful and dangerous: it makes the destination an exact mirror of the source, deleting anything extra — double-check the paths before running it.

---

## 4. Users, sudo and Least Privilege

### 4.1 Why you must stop using root **[B]**

You logged in as **root**, the superuser who can do anything. That is precisely the problem. Working as root all day means every typo is potentially catastrophic (`rm -rf /` deletes the entire system with no confirmation), every program you run has unlimited power (a compromised tool owns the whole machine), and there's no audit trail of *who* did *what* (everyone is "root"). The **principle of least privilege** says: run with the *minimum* power needed, and elevate to full power only for the specific commands that require it, deliberately. In practice that means: create a normal user for daily work, log in as *them*, and prefix the rare privileged command with `sudo`.

### 4.2 Creating a deploy user **[B]**

```bash
# On the SERVER, as root. Create a normal user named 'deploy'.
adduser deploy
#   → prompts for a password (set a strong one; it's a fallback, not your main auth)
#   → creates /home/deploy, a home directory the user owns

# Add 'deploy' to the 'sudo' group, granting the RIGHT to run commands as root
# (by prefixing them with `sudo`) — but only when they choose to.
usermod -aG sudo deploy
#         │└ a: APPEND to the group (without -a you'd REPLACE all their groups — a classic footgun)
#         └ G: the supplementary group to add
```

`adduser` (the friendly Debian/Ubuntu wrapper; the lower-level tool is `useradd`) creates the account, its home directory, and a private group. `usermod -aG sudo deploy` grants sudo rights. The `-a` (append) is critical: `usermod -G sudo deploy` *without* `-a` would remove `deploy` from every other group and set it to *only* `sudo` — always use `-aG` together.

### 4.3 Giving the deploy user your SSH key **[B]**

The `deploy` user needs your SSH public key too, or you can't log in as them with key auth. Copy root's authorized key over (since you already trust it), or add your key fresh:

```bash
# On the SERVER, as root: copy your key to the new user and fix ownership/permissions.
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh    # the files must be OWNED by deploy
chmod 700 /home/deploy/.ssh                  # dir: only owner may enter
chmod 600 /home/deploy/.ssh/authorized_keys  # file: only owner may read/write
```

The **permissions matter and SSH enforces them**: if `~/.ssh` or `authorized_keys` is readable by others, SSH *refuses* to use the key (a security feature — a world-readable key file is a red flag). `700` means "owner: read+write+execute; group/others: nothing"; `600` means "owner: read+write; others: nothing." §9.3 explains the permission numbers in full.

Now test from your laptop **in a new terminal** (keep the root session open as a safety net until you're sure): `ssh deploy@203.0.113.10`. You should land at `deploy@prod-api-1:~$` — note the `$`, meaning a normal (non-root) user. From here on, this is who you are.

### 4.4 Using sudo **[B]**

When you need root power for one command, prefix it with `sudo`:

```bash
sudo apt update           # runs apt as root; asks for YOUR password the first time
sudo systemctl restart caddy
sudo -i                   # start an interactive root shell (for a run of admin commands)
# ...but prefer per-command sudo; the fewer keystrokes as root, the fewer disasters.
```

`sudo` logs every use (in `/var/log/auth.log`), so there's an audit trail; it asks for *your* password (not root's), caches it for a few minutes, and runs just that one command elevated. This is least privilege in daily practice: you are a normal user who *borrows* root power, deliberately and briefly, rather than swimming in it constantly.

> **Best practice:** never enable password SSH for root again, and after §5 you'll disable root SSH entirely. The `deploy` user + `sudo` is how every professional operates. If multiple people administer the server, give each their own user (never a shared login) so the audit log means something.

---

## 5. Hardening SSH

### 5.1 Why SSH is the front door **[B/I]**

SSH on port 22 is, by default, exposed to the entire internet, and automated bots scan every IP constantly, trying to log in with common usernames and passwords thousands of times a day. If you allow password login, it is only a matter of time before a weak password is guessed. Hardening SSH closes this off almost completely. The configuration lives in **`/etc/ssh/sshd_config`** on the server (note the `d` — `sshd` is the SSH *daemon*/server; `ssh` without the `d` is the client). We'll make four changes, each removing an attack.

### 5.2 The four hardening changes **[B/I]**

Edit the config with `sudo nano /etc/ssh/sshd_config` (or `vim`), and set these directives (find and change them, or add them):

```text
# /etc/ssh/sshd_config  (on the SERVER)

# 1) Disable password login entirely — keys only. This ALONE stops ~all brute force.
PasswordAuthentication no

# 2) Never allow root to log in over SSH. Attackers always try 'root' first; deny it.
PermitRootLogin no

# 3) Don't even offer keyboard-interactive/challenge auth (another password vector).
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

# 4) (Optional but common) move SSH off port 22 to cut log noise from bots.
# Pick a high port and REMEMBER to open it in the firewall (§6) BEFORE you disconnect.
Port 2222
```

> **⚡ The lockout trap — read this twice.** If you change `Port` and don't open the new port in the firewall, or you disable password auth before confirming your key works, your *next* login attempt fails and you've locked yourself out of your own server. **Always keep your current SSH session open, apply the change, and test a NEW connection in a second terminal before closing the first.** If the new one works, you're safe; if not, you fix it from the still-open session. Providers offer a web "console" as a last resort, but avoid needing it.

Apply the changes by restarting the SSH service, then test:

```bash
sudo systemctl restart ssh      # reload the daemon with the new config (Ubuntu 24.04: 'ssh')
# On some systems the unit is 'sshd'. If unsure: systemctl restart ssh || systemctl restart sshd

# In a NEW laptop terminal, test the new settings (note the -p for a custom port):
ssh -p 2222 deploy@203.0.113.10
```

### 5.3 Extra SSH hardening **[I]**

For a production host, a few more directives tighten things further:

```text
# /etc/ssh/sshd_config — additional hardening
AllowUsers deploy               # ONLY this user may SSH in (whitelist). Add more as needed.
MaxAuthTries 3                  # disconnect after 3 failed attempts (slows bots)
LoginGraceTime 20               # 20s to authenticate, then drop the connection
X11Forwarding no                # you're not running GUI apps; disable this surface
ClientAliveInterval 300         # ping idle clients every 5 min...
ClientAliveCountMax 2           # ...drop after 2 missed pings (cleans up dead sessions)
```

`AllowUsers deploy` is especially strong: even if an attacker somehow obtained another user's key, SSH would refuse them because they're not on the whitelist. Restart `ssh` after editing. Between key-only auth, no-root, and a user whitelist, remote password guessing is off the table — §7's fail2ban then handles the residual noise, and §6's firewall ensures only the ports you chose are reachable at all.

---

## 6. The Firewall with ufw and nftables

### 6.1 What a firewall does and why **[B]**

A **firewall** controls which network connections are allowed in and out of the server. Its production job is simple and vital: **allow only the ports you actually serve, and block everything else.** Your VPS has many services that might listen on various ports; without a firewall, a misconfigured or forgotten service could be reachable from the internet. With a default-deny firewall, even if something starts listening on port 9999, nobody outside can reach it — you've closed every door except the few you deliberately open.

Under the hood, modern Linux filters packets with **nftables** (the successor to `iptables`), which is powerful but verbose. **ufw** ("Uncomplicated Firewall") is a friendly front-end that generates nftables rules from simple commands. On Ubuntu, use ufw; you rarely need raw nftables unless you have advanced needs (§6.4).

### 6.2 Configuring ufw **[B]**

```bash
# On the SERVER. Set the default policy FIRST: deny all incoming, allow all outgoing.
sudo ufw default deny incoming     # nobody gets in unless a rule allows it
sudo ufw default allow outgoing    # the server can reach out (updates, APIs) freely

# Allow ONLY what you serve. CRITICAL: allow SSH before enabling, or you lock yourself out.
sudo ufw allow 2222/tcp comment 'SSH'      # your (custom) SSH port from §5
sudo ufw allow 80/tcp   comment 'HTTP'     # Caddy — needed for HTTP→HTTPS redirect + ACME
sudo ufw allow 443/tcp  comment 'HTTPS'    # Caddy — the real traffic
sudo ufw allow 443/udp  comment 'HTTP/3'   # Caddy serves HTTP/3 over QUIC (UDP 443)

# Turn it on. It re-confirms because an SSH lockout is possible.
sudo ufw enable

# Inspect what's active, with rule numbers (for deleting later).
sudo ufw status verbose
sudo ufw status numbered
```

That is the entire production firewall: **SSH, HTTP, HTTPS, and nothing else.** Notice what is *not* here — Postgres (5432), Redis (6379), and your Go apps (8080) have **no firewall rule**, so they are unreachable from the internet. They don't need one because they only ever talk over Docker's *internal* network (§11–§12). This is the defense-in-depth payoff: even if you forgot to bind Postgres to localhost, the firewall would still block external access.

### 6.3 Managing rules **[B/I]**

```bash
sudo ufw status numbered            # list rules with numbers
sudo ufw delete 3                   # delete rule #3 (numbers shift after each delete!)
sudo ufw allow from 203.0.113.55 to any port 2222 proto tcp   # allow SSH only from ONE IP
sudo ufw deny 8080                  # explicitly deny a port
sudo ufw reload                     # reapply rules after manual config edits
sudo ufw disable                    # turn the firewall off (rarely; e.g. debugging)
```

A powerful hardening step for SSH: restrict it to your office/home IP with `sudo ufw allow from <your-ip> to any port 2222 proto tcp` and remove the open rule. Now SSH is only reachable from *you* — but be careful if your IP is dynamic (many home connections are), or you'll lock yourself out when it changes. A middle ground is a VPN/bastion (§22), but for a small team a static-IP allow-list is excellent.

### 6.4 A note on nftables and cloud firewalls **[I/A]**

`ufw` writes nftables rules; you can see them with `sudo nft list ruleset`. For most servers you never touch nft directly. Two things worth knowing: (1) many providers *also* offer a **cloud firewall** at the network edge (configured in their dashboard) — using both is good defense in depth (the cloud firewall stops traffic before it even reaches your VPS), just keep them consistent. (2) **Docker manipulates iptables/nftables directly** to publish container ports, and can *bypass* ufw for published ports — a notorious gotcha. The fix is to never publish container ports you don't want public (bind them to `127.0.0.1`, §12.4), which we do by design. §25 revisits this trap.

---

## 7. Fail2ban and Intrusion Prevention

### 7.1 What fail2ban does **[I]**

Even with key-only SSH, bots still hammer your ports with connection attempts, filling logs and wasting resources. **fail2ban** watches log files for patterns of malicious behavior (repeated failed logins, for example) and *automatically bans* the offending IP by adding a temporary firewall rule. It turns "someone is trying 10,000 passwords" into "that IP is blocked after 5 tries for an hour." It's a reflex the server does for you, around the clock.

### 7.2 Installing and configuring **[I]**

```bash
sudo apt update && sudo apt install -y fail2ban

# Never edit jail.conf directly (updates overwrite it). Create a local override:
sudo nano /etc/fail2ban/jail.local
```

```ini
# /etc/fail2ban/jail.local  (on the SERVER)
[DEFAULT]
bantime  = 1h          # how long a banned IP stays banned
findtime = 10m         # the window in which failures are counted
maxretry = 5           # ban after this many failures within findtime
# Never ban yourself: whitelist your office IP and localhost.
ignoreip = 127.0.0.1/8 ::1 203.0.113.55

[sshd]
enabled = true         # protect SSH (the most important jail)
port    = 2222         # match your custom SSH port from §5
```

```bash
sudo systemctl enable --now fail2ban    # start it now AND on every boot
sudo fail2ban-client status             # list active jails
sudo fail2ban-client status sshd        # see banned IPs for the ssh jail
sudo fail2ban-client set sshd unbanip 203.0.113.55   # manually unban (e.g. if you locked yourself)
```

`enable --now` is the common systemd idiom meaning "start the service immediately **and** configure it to start automatically at boot." After this, SSH brute-force noise vanishes from your logs, and the residual attack surface after §5–§7 is essentially nil. fail2ban can also protect Caddy, your app's login endpoint, and more via custom "jails" — but SSH is the essential one.

---

## 8. Automatic Security Updates

### 8.1 Why unattended upgrades **[I]**

Software has vulnerabilities; vendors ship patches; **the patched machine is safe and the unpatched one is not.** In production you cannot rely on remembering to run updates — you configure the server to install **security** updates automatically. Ubuntu's `unattended-upgrades` does exactly this: it applies security patches on a schedule, unattended, and can email you a report.

```bash
sudo apt update && sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades   # answer "Yes" to enable
```

The behavior is configured in `/etc/apt/apt.conf.d/50unattended-upgrades`. The defaults apply *security* updates only (conservative — non-security updates you apply deliberately). One important setting is whether to auto-reboot when a kernel update requires it:

```text
// /etc/apt/apt.conf.d/50unattended-upgrades  (on the SERVER)
Unattended-Upgrade::Automatic-Reboot "true";              // reboot if a patch needs it...
Unattended-Upgrade::Automatic-Reboot-Time "04:00";        // ...at 4am, your low-traffic hour
```

### 8.2 Manual updates you still run **[I]**

Security patches are automatic, but you'll still periodically apply *all* updates and clean up, deliberately, ideally right before a deploy so you're testing on current software:

```bash
sudo apt update            # refresh the list of available packages (doesn't install anything)
sudo apt upgrade -y        # install available upgrades for installed packages
sudo apt full-upgrade -y   # like upgrade but will add/remove packages if needed (kernels)
sudo apt autoremove --purge -y   # remove orphaned packages and their config (frees space)
sudo reboot                # reboot if a kernel/glibc update needs it (check with: needrestart)
```

`apt update` and `apt upgrade` are two different things people constantly conflate: **`update`** refreshes the *catalog* of what's available (installs nothing); **`upgrade`** actually installs the newer versions. You always `update` before `upgrade`. Because your app runs in Docker, an OS reboot restarts the host but your containers come back automatically (with the right restart policy, §12.5), so patching the host is low-risk — do it regularly.

> **The two-machine reminder (§2.3):** every command in §4–§8 runs **on the server**, over your SSH session. The next sections stay on the server too, until we reach DNS (§16), which is configured in your registrar's web dashboard, and deploys (§19), which run from your laptop or CI.

---

## 9. Linux Command Mastery for Operations

This section is your working reference for the Linux commands you'll actually use running a production server — grouped by task, each with *what it does, when you reach for it,* and the flags that matter. You don't need to memorize these; you need to recognize them and know where to look. (For the language-level deep dive, see [Linux Server Admin](LINUX_SERVER_ADMIN_GUIDE.md) and [Bash](BASH_SCRIPTING_GUIDE.md).)

### 9.1 Navigating and inspecting the filesystem **[B]**

Everything in Linux is a file, and the filesystem is a single tree rooted at `/`. You move around and look at it with:

```bash
pwd                 # "print working directory" — where am I right now?
ls -lah             # list: -l long form, -a include hidden (dotfiles), -h human sizes (K/M/G)
cd /etc/caddy       # change directory (absolute path, from /)
cd ~                # go home (~ = your home dir, /home/deploy); `cd -` = previous dir
tree -L 2 /home/deploy/app     # show the directory tree 2 levels deep (apt install tree)
find /var/log -name "*.log" -mtime -1   # find .log files modified in the last day
du -sh /var/lib/docker         # disk usage of a dir, summarized & human-readable
df -h                          # disk FREE space per filesystem — check before you run out!
stat Caddyfile                 # detailed metadata: size, owner, permissions, timestamps
```

Key production locations you'll live in: **`/etc`** (system + app configuration), **`/var/log`** (logs), **`/var/lib`** (application data, incl. Docker's `/var/lib/docker`), **`/home/deploy`** (your app files), **`/opt`** (optional/third-party software), and **`/tmp`** (scratch, wiped on reboot). Knowing where things live is half of operations.

### 9.2 Reading and editing files **[B]**

```bash
cat file.txt              # dump a whole (small) file to the screen
less /var/log/syslog      # page through a LARGE file (arrows to scroll, /term to search, q to quit)
head -n 50 file           # first 50 lines;  tail -n 50 file  = last 50
tail -f /var/log/caddy/access.log   # FOLLOW a log live — watch new lines appear (Ctrl-C to stop)
grep -i "error" app.log             # find lines containing "error" (-i = case-insensitive)
grep -rn "DATABASE_URL" /home/deploy/app   # -r recurse dirs, -n show line numbers
nano file.conf            # beginner-friendly editor (Ctrl-O save, Ctrl-X exit)
vim file.conf             # powerful modal editor (i to insert, Esc, :wq to save+quit)
```

`tail -f` is the single most-used operational command — it's how you *watch* what a service is doing right now. `grep` is how you *search* logs and configs. `less` is how you *read* big files without loading them into memory. Master these three and you can debug almost anything.

### 9.3 Permissions and ownership — the model **[B/I]**

Every file has an **owner** (a user), a **group**, and a set of **permissions** for three classes: the owner (`u`), the group (`g`), and everyone else (`o`). Each class can have **read (r=4)**, **write (w=2)**, and **execute (x=1)** permission. The numbers add up per class, giving the familiar three-digit modes:

| Mode | Meaning | Typical use |
|---|---|---|
| `600` | owner rw, others nothing | secrets, private keys, `.env` files |
| `644` | owner rw, others read | normal config/data files |
| `700` | owner rwx, others nothing | private directories, scripts only you run |
| `755` | owner rwx, others rx | directories, public executables |
| `640` | owner rw, group read | config a service's group may read |

```bash
ls -l secret.env                  # -rw------- 1 deploy deploy ...  → mode 600, owner deploy
chmod 600 secret.env              # set the mode numerically
chmod u+x deploy.sh               # or symbolically: give the owner execute
chown deploy:deploy file          # change owner:group (needs sudo)
chown -R deploy:deploy /app       # -R = recursive, whole tree
```

The rule for production secrets: **`chmod 600` and owned by the one user that needs them.** SSH enforces this on keys (§4.3); you enforce it on `.env` files and TLS certs. A world-readable secret is a leaked secret waiting for any local process to read it.

### 9.4 Processes — seeing and controlling what runs **[B/I]**

A **process** is a running program. You inspect and control them with:

```bash
ps aux                    # snapshot of ALL processes (user, PID, CPU%, MEM%, command)
ps aux | grep caddy       # filter for a specific process
top                       # live, updating process list sorted by CPU (q to quit)
htop                      # nicer top (apt install htop) — colors, tree view, click to kill
kill 12345                # politely ask process ID 12345 to stop (SIGTERM)
kill -9 12345             # forcibly kill it (SIGKILL) — last resort; no cleanup
pkill -f "myapp"          # kill by name/command pattern
```

**Every process has a PID** (process ID). `ps aux | grep X` finds X's PID; `kill` stops it. `top`/`htop` show you *what's eating CPU or memory right now* — the first thing you check when the server is slow or the provider emails "high load." In production, though, you rarely `kill` app processes directly — they run under **systemd** or **Docker**, which manage their lifecycle for you (below).

### 9.5 systemd — how services run and survive reboots **[B/I]**

**systemd** is the init system that starts and supervises long-running services ("units") on modern Linux. It's how SSH, Caddy (if installed natively), Docker, and fail2ban run — started at boot, restarted if they crash, with logs captured. You control services with `systemctl`:

```bash
sudo systemctl status docker      # is it running? recent logs? (the command you run most)
sudo systemctl start  docker      # start now
sudo systemctl stop   docker      # stop now
sudo systemctl restart docker     # stop then start (apply new config)
sudo systemctl reload  caddy      # reload config WITHOUT dropping connections (if supported)
sudo systemctl enable  docker     # start automatically at every boot
sudo systemctl disable docker     # don't start at boot
sudo systemctl enable --now docker   # enable AND start, in one command
systemctl list-units --type=service --state=running   # what's currently running?
```

The distinction that trips people up: **`start`/`stop`** affect the service *right now*; **`enable`/`disable`** affect whether it starts *at boot*. A service can be running now but not enabled (won't survive a reboot) or enabled but stopped. For anything important, you want it **enabled and running** — hence `enable --now`. This is why your Dockerized app survives a server reboot: Docker is enabled, and your containers have a restart policy (§12.5).

### 9.6 Logs — journalctl and log files **[B/I]**

systemd captures each service's output into the **journal**, queried with `journalctl`; some services also write to files in `/var/log`.

```bash
journalctl -u docker              # all logs for the docker unit
journalctl -u caddy -f            # FOLLOW caddy's logs live (-f), like tail -f
journalctl -u ssh --since "1 hour ago"     # time-filtered
journalctl -p err -b              # only errors (-p err), this boot (-b)
journalctl --disk-usage           # how much space the journal is using
sudo journalctl --vacuum-time=7d  # trim journal to the last 7 days (reclaim disk)
tail -f /var/log/auth.log         # SSH/sudo auth events (who logged in, sudo use)
```

When something is broken, the debugging loop is almost always: `systemctl status X` (is it running? what's the last error?) → `journalctl -u X -f` (watch it live while you reproduce the problem). For Docker containers, the equivalent is `docker logs -f <container>` (§10.6). Logs are the truth; learn to read them first, before guessing.

### 9.7 Networking — is it listening, is it reachable **[B/I]**

```bash
ip a                      # show network interfaces & IP addresses (replaces old `ifconfig`)
ss -tulpn                 # what's LISTENING? -t tcp -u udp -l listening -p process -n numeric
#   → e.g. shows caddy on :80/:443, sshd on :2222, and (bound to 127.0.0.1) your app on :8080
curl -I https://api.company.com    # make an HTTP request; -I = headers only (health check)
curl -v telnet://localhost:5432    # can I reach Postgres locally? (-v verbose)
ping 1.1.1.1              # basic reachability (some hosts block ICMP)
dig api.company.com       # DNS lookup — what IP does this name resolve to? (§16)
nc -zv localhost 6379     # netcat: is a TCP port open? -z scan -v verbose
traceroute 1.1.1.1        # the network path/hops to a destination
```

**`ss -tulpn` is the security-critical one:** it shows *everything listening on the server and which process owns it.* Run it after setup and confirm that only Caddy is bound to a public interface (`0.0.0.0:80`, `:443`) and everything else (your app `:8080`, Postgres `:5432`, Redis `:6379`) is either not host-published at all or bound only to `127.0.0.1`. If a database is listening on `0.0.0.0`, it's exposed — fix it immediately (§12.4).

### 9.8 Disk, memory, and the health-check trio **[B/I]**

```bash
df -h                     # disk space per filesystem — a full disk takes everything down
du -sh * | sort -h        # what's using space HERE, sorted smallest→largest
free -h                   # memory: used/free/available (watch 'available', not 'free')
uptime                    # load average (1/5/15 min) + how long the box has been up
nproc                     # number of CPU cores (load average is relative to this)
docker system df          # disk used by Docker images/containers/volumes (often the culprit)
```

The three questions to ask a struggling server, in order: **is the disk full?** (`df -h`), **is it out of memory?** (`free -h`), **is the CPU pegged?** (`top`/`uptime` load vs `nproc`). A shocking fraction of production incidents are simply "the disk filled up with logs or Docker images." `df -h` is the first command of every incident. Set up alerts for disk >80% (§20) so you're never surprised.

### 9.9 Package management and finding things **[B]**

```bash
sudo apt install -y htop tree jq   # install packages (-y = don't prompt)
sudo apt remove pkg                # remove a package
apt list --installed | grep docker # what's installed?
which caddy                        # where is a command's binary?
man ss                             # the MANUAL for a command — the offline source of truth
ss --help                          # quick flag reminder
history | grep ufw                 # what ufw commands did I run before?
```

`man <command>` is the offline documentation for essentially every tool on the system — when you forget a flag, `man` (or `--help`) is faster and more authoritative than a web search, and it works with no internet. `jq` (JSON processor) is worth installing early; you'll pipe Docker and API JSON through it constantly.

---

## 10. Docker From Zero

### 10.1 What Docker is and why production runs on it **[B]**

**Docker** packages an application together with *everything it needs to run* — the binary, its libraries, its runtime, its config — into a single, portable unit called an **image**. You run an image as a **container**: an isolated process that behaves identically on your laptop, in CI, and on the production VPS. This solves the oldest problem in deployment — "it works on my machine" — because the machine *is the image*, and it's the same everywhere.

For production, Docker buys you four concrete things: **isolation** (each service runs in its own sandbox; a crash or compromise is contained), **reproducibility** (the exact same image runs in every environment; no "the server had a different library version"), **easy multi-service orchestration** (run Postgres, Redis, and two app instances with one command, on a private network — §11), and **clean lifecycle management** (start, stop, restart, and auto-restart-on-crash, all standardized). This is why we run the entire backend — the two Go instances, Postgres, and Redis — as containers.

Three words to keep straight: an **image** is the built, immutable template (like a class); a **container** is a running instance of an image (like an object); a **volume** is persistent storage that outlives a container (so your database data isn't lost when the container is replaced).

### 10.2 Installing Docker on the server **[B]**

Install Docker Engine from Docker's official repository (not Ubuntu's older `docker.io` package):

```bash
# On the SERVER. Docker's convenience script installs Engine + Compose plugin + Buildx.
curl -fsSL https://get.docker.com | sudo sh
#    │└ f fail silently, s silent, S show errors, L follow redirects

# Let the 'deploy' user run docker WITHOUT sudo (adds them to the 'docker' group).
sudo usermod -aG docker deploy
# Log out and back in for the group change to take effect, then verify:
docker version                 # client + server (daemon) versions
docker compose version         # the Compose v2 PLUGIN (note: `docker compose`, a subcommand)
docker run --rm hello-world    # pull & run a test image; --rm auto-removes it after
```

> **⚡ Security note on the docker group.** Membership in the `docker` group is effectively **root** (you can mount the host filesystem into a container and escalate). Only give it to trusted admin users — treat "in the docker group" as "has root." That's fine for your `deploy` user; don't hand it out casually.

### 10.3 Images — pulling, listing, building **[B]**

```bash
docker pull postgres:17          # download an image (tag :17 pins the major version)
docker images                    # list local images (repository, tag, size)
docker build -t myapp:1.0 .      # build an image from the Dockerfile in the current dir (.)
docker rmi postgres:16           # remove an image
docker image prune -a            # delete ALL unused images (reclaim disk — do this regularly)
```

**Always pin image tags** (`postgres:17`, not `postgres:latest`) in production — `latest` is a moving target that can change under you and break a deploy. Pinning makes deploys reproducible: the image you tested is the image that runs.

### 10.4 A production Dockerfile for the Go app **[B/I]**

A **Dockerfile** is the recipe for building your app's image. For Go, the gold standard is a **multi-stage build**: compile in a full Go image, then copy just the tiny static binary into a minimal final image. The result is a container that's a few megabytes, has almost no attack surface, and starts instantly.

```dockerfile
# Dockerfile  (in your Go app's repo root)
# ── Stage 1: build ──────────────────────────────────────────────────────────
FROM golang:1.26 AS build
WORKDIR /src
# Copy go.mod/go.sum first and download deps — this layer caches unless deps change,
# so rebuilds are fast when only your code changed.
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Build a static, stripped binary. CGO off → no libc dependency → runs on 'scratch'.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/server

# ── Stage 2: run ────────────────────────────────────────────────────────────
# 'distroless' has no shell/package manager — minimal attack surface. Or use 'alpine'.
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot          # run as a NON-root user inside the container (defense in depth)
EXPOSE 8080                   # documents the port the app listens on (doesn't publish it)
ENTRYPOINT ["/app"]           # what runs when the container starts
```

Two production essentials are baked in: the **multi-stage build** keeps the final image tiny (no compiler, no source), and **`USER nonroot`** means that even if your app is exploited, the attacker is a powerless user inside a minimal container, not root. `CGO_ENABLED=0` produces a fully static binary so it runs in a `scratch`/`distroless` image with no OS libraries at all. A `.dockerignore` file (listing `.git`, `*.md`, local `.env`) keeps junk out of the build context.

### 10.5 Running containers **[B/I]**

```bash
docker run -d --name web -p 8080:8080 myapp:1.0     # -d detached (background), --name to reference it
#              -p HOST:CONTAINER publishes a port from the container to the host
docker ps                         # list RUNNING containers
docker ps -a                      # list ALL containers (incl. stopped)
docker stop web                   # graceful stop (SIGTERM, then SIGKILL after 10s)
docker start web                  # start a stopped container
docker restart web
docker rm web                     # remove a stopped container
docker rm -f web                  # force-remove a running one
docker exec -it web sh            # run a shell INSIDE a running container (-it interactive)
```

`docker exec -it <container> sh` (or `bash`) is how you "get inside" a running container to look around — invaluable for debugging. (Note: a `distroless` image has no shell, so `exec` won't give you one — a deliberate hardening trade-off; you debug those via logs and by running a debug variant.) In production you rarely run `docker run` by hand, though — you declare all of this in a **Compose file** (§11) so the whole stack is one versioned, reproducible document.

### 10.6 Logs, inspection, and volumes **[B/I]**

```bash
docker logs web                   # the container's stdout/stderr
docker logs -f --tail 100 web     # follow live, starting from the last 100 lines
docker inspect web                # full JSON: config, network, mounts, env (pipe to jq)
docker stats                      # LIVE CPU/memory/network per container (like top)

# Volumes — persistent storage that survives container removal.
docker volume create pgdata
docker volume ls
docker run -d -v pgdata:/var/lib/postgresql/data postgres:17   # mount the volume
docker volume inspect pgdata      # where does it live on the host? (/var/lib/docker/volumes/...)
```

**Volumes are how databases keep their data.** A container's own filesystem is ephemeral — remove the container and it's gone. A **volume** is storage Docker manages *outside* the container's lifecycle, so when you replace the Postgres container (to upgrade it, say), you attach the same `pgdata` volume and the data is intact. Getting this wrong — running a database without a volume — means losing all your data the first time the container is recreated. It is the single most important Docker concept for stateful services.

### 10.7 Cleaning up — reclaiming disk **[B/I]**

Docker accumulates images, stopped containers, and dangling volumes that silently fill the disk (a top cause of "the server died" — see §9.8):

```bash
docker system df                  # what's using space?
docker system prune               # remove stopped containers, unused networks, dangling images
docker system prune -a --volumes  # AGGRESSIVE: also remove unused images AND volumes — careful!
docker image prune -a             # just unused images
```

> **⚡ `--volumes` deletes data.** `docker system prune -a --volumes` will delete any volume not attached to a running container — which could include a database volume if the DB happens to be stopped. Never run it blindly on a production host. Prefer targeted `docker image prune -a` on a schedule, and back up before any aggressive cleanup.

---

## 11. Docker Compose for the Stack

### 11.1 Why Compose **[I]**

Running four containers (two Go instances, Postgres, Redis) with `docker run`, wiring them onto a shared network, setting env vars, volumes, restart policies, and health checks — by hand, in the right order, every time — is error-prone and unrepeatable. **Docker Compose** replaces all of it with a single declarative file, `compose.yaml`, that describes the *entire* stack: every service, its image, its config, its dependencies, and how they connect. One command (`docker compose up -d`) brings the whole thing up; the file lives in Git, so your production stack is **reproducible from source**. This is infrastructure-as-code for a single host, and it's exactly right for the 100k-user scale.

### 11.2 The anatomy of a Compose file **[I]**

```yaml
# compose.yaml  (on the SERVER, in /home/deploy/app/)
services:                    # each key is one container (a "service")
  postgres:                  # the service name IS its hostname on the private network
    image: postgres:17
    restart: unless-stopped  # auto-restart on crash/reboot (§12.5)
    environment:             # config passed as environment variables
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/pg_password   # read secret from a file (§18)
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume → data persists (§10.6)
    networks: [backend]      # attach to the private 'backend' network
    # NOTE: no `ports:` — Postgres is NOT published to the host. Only the apps reach it.

volumes:                     # declare the named volumes used above
  pgdata:

networks:                    # declare the private network the services share
  backend:
    driver: bridge
```

Two structural ideas do the heavy lifting. First, **a service's name is its DNS hostname** on the Compose network: the Go app connects to Postgres at `postgres:5432` — Compose runs an internal DNS so `postgres` resolves to that container's private IP. Second, **only services with a `ports:` mapping are reachable from the host/internet.** Postgres and Redis have no `ports:`, so they exist *only* on the private `backend` network — invisible to the outside world, no firewall rule needed. This is the whole security architecture from §1.2, expressed in YAML.

### 11.3 The essential Compose commands **[I]**

```bash
# All run in the directory containing compose.yaml (on the SERVER):
docker compose up -d            # create & start the WHOLE stack, detached
docker compose ps               # status of the stack's services
docker compose logs -f app-1    # follow one service's logs
docker compose logs -f          # follow ALL services (interleaved)
docker compose restart app-1    # restart one service
docker compose up -d --build    # rebuild images that changed, then apply
docker compose down             # stop & remove containers + network (KEEPS named volumes)
docker compose down -v          # also remove volumes — DELETES DATA; almost never in prod
docker compose exec app-1 sh    # shell into a running service
docker compose pull             # pull newer versions of the pinned images
docker compose config           # validate & print the fully-resolved config (great for debugging)
```

`docker compose up -d` is idempotent and smart: run it after editing `compose.yaml` and it changes *only* what differs — recreating the services you altered, leaving the rest running. That's how you apply config changes with minimal disruption. `docker compose config` is your syntax-checker — run it before every `up` to catch YAML mistakes.

> **⚡ `down -v` is the data-loss command.** `docker compose down` alone is safe (it keeps volumes). Adding `-v` removes the volumes too — deleting your database. Keep them straight: **`down` = stop the stack; `down -v` = stop the stack and erase its data.** In production you almost never use `-v`.

---

## 12. The Production Backend — Two Go Instances, Postgres and Redis

### 12.1 The complete Compose file **[I/A]**

Here is the entire production stack in one file. Read the comments — every line is a deliberate production choice. Caddy (the reverse proxy) is added in §15; this file is the application tier.

```yaml
# compose.yaml  — /home/deploy/app/compose.yaml  (on the SERVER)
services:
  # ── Two identical Go application instances (redundancy + zero-downtime deploys) ──
  app-1:
    image: registry.company.com/api:${APP_VERSION:-latest}   # pinned image from your registry
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://appuser@postgres:5432/appdb?sslmode=disable  # 'postgres' = the service name
      REDIS_ADDR: redis:6379                                                # 'redis'   = the service name
      PGPASSWORD_FILE: /run/secrets/pg_password
      JWT_SECRET_FILE: /run/secrets/jwt_secret
    secrets: [pg_password, jwt_secret]
    networks: [backend]
    depends_on:
      postgres: { condition: service_healthy }   # wait until PG is actually READY, not just started
      redis:    { condition: service_healthy }
    healthcheck:                                  # Caddy & Compose use this to know app-1 is alive
      test: ["CMD", "/app", "healthcheck"]        # your app exposes a tiny health subcommand/endpoint
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }     # cap so one service can't starve the others

  app-2:                                          # IDENTICAL to app-1 — the second instance
    image: registry.company.com/api:${APP_VERSION:-latest}
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://appuser@postgres:5432/appdb?sslmode=disable
      REDIS_ADDR: redis:6379
      PGPASSWORD_FILE: /run/secrets/pg_password
      JWT_SECRET_FILE: /run/secrets/jwt_secret
    secrets: [pg_password, jwt_secret]
    networks: [backend]
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    healthcheck:
      test: ["CMD", "/app", "healthcheck"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }

  # ── PostgreSQL — durable data. Private; no host port. ──
  postgres:
    image: postgres:17
    restart: unless-stopped
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/pg_password
    secrets: [pg_password]
    volumes:
      - pgdata:/var/lib/postgresql/data          # PERSISTENT data (survives container replacement)
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro   # tuned config (§13)
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]   # 'is the DB accepting connections?'
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Redis — cache, sessions, rate-limit counters. Private; no host port. ──
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - redisdata:/data                          # AOF/RDB persistence (§14)
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks: [backend]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]         # expects 'PONG'
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
  redisdata:

networks:
  backend:
    driver: bridge          # a private bridge network only these containers share

secrets:                    # secrets mounted as files at /run/secrets/<name> (§18)
  pg_password:
    file: ./secrets/pg_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

### 12.2 How the services find and talk to each other **[I/A]**

This is the part beginners find magical, so let's demystify it. When Compose creates the `backend` network, it runs an **embedded DNS server** on it. Every service is registered by its **service name**, so:

- `app-1` and `app-2` connect to Postgres using the hostname **`postgres`** (`postgres://appuser@postgres:5432/...`). Compose's DNS resolves `postgres` to that container's private IP on the `backend` network.
- They reach Redis at **`redis:6379`** the same way.
- The name resolution is *internal only* — `postgres` means nothing outside this network. There are no IP addresses to hardcode; if a container restarts with a new IP, the name still resolves. This is why you **never** put IP addresses in your config — you use service names.

Because all four services share the `backend` network and Postgres/Redis publish **no host ports**, the data stores are reachable *only* by `app-1` and `app-2`. An attacker on the internet cannot reach Postgres because it isn't on any public interface, isn't in the firewall, and only exists on a private Docker network. That is defense in depth: network isolation, no exposed ports, and the firewall, all reinforcing each other.

### 12.3 depends_on, health checks, and startup order **[I/A]**

A subtle production bug: if your Go app starts *before* Postgres is ready to accept connections, it crashes on its first query. `depends_on` with **`condition: service_healthy`** fixes this — Compose waits until Postgres's **health check** (`pg_isready`) passes before starting the apps. The health check is the difference between "the container process started" (useless — Postgres takes a few seconds to be *ready*) and "the service is actually accepting connections" (what you need).

Health checks do triple duty: they gate startup order (`depends_on`), they let Docker restart a container that has gone unhealthy, and — crucially — they let **Caddy** (§15) route traffic only to healthy app instances. Your Go app should expose a lightweight health endpoint (e.g. `GET /healthz` that pings the DB and returns 200) and/or a `healthcheck` subcommand; the [Gin guide](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) shows the handler. Keep it *cheap* — it runs every 10 seconds forever.

### 12.4 Binding to localhost — the exposure gotcha **[I/A]**

Suppose during debugging you *do* want to reach Postgres from the host (to run `psql`). The safe way is to publish it **only to localhost**, never to all interfaces:

```yaml
    # DEBUG ONLY — publish Postgres to the HOST's loopback, not the internet:
    ports:
      - "127.0.0.1:5432:5432"    # ✅ reachable via `psql -h 127.0.0.1` on the box, NOT from outside
      # - "5432:5432"            # ❌ NEVER: this binds 0.0.0.0 → the DB is on the public internet
```

The difference between `"127.0.0.1:5432:5432"` and `"5432:5432"` is the difference between "reachable only from the server itself" and "reachable from the entire internet." The second form is how databases get breached. **In production, publish nothing for Postgres/Redis at all** (as §12.1 does) — reach them via `docker compose exec` when you need a shell. Recall from §6.4 that Docker's port publishing can *bypass ufw*, which is exactly why the localhost-bind (or no-publish) rule is non-negotiable.

### 12.5 Restart policies — surviving crashes and reboots **[I/A]**

The `restart: unless-stopped` on every service is what makes the stack self-healing:

| Policy | Behavior |
|---|---|
| `no` | Never restart (the default). A crash = down until you intervene. |
| `on-failure` | Restart only if it exits non-zero; optionally limit attempts. |
| `always` | Always restart, even if *you* stopped it (comes back on daemon restart). |
| `unless-stopped` | Restart on crash and on reboot, **but** stay stopped if you deliberately stopped it. |

**`unless-stopped` is the production default.** It means: if a container crashes, Docker restarts it; if the server reboots (after a kernel patch, §8), Docker (which is `enabled`, §9.5) starts and brings the whole stack back automatically — no human needed. But if you *intentionally* `docker compose stop` a service to work on it, it won't surprise you by restarting. Combined with health checks, your stack recovers from crashes and reboots on its own, which is a big chunk of "availability" from the §1.4 checklist.

---

## 13. PostgreSQL in Production

### 13.1 Running Postgres as a container, safely **[I/A]**

The Compose service in §12.1 already runs Postgres correctly: pinned image, a **named volume** for durable data, a password from a **secret file** (not inline), a health check, no exposed port, and a custom config file. The three things that most distinguish a production Postgres from a toy one are **persistence** (the volume — lose it and you lose everything), **tuning** (the config below), and **backups** (§21 — untested backups don't count).

### 13.2 Tuning postgresql.conf **[I/A]**

Postgres ships with conservative defaults meant to start anywhere; production needs it tuned to the RAM you gave the box. Create `postgres/postgresql.conf` next to your compose file:

```ini
# postgres/postgresql.conf  — tuned for a box with ~8GB RAM dedicated-ish to PG
listen_addresses = '*'            # listen on the container's interfaces (still private to Docker)
max_connections = 100             # cap connections; use a POOLER (pgbouncer) past this (§22)

shared_buffers = 2GB              # ~25% of RAM — PG's own cache of pages
effective_cache_size = 6GB        # ~75% of RAM — hint: how much the OS+PG cache together
work_mem = 16MB                   # per-sort/hash memory; careful, it's PER operation
maintenance_work_mem = 512MB      # for VACUUM, index builds
wal_compression = on              # compress the write-ahead log
checkpoint_completion_target = 0.9
random_page_cost = 1.1            # SSD/NVMe: random reads are cheap (default 4 assumes spinning disk)
effective_io_concurrency = 200    # SSD can do many concurrent I/Os

# Logging — see slow queries (the #1 performance debugging tool)
log_min_duration_statement = 500  # log any query slower than 500ms
log_checkpoints = on
log_connections = on
log_lock_waits = on
```

The two biggest wins are `shared_buffers` (~25% of RAM) and `random_page_cost = 1.1` (tells the planner your disk is SSD, so it uses indexes more aggressively). `log_min_duration_statement = 500` surfaces slow queries so you can add indexes before they become an outage. Don't over-tune blindly — these defaults-plus are excellent for most workloads; the [PostgreSQL guide](POSTGRESQL_GUIDE.md) goes deeper on `EXPLAIN ANALYZE` and indexing.

### 13.3 Connecting, migrating, and a shell **[I/A]**

```bash
# Open a psql shell INSIDE the postgres container (no exposed port needed):
docker compose exec postgres psql -U appuser -d appdb
#   \dt   list tables    \d+ table   describe    \l  list DBs    \q  quit

# Run schema migrations. Your Go app should NOT auto-migrate in prod; run migrations
# as a deliberate deploy step (see the goose/sqlc guides). Example with goose:
docker compose run --rm app-1 /app migrate up    # a one-off container that runs & exits

# Quick sanity: is PG healthy and how big is the DB?
docker compose exec postgres pg_isready -U appuser -d appdb
docker compose exec postgres psql -U appuser -d appdb -c "SELECT pg_size_pretty(pg_database_size('appdb'));"
```

Run **migrations as an explicit, gated step** at deploy time (a `docker compose run --rm` one-off container), never automatically on app boot — auto-migrate-on-boot means every instance races to alter the schema simultaneously, and a bad migration ships coupled to a code deploy with no review. The [goose](GO_GOOSE_MIGRATIONS_GUIDE.md) and [sqlc](GO_SQLC_GOOSE_GUIDE.md) guides cover the migration discipline in full.

### 13.4 Connection pooling **[I/A]**

Each Postgres connection is a backend process costing memory, so `max_connections = 100` is a real ceiling. With two Go instances each holding a pool, plus migrations and admin sessions, you can hit it. Two defenses: size your Go pools sanely (the [pgx guide](GO_PGX_GUIDE.md) §22 has the math — total app connections across all instances must stay under `max_connections`), and at higher scale put **PgBouncer** (a lightweight connection pooler) in front of Postgres so thousands of app connections multiplex onto a few dozen real ones (§22). For 100k users this matters; below that, right-sized pgx pools are enough.

---

## 14. Redis in Production

### 14.1 What you use Redis for **[I/A]**

**Redis** is an in-memory data store — extremely fast, used here for the things Postgres shouldn't do: **session storage**, **caching** (avoid re-querying Postgres for hot data), **rate limiting** (token-bucket counters, per the JWT/Argon2 guide §23), **pub/sub** (a backplane so events reach users connected to *either* Go instance — the SSE/WebSocket guides), and **short-lived tokens** (the SSE-ticket pattern). Because both Go instances share one Redis, it's the shared brain that makes the stateless-app model (§1.3) work — session and rate-limit state lives in Redis, so any instance can serve any request.

### 14.2 A production redis.conf **[I/A]**

Create `redis/redis.conf` next to the compose file:

```ini
# redis/redis.conf  (on the SERVER)
bind 0.0.0.0                     # bind inside the container (still private to the Docker net)
protected-mode no               # safe ONLY because it's not exposed; the network is private
port 6379

# ── Security: require a password even on the private network (defense in depth) ──
requirepass ${REDIS_PASSWORD}   # set via env/secret; apps must AUTH

# ── Persistence: choose based on how much you'd hate to lose cached/session data ──
appendonly yes                  # AOF: log every write → durable, minimal data loss on crash
appendfsync everysec            # fsync once/sec — good durability/perf balance
save 900 1                      # also RDB snapshot: if ≥1 key changed in 900s
save 300 10

# ── Memory: never let Redis OOM-kill the box ──
maxmemory 512mb                 # hard cap on Redis's memory
maxmemory-policy allkeys-lru    # when full, evict least-recently-used keys (good for a cache)
```

The two production-critical settings: **`maxmemory` + `maxmemory-policy`** prevent Redis from consuming all RAM and getting the whole server OOM-killed (without them, a growing cache can take down the box), and **persistence** (`appendonly yes`) means a Redis restart doesn't wipe every session and log everyone out. Choose `allkeys-lru` if Redis is mostly a cache; if it holds sessions you can't lose, use `noeviction` and size `maxmemory` generously (and alert on it).

### 14.3 Operating Redis **[I/A]**

```bash
# A Redis shell inside the container (AUTH with the password):
docker compose exec redis redis-cli
#   AUTH yourpassword
#   PING            → PONG
#   INFO memory     → used_memory, maxmemory, eviction stats
#   DBSIZE          → number of keys
#   KEYS session:*  → ⚠️ NEVER in prod: KEYS scans everything and blocks Redis. Use SCAN.
#   MONITOR         → ⚠️ firehose of every command; debugging only, never leave it running

docker compose exec redis redis-cli INFO stats | grep keyspace   # hit/miss ratio
docker compose exec redis redis-cli --bigkeys                    # find memory-hogging keys
```

Two commands to *never* run casually in production: **`KEYS *`** (it scans the entire keyspace synchronously, blocking every other client — use `SCAN` for iteration) and **`MONITOR`** (it streams every command, crushing performance). The [Redis guide](REDIS_GUIDE.md) covers data types, pub/sub, and clustering in depth; here the point is running it *safely* as part of the stack.

---

## 15. Caddy — The Reverse Proxy From Zero to Hero

### 15.1 What a reverse proxy is, and why Caddy **[B]**

A **reverse proxy** sits in front of your application servers and is the single public entry point for all traffic. Every request from the internet hits the proxy first; the proxy then forwards it to one of your backend app instances, gets the response, and relays it back to the user. The word "reverse" distinguishes it from a *forward* proxy (which sits in front of *clients*): a reverse proxy fronts *servers*. It gives you, in one place: **TLS termination** (it handles HTTPS certificates so your app doesn't have to), **load balancing** (spreading requests across your two Go instances), **routing** (send `api.company.com` to the app, `company.com` to the marketing site), **security** (rate limiting, header hardening, hiding your backend), and **compression/caching**. It is the front door from §1.2, and everything public goes through it.

**Caddy** is a modern web server and reverse proxy whose headline feature is **automatic HTTPS**: point it at a domain and it obtains, installs, and renews a real TLS certificate from Let's Encrypt (or ZeroSSL) *automatically*, with zero configuration and zero cron jobs. Where Nginx needs a certbot setup, renewal timers, and a dozen lines of TLS config, Caddy needs one line — the domain name. Its config file (the **Caddyfile**) is famously simple and human-readable. For a modern backend, Caddy gets you to production HTTPS faster and with fewer moving parts than any alternative; §15.10 compares it to Nginx honestly.

### 15.2 Installing Caddy **[B]**

You can run Caddy either natively (systemd service) or as a container. For this stack we run it **as a container alongside the app** (so the whole edge is in Compose too), but installing it natively is also common and simple:

```bash
# Native install (Debian/Ubuntu), for reference:
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
caddy version                      # verify
systemctl status caddy             # it runs as a systemd service, serving /etc/caddy/Caddyfile
```

We'll use the **container** approach in §15.8, which fits the Compose stack. Either way, the Caddyfile syntax below is identical.

### 15.3 The Caddyfile — your first HTTPS site **[B]**

The **Caddyfile** is Caddy's configuration. Here is a complete, production HTTPS reverse proxy — yes, this is the *whole thing*:

```caddy
# Caddyfile
api.company.com {
	reverse_proxy app-1:8080 app-2:8080
}
```

That's it. Those three lines: serve `api.company.com` over **HTTPS with an auto-provisioned, auto-renewing certificate**, redirect HTTP→HTTPS automatically, and reverse-proxy every request to your two Go instances, **load-balanced** between them. Caddy sees two upstreams and round-robins across them out of the box. Compare that to the equivalent Nginx config (upstream block, server block, listen 443 ssl, certificate paths, a certbot cron, a redirect server block) — Caddy's defaults are the secure production defaults, so you write almost nothing.

The structure of a Caddyfile is: a **site address** (`api.company.com`) followed by a block `{ }` of **directives** (`reverse_proxy`, and others below) that configure how that site behaves. One file can hold many sites.

### 15.4 The reverse_proxy directive in depth **[B/I]**

`reverse_proxy` is the directive you'll use most. It has rich options for load balancing, health checks, and header handling:

```caddy
api.company.com {
	reverse_proxy app-1:8080 app-2:8080 {
		# ── Load balancing: how to pick which upstream gets the request ──
		lb_policy       round_robin      # default; also: least_conn, ip_hash, random, first
		lb_try_duration 5s               # keep retrying other upstreams for up to 5s on failure

		# ── Active health checks: Caddy polls each upstream and routes AROUND dead ones ──
		health_uri      /healthz         # your app's health endpoint (§12.3)
		health_interval 10s
		health_timeout  3s
		health_status   200              # a healthy upstream returns 200

		# ── Passive health checks: react to real request failures ──
		fail_duration   30s              # after a failure, stop sending to that upstream for 30s

		# ── Headers passed to the backend so your app sees the REAL client, not Caddy ──
		header_up X-Real-IP {remote_host}
		header_up X-Forwarded-For {remote_host}
		header_up X-Forwarded-Proto {scheme}
	}
}
```

Two things this buys you. **Load balancing** across `app-1` and `app-2` — `round_robin` alternates, `least_conn` sends to whichever instance has fewer in-flight requests (better for uneven request costs), `ip_hash` pins a client to one instance (useful if you have any per-instance state, though ideally you don't). **Health-aware routing** — with `health_uri`, Caddy actively checks each instance and *automatically stops routing to a dead or unhealthy one*, sending all traffic to the survivor until it recovers. That, combined with the two instances, is your zero-downtime, self-healing front end. The `header_up X-Forwarded-*` lines matter because behind a proxy your app would otherwise see *Caddy's* IP as the client; these headers pass the real client IP through (your Go app must be configured to trust them — the Gin guide covers `SetTrustedProxies`).

### 15.5 Subdomains and multiple sites **[B/I]**

A real company serves several things on one server: the API, an admin panel, maybe the marketing site and a status page — each on its own **subdomain**. Caddy handles them all in one Caddyfile, each getting its own automatic certificate:

```caddy
# The API → the Go backend
api.company.com {
	reverse_proxy app-1:8080 app-2:8080
}

# An admin dashboard → a different container
admin.company.com {
	reverse_proxy admin-ui:3000
}

# The marketing site → static files served directly by Caddy
company.com, www.company.com {
	root * /srv/www
	file_server
	encode gzip zstd            # compress responses
}

# Redirect the apex/www to a canonical host if you prefer
# (Caddy auto-redirects http→https already; this is host canonicalization)

# A wildcard: catch ANY subdomain (needs DNS-01 challenge — §17.4)
*.company.com {
	reverse_proxy app-1:8080 app-2:8080
}
```

Each site address gets its own TLS certificate automatically. Subdomains are just separate site blocks — `api.`, `admin.`, `status.` — and Caddy provisions a cert for each on first request. The marketing site shows a second superpower: `file_server` serves static files *directly* (no backend needed), so Caddy is both your reverse proxy *and* your static file host. §16 covers pointing these subdomains' DNS at your server; the wildcard `*.company.com` is powerful (one cert for all subdomains) but requires the DNS-01 ACME challenge (§17.4) because you can't HTTP-validate a wildcard.

### 15.6 Security headers, compression, and matchers **[I]**

Caddy makes hardening concise. **Matchers** (`@name`) let you apply directives to specific paths:

```caddy
api.company.com {
	# Compress responses (Caddy negotiates the best the client supports).
	encode zstd gzip

	# Security headers on every response (defense against XSS/clickjacking/sniffing).
	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
		X-Content-Type-Options    "nosniff"
		X-Frame-Options           "DENY"
		Referrer-Policy           "strict-origin-when-cross-origin"
		-Server                   # REMOVE the Server header (don't advertise what you run)
	}

	# A named matcher: requests whose path starts with /api/
	@api path /api/*
	reverse_proxy @api app-1:8080 app-2:8080

	# Serve static assets directly, cache them hard.
	@static path /static/*
	handle @static {
		root * /srv/assets
		header Cache-Control "public, max-age=31536000, immutable"
		file_server
	}
}
```

`encode zstd gzip` turns on compression (smaller responses, faster loads). The `header { }` block sets the standard security headers on every response — **HSTS** (force HTTPS in browsers), **nosniff**, **X-Frame-Options** (anti-clickjacking) — the same headers the JWT/Argon2 guide's security section prescribes, applied at the edge for *every* app behind Caddy. **Matchers** (`@api`, `@static`) route by path, so one site can serve the API from the backend and static files from disk. The `-Server` line strips the `Server` header so you're not telling attackers what you run.

### 15.7 Rate limiting and request hardening **[I/A]**

Rate limiting at the proxy protects your app from abuse and DoS before requests ever reach it. Caddy's rate limiting is provided by a module (bundle it into a custom Caddy build, §15.9):

```caddy
api.company.com {
	# Requires the caddy-ratelimit module (built in via xcaddy — §15.9).
	rate_limit {
		zone api_general {
			key    {remote_host}     # limit per client IP
			events 100               # ...to 100 requests...
			window 1m                # ...per minute
		}
	}

	# Stricter limit on the login endpoint (brute-force defense).
	@login path /api/login
	rate_limit @login {
		zone login {
			key    {remote_host}
			events 5
			window 1m
		}
	}

	request_body {
		max_size 10MB            # reject oversized uploads at the edge (DoS defense)
	}

	reverse_proxy app-1:8080 app-2:8080
}
```

Edge rate limiting is the first line of defense: an attacker hammering `/api/login` is throttled by Caddy before your Go app (or Postgres) sees the load, complementing the app-level Redis rate limiter. `request_body { max_size }` rejects giant payloads at the proxy so they never consume app memory. Together with §7's fail2ban and §6's firewall, you have layered abuse defense at the network, edge, and app levels.

### 15.8 Running Caddy in the Compose stack **[I/A]**

Now add Caddy to the Compose file as the public edge — the only service with published ports:

```yaml
# Add to compose.yaml (on the SERVER)
  caddy:
    image: caddy:2                 # or a custom build with plugins (§15.9)
    restart: unless-stopped
    ports:
      - "80:80"                    # HTTP (redirects to HTTPS + serves ACME challenges)
      - "443:443"                  # HTTPS
      - "443:443/udp"              # HTTP/3 over QUIC
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro   # your config (read-only)
      - caddy_data:/data           # PERSIST certificates! (lose this = re-issue certs, rate limits)
      - caddy_config:/config
    networks: [backend]            # same private net → it can reach app-1/app-2 by name
    depends_on: [app-1, app-2]

volumes:
  caddy_data:                      # add to the volumes: section
  caddy_config:
```

The `caddy_data` volume is **critical**: it stores the issued TLS certificates and ACME account. If you lose it, Caddy re-requests certificates on next start — and Let's Encrypt has rate limits (e.g. ~5 certs per domain per week), so repeatedly losing this volume can get you *temporarily blocked from issuing certs*. Because Caddy is on the same `backend` network, `reverse_proxy app-1:8080` resolves by service name exactly like the app→DB connection (§12.2). Caddy is now the only service exposing ports 80/443 to the host — everything else stays private. Apply with `docker compose up -d`; on first request to `api.company.com`, Caddy provisions the certificate and you're live on HTTPS.

### 15.9 The admin API, JSON config, and custom builds **[A]**

Under the hood, the Caddyfile is compiled to Caddy's native **JSON config**, and Caddy runs an **admin API** on `localhost:2019` for live, zero-downtime reconfiguration:

```bash
caddy validate --config /etc/caddy/Caddyfile   # check the config is valid BEFORE reloading
caddy reload   --config /etc/caddy/Caddyfile   # apply new config with ZERO dropped connections
caddy fmt --overwrite /etc/caddy/Caddyfile     # auto-format the Caddyfile
# In Docker:
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
curl localhost:2019/config/ | jq               # see the live JSON config (from the host)
```

`caddy reload` applies config changes **without dropping a single connection** — a graceful, atomic swap — which is why editing the Caddyfile and reloading is safe on a live server (always `caddy validate` first). For features not in the standard binary (rate limiting §15.7, DNS-provider plugins for wildcard certs §17.4, etc.), you build a custom Caddy with **`xcaddy`**:

```dockerfile
# caddy/Dockerfile — a custom Caddy image with the plugins you need
FROM caddy:2-builder AS build
RUN xcaddy build \
	--with github.com/mholt/caddy-ratelimit \
	--with github.com/caddy-dns/cloudflare      # DNS-01 provider for wildcard certs

FROM caddy:2
COPY --from=build /usr/bin/caddy /usr/bin/caddy
```

Point the Compose `caddy` service at `build: ./caddy` instead of `image: caddy:2` to use it. This is how you extend Caddy while keeping the tiny, secure base.

### 15.10 Caddy vs Nginx — an honest comparison **[I/A]**

| | **Caddy** | **Nginx** |
|---|---|---|
| Automatic HTTPS | Built-in, zero-config, auto-renew | Manual (certbot + renewal timer) |
| Config | Caddyfile — simple, secure defaults | Powerful but verbose; easy to misconfigure TLS |
| HTTP/3 | On by default | Available, more setup |
| Performance | Excellent for ~all workloads | Marginally faster at extreme static-file loads |
| Ecosystem/maturity | Younger, growing | Vast; more StackOverflow answers, modules |
| Plugins | `xcaddy` custom builds | Dynamic modules |

Choose **Caddy** (as this guide does) when you want production HTTPS with minimal config and fewer moving parts — which is most backends. Choose **Nginx** (see the [Nginx guide](NGINX_GUIDE.md)) when you have deep existing Nginx expertise, need a specific Nginx module, or run at a scale where its maturity and tuning knobs matter. For a company shipping a Go API to 100k users, Caddy's simplicity is a feature, not a compromise — less config is less to misconfigure, and misconfiguration is where security incidents come from.

---

## 16. DNS, Domains and Subdomains

### 16.1 What DNS is and how a request finds your server **[B]**

Users type `api.company.com`, not `203.0.113.10`. **DNS** (the Domain Name System) is the internet's phone book that translates human names into IP addresses. When a browser needs `api.company.com`, it asks a chain of DNS servers "what's the IP?" and eventually reaches the **authoritative nameservers** for `company.com` — the ones *you* control — which answer with your VPS's IP. The browser then connects to that IP. Setting up DNS is how you make your server reachable by name and enable HTTPS (certificates are issued for names, not IPs).

You get a domain from a **registrar** (Namecheap, Cloudflare Registrar, Porkbun, Google Domains' successors). DNS records are then managed either at the registrar or, commonly, at **Cloudflare** (free, fast, adds DDoS protection and a CDN — §22). The records live in a **zone** for your domain.

### 16.2 The DNS records you need **[B]**

DNS records are managed in your DNS provider's **web dashboard** (this is *not* a server command — it's the one big config that lives off the box). The record types that matter:

| Type | Purpose | Example |
|---|---|---|
| **A** | Name → IPv4 address | `api.company.com` → `203.0.113.10` |
| **AAAA** | Name → IPv6 address | `api.company.com` → `2001:db8::10` |
| **CNAME** | Name → another name (alias) | `www.company.com` → `company.com` |
| **MX** | Mail servers for the domain | (for email; use a provider) |
| **TXT** | Arbitrary text (SPF, DKIM, domain verification, ACME DNS-01) | `v=spf1 ...` |
| **NS** | Delegates the zone to nameservers | set at the registrar |

For this stack you create, in the dashboard:

```text
Type    Name (host)     Value (points to)        TTL
A       api             203.0.113.10             300     → api.company.com
A       admin           203.0.113.10             300     → admin.company.com
A       @               203.0.113.10             300     → company.com (the apex/root)
CNAME   www             company.com              300     → www.company.com
```

Each **subdomain is just another A record** pointing at the same server IP (or a `CNAME` to another name). `@` (or blank) means the **apex/root** domain itself (`company.com`). `www` is conventionally a CNAME to the apex. The **TTL** (Time To Live, in seconds) is how long resolvers cache the answer — keep it low (300s = 5 min) while setting things up so changes propagate fast, then raise it (3600+) once stable to reduce lookups.

### 16.3 Subdomains, wildcards, and propagation **[B/I]**

A **subdomain** (`api.`, `admin.`, `status.`) is a separate host under your domain, each with its own DNS record and — via Caddy (§15.5) — its own site block and TLS certificate. This is how one VPS serves many logical services: `api.company.com` → the Go backend, `admin.company.com` → an admin UI, `status.company.com` → a status page, all resolving to the same IP, all routed by Caddy based on the hostname.

A **wildcard** record `*.company.com` → your IP makes *every* subdomain resolve to the server (so you don't add a record per subdomain), pairing with Caddy's `*.company.com` site block. Wildcards need the DNS-01 certificate challenge (§17.4).

**DNS propagation:** after you create or change a record, it can take minutes to hours for the change to spread, bounded by the old record's TTL. Verify with:

```bash
dig api.company.com +short          # what IP does it resolve to right now?
dig api.company.com @1.1.1.1        # ask a specific resolver (bypass local cache)
nslookup api.company.com            # alternative lookup tool
```

`dig +short` returning your server's IP means DNS is working; then Caddy can obtain a certificate (which requires the name to already resolve to your box). A common first-deploy mistake is trying to get HTTPS working *before* DNS resolves — Caddy's ACME challenge fails because Let's Encrypt can't reach the name. **DNS first, then HTTPS.**

### 16.4 Cloudflare specifics **[I]**

If you use Cloudflare (recommended for the free CDN/DDoS protection), one setting trips everyone up: the **orange cloud** (proxy) toggle. When a record is "proxied" (orange), traffic flows *through* Cloudflare's network (great — CDN, DDoS shielding, hides your origin IP). When "DNS only" (grey), Cloudflare just answers DNS and traffic goes direct to your server. Two implications: proxied records hide your real IP (good) but mean Caddy sees Cloudflare's IPs as clients (configure trusted proxies and use Cloudflare's real-IP headers), and Caddy's HTTP-01 certificate challenge can conflict with the proxy — so with Cloudflare-proxied wildcards, use the **DNS-01 challenge** via Caddy's Cloudflare plugin (§17.4). For a simple start: set records **DNS-only (grey)** first, get Caddy issuing certs, then enable the proxy once it works.

---

## 17. TLS and Automatic HTTPS

### 17.1 What TLS/HTTPS gives you **[I]**

**TLS** (Transport Layer Security, the successor to SSL) encrypts the connection between the user and your server, so nobody in between (Wi-Fi snoops, ISPs, malicious proxies) can read or tamper with the traffic. **HTTPS** is just HTTP over TLS. In 2026 it is non-negotiable: browsers mark plain HTTP as "Not Secure," many APIs and features (service workers, HTTP/2, geolocation) require HTTPS, and sending credentials over plain HTTP is a breach. Every public endpoint must be HTTPS.

A TLS **certificate** is a file, signed by a trusted **Certificate Authority (CA)**, that proves you control the domain and contains the public key browsers use to establish the encrypted session. Historically you bought these; today **Let's Encrypt** issues them free and automatically, which is what Caddy uses.

### 17.2 How Caddy's automatic HTTPS works **[I]**

When Caddy serves a site with a real domain, it runs the **ACME protocol** to get a certificate automatically:

1. Caddy asks Let's Encrypt for a certificate for `api.company.com`.
2. Let's Encrypt issues a **challenge**: "prove you control this domain."
3. Caddy answers it — by default via **HTTP-01**: it serves a specific token at `http://api.company.com/.well-known/acme-challenge/...`, which Let's Encrypt fetches to confirm you control the domain (this is why **port 80 must be open** in the firewall, §6.2).
4. Let's Encrypt issues the certificate; Caddy installs it and starts serving HTTPS.
5. Caddy **auto-renews** ~30 days before expiry, forever, with no cron job.

This all happens on the first request to a new site, in seconds. The prerequisites you must get right: **DNS resolves** the name to your server (§16), **port 80 and 443 are open** (§6), and Caddy's **`caddy_data` volume persists** so certs and the ACME account survive restarts (§15.8). Get those three right and HTTPS is genuinely automatic.

### 17.3 The three ACME challenge types **[I/A]**

| Challenge | How it proves control | When Caddy uses it |
|---|---|---|
| **HTTP-01** | Serve a token over HTTP on port 80 | Default for single hostnames; needs port 80 reachable |
| **TLS-ALPN-01** | Prove control during the TLS handshake on port 443 | Fallback; needs port 443 |
| **DNS-01** | Create a TXT record in your DNS | Required for **wildcard** certs; works even if ports 80/443 are closed |

For normal subdomains, HTTP-01 (the default) just works. For a **wildcard** certificate (`*.company.com`), you *must* use **DNS-01**, because you can't HTTP-validate a name that doesn't exist yet — you prove control of the whole zone by writing a TXT record instead.

### 17.4 Wildcard certs with DNS-01 **[A]**

DNS-01 needs Caddy to create TXT records in your DNS automatically, which requires a **DNS-provider plugin** and an API token. Build Caddy with the plugin (§15.9) and configure it:

```caddy
# Caddyfile — wildcard cert via Cloudflare DNS-01
*.company.com {
	tls {
		dns cloudflare {env.CF_API_TOKEN}    # Caddy creates/removes the TXT record via the CF API
	}
	reverse_proxy app-1:8080 app-2:8080
}
```

```yaml
# In the caddy service in compose.yaml, provide the scoped API token as a secret/env:
    environment:
      CF_API_TOKEN_FILE: /run/secrets/cf_token   # a Cloudflare token with DNS-edit on this zone ONLY
    secrets: [cf_token]
```

The Cloudflare API token should be **scoped to editing DNS for this one zone** (least privilege — §4) so a leak can't touch anything else. With this, one certificate covers every subdomain, and adding `status.company.com` needs no new cert. This is the advanced setup; most teams start with per-subdomain HTTP-01 (zero plugins) and adopt wildcards only when they have many dynamic subdomains.

### 17.5 Testing and troubleshooting TLS **[I]**

```bash
curl -vI https://api.company.com          # -v shows the TLS handshake + cert; -I headers only
echo | openssl s_client -connect api.company.com:443 -servername api.company.com 2>/dev/null | openssl x509 -noout -dates   # cert validity dates
docker compose logs caddy | grep -i "certificate\|acme\|error"   # Caddy's cert provisioning logs
```

If HTTPS isn't working, the checklist is always: **does DNS resolve to this box?** (`dig +short`), **are ports 80/443 open?** (`sudo ufw status`, and reachable from outside), **what does Caddy's log say?** (it's verbose about ACME failures — usually "DNS doesn't point here yet" or "port 80 unreachable"). Nearly every "Caddy won't get a cert" issue is one of those three, and the log tells you which.

---

## 18. Secrets and Configuration Management

### 18.1 The rule — secrets never live in code or images **[I/A]**

Your app needs secrets: the database password, the JWT signing key, API tokens. The cardinal rule: **secrets never go in your Git repo, your Docker image, or your `compose.yaml`.** A secret committed to Git is compromised forever (it's in the history even after you delete it); a secret baked into an image leaks to anyone who can pull it. Secrets are injected at *runtime*, from a source that isn't in version control. The related rule (**12-factor config**) is that *configuration* — anything that differs between environments — comes from the environment, not hardcoded, so the same image runs in dev, staging, and prod with different config.

### 18.2 Docker secrets vs environment variables **[I/A]**

There are two common mechanisms, in increasing order of safety:

- **Environment variables** (`environment:` in Compose) — simple, but env vars are visible in `docker inspect`, can leak into logs and child processes, and end up in the shell history if set inline. Fine for *non-secret* config (ports, feature flags, `LOG_LEVEL`).
- **Docker secrets** (`secrets:` in Compose) — the value is mounted as a **file** at `/run/secrets/<name>` inside the container, readable only by the service. It doesn't show in `docker inspect`'s env, doesn't leak to child processes' environment, and the file lives on a tmpfs. This is what §12.1 uses (`POSTGRES_PASSWORD_FILE`, `JWT_SECRET_FILE`). Your app reads the file at startup.

```bash
# On the SERVER: create the secret files (outside Git), lock them down.
mkdir -p /home/deploy/app/secrets
openssl rand -base64 32 > /home/deploy/app/secrets/pg_password.txt    # a strong random password
openssl rand -base64 48 > /home/deploy/app/secrets/jwt_secret.txt     # a strong signing key
chmod 600 /home/deploy/app/secrets/*.txt                              # owner-only (§9.3)
# Add 'secrets/' to .gitignore so it can NEVER be committed.
```

Your Go app reads `os.Getenv("PGPASSWORD_FILE")`, then reads that file's contents — the `*_FILE` convention (supported by the Postgres image and easy to add in your app) that keeps the actual secret out of the environment. `openssl rand -base64 N` generates cryptographically strong random secrets; never hand-pick passwords for machines.

### 18.3 Non-secret config with an .env file **[I/A]**

Compose auto-loads a file named **`.env`** in the same directory for variable substitution (the `${APP_VERSION:-latest}` in §12.1). Use it for non-secret, per-environment settings:

```bash
# /home/deploy/app/.env  (on the SERVER — gitignored; a committed .env.example documents the keys)
APP_VERSION=1.4.2
LOG_LEVEL=info
CADDY_DOMAIN=api.company.com
```

Commit a **`.env.example`** with the *keys* and dummy values so anyone can see what config exists, and keep the real `.env` gitignored. For larger setups, dedicated secret managers (HashiCorp Vault, cloud KMS/Secrets Manager, Doppler, SOPS-encrypted files in Git) replace hand-managed files — the [JWT/Argon2 guide](GO_JWT_ARGON2_GUIDE.md) §22.5 covers KMS/envelope encryption. For a single-VPS deployment, gitignored secret files with `chmod 600` + Docker secrets is a perfectly respectable baseline.

---

## 19. Zero-Downtime Deploys and CI/CD

### 19.1 What "deploy" means here **[A]**

A **deploy** ships a new version of your Go app to production. Done naively (`docker compose up -d --build`) it recreates both instances at once, causing a brief outage while they restart. Done right — **rolling** one instance at a time — there is *zero downtime*: at every moment at least one instance is serving, and Caddy's health checks (§15.4) route around the one being replaced. This is the payoff of running two instances (§1.3).

### 19.2 The build-push-pull model **[A]**

The professional pattern separates *building* the image from *running* it:

1. **CI builds** the image from your Git repo (on a runner, not the prod server) and **pushes** it to a **container registry** (Docker Hub, GitHub Container Registry `ghcr.io`, or a private one) tagged with the version/commit.
2. The **prod server pulls** that exact, tested image and runs it.

This means the prod server never compiles code (no build tools, no source, smaller attack surface) and the image that ran in CI tests is byte-for-byte the image that runs in prod (reproducibility). Pin by immutable tag (the Git SHA) so a deploy is "run image `api:sha-abc123`," fully traceable.

### 19.3 A rolling deploy script **[A]**

```bash
#!/usr/bin/env bash
# deploy.sh — roll out a new version with zero downtime. Run on the SERVER (or via SSH from CI).
set -euo pipefail                      # fail fast: -e exit on error, -u undefined vars, -o pipefail

NEW_VERSION="$1"                       # e.g. sha-abc123, passed by CI
cd /home/deploy/app
export APP_VERSION="$NEW_VERSION"

echo "Pulling $NEW_VERSION..."
docker compose pull app-1 app-2        # fetch the new image (no downtime yet)

# Run migrations ONCE as a gated one-off (never auto-on-boot — §13.3).
docker compose run --rm app-1 /app migrate up

# Roll instance 1: recreate it, wait for it to be HEALTHY before touching instance 2.
echo "Rolling app-1..."
docker compose up -d --no-deps app-1
until [ "$(docker inspect -f '{{.State.Health.Status}}' app-1)" = "healthy" ]; do sleep 2; done

# Now app-1 is on the new version and healthy; roll instance 2 the same way.
echo "Rolling app-2..."
docker compose up -d --no-deps app-2
until [ "$(docker inspect -f '{{.State.Health.Status}}' app-2)" = "healthy" ]; do sleep 2; done

echo "Deployed $NEW_VERSION with zero downtime."
```

The key moves: `--no-deps app-1` recreates *only* that one service (leaving app-2, Postgres, Redis untouched), and the `until [ ... healthy ]` loop **waits for the new instance to pass its health check before rolling the second** — so you never take both down together. Caddy, meanwhile, has been routing all traffic to app-2 while app-1 restarts, then to app-1 while app-2 restarts. `set -euo pipefail` at the top makes the script abort on any error rather than blundering forward — mandatory for deploy scripts. If the new version is broken (health check never passes), the loop hangs on app-1 and app-2 is still serving the old version — a natural safety valve; add a timeout + rollback for full automation.

### 19.4 Automating it with CI/CD **[A]**

Wire the above into GitHub Actions (see the [CI/CD guide](GITHUB_ACTIONS_CICD_GUIDE.md)) so a push to `main` builds, tests, pushes the image, and triggers the deploy over SSH:

```yaml
# .github/workflows/deploy.yml (sketch — the CI/CD guide has the full version)
name: deploy
on: { push: { branches: [main] } }
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build & push image
        run: |
          echo "${{ secrets.REGISTRY_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker build -t ghcr.io/company/api:${{ github.sha }} .
          docker push ghcr.io/company/api:${{ github.sha }}
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: deploy
          key: ${{ secrets.DEPLOY_SSH_KEY }}     # a deploy-only SSH key
          script: /home/deploy/app/deploy.sh sha-${{ github.sha }}
```

Two security notes for CI deploys: use a **dedicated deploy SSH key** (its own key pair, added to `deploy`'s `authorized_keys`, revocable independently), and store all credentials in the CI provider's **encrypted secrets**, never in the workflow file. Now `git push` → tested, built, and rolled out with zero downtime, fully automated and auditable. That is the modern deploy loop.

---

## 20. Observability — Logs, Metrics and Health

### 20.1 The three pillars **[A]**

You cannot operate what you cannot see. **Observability** is your window into the running system, and it has three pillars: **logs** (what happened — discrete events), **metrics** (how much/how many — numbers over time, like requests/sec, CPU, error rate), and **traces** (the path of a single request across services). For a single-VPS backend you don't need a heavy stack; you need enough to answer "is it up?", "is it slow?", and "what broke?" quickly.

### 20.2 Logs — centralize and rotate **[A]**

Each container logs to stdout/stderr, captured by Docker. View and manage them:

```bash
docker compose logs -f --tail 200            # live, all services
docker compose logs -f app-1 | grep -i error # filter one service
docker inspect -f '{{.LogPath}}' app-1       # where Docker stores this container's log file
```

**Log rotation is essential** — unrotated container logs are a top cause of full disks (§9.8). Configure Docker's global logging driver to cap log size in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

That keeps at most 3 × 10 MB per container, then discards the oldest — so logs can never fill the disk. Restart Docker (`sudo systemctl restart docker`) after editing. For a company, ship logs off-box to a log service (Grafana Loki, or a hosted service) so they survive the server and are searchable across instances — but the rotation cap is the non-negotiable first step. Make your **Go app log structured JSON** (`log/slog`) so logs are machine-parseable; the [net/http guide](GO_NET_HTTP_REST_API_GUIDE.md) covers `slog`.

### 20.3 Metrics and health **[A]**

The lightweight, industry-standard approach: your Go app exposes a **Prometheus** metrics endpoint (`/metrics`), a **Prometheus** container scrapes it, and **Grafana** graphs it. Add them to Compose (private, behind Caddy with auth for the dashboards). Even without that, four things give you 80% of the value:

- **Health endpoints** (`/healthz`, §12.3) — Caddy and your uptime monitor hit these.
- **An external uptime monitor** (UptimeRobot, Better Uptime, or a `curl` cron elsewhere) that alerts you when `https://api.company.com/healthz` stops returning 200 — because if the whole box is down, *on-box* monitoring can't tell you.
- **`docker stats`** and **`htop`** for live resource use.
- **Disk alerts** — a simple cron that emails/pages when `df` shows >80% used. This one check prevents the most common outage.

### 20.4 Alerting — know before your users do **[A]**

The goal is that *you* find out about problems before your users tweet about them. At minimum, alert on: the health endpoint failing (external monitor), disk >80%, memory pressure/OOM kills, the certificate nearing expiry (Caddy auto-renews, but alert if it somehow fails), and error-rate spikes in logs. Route alerts to somewhere you'll actually see them (email + a phone push via the monitor's app, or Slack/PagerDuty for a team). An alert you ignore is worse than no alert; tune them so every page is real.

---

## 21. Backups and Disaster Recovery

### 21.1 The only backup rule that matters **[A]**

**A backup you have never restored is not a backup — it's a hope.** The most common and most devastating production failure is discovering, *during* a disaster, that your backups were empty, corrupt, or un-restorable. So the rule is: back up automatically, store copies **off the server** (a server-local backup dies with the server), and **periodically test-restore** to prove it works. Everything else is detail.

### 21.2 Backing up PostgreSQL **[A]**

The right tool is `pg_dump` (logical backup — a portable SQL/archive file) for most cases, plus the provider's disk snapshots as a coarse safety net:

```bash
#!/usr/bin/env bash
# backup-db.sh — run via cron on the SERVER. Dumps PG and ships the file off-box.
set -euo pipefail
STAMP=$(date +%F_%H%M%S)
DEST="/home/deploy/backups"
mkdir -p "$DEST"

# -Fc = custom compressed format (smaller, restorable selectively with pg_restore).
docker compose exec -T postgres pg_dump -U appuser -Fc appdb > "$DEST/appdb_$STAMP.dump"

# Ship it OFF the server — to object storage (S3/B2/Spaces). rclone is the swiss-army tool.
rclone copy "$DEST/appdb_$STAMP.dump" remote:company-backups/postgres/

# Retention: delete local dumps older than 7 days (object storage keeps the long tail).
find "$DEST" -name "appdb_*.dump" -mtime +7 -delete
```

```bash
# Schedule it: `crontab -e` as deploy, then add —
# Every day at 03:30, run the backup and log output.
30 3 * * * /home/deploy/app/backup-db.sh >> /home/deploy/backups/backup.log 2>&1
```

`pg_dump -Fc` produces a compressed, selectively-restorable archive. **`rclone copy` to object storage** is the critical step — a backup sitting on the same disk as the database is worthless if the disk (or server) dies. The cron runs it nightly; retention keeps disk usage bounded. The **`-T`** flag on `docker compose exec` disables pseudo-TTY allocation, which is required when redirecting output to a file in a script.

### 21.3 Restoring — and testing the restore **[A]**

```bash
# Restore into a fresh database (test this on a STAGING box regularly!).
docker compose exec -T postgres createdb -U appuser appdb_restore
docker compose exec -T postgres pg_restore -U appuser -d appdb_restore < appdb_20260119.dump
# Verify row counts, spot-check data, THEN (if it's a real recovery) swap it in.
```

Put a **quarterly calendar reminder to test-restore** a backup into a scratch database and confirm the data is intact and complete. This is the step everyone skips and everyone regrets. Redis, being mostly cache/session, matters less — its AOF (§14.2) recovers it on restart, and losing a cache is survivable; still back up the `redisdata` volume if it holds anything you'd miss. Also enable your **provider's automated snapshots** (whole-disk, point-in-time) as a coarse complement — they capture the entire server, not just the DB, and are a one-click rebuild for "the whole VPS is gone."

### 21.4 Disaster recovery — the rebuild drill **[A]**

Because the entire server is defined by files in Git (compose.yaml, Caddyfile, configs) plus secrets and the DB dump, rebuilding from total loss is a known procedure, not a panic: provision a fresh VPS (§2), run the hardening (§4–§8), install Docker (§10.2), `git clone` your infra repo, restore the secret files and the latest DB dump, `docker compose up -d`, point DNS at the new IP (§16). Write this down as a runbook and, ideally, *rehearse* it once — the confidence that you can rebuild in 30 minutes is what "disaster recovery" actually means. This is also why §1.4 lists **reproducible** as a core property: a server you can only build by hand, from memory, is one you cannot recover.

---

## 22. Scaling to 100k Users

### 22.1 Scale up before you scale out **[A]**

The good news: **100,000 users is not that many** for a well-built Go backend, and you'll get there on modest hardware if the fundamentals are right. Go is fast and concurrent; a single instance handles thousands of requests per second. So the first, cheapest scaling move is **vertical** — resize the VPS to more CPU/RAM (§2.1), a five-minute operation. Do this until one box is genuinely the bottleneck before adding the complexity of multiple machines. Premature horizontal scaling is how teams drown in infrastructure they don't need.

The second move is to **remove the obvious bottlenecks**, which are almost always the database and repeated work, not the app:

- **Right-size Postgres** and add **PgBouncer** (§13.4) so connection count stops being a ceiling.
- **Cache aggressively in Redis** (§14.1): the response you don't compute is infinitely fast. Cache hot queries, session lookups, computed pages.
- **Add database indexes** for your slow queries (surfaced by `log_min_duration_statement`, §13.2) — one missing index is the difference between 5ms and 5s.
- Put a **CDN** (Cloudflare, §16.4) in front for static assets and cacheable responses, offloading a huge fraction of traffic before it reaches your VPS at all.

### 22.2 Scaling out — more instances and more machines **[A]**

When one box isn't enough, scale **horizontally**. Because your app is **stateless** (§1.3), this is straightforward:

- **More app instances on the same box:** add `app-3`, `app-4` to Compose and Caddy's `reverse_proxy` list. Caddy load-balances across all of them. Limited by the box's CPU/RAM.
- **A separate database server:** move Postgres to its own VPS (databases love dedicated RAM and I/O). The app instances connect over the **private network** (§2.2). This is usually the *first* split, because the DB is the component that most wants its own machine.
- **Multiple app servers behind a load balancer:** run the app tier on several VPSes; put a load balancer (the provider's managed LB, or Caddy on a dedicated edge box) in front. Redis (shared session/cache) and Postgres (shared data) remain single shared services all app servers reach. This is where the stateless-app design pays off completely: any app server on any machine can handle any request, because all state is in the shared Postgres and Redis.
- **Read replicas:** for read-heavy loads, add Postgres **read replicas** and send read queries to them, writes to the primary (the [pgx](GO_PGX_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md) guides cover this).

### 22.3 When to reach for orchestration **[A]**

You may have heard "use Kubernetes." For most companies at 100k users, **you don't need it** — Docker Compose on a few well-run VPSes is simpler, cheaper, and easier to reason about, and this guide's architecture scales a long way. Reach for Kubernetes (or Nomad, or a managed container platform) when you genuinely have *many* services, need auto-scaling to fluctuating load across a fleet, or have a platform team to run it. Adopting it early trades a problem you have (shipping features) for a problem you don't (operating a cluster). Scale the boring way — bigger boxes, then more boxes, a CDN, caching, read replicas — until the boring way genuinely runs out, which is much later than the hype suggests.

### 22.4 The scaling ladder, summarized **[A]**

| Stage | Move | Why |
|---|---|---|
| 1 | Tune the app + DB, add indexes, cache in Redis | Cheapest wins; most "scale" problems are unindexed queries. |
| 2 | Vertically resize the VPS | Five minutes; buys a lot of headroom. |
| 3 | Add a CDN in front | Offloads static + cacheable traffic entirely. |
| 4 | Split Postgres onto its own server | The DB wants dedicated RAM/IO first. |
| 5 | Add app instances / app servers behind an LB | Stateless app = linear horizontal scaling. |
| 6 | Read replicas, PgBouncer, sharding | For read-heavy or very large datasets. |
| 7 | Orchestration (k8s) | Only when a fleet + a platform team justify it. |

Work down this ladder in order. Each rung is cheaper and simpler than the next, and most products never need the bottom rungs.

---

## 23. The Production Security Checklist

Everything security-related from this guide, in one place — the list to run down before you call a server production-ready. Each maps to a section for the how.

| # | Control | Section |
|---|---|---|
| 1 | SSH: key-only auth, no passwords | §5.2 |
| 2 | SSH: root login disabled | §5.2 |
| 3 | SSH: user whitelist (`AllowUsers`), moved port | §5.3 |
| 4 | A non-root `deploy` user; root not used for daily work | §4 |
| 5 | Firewall default-deny; only 22/80/443 open | §6.2 |
| 6 | fail2ban banning brute-force IPs | §7 |
| 7 | Automatic security updates enabled | §8 |
| 8 | Databases (PG/Redis) **not** exposed — no host ports | §12.1, §12.4 |
| 9 | Redis requires a password even on the private net | §14.2 |
| 10 | Secrets in files (`chmod 600`) / Docker secrets, never in Git or images | §18 |
| 11 | Containers run as **non-root** (`USER nonroot`) | §10.4 |
| 12 | Image tags pinned; minimal (distroless) base images | §10.3, §10.4 |
| 13 | HTTPS everywhere (Caddy auto-TLS); HSTS + security headers | §15.6, §17 |
| 14 | Edge rate limiting + request-size limits | §15.7 |
| 15 | Resource limits on containers (no single-service starvation) | §12.1 |
| 16 | Real client IP passed & trusted correctly behind the proxy | §15.4 |
| 17 | Off-site, test-restored backups | §21 |
| 18 | Log rotation configured (no disk-fill) | §20.2 |
| 19 | External uptime + disk/memory alerts | §20.3 |
| 20 | Docker-bypasses-ufw trap avoided (bind localhost / don't publish) | §6.4, §12.4 |
| 21 | Cloud firewall at the provider edge (defense in depth) | §6.4 |
| 22 | Least-privilege API tokens (e.g. scoped DNS token) | §17.4 |

If every box is checked, you have a genuinely hardened host — the same posture a professional infrastructure team ships. Security is layered on purpose: no single control is perfect, but an attacker must defeat *all* of them, and each is cheap to add.

---

## 24. The Complete Project Layout

### 24.1 Everything on disk, mapped **[I/A]**

Here is the full file/folder structure of the production deployment — the "infra repo" you keep in Git (minus the gitignored secrets) and check out onto the server at `/home/deploy/app/`. Every file has been built in this guide; the comment says which section.

```text
/home/deploy/app/                    # the deployment root (checked out from your infra repo)
├── compose.yaml                     # §12.1  the whole stack: app-1, app-2, postgres, redis, caddy
├── .env                             # §18.3  non-secret config (APP_VERSION, LOG_LEVEL) — gitignored
├── .env.example                     # §18.3  documents the .env keys — committed
├── deploy.sh                        # §19.3  the zero-downtime rolling-deploy script
├── backup-db.sh                     # §21.2  nightly pg_dump → object storage (run via cron)
│
├── caddy/
│   ├── Caddyfile                    # §15    reverse proxy: sites, TLS, LB, headers, rate limits
│   └── Dockerfile                   # §15.9  (optional) custom Caddy build with plugins (xcaddy)
│
├── postgres/
│   └── postgresql.conf              # §13.2  tuned Postgres config (shared_buffers, logging, SSD)
│
├── redis/
│   └── redis.conf                   # §14.2  maxmemory, eviction, AOF persistence, requirepass
│
├── secrets/                         # §18.2  GITIGNORED — never committed
│   ├── pg_password.txt              #         chmod 600, mounted as a Docker secret
│   ├── jwt_secret.txt               #         chmod 600
│   └── cf_token.txt                 #         §17.4 scoped Cloudflare DNS token (if wildcard certs)
│
├── backups/                         # §21    local staging for dumps (gitignored; shipped off-box)
│   └── backup.log
│
└── .gitignore                       # ignores: secrets/, .env, backups/, *.dump

# Elsewhere on the server (not in the app repo):
/etc/ssh/sshd_config                 # §5     hardened SSH daemon config
/etc/fail2ban/jail.local             # §7     fail2ban jails
/etc/docker/daemon.json              # §20.2  Docker log rotation
/etc/apt/apt.conf.d/50unattended-upgrades  # §8  auto security updates
# Docker-managed (don't touch directly):
/var/lib/docker/volumes/             #        pgdata, redisdata, caddy_data live here

# On YOUR LAPTOP (not the server):
~/.ssh/config                        # §3.3   the 'prod' host alias
~/.ssh/id_ed25519(.pub)              # §3.1   your SSH key pair

# The Go application (a SEPARATE repo, built into the image):
api/
├── Dockerfile                       # §10.4  multi-stage build → tiny non-root image
├── .dockerignore                    # §10.4  keep .git/secrets/junk out of the build
├── go.mod / go.sum
└── cmd/server/main.go               # the app (built in the Gin/pgx/JWT guides)
```

### 24.2 The two repos **[I/A]**

Note the deliberate split into **two Git repositories**: the **infra repo** (compose.yaml, Caddyfile, configs, scripts — the *how it runs*) and the **application repo** (the Go code + its Dockerfile — the *what runs*). CI builds the app repo into a versioned image and pushes it to the registry; the infra repo on the server pulls and runs that image. This separation means you can change how the app is *deployed* (add an instance, tweak Caddy) without touching app code, and ship app code without touching infra — and each has its own review and history. The secrets live in *neither* repo; they're placed on the server out-of-band (§18) and referenced by path. This is the reproducible, everything-in-Git posture that makes §21.4's rebuild-from-scratch possible.

---

## 25. Gotchas and Best Practices

The mistakes that actually take production down, distilled. Skim now; return when something breaks.

| Pitfall | Symptom | Fix |
|---|---|---|
| **Locked out of SSH** | Can't reconnect after an SSH/firewall change | Always keep the current session open; test a new one before closing (§5.2). |
| **Docker bypasses ufw** | A "firewalled" DB is reachable from the internet | Don't publish DB ports; bind `127.0.0.1` if you must (§6.4, §12.4). Verify with `ss -tulpn`. |
| **Database with no volume** | All data gone after a container recreate | Always mount a named volume for stateful services (§10.6). |
| **`docker compose down -v`** | Wiped the database | `down` keeps data; `-v` deletes it. Never `-v` in prod (§11.3). |
| **Disk full** | Everything stops; can't even log in cleanly | Rotate logs (§20.2), prune images (§10.7), alert at 80% (§20.3). `df -h` first, always. |
| **`latest` image tag** | A deploy silently pulls a different version | Pin tags (SHA/version) everywhere (§10.3). |
| **Secrets in Git/image** | Credential leak, forever in history | Docker secrets + gitignored files, `chmod 600` (§18). |
| **Auto-migrate on boot** | Instances race to alter schema; bad migration ships with code | Migrate as a gated one-off step (§13.3, §19.3). |
| **No health checks** | Traffic routed to a dead/starting instance | Health checks gate startup + LB routing (§12.3, §15.4). |
| **Lost `caddy_data` volume** | Certs re-issued; hit Let's Encrypt rate limit | Persist `caddy_data` (§15.8). |
| **HTTPS before DNS** | Caddy can't get a cert | DNS must resolve first; open port 80 (§16.3, §17.2). |
| **App sees proxy IP, not client** | Rate limits/logs show Caddy's IP | Pass & trust `X-Forwarded-For` (§15.4). |
| **Redis OOMs the box** | Server killed under memory pressure | Set `maxmemory` + eviction policy (§14.2). |
| **`KEYS *` / `MONITOR` in prod** | Redis stalls for all clients | Use `SCAN`; never leave `MONITOR` running (§14.3). |
| **Untested backups** | Discover during a disaster they don't restore | Test-restore quarterly; store off-box (§21). |
| **Running as root all day** | One typo destroys the system; no audit trail | `deploy` user + `sudo` (§4). |

**Best-practice summary:** harden SSH and the firewall *first*; expose nothing but 22/80/443; run everything in Docker with pinned images, non-root users, volumes for data, and `unless-stopped`; keep databases private on the Docker network; front it all with Caddy for automatic HTTPS; keep secrets out of Git; deploy by rolling one instance at a time; back up off-box and test the restore; watch disk, memory, and health; and keep the whole thing reproducible from Git so you can rebuild from zero. Do these and a single well-run VPS (or a few) serves a real company reliably.

---

## 26. Study Path and Build-to-Learn Projects

### 26.1 A staged path **[B→A]**

1. **Get in and get safe (§1–§8).** Rent the cheapest VPS, SSH in, create the `deploy` user, harden SSH, enable the firewall and fail2ban. *Deliberately* lock yourself out once (with a second session open) and recover — it teaches the failure mode viscerally.
2. **Live in the shell (§9).** Spend a day just navigating, reading logs with `tail -f`/`journalctl`, checking `ss -tulpn`, `df -h`, `htop`. Get fluent before you add complexity.
3. **Containerize one thing (§10).** Write the multi-stage Dockerfile for a tiny Go app, build it, run it, `exec` into a debug variant, read its logs. Understand images vs containers vs volumes by using them.
4. **Compose the stack (§11–§14).** Bring up app + Postgres + Redis with one `compose.yaml`. Prove the app reaches the DB by *service name*. Kill a container and watch `unless-stopped` restart it.
5. **Go live (§15–§17).** Buy a cheap domain, point an A record at the box, add Caddy, watch it get a real HTTPS certificate automatically on the first request. This is the magic moment — a real, secure, public URL.
6. **Operate it (§18–§21).** Add secrets properly, write the rolling deploy script and run it, set up log rotation and an uptime monitor, take a backup and *restore it* into a scratch DB.
7. **Break and fix (§25).** Fill the disk on purpose and recover. Simulate a crash. Do a rebuild-from-scratch drill (§21.4). Confidence comes from having recovered, not from never failing.

### 26.2 Build-to-learn projects **[A]**

- **The full stack, for real.** Deploy the Go app from the [Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)/[pgx](GO_PGX_GUIDE.md)/[JWT+Argon2](GO_JWT_ARGON2_GUIDE.md) guides onto a VPS exactly as this guide describes — two instances, Postgres, Redis, Caddy, HTTPS — and hit it from a real frontend. This is the capstone.
- **Zero-downtime proof.** Put a `/version` endpoint in the app, run `watch -n0.5 curl -s https://api.you.dev/version` from your laptop, and run a rolling deploy — watch it flip versions with *no failed request*. Feeling that work is the payoff of the whole two-instance design.
- **Chaos drill.** While traffic flows, `docker kill app-1` and confirm Caddy routes around it seamlessly; then reboot the whole VPS and confirm the stack comes back on its own.
- **Recovery drill.** Provision a *second* fresh VPS and rebuild the entire deployment from your infra repo + a backup, then cut DNS over to it. Time yourself. That number is your real disaster-recovery capability.

### 26.3 Where to go next **[A]**

Deepen each layer with its dedicated guide: [Linux Server Admin](LINUX_SERVER_ADMIN_GUIDE.md) for systemd/users/networking internals, [Docker](DOCKER_GUIDE.md) for advanced images and Compose, [Nginx](NGINX_GUIDE.md) for the alternative proxy, [PostgreSQL](POSTGRESQL_GUIDE.md) and [Redis](REDIS_GUIDE.md) for the data stores, [Networking](NETWORKING_GUIDE.md) for DNS/TLS/HTTP fundamentals, and [CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md) to fully automate the deploy. You now have the complete picture — a bare VPS turned into a hardened, observable, recoverable, HTTPS-fronted home for a real production backend, and the mental model to scale it from your first user to your hundred-thousandth.
