# Linux Server Administration — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Someone who can write code (or wants to) but has never run a server — and wants to go all the way from *"I just got SSH credentials and I'm scared to type anything"* to *"I run a hardened, monitored, automated fleet of production Linux servers for a company."* You will learn how a server boots and stays alive, how to get in safely over SSH, how the filesystem and permission model actually work, how to install and update software, how `systemd` supervises everything, how to carve up disks, configure networking, build a firewall, harden the box against attackers, ship logs and metrics, run web apps and containers in production, take backups you can actually restore, tune performance, and finally manage many machines at once with Ansible. Every topic is explained **in prose first** — *what it is, why it exists, when and how you use it, the key flags/options/parameters and what each does, the best practices, and the security implications* — and only **then** made concrete with heavily-commented, copy-pasteable commands and real config-file contents (`sshd_config`, `systemd` units, `nftables`, `fstab`, and more). Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** Everything here is current **as of 2026** and targets the distributions you will actually meet in production:
> - **Debian family:** **Ubuntu 24.04 LTS "Noble Numbat"** (and the 26.04 LTS arriving April 2026), **Debian 12 "Bookworm"** (Debian 13 "Trixie" is in late testing). Ubuntu LTS = 5 years standard support (10+ with Ubuntu Pro); Debian stable ≈ 3 years + 2 LTS.
> - **RHEL family:** **RHEL 9 and RHEL 10**, with the bug-for-bug rebuilds **Rocky Linux 9/10** and **AlmaLinux 9/10**. RHEL major releases get ~10 years of support.
> - **Init/service manager:** **systemd v255+** is universal across all of the above.
> - **SSH:** **OpenSSH 9.x** (9.6+), which by 2026 disables the legacy `ssh-rsa`/SHA-1 signatures by default and ships post-quantum key exchange (`sntrup761x25519`) on by default.
> - **Firewall:** **nftables** is the kernel default; `iptables` is a thin compatibility shim (`iptables-nft`). Front-ends: **ufw** (Ubuntu) and **firewalld** (RHEL).
> - **Networking:** **netplan + systemd-networkd / NetworkManager** on Ubuntu; **NetworkManager + nmcli** on RHEL. `ip`/`ss` everywhere (never `ifconfig`/`netstat`).
> - **Packages:** **apt** (Debian/Ubuntu), **dnf 5** (RHEL 10 / Fedora; `yum` is a symlink to `dnf`).
> Version-specific behavior is flagged inline with **⚡ Version note**.
>
> Cross-references to sibling guides in this library appear throughout: **[Networking](NETWORKING_GUIDE.md)**, **[Nginx](NGINX_GUIDE.md)**, **[Docker](DOCKER_GUIDE.md)**, **[Bash Scripting](BASH_SCRIPTING_GUIDE.md)**, **[PostgreSQL](POSTGRESQL_GUIDE.md)**, **[Git](GIT_GUIDE.md)**, **[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)**, and **[Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)**. Authoritative offline sources to confirm any detail: the `man` pages (`man 5 sshd_config`, `man systemd.service`, `man nft`), `/usr/share/doc`, and `--help` on every command.

---

## Table of Contents

1. [What a Linux Server Is & the Distro Landscape](#1-what-a-linux-server-is--the-distro-landscape) **[B]**
2. [Getting In: SSH In Depth](#2-getting-in-ssh-in-depth) **[B/I]**
3. [The Filesystem Hierarchy, Paths & Links](#3-the-filesystem-hierarchy-paths--links) **[B]**
4. [Users, Groups, Permissions & sudo](#4-users-groups-permissions--sudo) **[B/I]**
5. [Package Management](#5-package-management) **[B/I]**
6. [Processes, Signals & System Resources](#6-processes-signals--system-resources) **[B/I]**
7. [systemd In Depth: Units, Journald, Timers, cgroups](#7-systemd-in-depth-units-journald-timers-cgroups) **[I/A]**
8. [Storage: Disks, Filesystems, LVM, RAID, Swap](#8-storage-disks-filesystems-lvm-raid-swap) **[I/A]**
9. [Networking on the Server](#9-networking-on-the-server) **[I]**
10. [Firewalls: nftables, ufw, firewalld & fail2ban](#10-firewalls-nftables-ufw-firewalld--fail2ban) **[I/A]**
11. [Hardening & Security](#11-hardening--security) **[A]**
12. [Shell Scripting & Automation for Admins](#12-shell-scripting--automation-for-admins) **[I]**
13. [Logging & Monitoring](#13-logging--monitoring) **[I/A]**
14. [Web & App Serving in Production](#14-web--app-serving-in-production) **[I/A]**
15. [Containers on a Server](#15-containers-on-a-server) **[I/A]**
16. [Backups & Disaster Recovery](#16-backups--disaster-recovery) **[I/A]**
17. [Performance Tuning & Observability](#17-performance-tuning--observability) **[A]**
18. [Configuration Management & IaC at Scale](#18-configuration-management--iac-at-scale) **[A]**
19. [Production Operations & Company-Level Practices](#19-production-operations--company-level-practices) **[A]**
20. [Gotchas & Best Practices](#20-gotchas--best-practices) **[I/A]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

## 1. What a Linux Server Is & the Distro Landscape

### 1.1 What "a server" actually means **[B]**

A **server** is not a special kind of computer — it is a *role*. The exact same Linux kernel that runs on a laptop runs on a machine in a datacenter; what makes it a "server" is that it is set up to **provide a service to other machines over the network, unattended, continuously**. Three properties follow from that definition and shape everything in this guide:

1. **It is headless.** There is no monitor, keyboard, or mouse. You administer it entirely over the network, almost always via **SSH** (Section 2). If you can't reach it over the network, for most servers you can't reach it at all — which is why a mistake that breaks networking or locks out SSH is so dangerous, and why you always keep a second way in (a cloud "serial console," a hypervisor console, or physical IPMI/iDRAC/iLO out-of-band management).
2. **It runs without a logged-in human.** Services start at boot and keep running. That job — starting things in the right order, restarting them when they crash, and shutting them down cleanly — belongs to the **init system**, which on every modern distro is **systemd** (Section 7).
3. **It is shared and exposed.** It usually faces other machines, often the public internet, so **security is not optional** (Sections 10, 11). A desktop that gets compromised is your problem; a server that gets compromised is your *users'* problem, and possibly a data-breach disclosure.

A server can be **bare metal** (a physical machine you or a hosting company owns), a **virtual machine** (a software-emulated computer running on a hypervisor like KVM, VMware, or a cloud provider's — most servers today), or a **container** (a far lighter-weight isolated process; Section 15). The administration skills are nearly identical across all three; the differences are mostly in how you provision and access them.

### 1.2 Why Linux dominates servers **[B]**

Linux runs the overwhelming majority of the world's servers, all of the top supercomputers, and effectively all of the public cloud. It won for concrete reasons: it is **free and open-source** (no per-seat licensing across thousands of machines), **stable and long-lived** (a server can run for years), **scriptable to the core** (everything is a file or a command, so everything can be automated), **modular** (you install only what you need, keeping the attack surface small), and **transparent** (you can read exactly what every part does). The trade-off is that it expects you to *know what you're doing* — there are fewer guardrails than on a consumer OS, and a confident `rm -rf` does exactly what you told it to.

### 1.3 Distributions: the two great families **[B]**

A **Linux distribution ("distro")** is the Linux kernel plus a curated collection of system software, a package manager, default configuration, a release schedule, and a support policy. For servers, almost everything you'll meet descends from one of two families. The single most important practical difference between them is the **package manager** and the **default firewall/SELinux posture**.

| | **Debian family** | **RHEL family** |
|---|---|---|
| **Members** | Debian, **Ubuntu**, Linux Mint, Pop!_OS | **RHEL**, **Rocky Linux**, **AlmaLinux**, CentOS Stream, Fedora, Oracle Linux |
| **Package format** | `.deb` | `.rpm` |
| **Package manager** | `apt` (and low-level `dpkg`) | `dnf` (and low-level `rpm`); `yum` → `dnf` |
| **Default firewall front-end** | `ufw` (Ubuntu) | `firewalld` |
| **Default MAC (mandatory access control)** | **AppArmor** | **SELinux** |
| **Network config** | **netplan** (Ubuntu) / `/etc/network/interfaces` (Debian) | **NetworkManager** |
| **Release cadence** | Ubuntu: every 6 months, **LTS every 2 years**; Debian: ~every 2 years | RHEL: major every ~3 years, minor every ~6 months |
| **Cutting edge vs stable** | Ubuntu interim = newer; Debian stable = conservative | RHEL = very conservative; Fedora = bleeding edge upstream |

**Practical guidance.** For a brand-new production server in 2026, the safe, well-trodden choices are **Ubuntu 24.04 LTS** (huge community, newest hardware support, great docs) or **Rocky Linux / AlmaLinux 9 or 10** (a free, binary-compatible RHEL clone — pick this if your org standardizes on RHEL or you need that ecosystem's certifications and SELinux-first posture). **Debian 12** is the choice when you want maximum stability and minimal corporate involvement. You should be *literate in both families*, because real jobs mix them.

### 1.4 LTS, release cadence & support windows **[B]**

**LTS = Long-Term Support.** A normal Ubuntu release is supported for 9 months — useless for a server. An **LTS** release (the `.04` of even years: 20.04, 22.04, 24.04, 26.04) gets **5 years** of free security updates, extendable to 10+ with the (free for personal use) **Ubuntu Pro** subscription. **You always run an LTS on a server.** RHEL/Rocky/Alma give you ~10 years per major version. The lesson: **pick a release with a long support window and a clear upgrade path before its end-of-life (EOL)**, because a server running an EOL OS stops getting security patches and becomes a liability. Track EOL dates and budget an OS-upgrade project well before they hit.

> ⚡ **Version note:** Ubuntu **26.04 LTS** lands in April 2026. Don't deploy a brand-new LTS into production on day one — wait for the `.1` point release (e.g. 26.04.1, ~3 months later) when early bugs are shaken out, unless you have a strong reason and good test coverage.

### 1.5 How a Linux server boots (the 30-second tour) **[B]**

Knowing the boot chain helps you debug a server that won't come up:

1. **Firmware (UEFI/BIOS)** powers on, runs self-tests, and hands off to a bootloader on disk.
2. **Bootloader (GRUB2)** presents/loads a kernel and an `initramfs` (a tiny temporary root filesystem with just enough drivers to find the real disk).
3. The **kernel** initializes hardware, mounts the real **root filesystem**, and launches the very first userspace process, **PID 1**.
4. **PID 1 is `systemd`.** It brings the system up by activating a **target** (Section 7) — usually `multi-user.target` (full networked server, no GUI) — which pulls in all the enabled services: networking, SSH, your web app, your database, and so on.
5. You SSH in, and `sshd` (already started by systemd) authenticates you and gives you a shell.

When a server "won't boot," the failure is in one of those steps, and you diagnose it from the **console** (cloud serial console / hypervisor console), not from SSH — because SSH isn't up yet.

---

## 2. Getting In: SSH In Depth

### 2.1 What SSH is and why it exists **[B]**

**SSH (Secure Shell)** is the encrypted protocol — and the family of tools — you use to log in to and run commands on a remote machine over an untrusted network. It exists because its predecessors (`telnet`, `rlogin`, `rsh`, plain FTP) sent everything, *including your password*, in cleartext; anyone between you and the server could read it. SSH gives you three guarantees over a hostile network: **confidentiality** (the session is encrypted), **integrity** (tampering is detected), and **authentication of the server** (you can be sure you're talking to the real host, not an impostor — defeating man-in-the-middle attacks). It is the front door to every server you'll ever run, so it deserves the deepest treatment in this guide. For the protocol-level view (handshake, key exchange, ciphers), cross-reference the **[Networking](NETWORKING_GUIDE.md)** guide; here we focus on *operating* it.

The basic invocation:

```bash
# Connect as user "alice" to host "server.example.com" on the default port (22).
ssh alice@server.example.com

# Non-default port with -p, and run a single command instead of an interactive shell:
ssh -p 2222 alice@server.example.com 'uptime'   # prints uptime, then disconnects

# Verbose output (-v, -vv, -vvv) — your #1 tool when a connection mysteriously fails:
ssh -vvv alice@server.example.com
```

### 2.2 Host keys & "the authenticity of host… can't be established" **[B]**

The first time you connect to a server, SSH shows you the server's **host-key fingerprint** and asks you to confirm it:

```
The authenticity of host 'server.example.com (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:abcd...wxyz.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is **Trust On First Use (TOFU)**. The server has a long-lived **host key**; its public half's fingerprint is what you're shown. If you type `yes`, that key is saved to `~/.ssh/known_hosts`, and on every future connection SSH silently verifies the server presents the *same* key. The security point: on that **first** connection you should verify the fingerprint out-of-band (from the cloud console or the provisioning output) — that's the moment a man-in-the-middle could substitute their own key. If you later see the scary **"REMOTE HOST IDENTIFICATION HAS CHANGED!"** warning, it means the saved key no longer matches: either the server was legitimately rebuilt (reinstalled, new key generated) or you're being attacked. Investigate; don't blindly delete the line. To remove a stale entry after a legitimate rebuild:

```bash
ssh-keygen -R server.example.com   # surgically removes that host's lines from known_hosts
```

### 2.3 Key-based authentication — the only acceptable method **[B/I]**

**Passwords are the weakest link in server security.** They can be guessed, brute-forced, phished, and reused. The professional standard is **public-key authentication**: you generate a **key pair** — a *private key* that never leaves your laptop, and a *public key* you copy to the server. To log in, your SSH client proves it holds the private key (via a cryptographic challenge) without ever transmitting it. There is nothing to brute-force over the wire.

**Generate a modern key.** Use **Ed25519** (fast, small, secure, the 2026 default recommendation) and *always* protect it with a passphrase:

```bash
# -t ed25519 : the algorithm (preferred over older RSA).
# -a 100     : 100 KDF rounds, making a stolen private key much harder to brute-force.
# -C         : a comment so you can identify the key later (email/hostname).
# -f         : output filename.
ssh-keygen -t ed25519 -a 100 -C "alice@laptop-2026" -f ~/.ssh/id_ed25519
# It prompts for a passphrase — USE ONE. It encrypts the private key at rest.
```

This creates `~/.ssh/id_ed25519` (private — **guard with your life, never share, never commit to git**) and `~/.ssh/id_ed25519.pub` (public — safe to hand out).

**Copy the public key to the server.** The clean way:

```bash
# ssh-copy-id appends your PUBLIC key to the server's ~/.ssh/authorized_keys,
# fixing permissions automatically. Needs password auth to still be enabled (for now).
ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@server.example.com

# Manually (when ssh-copy-id isn't available), it's just an append to a specific file:
cat ~/.ssh/id_ed25519.pub | ssh alice@server.example.com \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

**The permissions matter and trip up everyone.** OpenSSH refuses to use a key if the files are too open (a security feature). On the server: `~/.ssh` must be `700`, `~/.ssh/authorized_keys` must be `600`, and the home directory must not be group/world-writable. If key auth "silently fails," check these first (and read `/var/log/auth.log` or `journalctl -u ssh`).

### 2.4 ssh-agent — type your passphrase once **[I]**

A passphrase-protected key is secure but annoying if you must type the passphrase on every connection. **`ssh-agent`** is a background process that holds your *decrypted* private keys in memory for the session, so you authenticate once. You add keys with `ssh-add`. Agent forwarding (`-A`) can pass your agent through to a remote host so you can hop further — but use it cautiously, as a compromised intermediate host could use your forwarded agent (prefer `ProxyJump`, §2.7, which avoids that risk).

```bash
eval "$(ssh-agent -s)"          # start the agent and export its env vars into this shell
ssh-add ~/.ssh/id_ed25519       # add your key; you type the passphrase ONCE here
ssh-add -l                      # list keys currently loaded in the agent
ssh-add -t 3600 ~/.ssh/key      # add with a 1-hour timeout, then it's auto-removed
ssh-add -D                      # remove ALL keys (e.g., when leaving your desk)
```

On modern desktops the agent usually starts automatically (GNOME Keyring, macOS Keychain via `UseKeychain`, or `gpg-agent`).

### 2.5 The client config file `~/.ssh/config` — stop typing long commands **[B/I]**

Instead of memorizing `ssh -p 2222 -i ~/.ssh/work alice@203.0.113.10`, define a **host alias** once in `~/.ssh/config`. This file is one of the biggest quality-of-life upgrades in a sysadmin's toolkit, and it's where you encode jump-host topology, per-host keys, and defaults.

```sshconfig
# ~/.ssh/config   (chmod 600)

# Sane global defaults applied to every host (Host * must be LAST to not override specifics
# for keys like IdentityFile, but options take the FIRST value seen — so put specifics first).
Host *
    ServerAliveInterval 30      # send a keepalive every 30s so idle sessions don't drop
    ServerAliveCountMax 3       # give up after 3 missed keepalives (~90s of silence)
    AddKeysToAgent yes          # auto-add a key to the agent on first use
    HashKnownHosts yes          # store hostnames hashed in known_hosts (privacy)

Host web1
    HostName 203.0.113.10       # the real address
    User alice
    Port 2222
    IdentityFile ~/.ssh/work_ed25519
    IdentitiesOnly yes          # only offer THIS key (don't spray every key at the server)

# A bastion/jump host that fronts a private network:
Host bastion
    HostName bastion.example.com
    User alice
    IdentityFile ~/.ssh/work_ed25519

# A private server reachable ONLY via the bastion — one hop, transparently:
Host db1
    HostName 10.0.5.20          # a private/internal IP
    User alice
    ProxyJump bastion           # SSH tunnels through 'bastion' to reach db1
```

Now `ssh web1` or `ssh db1` "just works." `IdentitiesOnly yes` is a security/usability best practice: without it, the client offers every loaded key in turn, which can trip the server's `MaxAuthTries` limit and lock you out.

### 2.6 Hardening `sshd` — the server side **[I/A]**

`sshd` (the SSH **daemon** running on the server) is configured in **`/etc/ssh/sshd_config`** (and drop-in snippets in `/etc/ssh/sshd_config.d/*.conf`, which on Ubuntu *override* the main file — a frequent gotcha). Because SSH is the most-attacked service on any internet-facing box, hardening it is the single highest-value security task you'll do. Below is a production-grade config with every directive explained. **Always keep your current session open and test in a second window** before relying on changes, so a typo can't lock you out.

```sshconfig
# /etc/ssh/sshd_config  (or a drop-in like /etc/ssh/sshd_config.d/99-hardening.conf)

# ── Authentication ───────────────────────────────────────────────
PasswordAuthentication no        # THE big one: keys only. Kills brute-force attacks dead.
KbdInteractiveAuthentication no  # also disable challenge-response/PAM password prompts
PubkeyAuthentication yes         # allow key auth (the method we DO want)
PermitRootLogin no               # never log in directly as root; use a normal user + sudo
PermitEmptyPasswords no          # obvious, but be explicit
MaxAuthTries 3                   # drop the connection after 3 failed attempts
MaxSessions 4                    # limit multiplexed sessions per connection

# ── Who may connect ──────────────────────────────────────────────
AllowUsers alice deploy          # ONLY these users may SSH in (allowlist > denylist)
# AllowGroups sshusers           # alternative: restrict by group membership

# ── Network ──────────────────────────────────────────────────────
Port 22                          # changing the port reduces log noise, NOT real security
AddressFamily inet               # inet=IPv4 only, inet6=IPv6 only, any=both
ListenAddress 0.0.0.0            # or a specific internal IP to not listen on the public NIC

# ── Session hygiene ──────────────────────────────────────────────
ClientAliveInterval 300          # ping idle clients every 5 min
ClientAliveCountMax 2            # disconnect after 2 missed (~10 min idle) → frees sessions
LoginGraceTime 30                # 30s to authenticate, then the connection is dropped
X11Forwarding no                 # servers don't need GUI forwarding; disable it
AllowAgentForwarding no          # disable unless you specifically need it (see §2.4)
AllowTcpForwarding no            # disable unless this host is a bastion/tunnel endpoint

# ── Logging ──────────────────────────────────────────────────────
LogLevel VERBOSE                 # logs key fingerprints used — invaluable for audits
```

Apply changes safely:

```bash
sudo sshd -t                     # SYNTAX-CHECK the config first. Catches typos before reload.
sudo systemctl reload ssh        # 'reload' keeps existing sessions alive (vs 'restart')
                                 # (the unit is 'ssh' on Debian/Ubuntu, 'sshd' on RHEL)
# Now open a NEW terminal and confirm you can still get in BEFORE closing the old one.
```

> ⚡ **Version note:** OpenSSH 9.x (2024+) **removed `ssh-rsa` (SHA-1) signatures from the defaults** and enables **post-quantum key exchange** (`sntrup761x25519-sha512`) by default. Very old clients connecting to a new server (or vice versa) may need an explicit `HostKeyAlgorithms`/`KexAlgorithms` line — but you should *upgrade the old side*, not weaken the new one. Also, OpenSSH 9.8+ split the listener into a small `sshd` plus a per-session `sshd-session` binary to shrink the privileged code's attack surface (a response to the 2024 "regreSSHion" CVE).

### 2.7 Bastions / jump hosts **[I/A]**

In a well-designed network, your application and database servers have **no public IP at all** — they live on a private subnet, unreachable from the internet. The only machine exposed to SSH is a hardened **bastion host** (a.k.a. **jump host**). You SSH to the bastion, and from there reach the internal machines. This shrinks your attack surface to a single, heavily-monitored door. The modern, secure way to hop is **`ProxyJump`** (the `-J` flag or the `ProxyJump` config directive from §2.5), which tunnels your real connection *through* the bastion without your traffic or keys ever resting on it:

```bash
# Connect to internal db1 by tunneling through the bastion, in one command:
ssh -J alice@bastion.example.com alice@10.0.5.20

# Chain multiple jumps (rare but possible):
ssh -J user@jump1,user@jump2 user@final-host
```

`ProxyJump` superseded the older `ProxyCommand … netcat` recipe and is strictly safer than agent forwarding because the intermediate host never sees your decrypted credentials.

### 2.8 Moving files: scp, sftp, rsync **[B/I]**

You constantly need to move files to and from servers. There are three tools, in increasing order of power.

**`scp`** copies files over SSH — simple, fine for one-offs. (Modern OpenSSH reimplemented `scp` on top of the SFTP protocol for safety.)

```bash
scp ./app.tar.gz alice@web1:/tmp/                 # local → remote
scp alice@web1:/var/log/app.log ./                # remote → local
scp -r ./site/ alice@web1:/var/www/               # -r = recurse a directory
```

**`sftp`** is an interactive, FTP-like file-transfer session over SSH (`get`, `put`, `ls`, `cd`). Useful for browsing and ad-hoc transfers. See the **[FTP Server (Go & Node)](FTP_SERVER_GO_AND_NODE_GUIDE.md)** guide for the protocol side.

**`rsync`** is the professional's tool and what you'll reach for 90% of the time. It transfers **only the differences** between source and destination (delta algorithm), so re-syncing a large tree is fast. It preserves permissions/timestamps, can delete files removed from the source, resume interrupted transfers, and exclude patterns. It runs over SSH transparently.

```bash
# -a : archive mode (recurse + preserve perms, times, symlinks, owners) — the usual default
# -v : verbose,  -z : compress in transit,  -P : progress + resume partial files
# -e : the remote shell to use (here, ssh on a custom port)
rsync -avzP -e 'ssh -p 2222' ./build/ alice@web1:/var/www/app/

# A trailing slash on the SOURCE means "the CONTENTS of build/", not the dir itself — a
# classic footgun. ./build/ → puts files INTO /var/www/app/. ./build (no slash) → creates
# /var/www/app/build/.

# --delete makes the destination an EXACT MIRROR (removes files not in source). Powerful
# and dangerous: ALWAYS dry-run first.
rsync -avzP --delete --dry-run ./build/ alice@web1:/var/www/app/   # -n / --dry-run = preview
```

### 2.9 Persistent sessions: tmux & screen **[B/I]**

Here's a scenario that bites every beginner: you SSH in, start a long-running task (a database migration, a big `apt upgrade`, a download), your laptop's Wi-Fi blips, the SSH session dies — **and your process dies with it.** The fix is a **terminal multiplexer**, which runs a shell *on the server* inside a session that survives disconnection. You attach and detach at will; the work keeps running while you're gone. The modern choice is **`tmux`**; `screen` is the older equivalent.

```bash
tmux new -s deploy        # start a NEW named session "deploy" on the server
# ...run your long task...
# Press Ctrl-b then d  →  DETACH. The session (and your task) keeps running.
# Now you can safely close the laptop / lose Wi-Fi.

tmux ls                   # list running sessions
tmux attach -t deploy     # RE-ATTACH from any future SSH login — your task is still there
```

Inside tmux, the prefix key is **`Ctrl-b`**: `Ctrl-b "` splits horizontally, `Ctrl-b %` vertically, `Ctrl-b c` opens a new window, `Ctrl-b n/p` switch windows, arrow keys (after the prefix) move between panes. **Always run risky or long operations inside tmux/screen** — it is the difference between "Wi-Fi dropped" and "the half-finished `apt upgrade` left my server unbootable."

---

## 3. The Filesystem Hierarchy, Paths & Links

### 3.1 "Everything is a file" and one unified tree **[B]**

Unlike Windows with its `C:\`, `D:\` drive letters, Linux has **one single directory tree** starting at the **root**, written `/`. Every disk, every partition, every USB stick, even many devices and kernel interfaces, are **mounted** somewhere into that one tree (Section 8). A second hard drive isn't "D:" — it might be the directory `/mnt/data`. This unified namespace is part of the deeper Unix philosophy that **almost everything is represented as a file**: regular files, directories, your hard disk (`/dev/sda`), your terminal (`/dev/tty`), random bytes (`/dev/urandom`), even running-kernel state (`/proc`, `/sys`). Because everything is a file, the same small set of tools (`cat`, `ls`, `cp`, redirection) works on all of it — that uniformity is what makes Linux so scriptable.

### 3.2 The Filesystem Hierarchy Standard (FHS) **[B]**

The **FHS** defines what each top-level directory is *for*, so that software (and you) know where things live on any distro. Memorizing this map turns "where on earth is that config?" from a hunt into a reflex.

| Path | Purpose |
|---|---|
| `/` | The root of everything. |
| `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin` | Executable programs (commands). `sbin` = system/admin binaries. On modern distros `/bin`→`/usr/bin` (the "usr merge"). |
| `/etc` | **System-wide configuration files.** Plain text. This is where you'll spend much of your life. |
| `/home/<user>` | Users' personal directories (your files, your `~/.ssh`, dotfiles). |
| `/root` | The **root** user's home (not in `/home`). |
| `/var` | **Variable** data that changes at runtime: `/var/log` (logs), `/var/lib` (app state/databases), `/var/spool` (queues), `/var/www` (web content). |
| `/tmp` | Temporary files, **world-writable, wiped on reboot** (often a RAM-backed tmpfs). |
| `/usr` | "Unix System Resources" — the bulk of installed software, libraries, docs. Read-only in spirit. |
| `/opt` | Optional/third-party self-contained software. |
| `/srv` | Data served by this machine (some sites put web/FTP data here). |
| `/dev` | Device files (`/dev/sda`, `/dev/null`, `/dev/urandom`). |
| `/proc` | Virtual filesystem exposing **per-process and kernel** info (`/proc/cpuinfo`, `/proc/meminfo`). |
| `/sys` | Virtual filesystem exposing **devices and kernel tunables** (`/sys/class/...`). |
| `/boot` | The kernel, `initramfs`, and bootloader (GRUB) files. |
| `/mnt`, `/media` | Mount points for temporary/removable filesystems. |
| `/run` | Volatile runtime data (PIDs, sockets), tmpfs, cleared on boot. |
| `/lib`, `/usr/lib` | Shared libraries and kernel modules. |

**Rules of thumb:** configuration lives in **`/etc`**, logs and app data live under **`/var`**, your stuff lives in **`/home`**, installed programs live under **`/usr`**, and **`/proc` + `/sys`** are windows into the live kernel, not real files on disk.

### 3.3 Navigating: absolute vs relative paths **[B]**

An **absolute path** starts at root and is unambiguous everywhere: `/etc/ssh/sshd_config`. A **relative path** is interpreted from your **current working directory** (shown by `pwd`): if you're in `/etc/ssh`, then `sshd_config` and `./sshd_config` mean the same file. Special tokens: `.` is "here," `..` is "the parent directory," `~` is "my home directory," and `-` (with `cd`) is "the previous directory."

```bash
pwd                    # print working directory — "where am I?"
cd /var/log            # go to an absolute path
cd ..                  # up one level → /var
cd ~                   # go home (just 'cd' alone also goes home)
cd -                   # jump back to the previous directory
ls -lah /etc           # -l long format, -a include hidden (dot)files, -h human-readable sizes
ls -lt                 # sort by modification time, newest first (great for "what changed?")
tree -L 2 /etc/ssh     # visual tree, 2 levels deep (install 'tree' if missing)
```

A leading dot makes a file **hidden** (`.bashrc`, `.ssh`) — it's just a naming convention, not a permission. `ls -a` reveals them.

### 3.4 Links: hard links and symbolic links **[B/I]**

A **link** lets one file be reachable by more than one name. There are two kinds, and confusing them causes real bugs.

A **symbolic link (symlink, soft link)** is a tiny file that **points to a path** — like a shortcut. It can cross filesystems, point to directories, and point to things that don't exist yet (a "dangling" link). If the target moves or is deleted, the symlink breaks. This is by far the more common kind, used constantly: `/usr/bin/python3` → `python3.12`, web roots pointing at a release directory, etc.

A **hard link** is a second *directory entry pointing at the very same data on disk* (the same **inode**). There's no "original" and "copy" — both names are equal, first-class references to identical bytes. The data is freed only when the **last** hard link is removed. Hard links can't cross filesystems or (normally) link directories.

```bash
ln -s /opt/app/releases/v42 /opt/app/current   # create a SYMLINK 'current' → the v42 release
ls -l /opt/app/current                         # shows: current -> /opt/app/releases/v42
readlink -f /opt/app/current                   # resolve a symlink to its final absolute target

ln /data/file.txt /data/also-file.txt          # create a HARD LINK (no -s): same inode
ls -li /data/*.txt                             # -i shows inode numbers; hard links share one
```

The atomic-symlink-swap is a famous **zero-downtime deploy** trick (Section 14): you upload the new release to a fresh directory, then atomically repoint the `current` symlink — the switch is instantaneous and a failed deploy never leaves a half-written live directory.

---

## 4. Users, Groups, Permissions & sudo

### 4.1 The Unix user/permission model and why it matters **[B]**

Linux is **multi-user** to its core, and its security model rests on **users**, **groups**, and **file permissions**. The reason this matters on a server is **least privilege**: every process runs *as some user*, and that user's permissions bound what the process can touch. If your web server runs as the unprivileged `www-data` user and gets exploited, the attacker gets `www-data`'s limited powers — not root's. Designing who-runs-as-whom, and what each user can read/write, is the foundation of server security.

Every user has a numeric **UID** and belongs to one **primary group** (a **GID**) plus any number of **secondary groups**. **UID 0 is special: it's `root`, the superuser, who bypasses all permission checks.** Three kinds of accounts exist: **root** (UID 0, total power), **system/service accounts** (low UIDs, no login shell — they exist solely to own and run a service, like `www-data`, `postgres`, `sshd`), and **human accounts** (UID ≥ 1000, real people). The account database is `/etc/passwd` (one line per user; *not* secret) and `/etc/shadow` (the hashed passwords; root-only).

```bash
id                       # show your UID, GID, and group memberships
whoami                   # current username
getent passwd alice      # look up a user (works whether the DB is local or LDAP)
cat /etc/passwd          # name:x:UID:GID:comment:home:shell  (x = password is in /etc/shadow)
```

### 4.2 Reading & understanding `rwx` permissions **[B]**

Run `ls -l` and you see permission strings like `-rwxr-xr--`. Decode them left to right:

```
-  rwx  r-x  r--
│   │    │    └── OTHER (everyone else): here r-- = read only
│   │    └─────── GROUP (members of the file's group): here r-x = read + execute
│   └──────────── OWNER (the user who owns the file): here rwx = read + write + execute
└──────────────── TYPE: - file, d directory, l symlink, c/b device, s socket, p pipe
```

The three permission bits mean different things for files vs directories — a crucial distinction:

| Bit | On a **file** | On a **directory** |
|---|---|---|
| **r** (read) | read its contents | *list* its entries (filenames) |
| **w** (write) | modify its contents | create/delete/rename files **inside** it |
| **x** (execute) | run it as a program | *enter/traverse* it (`cd` into it, access files by path) |

The directory rules surprise people: to `cd` into a directory you need **x** on it; to *list* it you need **r**; and crucially, **deleting a file depends on write permission of the directory it lives in, not the file itself.** That's why a file you don't own can still be deleted by anyone with write access to its folder (unless the sticky bit is set — §4.4).

### 4.3 Changing permissions & ownership: chmod, chown, umask **[B/I]**

**`chmod`** changes permission bits. You can use **symbolic** mode (readable) or **octal** mode (compact, what scripts use). Octal encodes each rwx triplet as a digit: **r=4, w=2, x=1**, summed. So `rwx`=7, `rw-`=6, `r-x`=5, `r--`=4. Three digits = owner/group/other.

```bash
chmod 644 file.txt       # rw-r--r--  : owner rw, group r, other r  (typical for files)
chmod 600 ~/.ssh/id_ed25519  # rw-------  : owner only (REQUIRED for private keys)
chmod 755 script.sh      # rwxr-xr-x  : owner all, others read+exec (typical for programs/dirs)
chmod 700 ~/.ssh         # rwx------  : owner only (REQUIRED for the .ssh dir)
chmod +x deploy.sh       # symbolic: add execute for all
chmod u+x,go-w file      # add exec for user; remove write for group & other
chmod -R 750 /srv/app    # -R = recurse (be careful: applies x to files too — see below)
```

> **Gotcha:** `chmod -R 755` on a tree makes *every file* executable, which is wrong and a minor security smell. To set dirs and files differently, use `find`: `find /srv/app -type d -exec chmod 755 {} +` and `find /srv/app -type f -exec chmod 644 {} +`.

**`chown`** changes the owner and/or group. Only root can give a file away to another user.

```bash
sudo chown alice file.txt           # change owner to alice
sudo chown alice:developers file    # change owner to alice AND group to developers
sudo chown -R www-data:www-data /var/www/app   # recurse — common for web roots
chgrp developers file               # change only the group
```

**`umask`** sets the *default* permissions for newly-created files by **masking off** bits. The common server umask is `022`, which removes write from group/other, so new files default to `644` and new directories to `755`. A tighter `077` (owner-only) is good for sensitive accounts.

### 4.4 Special bits: setuid, setgid, sticky **[I]**

Three extra permission bits unlock advanced (and security-sensitive) behavior:

- **setuid (`u+s`, octal 4xxx)** on an executable makes it run **as the file's owner**, not as the user who launched it. This is how `passwd` lets a normal user edit the root-owned `/etc/shadow`. Setuid-root binaries are a classic privilege-escalation target — **audit them and minimize them.**
- **setgid (`g+s`, octal 2xxx)** on an executable runs it as the file's *group*; on a **directory** it's far more useful: new files created inside **inherit the directory's group**, which is the standard way to build a shared team folder.
- **sticky bit (`+t`, octal 1xxx)** on a directory means **only a file's owner (or root) can delete it**, even if others have write access to the directory. This is why `/tmp` (mode `1777`) is world-writable yet users can't delete each other's temp files.

```bash
ls -l /usr/bin/passwd       # -rwsr-xr-x  ← the 's' in the owner triplet = setuid
ls -ld /tmp                 # drwxrwxrwt  ← the 't' at the end = sticky bit
chmod 2775 /srv/shared      # setgid dir: new files inherit the 'shared' group
chmod 1777 /scratch         # sticky: shared scratch space, users can't nuke each other's files
find / -perm -4000 -type f 2>/dev/null   # AUDIT: list all setuid-root binaries on the system
```

### 4.5 Managing users & groups **[B/I]**

```bash
# Create a human user with a home dir and bash shell:
sudo useradd -m -s /bin/bash -c "Bob Jones" bob   # -m make home, -s shell, -c comment
sudo passwd bob                                    # set/replace bob's password
# (Debian/Ubuntu also have the friendlier interactive 'adduser bob'.)

# Create a system/service account with NO login (it only runs a daemon):
sudo useradd --system --no-create-home --shell /usr/sbin/nologin appsvc

sudo usermod -aG sudo bob       # ADD bob to the 'sudo' group (-a APPEND, -G secondary group)
                                # ⚠ Forgetting -a REPLACES all secondary groups — a classic
                                # way to accidentally remove someone from every group.
sudo usermod -aG docker bob     # let bob use Docker without sudo (note: this == root-equiv!)
groups bob                      # show bob's groups
sudo gpasswd -d bob docker      # remove bob from a group

sudo userdel -r bob             # delete user AND their home dir (-r). Omit -r to keep files.
sudo groupadd developers        # create a group
```

On **Ubuntu/Debian**, the admin group is **`sudo`**; on **RHEL/Rocky/Alma** it's **`wheel`**. Adding a user to that group is how you grant them administrative rights.

### 4.6 `sudo` and `/etc/sudoers` — controlled privilege escalation **[B/I]**

**You should never log in as root, and you should disable root SSH login entirely** (§2.6). Instead, normal users run individual commands with elevated privilege via **`sudo`** ("substitute user do"). Why this is better than a shared root password: every `sudo` use is **logged** (who ran what, when — in `/var/log/auth.log` or the journal), you can grant **fine-grained** rights (this user may only restart nginx), there's no shared secret to leak, and accountability is preserved.

```bash
sudo apt update                 # run one command as root
sudo -u postgres psql           # run as a SPECIFIC user (here, the postgres service account)
sudo -i                         # start a root login shell (for a series of admin commands)
sudo -l                         # list what YOU are allowed to run via sudo
sudo !!                         # re-run the previous command with sudo (shell shortcut)
```

`sudo` rights are defined in **`/etc/sudoers`**, which you must **always edit with `visudo`** — never a plain editor. `visudo` syntax-checks the file on save, so a typo can't lock everyone out of root forever (a real, unrecoverable-without-a-rescue-disk mistake). Better still, drop your rules into a file under `/etc/sudoers.d/`:

```bash
sudo visudo                                  # edit the main file safely
sudo visudo -f /etc/sudoers.d/deploy         # edit/validate a drop-in file (preferred)
```

```sudoers
# /etc/sudoers.d/deploy  — examples of common rules

# Full sudo for everyone in the admin group (default on most distros):
%sudo   ALL=(ALL:ALL) ALL          # %group  hosts=(runas) commands

# Let the 'deploy' user restart ONLY the app & nginx, WITHOUT a password prompt
# (essential for non-interactive CI/CD; restrict the command list tightly):
deploy  ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl reload nginx

# Let the 'dba' group run psql as the postgres user, password required:
%dba    ALL=(postgres) /usr/bin/psql
```

**Security implications:** the `NOPASSWD` convenience is also a risk — if that account is compromised, the attacker gets those commands for free, so keep the command list minimal and never `NOPASSWD: ALL`. Watch out for **wildcards and shell-outs**: granting `sudo vim` effectively grants root (you can spawn a shell from inside vim). Grant *specific binaries with specific arguments*, not editors or interpreters.

### 4.7 PAM — the authentication framework (the basics) **[I]**

**PAM (Pluggable Authentication Modules)** is the framework that decides *how* login, sudo, SSH, and other services authenticate, authorize, and set up a session. Each service has a config in **`/etc/pam.d/`** (e.g., `/etc/pam.d/sshd`, `/etc/pam.d/sudo`) listing a stack of modules across four **management groups**: **auth** (prove identity), **account** (is the account allowed/expired?), **password** (changing credentials, e.g. complexity rules), and **session** (set up/tear down the session, e.g. mount home, set limits). You rarely write PAM configs from scratch, but you *do* edit them to enforce things like **password complexity** (`pam_pwquality`), **account lockout after failed logins** (`pam_faillock`), **two-factor authentication** (`pam_google_authenticator`), or **resource limits** (`pam_limits`, which reads `/etc/security/limits.conf` — see §17). Knowing PAM exists, and that it's the layer where "lock the account after 5 bad passwords" or "require an OTP" gets implemented, is the goal at this level.

---

## 5. Package Management

### 5.1 What a package manager is & why it matters **[B]**

A **package manager** installs, upgrades, configures, and removes software, while **automatically resolving dependencies** (the other libraries a program needs) and **cryptographically verifying** that packages come from a trusted, signed source. Before package managers, installing software meant manually downloading source, chasing down a tangle of dependencies ("dependency hell"), and compiling by hand. The package manager turns all of that into one command, keeps a database of what's installed and which files belong to which package, and — critically for a server — lets you **patch every piece of software in one sweep** when a vulnerability drops. Mastering it is non-negotiable: it's how you install your stack and, more importantly, how you keep it secure and up to date.

Software comes from **repositories ("repos")**: signed collections of packages hosted by the distro and third parties. The package manager checks each package's signature against trusted keys, so you can't be silently fed a malicious binary — *as long as you only add repos and keys you trust* (a key supply-chain security point).

### 5.2 APT (Debian / Ubuntu) **[B/I]**

`apt` is the high-level front-end (`dpkg` is the low-level engine that handles individual `.deb` files).

```bash
sudo apt update                  # refresh the list of available packages from repos.
                                 # ALWAYS run this before installing/upgrading — it only
                                 # updates the INDEX, it does NOT change installed software.
sudo apt upgrade                 # install available updates for installed packages
sudo apt full-upgrade            # like upgrade but may add/remove packages to satisfy deps
sudo apt install nginx git htop  # install one or more packages (+ their dependencies)
sudo apt remove nginx            # remove a package but KEEP its config files
sudo apt purge nginx             # remove the package AND its config files
sudo apt autoremove              # remove orphaned dependencies nothing needs anymore
apt search '^postgresql'         # search package names/descriptions (regex)
apt show nginx                   # detailed info: version, deps, size, description
apt list --installed             # everything currently installed
dpkg -l                          # low-level: list installed packages
dpkg -L nginx                    # which FILES did the nginx package install?
dpkg -S /usr/sbin/nginx          # which package OWNS this file? (reverse lookup)
```

Repos are configured in `/etc/apt/sources.list` and `/etc/apt/sources.list.d/*` (the newer `deb822` `.sources` format on 24.04+). Third-party repo signing keys live under `/etc/apt/keyrings/` — **the modern, secure way** is to download a vendor's key to a keyring file and reference it in the repo line with `signed-by=`, rather than the deprecated, system-wide `apt-key add`.

### 5.3 DNF / YUM (RHEL / Rocky / Alma / Fedora) **[B/I]**

`dnf` is the RHEL-family equivalent. `yum` is now just a compatibility alias for `dnf`. The command shapes are deliberately similar to apt's ideas, but you do **not** need a separate "update the index" step — `dnf` refreshes metadata as needed.

```bash
sudo dnf install nginx git       # install (auto-refreshes metadata, resolves deps)
sudo dnf upgrade                 # update ALL packages (this single command = apt update+upgrade)
sudo dnf remove nginx            # remove a package
sudo dnf autoremove              # remove orphaned dependencies
dnf search nginx                 # search
dnf info nginx                   # detailed package info
dnf list installed               # list installed packages
rpm -qa                          # low-level: query all installed packages
rpm -ql nginx                    # which FILES does nginx own?
rpm -qf /usr/sbin/nginx          # which package owns this file?
sudo dnf history                 # ★ DNF logs every transaction...
sudo dnf history undo 42         # ...and can ROLL BACK a transaction by ID. Lifesaver.
sudo dnf config-manager --add-repo https://example.com/repo.repo   # add a repo
sudo dnf module list             # AppStream "modules" (alternate versions of e.g. nodejs)
```

> ⚡ **Version note:** **DNF 5** is the default on **Fedora 41+ and RHEL 10** — much faster, with refined output, but the everyday commands above are unchanged. RHEL/Rocky/Alma 8–9 ship DNF 4.

### 5.4 Pinning, holding & version control **[I]**

Sometimes you must **prevent** a package from upgrading — to freeze a known-good kernel, hold a database at a tested version, or avoid an upgrade that breaks a dependency. This is "pinning" or "holding."

```bash
# Debian/Ubuntu — hold/unhold:
sudo apt-mark hold postgresql-16    # never auto-upgrade this package
apt-mark showhold                   # list held packages
sudo apt-mark unhold postgresql-16  # release the hold
# Finer control via /etc/apt/preferences.d/ ("apt pinning") to prefer specific versions.

# RHEL family — exclude or use the versionlock plugin:
sudo dnf install python3-dnf-plugin-versionlock
sudo dnf versionlock add postgresql-server   # lock the current version
sudo dnf versionlock list
```

### 5.5 Universal/sandboxed packages: Snap & Flatpak **[I]**

Distro repos can lag behind upstream releases. **Snap** (Canonical, default on Ubuntu) and **Flatpak** (more common on Fedora/desktops) bundle an app with its dependencies and run it **sandboxed**, so you get newer versions and stronger isolation at the cost of larger size and slower startup. On servers, Snap shows up most for things like `certbot` and `lxd`; Flatpak is mostly a desktop technology.

```bash
sudo snap install certbot --classic   # --classic = no sandbox (needs full host access)
snap list                             # installed snaps
sudo snap refresh                     # update all snaps (they also auto-update)
```

### 5.6 Building from source (when you must) **[I]**

Occasionally you need software that isn't packaged, or a newer/custom build. The classic GNU pattern is `./configure && make && make install`. **Prefer packaged software when at all possible** — source installs aren't tracked by the package manager, won't get security updates, and clutter the system. If you must, install into `/usr/local` (not `/usr`) to keep it separate, and strongly consider building a `.deb`/`.rpm` (or a container image) instead so it's trackable.

```bash
sudo apt install build-essential   # the compiler toolchain (gcc, make, headers) on Debian/Ubuntu
sudo dnf groupinstall "Development Tools"   # the RHEL-family equivalent
tar xzf app-1.2.3.tar.gz && cd app-1.2.3
./configure --prefix=/usr/local    # detect the system & set the install location
make -j"$(nproc)"                  # compile, using all CPU cores (nproc = core count)
sudo make install                  # copy built files into /usr/local
```

### 5.7 Automatic security updates **[B/I]**

A server that never gets patched becomes a sitting duck. Configure **unattended security updates** so critical fixes apply automatically (covered fully in §11.3): `unattended-upgrades` on Debian/Ubuntu, `dnf-automatic` on RHEL-family. Apply *security* updates automatically; schedule a maintenance window for larger upgrades you want to watch.

---

## 6. Processes, Signals & System Resources

### 6.1 What a process is **[B]**

A **process** is a running instance of a program. Each has a unique **PID** (process ID), a **parent** (PPID — every process except PID 1 is started by another), an owner (the UID it runs as, which bounds its powers), its own memory space, open file descriptors, and a **state** (running, sleeping, stopped, or the dreaded **zombie** — a finished child whose parent hasn't yet collected its exit status). Understanding processes is how you answer the daily server questions: *what's using all the CPU? why is the box out of memory? why won't this service die? what got killed and why?*

### 6.2 Inspecting processes: ps, top, htop **[B/I]**

**`ps`** takes a snapshot of processes. The cryptic-but-universal incantation is `ps aux` (BSD style) or `ps -ef` (System V style):

```bash
ps aux                  # a=all users, u=user-oriented columns, x=incl. processes w/o a terminal
ps aux | grep nginx     # find a specific process (the grep itself shows up too)
ps -ef --forest         # show the parent/child PROCESS TREE
ps -eo pid,ppid,user,%cpu,%mem,etime,cmd --sort=-%mem | head   # custom columns, sorted by RAM
pgrep -af nginx         # find PIDs by name (cleaner than ps|grep)
```

**`top`** is the live, refreshing dashboard built into every system. **`htop`** (install it) is the friendlier color version with mouse support, tree view, and easy killing/renicing — most admins install it on every box. Key things to read in `top`/`htop`: the **load average** (§6.5), per-process **%CPU** and **%MEM** (RES = real resident RAM, the number that matters), and the process **state**.

```bash
htop                    # interactive: F6 sort, F4 filter, F9 kill, F5 tree view, q quit
top -o %MEM             # sort by memory; press 'M' (mem) / 'P' (cpu) / 'k' (kill) interactively
```

### 6.3 Signals: how you talk to a process **[B/I]**

You control processes by sending them **signals** — small asynchronous notifications. The program can choose how to handle most of them (e.g., re-read its config), but a couple are unstoppable. The everyday ones:

| Signal | Number | Meaning | Catchable? |
|---|---|---|---|
| `SIGTERM` | 15 | **Polite "please shut down."** The default. Lets the program clean up. | Yes |
| `SIGKILL` | 9 | **Forceful "die now."** Kernel kills it immediately, no cleanup. Last resort. | **No** |
| `SIGINT` | 2 | Interrupt — what `Ctrl-C` sends. | Yes |
| `SIGHUP` | 1 | "Hang up." Conventionally means **"reload your config"** for daemons. | Yes |
| `SIGSTOP`/`SIGCONT` | 19/18 | Pause / resume a process. | STOP: No |
| `SIGUSR1`/`SIGUSR2` | 10/12 | App-defined (e.g., nginx uses them to rotate logs / upgrade binary). | Yes |

```bash
kill 12345              # send SIGTERM (the default) — the RIGHT first thing to try
kill -HUP 12345         # tell a daemon to reload its config (no restart, no downtime)
kill -9 12345           # SIGKILL — only when SIGTERM is ignored. Skips cleanup → may corrupt.
killall nginx           # signal all processes by NAME
pkill -f 'python app.py'  # signal by command-line pattern
```

**Best practice:** always try `SIGTERM` first and give the process a few seconds. Jumping straight to `kill -9` denies the program its chance to flush buffers, close files, and finish transactions — which is how you corrupt a database. On a server, you'll usually stop services via `systemctl stop` (§7) rather than killing PIDs directly, because systemd sends the signals *and* tracks the whole process group.

### 6.4 Priorities: nice & renice **[I]**

Every process has a **niceness** from **-20 (highest priority, greediest)** to **+19 (lowest, most "nice" to others)**, default 0. A "nicer" process yields CPU to others. You use this to keep a heavy batch job (a backup, a big compile) from starving your web server. Only root can set negative (higher-priority) niceness.

```bash
nice -n 10 ./backup.sh          # START a job at lower priority (+10)
renice -n 5 -p 12345            # CHANGE a running process's niceness
ionice -c2 -n7 -p 12345         # also de-prioritize its DISK I/O (class 2, level 7)
```

### 6.5 Load average — the most misread number on a server **[B/I]**

`uptime`, `top`, and `w` show three **load averages** (1, 5, 15-minute). Load is the average number of processes **running or waiting** for the CPU (and, on Linux, also waiting on uninterruptible disk I/O). The key to reading it: **compare load to your core count.** On a 4-core box, a load of 4.0 means "fully busy, no queue"; 8.0 means "twice as much work as cores — things are queuing and getting slow"; 1.0 means "mostly idle." The three numbers show the *trend*: a 1-min load far above the 15-min means a spike is building; the reverse means it's subsiding. **High load with low CPU usage** is the classic fingerprint of **I/O wait** — the CPU is idle because everything's blocked on a slow disk or network (confirm with the `%wa` field in `top` or `iostat`, §17).

```bash
uptime                  # ... load average: 0.52, 0.71, 0.68
nproc                   # how many CPU cores you have (the number to compare load against)
cat /proc/loadavg       # the raw source of those numbers
```

### 6.6 Memory: free, the page cache, and not panicking **[B/I]**

Reading memory correctly saves you from a beginner's heart attack. Linux deliberately **uses "free" RAM as disk cache** (the **page cache**) to speed up file access — so a healthy server often shows almost no "free" memory, and that's *good*, not a leak. The number that matters is **`available`**: how much RAM apps could claim *right now*, counting reclaimable cache.

```bash
free -h                 # -h = human-readable
#               total        used        free      shared  buff/cache   available
# Mem:           7.8Gi       2.1Gi       180Mi        45Mi       5.5Gi       5.4Gi
#                                         ▲ "free" looks tiny — DON'T PANIC...
#                                                                            ▲ ...look at
#                                                                              "available"
cat /proc/meminfo       # the full, detailed memory breakdown
```

`buff/cache` is reclaimable on demand; only when **`available`** runs low are you genuinely short on RAM.

### 6.7 Swap and the OOM killer **[I]**

**Swap** is disk space the kernel uses as overflow when RAM fills — it moves idle memory pages to disk to free RAM. It's a safety cushion, not a substitute for RAM: if a busy server is actively **swapping** (thrashing), performance collapses because disk is orders of magnitude slower than RAM. When memory is *truly* exhausted and swap can't save it, the kernel invokes the **OOM (Out-Of-Memory) killer**, which sacrifices a process to keep the system alive. It picks a victim by an `oom_score` heuristic (roughly, "biggest, least-important memory hog"). Discovering that your database was "killed for no reason" usually means the OOM killer struck — always check:

```bash
free -h                          # is swap in use? how much?
swapon --show                    # active swap devices/files
dmesg -T | grep -i 'killed process'   # ★ did the OOM killer strike? what did it kill?
journalctl -k | grep -i oom           # same, from the kernel journal
# Bias a critical process AWAY from being killed (lower = safer; -1000..1000):
echo -500 | sudo tee /proc/$(pgrep -f postgres | head -1)/oom_score_adj
```

**Best practice:** size RAM so you rarely swap; keep a *little* swap as a cushion (or use `zram`, compressed RAM-based swap); on a memory-critical service, protect it with systemd's `OOMScoreAdjust=` and `MemoryMax=` (§7). The `vm.swappiness` sysctl (§17) tunes how eagerly the kernel swaps; 10 is a common server value (vs the default 60).

### 6.8 /proc and /sys — the live kernel as files **[I]**

`/proc` exposes per-process and system info as virtual files: `/proc/<pid>/` has everything about a process (`status`, `limits`, `environ`, `fd/` open file descriptors, `cmdline`). `/proc/cpuinfo`, `/proc/meminfo`, `/proc/loadavg`, `/proc/mounts` describe the system. `/sys` exposes devices and **kernel tunables** you can read and write. Many monitoring tools are just pretty front-ends over these files — knowing they exist lets you script anything.

---

## 7. systemd In Depth: Units, Journald, Timers, cgroups

### 7.1 What systemd is and why it runs everything **[I]**

**systemd** is the **init system and service manager** — **PID 1**, the first process the kernel starts and the ancestor of everything else. Its job is to bring the machine up (start services in dependency order, in parallel where possible), keep services running (restart them on failure), shut down cleanly, and provide a uniform interface to manage all of it. It replaced the old SysV init scripts because those were slow (strictly sequential), inconsistent (every service a hand-written shell script), and dumb about dependencies and crashed services. systemd is now universal across Ubuntu, Debian, RHEL, Rocky, and Alma, so **`systemctl` and `journalctl` are the two commands you'll type more than any others on a server.** It also absorbed logging (`journald`), scheduling (timers), network config (`networkd`/`resolved`), and more — a lot of surface area, but a consistent one.

### 7.2 Units — the universal abstraction **[I]**

systemd models everything as a **unit**, an object described by a small declarative config file. The type is the file extension:

| Unit type | What it represents |
|---|---|
| `.service` | A daemon/process to run (nginx, postgres, your app). **The one you'll use most.** |
| `.target` | A named group/sync point of units (≈ the old "runlevels"): `multi-user.target`, `graphical.target`. |
| `.timer` | A schedule that activates another unit (the modern cron). |
| `.socket` | A socket systemd listens on, starting the service on first connection (socket activation). |
| `.mount` / `.automount` | A filesystem mount point (often auto-generated from `/etc/fstab`). |
| `.path` | Watches a file/dir and triggers a unit on change. |
| `.device`, `.swap`, `.slice`, `.scope` | Devices, swap, and cgroup resource groupings. |

Unit files live in three layers, in increasing precedence: **`/usr/lib/systemd/system/`** (shipped by packages — don't edit), **`/etc/systemd/system/`** (your local units and overrides — **edit here**), and runtime `/run/systemd/system/`. To tweak a packaged unit, never edit the vendor file; use a **drop-in override** (§7.5) so upgrades don't clobber your change.

### 7.3 Driving services with systemctl **[B/I]**

```bash
sudo systemctl start nginx       # start now
sudo systemctl stop nginx        # stop now
sudo systemctl restart nginx     # stop then start (brief downtime)
sudo systemctl reload nginx      # tell it to re-read config WITHOUT dropping connections
sudo systemctl reload-or-restart nginx   # reload if supported, else restart

sudo systemctl enable nginx      # start automatically AT BOOT (creates symlinks)
sudo systemctl disable nginx     # don't start at boot
sudo systemctl enable --now nginx   # ★ enable AND start, in one go
sudo systemctl mask nginx        # forcibly DISABLE — can't be started even manually
sudo systemctl unmask nginx      # undo a mask

systemctl status nginx           # ★ is it running? recent logs? PID? memory? Read this CONSTANTLY.
systemctl is-active nginx        # scriptable: prints "active"/"inactive", sets exit code
systemctl is-enabled nginx       # will it start at boot?
systemctl list-units --type=service          # all loaded services
systemctl list-units --state=failed          # ★ what's BROKEN right now?
systemctl list-unit-files --state=enabled    # everything set to start at boot
systemctl cat nginx              # show the effective unit file(s), incl. drop-ins
sudo systemctl daemon-reload     # ★ RE-READ unit files after you EDIT one (easy to forget!)
```

**The two most important muscle-memory habits:** read `systemctl status <svc>` whenever something's wrong (it shows state, the last few log lines, and the exit code), and run `sudo systemctl daemon-reload` after **editing** any unit file — otherwise systemd keeps using the old version and you'll be baffled.

### 7.4 Writing your own service unit **[I/A]**

This is a core production skill: you have an app (a Go binary, a Node server, a Python service) and you want systemd to run it as an unprivileged user, restart it if it crashes, start it at boot, and capture its logs. Here is a fully-commented, hardened template:

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My Web Application          # human-readable; shows in status/logs
Documentation=https://internal.wiki/myapp
After=network-online.target postgresql.service   # start only AFTER these are up...
Wants=network-online.target            # ...and actively pull network-online in
Requires=postgresql.service            # HARD dep: if postgres fails, we fail too
                                       # (use Wants= for a SOFT dep that won't block us)

[Service]
Type=simple                            # the process stays in the foreground (most apps).
                                       # Other types: 'forking' (classic daemons that fork),
                                       # 'notify' (app signals readiness via sd_notify),
                                       # 'oneshot' (runs once and exits, for setup tasks).
User=appsvc                            # ★ run as an UNPRIVILEGED service account, never root
Group=appsvc
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/env         # load KEY=VALUE secrets/config from a file (chmod 600)
ExecStart=/opt/myapp/bin/myapp --port 8080   # the command to run (MUST be an absolute path)
ExecReload=/bin/kill -HUP $MAINPID     # how 'systemctl reload' should signal the app

Restart=on-failure                     # ★ auto-restart if it crashes (not on clean exit)
RestartSec=5                           # wait 5s between restarts...
StartLimitIntervalSec=60               # ...and if it crashes 5 times in 60s...
StartLimitBurst=5                      # ...give up (stop flapping forever)

# ── Security sandboxing (cheap, powerful hardening — apply to EVERY service) ──
NoNewPrivileges=true                   # process can never gain new privileges (no setuid esc.)
ProtectSystem=strict                   # make almost the entire filesystem READ-ONLY to it
ProtectHome=true                       # hide /home, /root, /run/user from the service
ReadWritePaths=/var/lib/myapp          # ...except this dir it legitimately needs to write
PrivateTmp=true                        # give it its OWN /tmp (isolated from other services)
PrivateDevices=true                    # no access to physical devices
ProtectKernelTunables=true             # can't write /proc/sys, /sys
ProtectControlGroups=true              # can't tamper with cgroups
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX   # only these socket types
CapabilityBoundingSet=                 # drop ALL Linux capabilities (add back only if needed)

# ── Resource limits (cgroup-enforced — see §7.8) ──
MemoryMax=512M                         # HARD cap: exceed it → the service is OOM-killed (not the box)
CPUQuota=50%                           # at most half of one CPU core
LimitNOFILE=65536                      # max open file descriptors (raise for network servers)

[Install]
WantedBy=multi-user.target             # 'systemctl enable' hooks it into normal multi-user boot
```

```bash
sudo systemctl daemon-reload           # make systemd read the new file
sudo systemctl enable --now myapp      # start it now AND at every boot
systemctl status myapp                 # verify it's running
journalctl -u myapp -f                 # watch its logs live (see §7.6)
systemd-analyze security myapp         # ★ SCORE this unit's sandboxing; tighten the worst bits
```

That sandboxing block is one of the highest-leverage security wins available: a few declarative lines turn a process that could (if exploited) roam the whole filesystem into one boxed into a read-only world with a private `/tmp`, no extra privileges, and capped resources. Run `systemd-analyze security <unit>` to grade it.

### 7.5 Drop-in overrides — customize without forking **[I]**

To change *one setting* of a packaged unit, don't copy and edit the whole file (you'd miss upstream improvements and security fixes). Create a **drop-in**: a small file in `…/<unit>.d/override.conf` that adds to or overrides specific directives. `systemctl edit` does this for you:

```bash
sudo systemctl edit nginx          # opens an editor; creates /etc/systemd/system/nginx.service.d/override.conf
```

```ini
# This snippet ONLY changes these directives; everything else stays as the package shipped it.
[Service]
MemoryMax=1G
Restart=always
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart nginx
systemctl cat nginx                # confirm: shows the base unit + your drop-in merged
```

### 7.6 journald — querying logs like a pro **[B/I]**

systemd captures every service's stdout/stderr and structured metadata into a binary **journal**, queried with **`journalctl`**. This is a massive upgrade over hunting through scattered text files: you can filter by unit, time, priority, boot, or PID, follow live, and it's all structured. (Many systems *also* run `rsyslog` to write traditional `/var/log/*` files — §13 — but `journalctl` is your first stop.)

```bash
journalctl -u myapp              # all logs for one unit
journalctl -u myapp -f           # ★ FOLLOW live (like tail -f)
journalctl -u myapp -n 100       # last 100 lines
journalctl -u myapp --since "10 min ago"          # time-window filtering...
journalctl -u myapp --since today --until "1 hour ago"
journalctl -p err -b             # only priority 'error' and worse, THIS boot (-b)
journalctl -b -1                 # logs from the PREVIOUS boot (did it crash last time?)
journalctl -k                    # KERNEL messages (= dmesg, but persistent)
journalctl --disk-usage          # how much space the journal is using
journalctl _PID=12345            # filter by structured fields (PID, UID, etc.)
journalctl -u myapp -o json-pretty   # structured output for tooling
```

By default the journal may be **volatile** (lost on reboot, stored in `/run`). Make it **persistent** so you can investigate crashes after a reboot, and cap its size:

```bash
sudo mkdir -p /var/log/journal   # its existence makes the journal persistent
# /etc/systemd/journald.conf:  Storage=persistent  SystemMaxUse=1G   then:
sudo systemctl restart systemd-journald
```

### 7.7 Timers vs cron — scheduling the systemd way **[I]**

systemd **timers** are the modern replacement for cron (§12 covers cron itself). A timer is a pair of units: a `.timer` (the schedule) that activates a `.service` (the work). Timers beat cron for: **logging** (output lands in the journal, queryable per-unit), **dependencies** (run only after the network/database is up), **`Persistent=true`** (run a missed job after downtime — cron just skips it), **randomized delays** (spread load across a fleet), and **resource control** (the job inherits cgroup limits). cron is still fine for simple, portable jobs; timers win when you need observability and reliability.

```ini
# /etc/systemd/system/backup.service  — WHAT to do (Type=oneshot: run and exit)
[Unit]
Description=Nightly database backup
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer  — WHEN to do it
[Unit]
Description=Run the nightly backup
[Timer]
OnCalendar=*-*-* 02:30:00      # every day at 02:30 (systemd's calendar syntax)
RandomizedDelaySec=600         # jitter up to 10 min (avoid a thundering-herd across servers)
Persistent=true                # if the machine was off at 02:30, run it on next boot
[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now backup.timer
systemctl list-timers            # ★ when did each timer last run, when does it run next?
sudo systemd-run --on-active=10m /usr/local/bin/task.sh   # one-off "run this in 10 minutes"
```

`OnCalendar` examples: `hourly`, `daily`, `weekly`, `Mon *-*-* 09:00`, `*-*-01 00:00` (1st of month). Test an expression with `systemd-analyze calendar 'Mon *-*-* 09:00'`.

### 7.8 Resource control with cgroups **[I/A]**

**cgroups (control groups)** are the kernel feature that **limits and accounts for** the CPU, memory, and I/O a group of processes may use — the same mechanism that powers containers (§15). systemd puts every service in its own cgroup automatically, so you can cap resources *per service* declaratively (no wrapper scripts), and you get free accounting:

```bash
systemctl set-property myapp.service MemoryMax=512M CPUQuota=50%   # apply live (persists)
systemd-cgtop                    # ★ live, per-cgroup CPU/MEM/IO — "which SERVICE is the hog?"
systemctl status myapp           # the status output now shows the service's memory & tasks
```

The key directives (use them in unit files, §7.4): **`MemoryMax=`** (hard cap; over it → that service's processes are OOM-killed, protecting the rest of the box), **`MemoryHigh=`** (soft throttle), **`CPUQuota=`** (CPU ceiling), **`CPUWeight=`** (relative share under contention), **`IOWeight=`** / **`IOReadBandwidthMax=`** (disk I/O), and **`TasksMax=`** (process/thread count, a fork-bomb guard). This is how you stop one misbehaving service from taking down everything else on a shared box — a foundational reliability technique.

> ⚡ **Version note:** Modern distros use **cgroup v2** (the "unified hierarchy") exclusively, which is what the directives above target. You may still see cgroup-v1 references in old docs — ignore them on a 2024+ system.

---

## 8. Storage: Disks, Filesystems, LVM, RAID, Swap

### 8.1 The storage stack, top to bottom **[I]**

Before commands, hold the layered mental model, because storage is layered and each layer has its own tools:

1. **Physical/virtual disk** — `/dev/sda`, `/dev/nvme0n1`, `/dev/vda` (a whole device).
2. **Partition** — a slice of a disk: `/dev/sda1`, `/dev/nvme0n1p1`.
3. **(Optional) LVM or RAID** — a flexible/redundant layer *between* partitions and filesystems.
4. **Filesystem** — the structure that turns raw blocks into files and directories (ext4, xfs, btrfs, zfs).
5. **Mount point** — the directory in the unified tree where that filesystem appears (`/`, `/home`, `/var`).

"Adding a disk to a server" means walking down this stack: identify the device, partition it, (optionally) put it in LVM/RAID, create a filesystem, mount it, and add it to `/etc/fstab` so it mounts at every boot.

### 8.2 Seeing what you have **[B/I]**

```bash
lsblk                    # ★ tree of disks → partitions → mountpoints. Your first storage command.
lsblk -f                 # also show filesystem TYPE and UUID (great overview)
sudo fdisk -l            # detailed partition tables of all disks
df -h                    # ★ disk space FREE per mounted filesystem, human-readable
df -i                    # INODE usage — you can run out of inodes (too many tiny files)
                         #   while still showing free space! A real, confusing outage cause.
sudo du -sh /var/*       # ★ where did my space GO? size of each item under /var, summarized
sudo du -ahx / | sort -rh | head -20   # the 20 biggest files/dirs on the root fs
ncdu /var                # interactive disk-usage explorer (install it — far nicer than du)
blkid                    # show UUIDs and types of block devices (for fstab)
```

`df` vs `du`: **`df`** asks the *filesystem* how full it is; **`du`** adds up *files*. They can disagree when a deleted-but-still-open file holds space (a process keeps a deleted log open) — in which case `df` shows full but `du` doesn't, and the fix is to restart the process holding the deleted file (`lsof | grep deleted`).

### 8.3 Partitioning: fdisk, parted, gdisk **[I]**

Two partition-table formats: the legacy **MBR** (max 2 TB, 4 primary partitions) and the modern **GPT** (huge disks, 128 partitions, required for UEFI). **Use GPT** for anything new. `fdisk` now handles both; `parted` is scriptable and good for big disks; `gdisk` is GPT-specialized.

```bash
sudo fdisk /dev/sdb      # interactive: 'g' new GPT table, 'n' new partition, 'w' WRITE (commit), 'q' quit w/o saving
# Non-interactive with parted (e.g. for automation): one big partition spanning the disk:
sudo parted -s /dev/sdb mklabel gpt mkpart primary ext4 0% 100%
sudo partprobe /dev/sdb  # tell the kernel to re-read the partition table without a reboot
```

> ⚠ **Danger:** partitioning operates on whole devices and is destructive. *Triple-check the device name* (`/dev/sdb`, not `/dev/sda` which is usually your system disk). The wrong letter wipes your server. Always `lsblk` first.

### 8.4 Filesystems: ext4, xfs, btrfs, zfs **[I/A]**

The filesystem decides how files are stored, its max sizes, and features like snapshots and checksums. Pick by use case:

| Filesystem | Strengths | When to use |
|---|---|---|
| **ext4** | Rock-solid, mature, fast, universal default on Debian/Ubuntu | The safe default for almost everything |
| **xfs** | Excellent for large files & high parallelism; default on RHEL family | Big data, databases, RHEL servers |
| **btrfs** | Built-in snapshots, checksums, compression, subvolumes (copy-on-write) | When you want cheap snapshots/rollback |
| **zfs** | Best-in-class data integrity, snapshots, compression, software RAID-Z | Storage/NAS servers, max data safety |

```bash
sudo mkfs.ext4 -L data /dev/sdb1     # create an ext4 filesystem, label "data"
sudo mkfs.xfs  -L data /dev/sdb1     # or xfs
sudo tune2fs -l /dev/sdb1            # inspect ext4 parameters (reserved blocks, etc.)
```

**Best practice:** if you don't have a specific reason, **ext4** (Debian/Ubuntu) or **xfs** (RHEL) is the boring, correct choice. Reach for **btrfs/zfs** when you specifically want their snapshot/integrity features and accept the extra complexity.

### 8.5 Mounting and `/etc/fstab` **[B/I]**

**Mounting** attaches a filesystem to a directory (mount point). `mount` does it *temporarily* (lost on reboot); **`/etc/fstab`** makes it *permanent* — the file systemd reads at boot to mount everything. **Reference filesystems by UUID, not `/dev/sdb1`**, because device letters can change between boots (add a disk and `sdb` might become `sdc`), but UUIDs are stable — a UUID mistake here is a top cause of "my server won't boot."

```bash
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data       # temporary mount, to test before committing to fstab
mount | grep sdb1                    # confirm it's mounted and with what options
blkid /dev/sdb1                      # get the UUID to put in fstab
```

```fstab
# /etc/fstab — columns: <device>  <mountpoint>  <fstype>  <options>  <dump>  <fsck-pass>
UUID=1234-abcd-...   /mnt/data   ext4   defaults,noatime          0  2
# Hardening a writable-but-no-exec data partition:
UUID=5678-ef01-...   /srv/uploads ext4  defaults,nosuid,nodev,noexec  0  2
tmpfs                /tmp        tmpfs  defaults,nosuid,nodev,size=2G 0  0
```

Key mount **options** and what they do: `noatime` (don't write a timestamp on every read — a real performance win, especially for busy/SSD storage), `nosuid` (ignore setuid bits — harden data partitions), `nodev` (don't honor device files), **`noexec`** (forbid running programs from here — apply to `/tmp`, upload dirs, to blunt a class of attacks), `ro` (read-only). The last two fstab columns: `dump` (legacy backup flag, leave `0`) and `fsck` pass (`1` for root, `2` for others, `0` to skip).

```bash
sudo mount -a            # ★ mount everything in fstab NOW — TEST your fstab edits with this
                         # BEFORE rebooting. If it errors, fix it now, not from a rescue console.
sudo systemctl daemon-reload   # systemd reads fstab into mount units; reload after edits
```

> ⚠ A bad `/etc/fstab` entry can make a server **fail to boot** (it drops to an emergency shell waiting for a mount that can't happen). Add `nofail` to the options of non-critical mounts so a missing disk doesn't block boot, and **always `mount -a` before you reboot.**

### 8.6 LVM — flexible volumes **[I/A]**

**LVM (Logical Volume Manager)** inserts an abstraction layer between physical disks and filesystems so you can **resize, span, and snapshot** storage without repartitioning. Its three concepts: **PV (Physical Volume)** = a disk/partition handed to LVM; **VG (Volume Group)** = a pool combining one or more PVs; **LV (Logical Volume)** = a virtual "partition" carved from the pool, which you format and mount. The payoff: you can **grow a filesystem online** by adding a disk to the VG and extending the LV — no downtime, no repartition dance. This is why production servers almost always use LVM.

```bash
sudo pvcreate /dev/sdb /dev/sdc                 # 1. mark two disks as LVM physical volumes
sudo vgcreate data_vg /dev/sdb /dev/sdc         # 2. pool them into volume group "data_vg"
sudo lvcreate -L 50G -n app_lv data_vg          # 3. carve a 50G logical volume "app_lv"
sudo mkfs.ext4 /dev/data_vg/app_lv              # 4. format it
sudo mount /dev/data_vg/app_lv /srv/app         # 5. mount it (then add to fstab by UUID)

# GROW it later, ONLINE, with zero downtime — the killer feature:
sudo lvextend -L +20G /dev/data_vg/app_lv       # add 20G to the logical volume...
sudo resize2fs /dev/data_vg/app_lv              # ...then grow the ext4 filesystem onto it
                                                # (xfs: use 'xfs_growfs /srv/app' instead)
sudo vgs; sudo lvs; sudo pvs                    # inspect groups / volumes / physical volumes
```

LVM **snapshots** give a point-in-time, copy-on-write view of a volume — perfect for taking a *consistent* backup of a database while it's running (snapshot, back up the snapshot, drop the snapshot).

### 8.7 RAID — redundancy & performance (overview) **[I/A]**

**RAID (Redundant Array of Independent Disks)** combines multiple disks for **redundancy** (survive a disk failure) and/or **performance**. The levels you must know:

| Level | What it does | Survives | Cost |
|---|---|---|---|
| **RAID 0** | Stripes across disks (speed only) | **0** disk failures (worse than one disk!) | none, but risky |
| **RAID 1** | Mirrors (exact copies) | 1+ failures | 50% capacity |
| **RAID 5** | Stripe + 1 parity disk | 1 failure | one disk's worth |
| **RAID 6** | Stripe + 2 parity | 2 failures | two disks' worth |
| **RAID 10** | Mirror + stripe | 1 per mirror | 50% capacity, fast |

Linux software RAID uses **`mdadm`**. **Crucial caveat: RAID is not a backup.** It protects against *hardware* failure, not against `rm -rf`, ransomware, corruption, or a fire — all of which replicate instantly across the mirror. You still need real backups (§16).

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc   # RAID 1 mirror
cat /proc/mdstat                 # ★ array status & rebuild progress
sudo mdadm --detail /dev/md0     # detailed health of the array
```

### 8.8 Swap configuration **[I]**

Swap (§6.7) is a partition or a **swap file**. A swap file is easier to add/resize on a running server:

```bash
sudo fallocate -l 2G /swapfile       # create a 2 GB file
sudo chmod 600 /swapfile             # only root may read/write it
sudo mkswap /swapfile                # format it as swap
sudo swapon /swapfile                # activate it now
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # persist across reboots
swapon --show; free -h               # verify
```

---

## 9. Networking on the Server

This section covers *operating* the network on the box; for the protocols themselves (IP, TCP/UDP, DNS, TLS, routing) read the **[Networking](NETWORKING_GUIDE.md)** guide — it's the companion to this one.

### 9.1 The modern toolset: `ip` and `ss` (never ifconfig/netstat) **[B/I]**

The old `net-tools` commands (`ifconfig`, `route`, `netstat`, `arp`) are **deprecated and often not even installed** on modern servers. The replacements live in `iproute2`: **`ip`** for addresses/links/routes and **`ss`** for sockets. Learn these; the old ones will eventually be muscle-memory that fails you on a fresh box.

```bash
ip addr                  # ★ show all interfaces and their IP addresses (alias: 'ip a')
ip -br addr              # -br = brief, one tidy line per interface
ip link                  # show interfaces and their state (UP/DOWN, MAC)
ip route                 # ★ the routing table — incl. the default gateway ('default via ...')
ip neigh                 # the ARP/neighbor table (IP ↔ MAC)

ss -tulpn                # ★★ THE command: what's LISTENING? -t TCP -u UDP -l listening
                         #     -p process -n numeric. "What ports are open and who owns them?"
ss -tan state established # all established TCP connections
ss -s                    # socket summary statistics
```

`ss -tulpn` is the most important troubleshooting command in this whole section: it answers "is my service actually listening, on which address and port, and which process owns it?" — the first thing you check when "I can't connect to my app."

### 9.2 Static vs DHCP, and why servers want static **[I]**

A client laptop gets its IP automatically via **DHCP** (a server hands it an address from a pool). A **server**, though, usually needs a **static IP** (or a DHCP *reservation* tied to its MAC) so its address never changes — because DNS records, firewall rules, and other machines' configs all point at that address. A server whose IP silently changed overnight is an outage. Cloud VMs are the nuance: they typically get a *stable* private IP via DHCP from the cloud's control plane, so you leave DHCP on and let the cloud manage it — don't hard-code static config on a cloud instance unless you know the platform expects it.

### 9.3 Configuring the network — netplan (Ubuntu) vs NetworkManager (RHEL) **[I]**

How you persist network config differs by family. **Ubuntu** uses **netplan**, a YAML layer that *renders* config for a backend (`systemd-networkd` on servers, `NetworkManager` on desktops). **RHEL/Rocky/Alma** use **NetworkManager**, driven by `nmcli`.

```yaml
# Ubuntu: /etc/netplan/01-netcfg.yaml   (YAML — indentation matters!)
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false                    # turn DHCP off for a static config
      addresses: [203.0.113.10/24]    # this host's IP + prefix length
      routes:
        - to: default
          via: 203.0.113.1            # the gateway/router
      nameservers:
        addresses: [1.1.1.1, 9.9.9.9] # DNS resolvers
```

```bash
sudo netplan try                      # ★ apply, but AUTO-ROLLBACK in 120s if you don't confirm.
                                      #   Saves you from locking yourself out with a bad config.
sudo netplan apply                    # apply permanently (only after 'try' looked good)
```

```bash
# RHEL family with nmcli (imperative, no YAML):
nmcli connection show                            # list connections ("profiles")
nmcli con mod eth0 ipv4.addresses 203.0.113.10/24 ipv4.gateway 203.0.113.1 \
       ipv4.dns "1.1.1.1 9.9.9.9" ipv4.method manual   # set a static config
nmcli con up eth0                                # apply it
nmcli device status                              # interface states
```

> ⚠ **Always have a fallback before changing networking remotely.** A typo can drop your SSH session with no way back. Use `netplan try` (auto-rollback), or schedule a `reboot` in a few minutes (`sudo shutdown -r +5`) that you cancel (`shutdown -c`) only once you confirm the new config works, or keep a cloud serial console open.

### 9.4 DNS resolution on the server **[I]**

When your server resolves a name (to reach a database, an API, a package mirror), it consults its **resolver** config. Historically that was the static **`/etc/resolv.conf`** (a list of `nameserver` IPs and a `search` domain). Modern Ubuntu runs **`systemd-resolved`**, a local caching stub resolver, and `/etc/resolv.conf` is a *symlink* to a stub at `127.0.0.53` — so editing `/etc/resolv.conf` directly does nothing useful. Use `resolvectl` instead.

```bash
resolvectl status                # ★ which DNS servers is each interface actually using?
resolvectl query example.com     # resolve a name THROUGH systemd-resolved (uses its cache)
resolvectl flush-caches          # clear the DNS cache (after changing records)
cat /etc/resolv.conf             # note: a symlink to systemd-resolved's stub on Ubuntu
dig example.com                  # the proper DNS debugging tool (from bind9-dnsutils)
dig +short example.com           # just the answer
getent hosts example.com         # resolve the way the SYSTEM does (honors /etc/hosts + nsswitch)
```

`/etc/hosts` still provides static, local overrides (it's checked before DNS, per `/etc/nsswitch.conf`) — handy for pinning a name during testing.

### 9.5 Quick on-server connectivity troubleshooting **[I]**

A reliable top-down checklist when "the server can't reach X" or "I can't reach the server":

```bash
ip -br addr                      # 1. Do I have the IP I expect, interface UP?
ip route                         # 2. Is there a default route (gateway)?
ping -c3 203.0.113.1             # 3. Can I reach the GATEWAY? (layer 3 to the local net)
ping -c3 1.1.1.1                 # 4. Can I reach the wider internet by IP? (routing/firewall)
resolvectl query example.com     # 5. Does DNS resolve? (if 4 works but names don't → DNS)
ss -tulpn | grep :443            # 6. Is my service actually LISTENING on the port?
curl -v https://localhost:443    # 7. Does it respond locally? (isolates app vs network/firewall)
sudo nft list ruleset            # 8. Is the FIREWALL blocking it? (§10)
```

This sequence isolates the failure to a layer fast. For deeper packet-level work (`tcpdump`, `traceroute`, `mtr`), see the **[Networking](NETWORKING_GUIDE.md)** guide.

---

## 10. Firewalls: nftables, ufw, firewalld & fail2ban

### 10.1 Why a host firewall, and the default-deny principle **[I]**

A **firewall** filters network traffic against a set of rules. On a server it enforces the single most important network-security rule: **default-deny inbound** — block *everything* coming in, then explicitly allow only the handful of ports your services actually need (typically SSH, HTTP, HTTPS). The reasoning is least exposure: every open port is a potential entry point, and a service you forgot was listening (a debug endpoint, a database bound to `0.0.0.0`) is exactly how breaches happen. With default-deny, *forgetting* to lock something down fails *safe* instead of fails *open*. **Outbound** is usually left open (the server needs to reach package mirrors, APIs, DNS), though high-security environments restrict that too (egress filtering).

> Note: cloud providers add a *second* firewall *outside* the VM — **Security Groups** (AWS), **Network Security Groups** (Azure), **VPC firewall rules** (GCP). Best practice is **defense in depth**: lock down both the cloud-level firewall *and* the host firewall. A common "why can't I connect?!" is the cloud security group blocking a port the host firewall allows (or vice versa).

### 10.2 nftables — the modern kernel firewall **[I/A]**

**`nftables`** replaced `iptables` as the Linux kernel's native packet-filtering framework. It's faster, has cleaner syntax, unifies IPv4/IPv6, and uses atomic rule replacement. `iptables` still "works" but is now just a compatibility shim (`iptables-nft`) that translates to nftables under the hood — write **nftables** for anything new. Its model: **tables** (a namespace for a protocol family) contain **chains** (attached to hooks like `input`/`forward`/`output`) which hold **rules**. Here's a complete, commented baseline server ruleset:

```nft
#!/usr/sbin/nft -f
# /etc/nftables.conf  — a default-deny inbound firewall for a web server

flush ruleset                          # start clean

table inet filter {                    # 'inet' = handles both IPv4 and IPv6 in one table
    chain input {
        type filter hook input priority 0; policy drop;   # ★ DEFAULT-DENY: drop unmatched input

        ct state established,related accept   # allow replies to connections WE initiated
        ct state invalid drop                 # drop malformed/invalid packets
        iif lo accept                         # allow all loopback (localhost) traffic

        ip protocol icmp icmp type echo-request limit rate 5/second accept  # rate-limited ping
        ip6 nexthdr icmpv6 accept             # ICMPv6 is REQUIRED for IPv6 to work — don't block it

        tcp dport 22  ct state new limit rate 15/minute accept   # SSH, brute-force rate-limited
        tcp dport 80  accept                  # HTTP
        tcp dport 443 accept                  # HTTPS
        # everything else hits 'policy drop' and is silently dropped
    }
    chain forward { type filter hook forward priority 0; policy drop; }  # not a router → drop
    chain output  { type filter hook output  priority 0; policy accept; } # allow our outbound
}
```

```bash
sudo nft -f /etc/nftables.conf       # load the ruleset from the file (atomic replace)
sudo nft list ruleset                # ★ show the entire live ruleset
sudo systemctl enable --now nftables # ★ load /etc/nftables.conf at every boot (CRITICAL — else
                                     #   your firewall vanishes on reboot)
```

> ⚠ **Don't lock yourself out.** When editing firewall rules over SSH, the `ct state established,related accept` line keeps your *current* session alive even if you fumble the SSH rule — but a fresh login could be blocked. Test in a second session, and on a remote box consider scheduling `sudo shutdown -r +5` (a reboot that reverts un-persisted rules) and cancelling it (`shutdown -c`) only once you've confirmed you're not locked out.

### 10.3 ufw — the friendly Ubuntu front-end **[I]**

Writing raw nftables is powerful but verbose. **`ufw` (Uncomplicated Firewall)** is Ubuntu's simple front-end that generates the rules for you — ideal for single servers where you don't need nftables' full expressiveness.

```bash
sudo ufw default deny incoming      # ★ default-deny inbound
sudo ufw default allow outgoing     # allow outbound
sudo ufw allow OpenSSH              # allow SSH by named profile (or: ufw allow 22/tcp)
sudo ufw allow 80,443/tcp           # allow HTTP & HTTPS
sudo ufw limit ssh                  # rate-limit SSH (auto-blocks brute-forcers)
sudo ufw allow from 10.0.0.0/24 to any port 5432  # only this subnet may reach Postgres
sudo ufw enable                     # turn it on (warns it may disrupt SSH — see above!)
sudo ufw status verbose             # ★ show rules and defaults
sudo ufw delete allow 80/tcp        # remove a rule
```

### 10.4 firewalld — the RHEL front-end & zones **[I]**

**`firewalld`** is the RHEL-family front-end, built around **zones** (named trust levels — `public`, `internal`, `dmz`, `trusted` — each with its own rules; interfaces and sources are assigned to zones). You make changes to the *permanent* config, then `--reload`.

```bash
sudo firewall-cmd --get-active-zones                       # which zone is each interface in?
sudo firewall-cmd --zone=public --add-service=https --permanent   # allow HTTPS, persistently
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent   # allow a custom port
sudo firewall-cmd --zone=public --add-rich-rule='rule family=ipv4 source address=10.0.0.0/24 port port=5432 protocol=tcp accept' --permanent
sudo firewall-cmd --reload                                 # ★ apply permanent changes
sudo firewall-cmd --list-all                               # show the active zone's full config
```

> ⚡ **Version note:** firewalld on RHEL 9/10 uses the **nftables** backend by default (not the old iptables backend) — the front-end is unchanged, but the rules it generates are native nftables.

### 10.5 NAT & port forwarding (when the server is a router) **[A]**

If your server routes traffic for others (a VPN gateway, a NAT box, a container host without Docker's own NAT), you need **NAT (Network Address Translation)**: **SNAT/masquerade** rewrites outgoing source addresses so a private network can share one public IP; **DNAT/port-forwarding** redirects an incoming port to an internal host. This requires enabling IP forwarding plus nftables NAT chains:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system            # enable the kernel to forward packets between interfaces
```

```nft
table inet nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "eth0" masquerade            # SNAT: share eth0's public IP for outbound
    }
    chain prerouting {
        type nat hook prerouting priority dstnat;
        iifname "eth0" tcp dport 8080 dnat to 10.0.0.5:80   # forward :8080 → internal :80
    }
}
```

### 10.6 fail2ban — auto-banning brute-forcers **[I/A]**

Even with key-only SSH and a firewall, bots will hammer your open ports endlessly, filling logs and occasionally finding a weak service. **`fail2ban`** watches log files, detects patterns of failure (repeated auth failures, scanner signatures), and **dynamically adds a firewall ban** for the offending IP for a set time. It's a near-mandatory layer on any internet-facing host. A "jail" couples a log filter to a ban action:

```ini
# /etc/fail2ban/jail.local   (override jail.conf here, never edit jail.conf directly)
[DEFAULT]
bantime  = 1h             # how long an IP stays banned
findtime = 10m            # the window in which failures are counted
maxretry = 5              # ban after this many failures within findtime
backend  = systemd        # read failures from the journal (modern default)
banaction = nftables-multiport   # ban using nftables (not legacy iptables)

[sshd]
enabled = true            # protect SSH (the most-attacked service)
# Jails also exist for nginx (bad bots, auth), postfix, and many more.
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd     # ★ how many IPs are currently banned, and which?
sudo fail2ban-client set sshd unbanip 203.0.113.7   # un-ban an IP (e.g., you locked yourself out)
```

---

## 11. Hardening & Security

### 11.1 The mindset: least privilege, defense in depth, reduce attack surface **[A]**

Hardening isn't one task; it's a posture made of overlapping principles. **Least privilege:** every user, service, and process gets the *minimum* access it needs and no more (run services as dedicated unprivileged accounts, grant narrow `sudo`, drop Linux capabilities). **Defense in depth:** assume any single control will fail, so layer several — a firewall *and* fail2ban *and* key-only SSH *and* SELinux *and* sandboxed services, so one breach doesn't equal full compromise. **Reduce attack surface:** every installed package, open port, and running service is a potential vulnerability, so install and run the *minimum*. **Assume breach:** design so that *when* (not if) something is compromised, the blast radius is small and you'll detect it (auditd, monitoring). The professional benchmark for "what specifically to do" is the **CIS Benchmarks** (Center for Internet Security) — distro-specific, prescriptive checklists; tools like `OpenSCAP`/`Lynis` audit a box against them.

### 11.2 The SSH & account hardening checklist (recap + extend) **[A]**

Most server compromises start at a login. Lock the doors (cross-ref §2.6, §4):

- **Key-only SSH** (`PasswordAuthentication no`), **no root login** (`PermitRootLogin no`), **user allowlist** (`AllowUsers`/`AllowGroups`), **fail2ban** on `sshd`, optional **non-standard port** (cuts log noise, not real security), optional **MFA** via `pam_google_authenticator`.
- **No shared accounts.** Every human has their own login and uses `sudo` — for accountability and clean off-boarding.
- **Disable/lock unused accounts**, set login.defs password aging, enforce complexity via `pam_pwquality`, lock after failed attempts via `pam_faillock`.
- **Audit setuid binaries** (`find / -perm -4000`) and **listening ports** (`ss -tulpn`) — close or remove anything you don't recognize.

### 11.3 Automatic security updates — the highest-value patch habit **[A]**

The most common real-world compromise vector is an **unpatched known vulnerability**. Automating *security* patching closes that window without a human in the loop.

```bash
# Debian/Ubuntu:
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades     # enable it
# /etc/apt/apt.conf.d/50unattended-upgrades controls WHICH updates auto-apply
# (security only by default — the right choice) and whether to auto-reboot for kernels.

# RHEL/Rocky/Alma:
sudo dnf install dnf-automatic
# set apply_updates = yes and upgrade_type = security in /etc/dnf/automatic.conf, then:
sudo systemctl enable --now dnf-automatic.timer
```

**Best practice:** auto-apply *security* updates everywhere; for kernel updates that need a reboot, either enable a controlled auto-reboot window (`Unattended-Upgrade::Automatic-Reboot-Time "03:00"`) or use **`needrestart`**/**kpatch/live-patching** to know when a reboot is pending. Test bigger upgrades in staging first.

### 11.4 Mandatory Access Control: SELinux & AppArmor **[A]**

Standard Unix permissions are **Discretionary** Access Control (DAC) — the *owner* decides who can access a file, and **root bypasses everything**. **MAC (Mandatory Access Control)** adds a second, system-wide policy that even root can't override and that *confines each program to only the files/ports/operations it's supposed to use*. The payoff: if your web server is exploited, MAC can stop the attacker from reading `/etc/shadow` or writing outside the web root **even though the process technically has the Unix permissions** — because the MAC policy forbids that *program* from doing so. Two implementations:

- **SELinux** (RHEL family, default **enforcing**): label-based, very granular, powerful but with a learning curve. Every file and process has a security context (label); policy governs which label may do what to which.
- **AppArmor** (Ubuntu/Debian, default): path-based profiles per program — simpler to read and write.

```bash
# ── SELinux (RHEL/Rocky/Alma) ──
getenforce                       # Enforcing | Permissive | Disabled
sudo setenforce 0                # TEMPORARILY → Permissive (logs violations but allows them)
sudo ausearch -m AVC -ts recent  # ★ show recent SELinux DENIALS (your debugging starting point)
sudo restorecon -Rv /var/www     # reset files to their CORRECT SELinux labels (fixes most issues)
sudo semanage port -a -t http_port_t -p tcp 8080   # tell SELinux a new port is "an http port"
sudo setsebool -P httpd_can_network_connect on     # flip a tunable SELinux boolean, persistently

# ── AppArmor (Ubuntu/Debian) ──
sudo aa-status                   # which profiles are loaded and enforcing?
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx   # put a profile in COMPLAIN mode (log-only)
sudo aa-enforce  /etc/apparmor.d/usr.sbin.nginx   # back to ENFORCE
```

> **Critical advice — do NOT just disable it.** The reflex when an app "mysteriously" fails is to `setenforce 0` or disable AppArmor. **Don't.** That throws away a major security layer. Instead, *read the denial* (`ausearch`/journal), then fix the *label*/*port*/*boolean* (SELinux) or the *profile path* (AppArmor). The fix is almost always a one-liner. Permissive/complain mode is for *diagnosing*, not a destination.

### 11.5 auditd — the forensic record **[A]**

**`auditd`** is the Linux kernel's auditing subsystem: it records security-relevant events — who ran what, who accessed which sensitive file, who changed a config — to a tamper-evident log. It's how you answer "what exactly happened?" after an incident, and it's frequently *required* by compliance frameworks (PCI-DSS, HIPAA, SOC 2). You define **watch rules** on sensitive paths and syscalls:

```bash
sudo auditctl -w /etc/passwd -p wa -k passwd_changes   # WATCH /etc/passwd for Writes/Attr changes
sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes # and the sudoers file
# Persist rules in /etc/audit/rules.d/*.rules, then:
sudo augenrules --load
sudo ausearch -k passwd_changes      # ★ search the audit log by your key
sudo aureport --auth                  # summarize authentication events
```

### 11.6 Secrets management **[A]**

Secrets — database passwords, API keys, TLS private keys — must **never** be committed to git, baked into images, or world-readable on disk. Layers of good practice, weakest to strongest:

- **File permissions:** secrets in a `chmod 600`, root- or service-owned file (e.g., the `EnvironmentFile=` of a systemd unit, §7.4). The minimum bar.
- **systemd credentials:** `LoadCredential=`/`SetCredential=` deliver secrets to a service in a private, non-world-readable tmpfs, kept out of the environment (which can leak via `/proc/<pid>/environ`).
- **A dedicated secrets manager:** **HashiCorp Vault**, cloud KMS/Secrets Manager (AWS/GCP/Azure), or **SOPS** (encrypt secrets *in* git with age/PGP). These add rotation, access auditing, and short-lived dynamic credentials.

**Never** put secrets in command-line arguments (visible in `ps` to every user) or in shell history. Restrict who can read service env files, and rotate anything that may have leaked.

### 11.7 Kernel hardening via sysctl **[A]**

The kernel exposes hundreds of tunables under `/proc/sys` (set via **`sysctl`**), several of which harden the network stack and memory. Persist them in `/etc/sysctl.d/*.conf`:

```ini
# /etc/sysctl.d/99-hardening.conf
net.ipv4.conf.all.rp_filter = 1            # reverse-path filter: drop spoofed source addresses
net.ipv4.icmp_echo_ignore_broadcasts = 1   # ignore broadcast pings (Smurf-attack hygiene)
net.ipv4.conf.all.accept_redirects = 0     # don't accept ICMP redirects (route hijack defense)
net.ipv4.conf.all.send_redirects = 0       # don't send them either (we're not a router)
net.ipv4.conf.all.accept_source_route = 0  # reject source-routed packets (spoofing vector)
net.ipv4.tcp_syncookies = 1                # SYN-flood mitigation
kernel.randomize_va_space = 2              # full ASLR (memory-layout randomization)
kernel.kptr_restrict = 2                   # hide kernel pointers from unprivileged users
kernel.dmesg_restrict = 1                  # restrict who can read the kernel log
fs.protected_hardlinks = 1                 # block a class of hardlink-based privilege attacks
fs.protected_symlinks = 1                  # and symlink-based ones
```

```bash
sudo sysctl --system             # load all /etc/sysctl.d/*.conf now
sudo sysctl net.ipv4.tcp_syncookies   # read a single current value
```

### 11.8 A pragmatic hardening order **[A]**

You don't do all of this on day one. A sane order for a new production box: (1) key-only SSH + no root + firewall default-deny + fail2ban; (2) automatic security updates; (3) run every service as an unprivileged user under a sandboxed systemd unit (§7.4); (4) keep SELinux/AppArmor *enforcing*; (5) sysctl hardening; (6) auditd + centralized logging; (7) run `Lynis`/OpenSCAP against the CIS benchmark and fix the findings; (8) document it all in a runbook (§19). Automate the whole list with Ansible (§18) so every new box is born hardened.

---

## 12. Shell Scripting & Automation for Admins

### 12.1 Why every admin scripts — and the prime directive **[I]**

Administration *is* automation. Any task you do twice should become a script: it's faster, repeatable, documented (the script *is* the documentation), and free of the typos that creep into by-hand work. The deeper guide to the language is **[Bash Scripting](BASH_SCRIPTING_GUIDE.md)**; here we cover the admin essentials and the **prime directive of safe scripting**: start every serious script with `set -euo pipefail`.

```bash
#!/usr/bin/env bash
set -euo pipefail        # ★ THE safety preamble for admin scripts:
#   -e            exit immediately if any command fails (don't blunder onward after an error)
#   -u            error on use of an UNSET variable (catches typos like $HOEM)
#   -o pipefail   a pipeline fails if ANY stage fails (not just the last) — e.g. catches a
#                 failing 'curl' in 'curl ... | tar ...'
IFS=$'\n\t'              # safer word-splitting (spaces in filenames won't bite you)

log() { printf '%(%Y-%m-%dT%H:%M:%S%z)T %s\n' -1 "$*"; }   # timestamped logging helper

readonly BACKUP_DIR="/var/backups/app"
log "Starting backup to ${BACKUP_DIR}"
mkdir -p "${BACKUP_DIR}"                          # always quote variables: "${VAR}"
tar -czf "${BACKUP_DIR}/app-$(date +%F).tar.gz" /opt/myapp/data
log "Backup complete"
```

Without `set -e`, a script whose `cd /important/dir` *failed* would happily run the next line (`rm -rf *`) in the wrong directory — a legendary disaster. The preamble turns silent, dangerous failures into loud, safe stops. **Always quote `"${variables}"`** to survive spaces and empty values, and lint your scripts with **`shellcheck`** (it catches a remarkable share of bugs before they run).

### 12.2 cron — the classic scheduler **[I]**

**cron** runs commands on a schedule. Each user has a **crontab**; there are also system crontabs in `/etc/cron.d/` and convenience directories `/etc/cron.{hourly,daily,weekly,monthly}/` (drop a script in and it just runs). A crontab line is **five time fields + a command**:

```cron
# ┌──────── minute (0–59)
# │ ┌────── hour (0–23)
# │ │ ┌──── day of month (1–31)
# │ │ │ ┌── month (1–12)
# │ │ │ │ ┌ day of week (0–7, 0 & 7 = Sunday)
# │ │ │ │ │
  30 2 * * *   /usr/local/bin/backup.sh           # every day at 02:30
  */15 * * * * /usr/local/bin/healthcheck.sh       # every 15 minutes
  0 4 * * 0    /usr/local/bin/weekly-report.sh      # Sundays at 04:00
  @reboot      /usr/local/bin/on-boot.sh            # once, at every boot
```

```bash
crontab -e               # edit YOUR crontab (use this, not a raw file)
crontab -l               # list your cron jobs
sudo crontab -e -u www-data   # edit another user's crontab
```

**Cron gotchas that bite everyone:** cron runs with a **minimal environment** (a bare `PATH`, no profile sourced), so **always use absolute paths** (`/usr/bin/python3`, not `python3`) and set any needed env explicitly. **Output is emailed** to the user (and lost if mail isn't set up) unless you redirect it — so redirect to a log: `>> /var/log/myjob.log 2>&1`. And cron **silently skips** jobs if the machine was off at the scheduled time — if that matters, use a systemd timer with `Persistent=true` (§7.7) instead.

### 12.3 The environment & shell startup files **[I]**

Understanding *where environment variables and PATH come from* saves hours of "it works in my shell but not in cron/the service." Login shells read `/etc/profile` then `~/.bash_profile`/`~/.profile`; interactive non-login shells read `/etc/bash.bashrc` then `~/.bashrc`; **systemd services and cron read NONE of these** — they start with a near-empty environment. That's why a service needs its env spelled out via `Environment=`/`EnvironmentFile=` (§7.4) and cron needs absolute paths. System-wide environment for everyone goes in `/etc/environment` (simple `KEY=value`, no scripting) or drop-ins in `/etc/profile.d/*.sh`.

---

## 13. Logging & Monitoring

### 13.1 Why logging & monitoring are non-negotiable **[I]**

You cannot operate what you cannot see. **Logs** tell you *what happened* (an error, an auth failure, a deploy); **metrics** tell you *how the system is behaving over time* (CPU, memory, request rate, latency); **alerts** tell you *something is wrong* before (ideally) your users do. Without these, you're flying blind: outages surprise you, root-cause analysis is guesswork, and capacity planning is a wish. The progression in this section: master the local logs (journald + the `/var/log` files), keep them from filling the disk (logrotate), ship them somewhere central, then layer metrics and alerting on top.

### 13.2 Where logs live: journald and /var/log **[B/I]**

Two overlapping systems coexist. **journald** (§7.6) holds structured, queryable logs for everything systemd supervises — your first stop (`journalctl -u <svc>`). Many systems *also* run a traditional syslog daemon (**rsyslog**, sometimes `syslog-ng`) that writes plain-text files under **`/var/log`**, which lots of tooling and humans still expect:

| Path | What it holds |
|---|---|
| `/var/log/syslog` (Deb) / `/var/log/messages` (RHEL) | General system messages |
| `/var/log/auth.log` (Deb) / `/var/log/secure` (RHEL) | **Authentication & sudo** — check after any security event |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dpkg.log` / `/var/log/dnf.log` | Package operations |
| `/var/log/nginx/`, `/var/log/postgresql/` | Per-application logs |

```bash
sudo tail -f /var/log/auth.log         # watch auth events live (failed logins, sudo use)
sudo grep -i 'fail\|invalid' /var/log/auth.log | tail   # hunt brute-force attempts
journalctl -p warning -b               # the journald equivalent across all services
```

### 13.3 rsyslog & centralized logging **[I/A]**

`rsyslog` routes log messages by **facility** (auth, cron, mail, kern, local0–7) and **severity** (emerg…debug) to files, or **over the network to a central log server**. Centralizing logs is a production must: it lets you search across your whole fleet in one place, and — crucially for security — an attacker who compromises a box **can't erase the logs that are already shipped off it**. The modern stack is usually an agent (rsyslog, **Vector**, **Fluent Bit**, or **Promtail**) shipping into **Loki/Grafana**, the **ELK/OpenSearch** stack, or a SaaS.

```rsyslog
# /etc/rsyslog.d/60-remote.conf — forward ALL logs to a central server over TCP (@@ = TCP)
*.*  @@logserver.internal:514
# (Authenticate & encrypt this with TLS in production; plain :514 is for trusted networks only.)
```

### 13.4 Log rotation with logrotate **[I]**

Logs grow forever and will **fill your disk** — a classic, embarrassing outage ("server down: `/var` is 100% full of logs"). **`logrotate`** (run daily via cron/timer) periodically rotates logs: renames the current file, compresses old ones, deletes ones past a retention limit, and signals the service to reopen its log. Packages drop their config in `/etc/logrotate.d/`:

```
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily                  # rotate every day
    rotate 14              # keep 14 old files, then delete (≈ two weeks retention)
    compress               # gzip rotated logs to save space
    delaycompress          # don't compress the most-recent rotation (it may still be written)
    missingok              # don't error if the log is absent
    notifempty             # skip rotation if the log is empty
    create 0640 myapp myapp   # recreate the fresh log with these perms/owner
    sharedscripts
    postrotate             # after rotating, tell the app to reopen its log file
        systemctl reload myapp >/dev/null 2>&1 || true
    endscript
}
```

```bash
sudo logrotate -d /etc/logrotate.d/myapp   # -d = DEBUG/dry-run: see what WOULD happen
sudo logrotate -f /etc/logrotate.conf      # -f = force a rotation now (testing)
```

(journald rotates *itself* by size — `SystemMaxUse=` in `journald.conf`, §7.6 — so logrotate is for the `/var/log` text files.)

### 13.5 Metrics: Prometheus, node_exporter, Grafana (overview) **[I/A]**

Logs are events; **metrics** are numeric time-series you watch for trends and alerting. The de-facto open-source stack:

- **node_exporter** runs on each server and exposes hundreds of host metrics (CPU, memory, disk, network, filesystem fullness, load) over HTTP at `/metrics`.
- **Prometheus** is a central server that **scrapes** each exporter on an interval, stores the time-series, and runs an alerting-rule engine (**PromQL** queries). You also expose **app metrics** (request rate, error rate, latency) from your services in the same format.
- **Grafana** is the dashboard/visualization layer querying Prometheus (and logs from Loki) — the "single pane of glass."
- **Alertmanager** takes Prometheus alerts and routes/deduplicates them to email, Slack, PagerDuty, etc.

```bash
# A typical install runs node_exporter as a hardened systemd service:
# ExecStart=/usr/local/bin/node_exporter --web.listen-address=127.0.0.1:9100
# Bind it to LOCALHOST and scrape over a private network/VPN — never expose /metrics publicly.
curl -s localhost:9100/metrics | head    # see the raw metrics it serves
```

### 13.6 What to actually watch & alert on **[I/A]**

The hardest part isn't collecting metrics — it's choosing *which* to alert on so you get paged for real problems, not noise. The widely-used frameworks are the **USE method** (for resources: **U**tilization, **S**aturation, **E**rrors) and the **RED method** (for services: **R**ate, **E**rrors, **D**uration). A solid baseline alert set for a server:

| Watch | Alert when | Why |
|---|---|---|
| **Disk space** (`/`, `/var`) | > 85% full | The #1 avoidable outage |
| **Disk inodes** | > 85% used | Out of inodes ≈ out of disk |
| **Memory available** | persistently low / swapping hard | Imminent OOM kills |
| **Load average / CPU** | sustained > core count | Saturation, queuing |
| **Service up?** | a critical unit is `failed`/down | The service itself died |
| **HTTP error rate** | 5xx rate spikes | Users are seeing failures |
| **Latency (p95/p99)** | crosses your SLO | Slowness before it becomes downtime |
| **Certificate expiry** | < 14 days | Avoid the classic "site down — cert expired" |
| **Failed logins** | abnormal spike | Possible attack in progress |

**Alert on *symptoms users feel* (error rate, latency, "is the page up"), not just causes** — and make every alert *actionable* (it points to a runbook, §19). Pages that fire for non-problems train people to ignore alerts, which is how real incidents get missed.

---

## 14. Web & App Serving in Production

### 14.1 The production web stack shape **[I]**

Almost every production web service follows the same topology: the public internet hits a **reverse proxy** (Nginx, sometimes a cloud load balancer) on ports 80/443; the proxy **terminates TLS**, serves static files, and **forwards** dynamic requests to one or more **application processes** (your Go/Node/Python/Java app) listening on a private localhost port; the app talks to a **database** on a private network. Each tier runs as an unprivileged user under systemd, behind the firewall, with only 80/443 (and SSH) exposed. This section ties the pieces together; the deep dives live in **[Nginx](NGINX_GUIDE.md)**, **[Networking](NETWORKING_GUIDE.md)**, **[PostgreSQL](POSTGRESQL_GUIDE.md)**, and **[Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)**.

### 14.2 Reverse proxy: why you don't expose your app directly **[I]**

You almost never let the public talk straight to your application process. A **reverse proxy** in front buys you: **TLS termination** (one place to manage certs, modern ciphers, HTTP/2/3 — §14.4), **serving static assets** efficiently (don't burn app workers on CSS), **load balancing** across app replicas (§14.5), **buffering** slow clients so they don't tie up app workers, **rate limiting and basic WAF** protection, clean **request logging**, and the freedom to restart/redeploy the app behind a stable front door. Nginx is the canonical choice — see the **[Nginx](NGINX_GUIDE.md)** guide for the full configuration. The skeleton:

```nginx
# /etc/nginx/sites-available/myapp  — proxy HTTPS public traffic to a local app on :8080
upstream myapp { server 127.0.0.1:8080; keepalive 32; }   # the app, on private localhost

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://myapp;
        proxy_set_header Host $host;                         # pass the original Host...
        proxy_set_header X-Real-IP $remote_addr;             # ...and the real client IP...
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;          # so the app knows it's HTTPS
    }
}
server { listen 80; server_name app.example.com; return 301 https://$host$request_uri; }  # force HTTPS
```

### 14.3 Running your app as a service **[I]**

Your application must run as a supervised systemd service (§7.4): an **unprivileged user**, **auto-restart on crash**, **start at boot**, **logs to journald**, **resource caps**, and **sandboxing**. It should **bind to `127.0.0.1`** (localhost), not `0.0.0.0`, so only the local Nginx can reach it — the firewall then never even needs a rule for the app port. Secrets come from a `chmod 600` `EnvironmentFile=` (§7.4, §11.6). This is the single most important "make it production" step beyond the proxy.

### 14.4 TLS certificates with Let's Encrypt / certbot **[I/A]**

Public HTTPS requires a TLS certificate signed by a trusted CA. **Let's Encrypt** issues them **free and automatically** via the **ACME** protocol; **`certbot`** (or `acme.sh`, or Caddy's built-in ACME) is the client that requests, installs, and **auto-renews** them. Certs are valid for **90 days**, so **automated renewal is mandatory** — the renewal timer is the part people forget, and "site down: certificate expired" is an avoidable classic.

```bash
sudo apt install certbot python3-certbot-nginx        # (or: snap install certbot)
sudo certbot --nginx -d app.example.com -d www.app.example.com   # obtain + auto-configure Nginx
# certbot installs a systemd TIMER that renews automatically. VERIFY it actually works:
sudo certbot renew --dry-run                           # ★ simulate renewal end-to-end
systemctl list-timers | grep certbot                   # confirm the renewal timer is scheduled
```

For wildcard certs or hosts not reachable on port 80, use the **DNS-01 challenge** (certbot proves control by creating a DNS TXT record via your provider's API) instead of HTTP-01. Harden the resulting TLS config (TLS 1.2+ only, modern ciphers, OCSP stapling, HSTS) per the **[Nginx](NGINX_GUIDE.md)** and **[Networking](NETWORKING_GUIDE.md)** guides; verify it offline with `openssl s_client -connect app.example.com:443`.

### 14.5 Zero-downtime deploys **[I/A]**

A deploy shouldn't drop requests. The common patterns, in increasing sophistication:

- **Reload, not restart.** Nginx (`systemctl reload nginx`) and many apps re-read config or gracefully drain on `SIGHUP` with no dropped connections. Always prefer reload.
- **Atomic symlink swap** (§3.4) for releases: upload to `/opt/app/releases/<sha>/`, repoint the `current` symlink atomically, restart the worker. A failed build never touches the live path; rollback is repointing the symlink.
- **Rolling restart** across replicas: with multiple app instances behind the proxy, drain and restart them **one at a time** so capacity stays up. The proxy's health checks route traffic away from the draining instance.
- **Blue-green / canary:** run the new version alongside the old (blue=current, green=new), shift a fraction of traffic to it, watch error rate/latency, then cut over fully or roll back. Often orchestrated by your **[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)** pipeline.

The non-negotiable enabler is a **graceful-shutdown** app: on `SIGTERM` it stops accepting new connections, finishes in-flight requests, then exits — so systemd/the orchestrator can cycle it without dropping work.

### 14.6 Environment & config separation **[I]**

Per the Twelve-Factor app principles, keep **config in the environment**, not the code: the same artifact (binary/image) runs in dev, staging, and prod, differing only by environment variables and mounted secrets. On a single server that's a `chmod 600` `EnvironmentFile=` per environment; at scale it's the config layer of your orchestrator or a secrets manager (§11.6). This makes promotion between environments safe and rollbacks clean.

---

## 15. Containers on a Server

### 15.1 Why containers, and how they relate to the host **[I]**

A **container** packages an application with *all* its dependencies into one image that runs identically anywhere, isolated from the host and from other containers — but, unlike a VM, **sharing the host kernel**, so it's far lighter (megabytes, millisecond start) than a full virtual machine. On a server this solves "works on my machine" forever, makes deployments atomic and rollback trivial (just run the previous image tag), and cleanly isolates dependencies. The isolation is built from the same kernel primitives you've already met: **namespaces** (separate views of processes, network, mounts, users) and **cgroups** (§7.8, resource limits). This is an operations overview; the full guide is **[Docker](DOCKER_GUIDE.md)**.

### 15.2 Docker vs Podman, and the rootless argument **[I/A]**

Two main runtimes. **Docker** is ubiquitous and runs a privileged daemon (`dockerd`) as **root** — which means **anyone in the `docker` group effectively has root** on the host (they can mount `/` into a container), a frequently-overlooked privilege-escalation risk. **Podman** (default on RHEL family) is **daemonless** and runs **rootless** by default: containers run as your unprivileged user via user namespaces, so a container breakout doesn't hand over root. For new server deployments, **rootless Podman** is the more secure default; Docker can also run rootless. Their CLIs are nearly identical (`alias docker=podman` usually works).

```bash
docker run -d --name web -p 127.0.0.1:8080:80 \      # bind to LOCALHOST, not 0.0.0.0...
  --restart=unless-stopped \                          # ...auto-restart...
  --memory=512m --cpus=1 \                            # ...with resource limits (cgroups)...
  --read-only --cap-drop=ALL \                        # ...read-only rootfs, drop all capabilities
  nginx:1.27-alpine
docker ps                                             # running containers
docker logs -f web                                    # follow a container's logs
docker exec -it web sh                                # get a shell inside (debugging)
# Podman: replace 'docker' with 'podman' above — same flags.
```

### 15.3 Running containers under systemd (the production pattern) **[I/A]**

On a server you want containers supervised like any other service — start at boot, restart on failure, log to the journal. The clean way is to let systemd manage them. With **Podman**, **Quadlet** (a `.container` unit type) generates a proper systemd service from a simple declarative file:

```ini
# /etc/containers/systemd/myapp.container   (Podman Quadlet — systemd generates the service)
[Container]
Image=registry.example.com/myapp:1.4.2
PublishPort=127.0.0.1:8080:8080
Environment=NODE_ENV=production
Memory=512M
Volume=/srv/myapp/data:/data:Z      # ':Z' sets the right SELinux label on the volume

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp           # systemd now supervises the container like any service
journalctl -u myapp -f                       # its logs are in the journal
```

(With Docker, you instead write a normal `.service` that runs `docker run` with `Restart=` and proper `ExecStop=`, or use Docker Compose with a wrapping unit.)

### 15.4 Registries & images **[I]**

Container images live in **registries** — Docker Hub, GitHub Container Registry (`ghcr.io`), cloud registries, or a self-hosted one. Your **[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)** pipeline builds an image, tags it (by git SHA *and* a semantic version — avoid relying on `latest` in production, which is ambiguous and unrepeatable), pushes it to the registry, and the server pulls that exact tag. **Scan images for vulnerabilities** (Trivy, Grype) in the pipeline, prefer **minimal base images** (`alpine`, `distroless`) to shrink attack surface, and **never bake secrets into an image** (anyone who pulls it can extract them — §11.6).

```bash
docker login ghcr.io                          # authenticate to a registry
docker pull ghcr.io/acme/myapp:1.4.2          # pull a SPECIFIC, immutable tag
docker image ls                               # local images & their sizes
docker system prune -af                       # ★ reclaim disk from stopped containers/old images
```

> **Beyond a single host:** when you outgrow one server, container orchestration (**Kubernetes**, Nomad, or Docker Swarm) schedules containers across a fleet, handles rolling updates, self-healing, and service discovery. That's a large topic of its own; the single-host patterns here are the foundation it builds on.

---

## 16. Backups & Disaster Recovery

### 16.1 The mindset: you don't have backups until you've restored one **[I]**

Backups are insurance against the day something goes irreversibly wrong — a bad `rm`, ransomware, a failed disk, a botched migration, a datacenter fire. The brutal truth every veteran learns: **an untested backup is not a backup, it's a hope.** Backups silently fail constantly — the job errored months ago, the archive is corrupt, a critical directory was excluded, the encryption key is lost. **The only thing that proves a backup works is restoring it.** So the discipline is: automate backups, *and* automate (or at least schedule) **restore drills**, and treat a successful restore — not a successful backup job — as the success metric.

### 16.2 The 3-2-1 rule **[I]**

The durable, decades-proven standard: **3** copies of your data, on **2** different media/systems, with **1** copy **off-site** (and ideally **off-line/immutable** to survive ransomware that encrypts everything it can reach). Concretely for a server: the live data (copy 1), a local/nearby backup for fast restores (copy 2), and an off-site copy in another region or provider (copy 3, off-site). Add **immutability** (object-lock / write-once storage) so an attacker who gets root can't delete your backups too.

### 16.3 RPO & RTO — the two numbers that drive everything **[I]**

Two objectives define your backup strategy and budget:

- **RPO (Recovery Point Objective):** how much *data* you can afford to lose, measured in time. RPO = 24h means daily backups are fine; RPO = 5 min demands continuous replication / frequent WAL shipping. It dictates **backup frequency**.
- **RTO (Recovery Time Objective):** how long you can afford to be *down* while restoring. RTO = 1h means your restore process must be fast and rehearsed; RTO = 1 week tolerates slow cold storage. It dictates **restore architecture** (warm standby vs cold archive).

Define these *per dataset* with the business, then design backups to meet them — and **test that you actually hit the RTO** in a drill.

### 16.4 Tools: rsync, restic, borg **[I/A]**

| Tool | Model | Best for |
|---|---|---|
| **rsync** | File mirror/copy (delta transfer) | Simple mirrors, staging, moving data; *not* versioned by itself |
| **borg** (BorgBackup) | Deduplicating, compressed, encrypted, versioned local/SSH repo | Efficient versioned backups to your own storage |
| **restic** | Like borg, but first-class **cloud/object-store** backends (S3, B2, etc.) | Encrypted, deduplicated backups straight to the cloud |

**restic** and **borg** both deduplicate (store each unique chunk once, so 30 daily snapshots cost far less than 30 full copies), compress, and **encrypt** at rest — the modern default. Example with restic to an off-site S3-compatible store:

```bash
export RESTIC_REPOSITORY="s3:https://s3.example.com/mybackups"
export RESTIC_PASSWORD_FILE="/etc/restic/password"     # chmod 600, NEVER lose this key
restic init                                            # one-time: create the encrypted repo
restic backup /etc /home /var/lib/myapp \              # back up these paths...
       --exclude-caches --exclude '*.tmp'              # ...skipping junk
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune   # retention policy
restic snapshots                                       # list recovery points
restic check                                           # ★ VERIFY repo integrity (run regularly)
# THE PART PEOPLE SKIP — practice the restore:
restic restore latest --target /tmp/restore-test --include /etc/ssh/sshd_config
```

Wire the backup into a systemd timer (§7.7) with `Persistent=true`, alert if it fails, and run `restic check` and a sample restore on a schedule.

### 16.5 Database backups need special care **[I/A]**

You **cannot** safely back up a live database by copying its files — you'll capture a torn, inconsistent state mid-write. Databases provide **consistent** backup tools: `pg_dump`/`pg_basebackup` + WAL archiving for PostgreSQL, `mysqldump`/Percona XtraBackup for MySQL, filesystem/LVM **snapshots** for a crash-consistent point-in-time. For low RPO, PostgreSQL's **continuous archiving (WAL) + Point-In-Time Recovery (PITR)** lets you restore to *any moment*. This is important enough to have its own guide — see **[PostgreSQL](POSTGRESQL_GUIDE.md)** and **[Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)**.

### 16.6 Snapshots are not backups **[I]**

Filesystem/LVM/cloud **snapshots** (§8.6) are fast and great for quick rollback, but a snapshot usually lives on (or beside) the same storage as the original — so a disk failure, a deleted volume, or a region outage destroys both. Snapshots are a *complement* to, not a *replacement* for, real off-site backups. Use snapshots for instant local recovery and the speed they give a *consistent* backup; use 3-2-1 off-site backups for actual disaster recovery.

### 16.7 The disaster-recovery plan **[I/A]**

A DR plan is the documented, *tested* procedure to rebuild service from nothing: where the backups are and how to access them (and where the **encryption keys** are — losing the key loses the backup), the exact restore steps, the order to bring systems up (database before app before proxy), how DNS/load-balancers get repointed, who does what, and the target RTO/RPO. Rehearse it — a game-day where you restore into a clean environment — because the worst time to discover your runbook is wrong is during a real outage. Store the DR plan somewhere reachable when your infrastructure is *down* (not only on the server that's on fire).

---

## 17. Performance Tuning & Observability

### 17.1 The method: measure, find the bottleneck, fix, re-measure **[A]**

The cardinal rule of performance work: **never tune blind.** Premature, guess-based tuning usually makes things worse and always wastes time. The loop is: **measure** to establish a baseline, **find the actual bottleneck** (it's always one of four resources — CPU, memory, disk I/O, or network), **change one thing**, **re-measure** to confirm it helped, repeat. A system is bottlenecked on exactly one resource at a time; your whole job is to find *which* before touching anything. Brendan Gregg's **USE method** (for every resource, check **U**tilization, **S**aturation, **E**rrors) is the systematic way to localize it.

### 17.2 The observability toolkit **[A]**

```bash
vmstat 1                 # ★ system-wide: CPU (us/sy/id/wa), memory, swap in/out, per second.
                         #   High 'wa' (I/O wait) → disk-bound. High 'sy' → kernel/syscall heavy.
                         #   Nonzero si/so (swap in/out) → memory pressure, you're thrashing.
iostat -xz 1             # ★ PER-DISK: %util (≈100 = saturated), await (latency ms), r/s w/s.
                         #   (from the 'sysstat' package)
mpstat -P ALL 1          # per-CPU-core utilization — spot a single hot core (single-threaded bottleneck)
sar -u 1 3               # historical/sampled stats; 'sar' records system activity over time
pidstat 1                # per-PROCESS CPU/IO/memory — "WHICH process is the hog?"
free -h                  # memory & swap (§6.6)
ss -s                    # socket summary (§9)
```

**The diagnostic flow:** `top`/`vmstat` says *which resource* is the problem; then the specialist tool (`iostat` for disk, `mpstat` for per-core CPU, `pidstat` for which process, `ss` for network) says *where exactly*. **High load + high `wa` + a disk at ~100% `%util`** = disk-bound; the fix is faster storage, less I/O, or more caching — *not* a bigger CPU.

> ⚡ **Modern tooling:** on recent kernels, **eBPF**-based tools (`bcc`, `bpftrace`, and the `*-bpfcc` scripts like `execsnoop`, `biolatency`, `tcptop`) give low-overhead, deep visibility into exactly what the kernel is doing — the cutting edge of Linux observability when the classic tools aren't precise enough.

### 17.3 Tuning levers: sysctl, ulimits, I/O scheduler **[A]**

Once you've *proven* the bottleneck, the relevant knobs:

```ini
# /etc/sysctl.d/99-perf.conf  — apply with: sudo sysctl --system
vm.swappiness = 10                       # swap less eagerly; keep working set in RAM (server default)
vm.dirty_ratio = 10                      # cap dirty (unwritten) page cache before forcing writeback
net.core.somaxconn = 4096                # bigger listen backlog for high-connection-rate servers
net.ipv4.tcp_max_syn_backlog = 8192      # tolerate connection bursts
net.core.netdev_max_backlog = 16384      # NIC packet-queue depth under heavy ingress
fs.file-max = 2097152                    # system-wide open-file-descriptor ceiling
```

**File-descriptor limits (`ulimit`)** routinely bite network servers: each connection/open file uses a descriptor, and the per-process default (often 1024) is far too low for a busy web/DB server — you hit *"too many open files"* and connections fail. Raise it via the service's `LimitNOFILE=` (systemd, §7.4 — the right place for services) or `/etc/security/limits.conf` (for login shells, enforced by PAM's `pam_limits`).

```
# /etc/security/limits.conf — raise limits for the app user's login sessions
appsvc   soft   nofile   65536
appsvc   hard   nofile   65536
```

The **I/O scheduler** (`/sys/block/<dev>/queue/scheduler`) orders disk requests: use **`none`/`mq-deadline`** for SSDs/NVMe (they don't benefit from the heavy reordering spinning disks needed), **`bfq`** where fairness matters. Modern distros usually pick well automatically.

### 17.4 Benchmarking — measure before *and* after **[A]**

To know a change helped, benchmark the same way before and after, under realistic load:

```bash
# HTTP load: requests/sec and latency distribution
wrk -t4 -c100 -d30s https://app.example.com/    # 4 threads, 100 connections, 30s
ab  -n 10000 -c 100 https://app.example.com/     # Apache Bench (simpler)
# Disk throughput/IOPS/latency (careful: --filename to a SCRATCH path, fio writes real data):
fio --name=randread --rw=randread --bs=4k --size=1G --numjobs=4 --runtime=30 --filename=/scratch/testfile
# CPU/memory stress for headroom testing:
stress-ng --cpu 4 --vm 2 --vm-bytes 1G --timeout 60s
```

Benchmark a **realistic** workload (your real request mix), not a synthetic one that hits a code path you don't care about — and change one variable at a time so you know what moved the number.

---

## 18. Configuration Management & IaC at Scale

### 18.1 Why automate: from pets to cattle **[A]**

Everything so far assumed you could SSH into a box and run commands. That breaks the moment you have *more than a handful* of servers. Configuring machines by hand doesn't scale and produces **configuration drift** — each box accumulates undocumented, slightly different tweaks until none are reproducible and "it works on server 3 but not server 7" becomes unanswerable. The cultural shift is **"pets vs cattle":** a *pet* is a hand-raised, irreplaceable, named server you nurse back to health; *cattle* are identical, disposable, numbered servers you destroy and recreate without a second thought. Production aims for **cattle**: machines defined entirely by code, so any one can be rebuilt from scratch, identically, in minutes. **Infrastructure as Code (IaC)** makes the configuration a versioned, reviewed, testable artifact in **[Git](GIT_GUIDE.md)** — your infrastructure becomes as reproducible and auditable as your application code.

### 18.2 The two layers: provisioning vs configuration **[A]**

Two complementary kinds of tooling, often used together:

- **Provisioning / infrastructure (Terraform, OpenTofu, Pulumi, CloudFormation):** *creates* the infrastructure itself — VMs, networks, load balancers, DNS, databases — declaratively from code against cloud APIs. "Make 5 servers, a load balancer, and a database exist."
- **Configuration management (Ansible, Puppet, Chef, Salt):** *configures what's inside* a machine — installs packages, writes config files, manages services, applies your hardening. "On those 5 servers, install Nginx, deploy the app, and apply the security baseline."
- **cloud-init** is the bridge: a standard that runs at a VM's *first boot* to do initial setup (create users, install your SSH key, run a bootstrap script), commonly used to hand a fresh instance off to Ansible.

### 18.3 Idempotence — the central idea **[A]**

The defining property of good config-management is **idempotence**: running the same configuration **any number of times** converges the system to the same desired state, changing nothing if it's already correct. A shell script that does `useradd bob` *fails* the second time (the user exists); the idempotent equivalent says "ensure user bob *exists*" and is a safe no-op when it already does. This is *why* you use Ansible instead of shell scripts at scale: you can run it repeatedly and safely to *enforce* state (and to detect drift — anything it changes was something that had drifted), without fear of "running it twice breaks things."

### 18.4 Ansible — the pragmatic standard **[A]**

**Ansible** is the most popular choice because it's **agentless** (it just SSHes in and runs Python — nothing to install on the targets beyond Python and SSH) and uses readable **YAML**. Its pieces: an **inventory** (the list of hosts, grouped), **playbooks** (YAML describing the desired state via **tasks**), **modules** (the idempotent building blocks — `apt`, `copy`, `template`, `service`, `user`…), **roles** (reusable bundles of tasks/templates/handlers), and **handlers** (actions like "restart nginx" that run only when something they're notified by actually changed). It turns your whole §1–§17 hardening checklist into versioned code applied identically to every box.

```yaml
# inventory.yml — the fleet
all:
  children:
    web:
      hosts:
        web1.example.com:
        web2.example.com:
```

```yaml
# webserver.yml — a playbook: the DESIRED STATE of every 'web' host
- hosts: web
  become: true                          # use sudo/privilege escalation
  vars:
    app_port: 8080
  tasks:
    - name: Ensure required packages are present     # idempotent: installs only if missing
      ansible.builtin.apt:
        name: [nginx, fail2ban, ufw]
        state: present
        update_cache: true

    - name: Deploy the Nginx site config from a template
      ansible.builtin.template:
        src: nginx-site.j2                # a Jinja2 template (can reference {{ app_port }})
        dest: /etc/nginx/sites-available/myapp
        mode: "0644"
      notify: reload nginx                # only fires the handler IF this task changed something

    - name: Ensure nginx is started and enabled at boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Default-deny firewall, allow SSH + web
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
      loop: ["22", "80", "443"]

  handlers:
    - name: reload nginx                  # runs at most once, at the end, only if notified
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```bash
ansible-inventory -i inventory.yml --list      # sanity-check the inventory
ansible all -i inventory.yml -m ping            # ★ can I reach every host? (connectivity test)
ansible-playbook -i inventory.yml webserver.yml --check --diff   # ★ DRY-RUN: what WOULD change?
ansible-playbook -i inventory.yml webserver.yml                  # apply for real
```

The `--check --diff` dry-run is the safety habit: it shows exactly what *would* change before you commit, and on a stable fleet it should report **zero changes** — anything it *would* change is configuration drift you've just caught.

### 18.5 Immutable infrastructure **[A]**

The most advanced posture takes "cattle" to its conclusion: **immutable infrastructure** — you never modify a running server *at all*. To change anything (even a patch), you **build a brand-new machine image** (with Packer, or a container image) carrying the change, deploy fresh instances from it, shift traffic over, and destroy the old ones. Benefits: **zero drift** (every instance is byte-identical to a tested image), trivial rollback (redeploy the previous image), and no "snowflake" servers nobody dares touch. Containers (§15) are immutable infrastructure at the process level; image-based VM deploys apply the same idea to whole machines. This pairs naturally with your **[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)** pipeline, which builds and publishes the images.

---

## 19. Production Operations & Company-Level Practices

### 19.1 What "company-level" actually means **[A]**

The gap between "I can configure a server" and "I run production for a company" is **process and discipline**, not more commands. At company scale, servers carry real money, real user data, and real legal obligations; mistakes have consequences; and *you are not the only person* touching the systems. The practices below exist to make operations **reliable, repeatable, auditable, and survivable** — so the system keeps running across team changes, scales without chaos, recovers fast from incidents, and satisfies auditors. This is the **SRE/DevOps** discipline: treating operations as an engineering problem.

### 19.2 Change management **[A]**

Most outages are **caused by a change** (a deploy, a config edit, an "I'll just quickly fix this in prod"). Mature orgs therefore control change: changes are **code-reviewed** and merged via **[Git](GIT_GUIDE.md)** (no editing prod by hand), deployed through an **automated, tested pipeline** (**[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)**) rather than manual SSH, made during agreed **windows** with a written **rollback plan**, and (for risky ones) reviewed by peers. The cultural anchor: **no manual changes to production** — if it's worth doing, it's worth doing in code so it's reviewed, repeatable, and revertible. This is exactly what IaC (§18) enables.

### 19.3 Runbooks & documentation **[A]**

A **runbook** is a step-by-step procedure for a specific operational task or failure: "the disk is full," "rotate the TLS cert," "fail over the database," "restart the payment service." Its purpose is to let *anyone* on the team — including a half-asleep on-call engineer at 3 a.m. — execute correctly without the original author's tribal knowledge. Good runbooks are **specific** (exact commands, exact paths), **tested** (someone other than the author followed them successfully), and **kept current**. Every alert (§13.6) should link to a runbook. Documentation more broadly — architecture diagrams, network maps, the DR plan (§16.7), who-owns-what — is what turns a fragile pile of servers into an operable system, and is essential when people leave or join.

### 19.4 On-call & incident response **[A]**

Production runs 24/7, so someone is **on-call** to respond when alerts fire. A healthy on-call rotation shares the load fairly, **pages only for actionable, user-impacting problems** (alert fatigue from noise is dangerous — it trains people to ignore pages), and has clear **escalation** paths. When an incident hits, a standard **incident-response** flow keeps it from becoming chaos: **detect** (alert/report) → **assess severity** → appoint an **incident commander** (coordinates; doesn't fix) → **mitigate first** (stop the bleeding — roll back, fail over, scale up — *before* root-causing) → **communicate** (status updates to stakeholders) → **resolve** → **blameless post-mortem**. The **blameless** post-mortem is cultural bedrock: you analyze *what in the system* allowed the failure and how to prevent the class of it, **without blaming individuals** — because blame makes people hide problems, and hidden problems recur.

### 19.5 Patching cadence & compliance **[A]**

A defined **patching cadence** keeps the fleet current without chaos: critical security patches fast (automated, §11.3), routine updates on a regular cycle (e.g., monthly), tested in staging, rolled out progressively (canary → fleet) with monitoring. **Compliance** frameworks — **SOC 2**, **ISO 27001**, **PCI-DSS** (payment data), **HIPAA** (health data), **GDPR** (EU personal data) — impose concrete, auditable requirements: enforced access controls, audit logging (§11.5), encryption at rest and in transit, tested backups (§16), documented change management, and evidence that you actually do all of it. The practical upshot: build these controls in from the start (it's far cheaper than retrofitting), and keep the *evidence* (logs, tickets, IaC history) auditors will ask for.

### 19.6 Capacity planning & cost **[A]**

Operating well also means the system **has enough headroom** before it's overwhelmed and **doesn't waste money** on idle capacity. Use your metrics (§13) to track growth trends, plan capacity ahead of need (don't discover the limit during a traffic spike), and right-size resources. Autoscaling handles bursty load automatically where the platform supports it; FinOps practices keep cloud spend visible and accountable. The goal is the balance point: reliable under peak load, not paying for peak capacity 24/7.

### 19.7 The operational mindset, distilled **[A]**

Everything above reduces to a few habits: **automate everything** (humans make mistakes; code is reviewable and repeatable — §12, §18); **measure everything** (you can't manage what you don't see — §13, §17); **document everything** (so the system outlives any individual — §19.3); **assume failure** (build for it, test your recovery, run game-days — §16); **least privilege and defense in depth** everywhere (§11); and **change carefully** (most outages are self-inflicted — §19.2). Master those, and you're not just administering Linux servers — you're *running production* the way a company needs.

---

## 20. Gotchas & Best Practices

### 20.1 Locking yourself out (the cardinal sins) **[I/A]**
- **Editing SSH/firewall/network config without a second way in.** A typo in `sshd_config`, an `nft` mistake, or a bad `netplan` can end your only session permanently. Always **keep your current session open**, **test in a second terminal**, use **`sshd -t`** / **`netplan try`** (auto-rollback) / **`nft -c`**, and on remote boxes arm a **`sudo shutdown -r +5`** you cancel only after confirming you're not locked out. Keep a **console** (cloud serial / hypervisor / IPMI) available.
- **`PermitRootLogin yes` + password auth** — the combination bots find within minutes. Key-only, no root, fail2ban (§2, §10).
- **`ufw enable` / `firewall enable` over SSH without first allowing SSH** — instant disconnect. Allow `22` (or `OpenSSH`) *before* enabling (§10.3).

### 20.2 Filesystem & storage footguns **[I/A]**
- **`rm -rf` with a stray space or variable.** `rm -rf "$DIR"/` when `$DIR` is empty becomes `rm -rf /`. Always quote, double-check, and prefer `--one-file-system`. Newer `rm` refuses `/` by default — don't rely on it.
- **A bad `/etc/fstab` makes the box fail to boot.** Always `sudo mount -a` to test *before* rebooting; add **`nofail`** to non-critical mounts; reference disks by **UUID**, never `/dev/sdX` (letters reorder).
- **`/var` (or `/`) fills up and everything breaks.** Logs, journald, Docker images, and old kernels are the usual culprits. Monitor disk at **85%** (§13.6), set up **logrotate** (§13.4) and `journald` size caps, and `docker system prune`. Watch **inodes** (`df -i`) too.
- **Confusing `df` and `du`:** a deleted-but-open file keeps space `df` sees but `du` doesn't — restart the process holding it (`lsof | grep deleted`).
- **RAID/snapshots are not backups.** They don't survive `rm -rf`, ransomware, or a region loss. Keep real, tested, off-site **3-2-1** backups (§16).

### 20.3 Permissions & users **[I/A]**
- **`chmod -R 777` to "fix" a permission problem.** Never. It's a security hole and rarely the real fix — find the actual owner/group/mode the app needs.
- **`usermod -G` without `-a`** wipes a user's other groups. Always `usermod -aG`.
- **`sudo` granting an editor/interpreter** (`sudo vim`, `sudo less`, `sudo python`) is effectively full root (shell escapes). Grant specific binaries with specific args; never `NOPASSWD: ALL`.
- **Private keys / secret files not `chmod 600`.** OpenSSH refuses over-permissive keys; secrets readable by others leak. `~/.ssh` = `700`, keys = `600`.

### 20.4 systemd & services **[I/A]**
- **Forgetting `systemctl daemon-reload` after editing a unit** — systemd keeps using the old version and you debug a ghost. Reload after every unit edit.
- **`restart` when you meant `reload`** drops connections needlessly. Prefer `reload` for nginx/config changes.
- **Editing vendor unit files in `/usr/lib/systemd/system`** — upgrades overwrite your change. Use `systemctl edit` drop-ins (§7.5).
- **`enable` ≠ `start`** (and vice-versa). `enable` is "at boot," `start` is "now." Use `enable --now` for both.
- **Drop-in override files on Ubuntu's `sshd_config.d/` silently override** the main file — check `sshd -T` for the *effective* config.

### 20.5 Networking & firewall **[I/A]**
- **Binding services to `0.0.0.0` when they should be `127.0.0.1`.** A database or admin port on all interfaces is a breach waiting to happen — bind app/DB ports to localhost or a private interface, never the public NIC (`ss -tulpn` to audit).
- **Editing `/etc/resolv.conf` on systemd-resolved systems** does nothing (it's a symlink). Use `resolvectl` / netplan (§9.4).
- **Cloud security group vs host firewall mismatch** — "why can't I connect?!" is often the *other* firewall. Check both layers (§10.1).
- **Forgetting to `systemctl enable nftables`/`ufw`** — your firewall vanishes on the next reboot.

### 20.6 Operations discipline **[I/A]**
- **Running long jobs without tmux/screen** — a dropped SSH session kills a half-finished migration/upgrade. Always wrap risky/long work (§2.9).
- **Manual changes to production** — undocumented, unreproducible, drift-inducing. Do it in code (Ansible/Git), reviewed and revertible (§18, §19.2).
- **Untested backups.** Schedule **restore drills**; a backup you've never restored is a guess (§16.1).
- **Certs that expire because the renewal timer wasn't verified.** `certbot renew --dry-run` and alert on `< 14 days` (§14.4, §13.6).
- **Disabling SELinux/AppArmor to "make it work."** Read the denial and fix the label/port/boolean instead (§11.4).
- **Noisy alerts** train people to ignore pages — alert only on actionable, user-impacting symptoms (§13.6).

### 20.7 The positive best-practice checklist **[I/A]**
- **Least privilege & defense in depth** everywhere: dedicated unprivileged service users, sandboxed systemd units, narrow sudo, dropped capabilities, MAC enforcing.
- **Automate everything**; treat servers as **cattle**, config as **code** in Git; aim for **idempotent**, drift-free, reproducible builds.
- **Default-deny firewall**, key-only SSH, automatic security updates, fail2ban — the baseline on *every* internet-facing box.
- **Measure everything** (metrics + logs + alerts) and **document everything** (runbooks, DR plan, architecture).
- **Test the scary paths** (restores, failovers, deploys) *before* you need them, in drills.
- **Change carefully**: reviewed, automated, reversible, in a window — most outages are self-inflicted.

---

## 21. Study Path & Build-to-Learn Projects

### 21.1 A suggested learning order

Work through this in order; each stage assumes the last. Do everything on a **cheap throwaway VM** (a small cloud instance, or a local VM in VirtualBox/Multipass/`virt-manager`) so you can break and rebuild it fearlessly — the willingness to destroy and recreate *is* the "cattle" mindset (§18).

1. **[B] Get in and look around.** Spin up an Ubuntu 24.04 (and separately a Rocky 10) VM. Master SSH key auth, `~/.ssh/config`, and tmux (§2). Navigate the FHS until `/etc`, `/var/log`, `/home` are reflexes (§3). Read `man` pages and `--help` constantly.
2. **[B] Users, permissions, packages.** Create human and service users, master `rwx`/octal/`chmod`/`chown`, set up `sudo` via `visudo` (§4). Install, search, update, and remove software with both `apt` and `dnf`; understand repos (§5).
3. **[B/I] Processes & systemd.** Read `top`/`htop`, send signals, understand load and memory (§6). Then live in `systemctl`/`journalctl`: start/enable services, read status, and **write your own hardened service unit** for a tiny app — the single most important hands-on skill in this guide (§7).
4. **[I] Storage & networking.** Add a virtual disk: partition it, make a filesystem, mount it, persist it in `fstab` by UUID, then put it under **LVM and grow it online** (§8). Configure a static IP with netplan/nmcli (using `netplan try` so you don't lock out), and learn `ip`/`ss`/`resolvectl` cold (§9).
5. **[I/A] Firewall & hardening.** Build a default-deny **nftables** ruleset (and the `ufw`/`firewalld` equivalents), add **fail2ban** (§10). Then do a full hardening pass: SSH lockdown, automatic updates, keep **SELinux/AppArmor enforcing** (debug a denial instead of disabling it), sysctl hardening, and run **Lynis** to grade yourself against CIS (§11).
6. **[I] Automation & scheduling.** Write `set -euo pipefail` admin scripts (lint with shellcheck), schedule with both **cron** and **systemd timers**, and feel the difference (§12, §7.7).
7. **[I/A] Production serving.** Put Nginx in front of your app, get a **Let's Encrypt** cert with auto-renewal, and verify zero-downtime reloads (§14). Then run the app as a **container** under systemd/Podman (§15).
8. **[I/A] Observability & backups.** Stand up node_exporter + Prometheus + Grafana, build a dashboard, and wire one real alert with a runbook (§13). Set up **restic** backups to off-site storage and — crucially — **restore one** (§16).
9. **[A] Performance & scale.** Load-test with `wrk`, find a bottleneck with `vmstat`/`iostat`/`pidstat`, fix it, and prove the win by re-measuring (§17).
10. **[A] Fleet & operations.** Re-express *everything you did by hand above* as an **Ansible** playbook, run it against two fresh VMs, and confirm `--check` reports zero drift (§18). Write the runbooks and a DR plan; practice an incident (§19).

### 21.2 Build-to-learn projects

- **Project 1 — The hardened baseline host.** From a fresh Ubuntu 24.04 *and* a fresh Rocky 10 VM, manually bring each to a production baseline: key-only SSH with no root login, a dedicated admin user with sudo, a default-deny firewall (nftables, then ufw/firewalld), fail2ban, automatic security updates, persistent journald, and sysctl hardening. Run **Lynis** before and after and watch the score climb. *Deliverable: a documented checklist of every change.*
- **Project 2 — Production web service, end to end.** Deploy a small app (a Go/Node/Python "hello + DB" service — cross-ref the language guides) as a **sandboxed systemd unit** running as an unprivileged user, behind **Nginx** with a **Let's Encrypt** cert and auto-renewal, talking to **PostgreSQL** on a private interface (cross-ref **[PostgreSQL](POSTGRESQL_GUIDE.md)** / **[Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)**). Achieve a **zero-downtime deploy** via atomic symlink swap or reload. *Deliverable: a fresh request never drops during a redeploy.*
- **Project 3 — Storage drill.** Add three virtual disks; build an **LVM** volume group across two, format and mount it via `fstab`/UUID, then **grow the filesystem online**. Separately, build an **mdadm RAID 1** mirror, *fail a disk on purpose*, and rebuild the array. Take an LVM snapshot and use it for a consistent backup. *Deliverable: a notebook of exactly which commands recovered each scenario.*
- **Project 4 — Observability stack.** Install node_exporter on two servers, Prometheus + Grafana to scrape them, and build a dashboard covering CPU, memory, disk fullness, load, and your app's request rate/error rate/p95 latency (USE + RED, §13.6). Wire **one** alert (disk > 85%) to fire and link a runbook. *Deliverable: a screenshot of the alert firing and the runbook that resolves it.*
- **Project 5 — Backup & disaster recovery.** Set up **restic** (or borg) backups of your app's data and configs to an off-site/object store, on a systemd timer, with retention and `restic check`. Then **simulate a disaster**: destroy the server entirely and **rebuild service from backups** on a brand-new VM, timing yourself against a target **RTO**. *Deliverable: a tested, written DR runbook and your measured restore time.*
- **Project 6 — Containerize it.** Repackage Project 2's app as a container image (cross-ref **[Docker](DOCKER_GUIDE.md)**), build and push it via **[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)**, scan it with Trivy, and run it under **rootless Podman + Quadlet** as a supervised systemd service with resource limits, behind the same Nginx. *Deliverable: a `git push` that ends with a new image running in production.*
- **Project 7 — Fleet as code (capstone).** Express the *entire* stack from Projects 1–2 (baseline hardening + web service + firewall + monitoring agent) as an idempotent **Ansible** project (inventory, roles, templates, handlers). Provision **two** identical fresh VMs and bring them both to full production state with one `ansible-playbook` run. Re-run it and confirm **zero changes** (no drift). Make one change *only in code*, review it like a PR in **[Git](GIT_GUIDE.md)**, and roll it out. *Deliverable: a Git repo that builds your whole production environment from nothing — pets become cattle.*

---

*End of the Linux Server Administration reference. You started at a nervous first SSH login and ended able to provision, secure, operate, observe, and automate production Linux fleets at company level. Cross-references throughout: **[Networking](NETWORKING_GUIDE.md)** (the protocols under §2/§9/§10), **[Nginx](NGINX_GUIDE.md)** (the reverse proxy in §14), **[Docker](DOCKER_GUIDE.md)** (containers in §15), **[Bash Scripting](BASH_SCRIPTING_GUIDE.md)** (the automation in §12), **[PostgreSQL](POSTGRESQL_GUIDE.md)** and **[Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)** (the data tier in §14/§16), **[Git](GIT_GUIDE.md)** (versioned infrastructure in §18/§19), and **[CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md)** (the pipeline in §14/§15/§18). The loop that builds mastery: provision a throwaway VM, harden it, break it, read the logs, restore it, then write the Ansible that does it all for you — and never touch production by hand.*
