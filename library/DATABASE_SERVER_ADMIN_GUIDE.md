# Database Server Administration (DBA) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I can write a `SELECT`" to "I can install, configure, secure, back up, replicate, monitor, tune, upgrade and operate a production database server — and be the person on call when it breaks at 3 a.m." — entirely offline. This is the **operations / administration** guide: it teaches the **DBA discipline** of *running database servers*, which is mostly engine-agnostic but shown concretely on **PostgreSQL** and **MySQL/MariaDB**, with notes for **MongoDB**, **Redis** and **Microsoft SQL Server** where the ops differ. Every concept is explained in **prose first** — *what it is, why it exists, when and how to use it, the key config parameters and what they do, best practices, and the security implications* — and only then shown with **heavily-commented, runnable commands and config**. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Scope note — this is the OPS guide, not a SQL course.** This library already teaches the *data model, SQL, querying, indexing theory and schema design* for each engine in dedicated guides: **[PostgreSQL](POSTGRESQL_GUIDE.md)**, **[MongoDB](MONGODB_GUIDE.md)**, **[SQLite3](SQLITE3_GUIDE.md)**, **[Redis](REDIS_GUIDE.md)** and **[Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)**. This guide does **not** re-teach `SELECT`/`JOIN`/normalization; it cross-references those guides and focuses on *administering the server*: install, config files, auth, networking, backups, replication, HA, scaling, monitoring, tuning, security, upgrades and production practice. It also leans on the **[Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md)**, **[Windows Server Administration](WINDOWS_SERVER_ADMIN_GUIDE.md)**, **[Networking](NETWORKING_GUIDE.md)**, **[Docker](DOCKER_GUIDE.md)** and **[Nginx](NGINX_GUIDE.md)** guides for the surrounding system.
>
> **Version note (2026-current):** Commands and config target **PostgreSQL 17** (released Sept 2024; PG 18 ships late 2025 and is noted where it changes things), **MySQL 8.4 LTS** (the long-term-support line) and **MySQL 9.x Innovation**, **MariaDB 11.x** (11.4 is the current LTS), **MongoDB 8.0**, **Redis 7.4 / Redis 8** (note the 2024 licensing split — see §13, plus the **Valkey** fork), and **Microsoft SQL Server 2022** (with **SQL Server 2025** in preview). Where behaviour is version-sensitive it is flagged **⚡ Version note**. The author works on **Windows 11**; production databases overwhelmingly run on **Linux**, so most examples assume a Debian/Ubuntu or RHEL-family Linux host, with Windows/SQL-Server specifics called out. Always confirm exact syntax against the official docs for your exact version and OS.

---

## Table of Contents

1. [What a DBA Does & the Database-Server Landscape](#1-what-a-dba-does--the-database-server-landscape) **[B]**
2. [Installation & Setup: Packages, Docker, Managed, Cluster Init](#2-installation--setup-packages-docker-managed-cluster-init) **[B/I]**
3. [Configuration Fundamentals & Connection Pooling](#3-configuration-fundamentals--connection-pooling) **[I/A]**
4. [Authentication, Users / Roles & Authorization](#4-authentication-users--roles--authorization) **[I/A]**
5. [Connections, Networking & Access Control](#5-connections-networking--access-control) **[I/A]**
6. [Backups & Recovery — The Most Important DBA Skill](#6-backups--recovery--the-most-important-dba-skill) **[I/A]**
7. [High Availability & Replication](#7-high-availability--replication) **[A]**
8. [Scaling: Read Replicas, Sharding & Partitioning](#8-scaling-read-replicas-sharding--partitioning) **[A]**
9. [Monitoring & Observability](#9-monitoring--observability) **[I/A]**
10. [Performance Tuning & Maintenance](#10-performance-tuning--maintenance) **[A]**
11. [Security & Hardening](#11-security--hardening) **[A]**
12. [Upgrades & Migrations](#12-upgrades--migrations) **[A]**
13. [Operating MongoDB & Redis as Servers](#13-operating-mongodb--redis-as-servers) **[I/A]**
14. [Microsoft SQL Server Basics for DBAs](#14-microsoft-sql-server-basics-for-dbas) **[I/A]**
15. [Automation & Running Databases in Containers / Cloud](#15-automation--running-databases-in-containers--cloud) **[A]**
16. [Production Operations & Company-Level Practices](#16-production-operations--company-level-practices) **[A]**
17. [Gotchas & Best Practices](#17-gotchas--best-practices) **[I/A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What a DBA Does & the Database-Server Landscape

### 1.1 What a database administrator actually does **[B]**

A **database administrator (DBA)** is the person (or team, or — increasingly — the SRE/platform role) responsible for keeping the company's databases **available, correct, fast, secure and recoverable**. Writing SQL is a *developer* skill; this library's engine guides teach it. **Running the server that executes that SQL in production** is the DBA skill, and it is a different job. A useful one-line definition: *a DBA owns the data's durability, availability and performance, and the operational machinery around it.*

Concretely, the daily and lifecycle responsibilities break into a handful of buckets you will meet again as the chapters of this guide:

- **Provisioning & configuration** — installing the engine, sizing the machine, setting the config so the database uses the hardware well without falling over.
- **Access control** — creating roles, granting least-privilege permissions, managing who and what can connect and from where.
- **Backups & recovery** — the single most important duty. Taking backups, *testing restores*, and being able to recover to a point in time after a disaster or a bad `DELETE`.
- **High availability & replication** — making the database survive the loss of a server, a disk, or a data centre, ideally with automatic failover.
- **Performance** — finding and fixing slow queries, tuning memory and I/O, managing locks and bloat, planning capacity.
- **Monitoring & alerting** — knowing the database is sick *before* users do, with metrics, logs and alerts.
- **Security & compliance** — encryption, auditing, patching, and meeting PCI/HIPAA/GDPR obligations.
- **Upgrades & migrations** — moving to new engine versions and applying schema changes safely, ideally with little or no downtime.
- **Incident response** — being on call, running the runbook, restoring service, and writing the postmortem.

The cultural shift to **DevOps/SRE/platform engineering** means in many companies there is no person with the literal job title "DBA"; instead, app teams own their databases with a platform team providing the managed building blocks. The *work in this guide does not go away* — it gets distributed and automated. Knowing it makes you the person everyone asks when the database misbehaves.

### 1.2 The reliability vocabulary every DBA must speak **[B]**

Three acronyms anchor every conversation about database operations. Internalise them now; they recur throughout.

| Term | Stands for | Means | Example |
|---|---|---|---|
| **RPO** | Recovery **Point** Objective | How much *data* you can afford to lose, measured in time | "RPO = 5 min" → after a disaster you may lose at most the last 5 minutes of writes |
| **RTO** | Recovery **Time** Objective | How long you can afford to be *down* while recovering | "RTO = 30 min" → service must be back within 30 minutes |
| **SLA / SLO** | Service-Level Agreement / Objective | The availability promise, often as "nines" | "99.9 %" ≈ 8.7 h downtime/year; "99.99 %" ≈ 52 min/year |
| **MTTR** | Mean Time To Recovery | Average time to recover from failure | Lower is better; drives RTO |
| **Durability** | — | Once the DB says "committed", the data survives crashes | Guaranteed by the WAL/redo log + `fsync` |
| **Availability** | — | The fraction of time the DB is reachable and serving | Improved by HA/replication/failover |

RPO and RTO are the two numbers that *design your backup and HA strategy*. If the business needs RPO = 0 (lose no committed transaction), you need synchronous replication. If RTO = a few minutes, a nightly logical dump that takes an hour to restore is unacceptable and you need physical backups + standbys. **Always derive the technology from the RPO/RTO the business actually needs** — over-engineering wastes money, under-engineering loses the company data.

### 1.3 The landscape: relational vs document vs key-value **[B]**

You administer different *kinds* of database server differently because their internal architecture differs. The three families you will meet most:

**Relational (RDBMS) — PostgreSQL, MySQL/MariaDB, SQL Server, Oracle.** Data in tables with rows and columns, strong schemas, ACID transactions, SQL. These are the workhorses of business data. Ops centre on: a **write-ahead log** for durability, a **buffer pool/shared buffers** cache in RAM, connection management, vacuum/cleanup of dead rows, and physical/logical replication. This guide's primary subjects.

**Document — MongoDB, Couchbase.** Data as JSON-like documents (BSON in Mongo) grouped in collections, flexible schema. Ops centre on **replica sets** (a primary + secondaries electing among themselves), **sharded clusters**, the **oplog** (operations log used for replication), and memory/working-set management. Covered in §13.

**Key-value / cache / in-memory — Redis, Valkey, Memcached.** Extremely fast, data primarily in RAM. Often used as a *cache* in front of a relational DB rather than the system of record. Ops centre on **memory limits and eviction**, **persistence** (RDB snapshots / AOF logs — and the trade-off of *whether you even want* durability), **Sentinel** or **Cluster** for HA. Covered in §13.

There are more families — **wide-column** (Cassandra/ScyllaDB), **time-series** (TimescaleDB, InfluxDB, Prometheus), **graph** (Neo4j), **search** (Elasticsearch/OpenSearch) — but the four above cover the vast majority of what a generalist DBA runs. The operational *principles* (backups, replication, monitoring, security) transfer; the *mechanics* differ.

### 1.4 OLTP vs OLAP — two opposite workloads **[B]**

A workload's *shape* dictates how you tune, scale and even which engine you pick.

| | **OLTP** (Online Transaction Processing) | **OLAP** (Online Analytical Processing) |
|---|---|---|
| Pattern | Many tiny, fast read/write transactions | Few huge, scan-heavy analytical queries |
| Example | "Place an order", "fetch a user" | "Total revenue by region by month, last 5 years" |
| Rows touched | A handful per query | Millions per query |
| Tuning goal | Low latency, high concurrency, fast point lookups | High throughput on big scans, columnar storage |
| Engines | PostgreSQL, MySQL, SQL Server (row stores) | Snowflake, BigQuery, ClickHouse, Redshift, DuckDB (column stores) |
| Index strategy | B-tree on selective columns | Often few indexes; columnar + partitioning |

Most application databases are **OLTP**. The classic mistake is running heavy analytics directly against the OLTP primary, where a single report scan trashes the buffer cache and starves the latency-sensitive transactional traffic. The standard DBA answer is to **offload analytics to a read replica or a separate analytics warehouse** (see §8). Knowing which workload you have tells you whether to optimise for latency or throughput.

### 1.5 Self-managed vs managed — the central tradeoff **[B]**

The biggest strategic decision a modern DBA influences is **self-managed vs managed**.

**Self-managed** (you install Postgres/MySQL on a VM or bare metal, or in your own Kubernetes): you control everything — every config knob, every extension, the exact version, the kernel, the storage. You also *own* everything — patching, backups, replication, failover, monitoring, 3 a.m. pages. Maximum control and (at scale) lowest cost, maximum operational burden.

**Managed** (AWS **RDS**/**Aurora**, Google **Cloud SQL**/**AlloyDB**, Azure **Database for PostgreSQL/MySQL**, MongoDB **Atlas**, Redis **Cloud**): the provider runs the server. They handle provisioning, automated backups, patching, replication and failover, monitoring dashboards — behind a console and an API. You give up: superuser access (no `SUPERUSER` role, restricted extensions, no shell on the box), some config knobs, and you pay a premium. You gain: most of the undifferentiated ops toil disappears.

| Concern | Self-managed | Managed (RDS / Cloud SQL / Atlas) |
|---|---|---|
| Cost (raw) | Lower per-GB/CPU | Higher (you pay for the service) |
| Cost (total, incl. people) | Often higher (needs DBA time) | Often lower for small teams |
| Control | Total (superuser, any extension, OS access) | Limited (no superuser, curated extensions, no OS) |
| Patching & minor upgrades | You do it | Automated (in a maintenance window) |
| Backups / PITR | You build & test it | Built in (still **verify** it!) |
| HA / failover | You build (Patroni, repmgr…) | One checkbox (Multi-AZ) |
| Lock-in | Low | Higher (esp. Aurora/Atlas-specific features) |
| Best for | Large scale, special needs, cost control, regulated air-gapped | Small/medium teams, fast-moving startups, "boring" infra |

**The honest default for most companies in 2026 is managed**, because engineer time is more expensive than the managed premium and the providers do backups/HA better than a small team will. You go self-managed when you have the scale to justify a real DBA team, special requirements (an unusual extension, an air-gapped/regulated environment, on-prem), or a cost profile where the managed premium dominates. **Even on managed services you must still understand everything in this guide** — you still design the backup/RPO strategy, still tune queries, still set up monitoring and alerting, still respond to incidents. Managed removes the *toil*, not the *responsibility*.

---

## 2. Installation & Setup: Packages, Docker, Managed, Cluster Init

### 2.1 The three ways to get a database server **[B]**

There are three routes, and choosing well is an ops decision:

1. **OS packages** (`apt`, `dnf`, `zypper`, MSI on Windows) — the classic. The engine runs as a native service managed by **systemd** (Linux) or the Service Control Manager (Windows). Best for **dedicated database hosts** and production servers where you want the OS package manager handling security patches and the database to "own" the machine. This is what most production self-managed databases use.
2. **Docker / containers** — fast to spin up, perfect for **local dev, CI, and ephemeral test databases**, and increasingly for production *via Kubernetes operators* (§15). The catch for production: a database is **stateful**, so you must attach a persistent **volume** for the data directory or you lose everything on container restart. See the **[Docker guide](DOCKER_GUIDE.md)**.
3. **Managed service** — you don't install anything; you click "create instance" and get a connection string. See §1.5 and §15.

For learning DBA work, install at least once **from packages on Linux** so you see the data directory, the service unit, the config files and the logs — the things managed services hide. Then use Docker for throwaway experiments.

### 2.2 Installing PostgreSQL from packages **[B]**

The PostgreSQL Global Development Group (PGDG) ships an official apt/dnf repository that is more current than the distro's default and lets you pick the major version. Use it for production.

```bash
# --- Debian / Ubuntu: install PostgreSQL 17 from the official PGDG repo ---
sudo apt update && sudo apt install -y curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
# Import the PGDG signing key (so apt trusts the repo):
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  https://www.postgresql.org/media/keys/ACCC4CF8.asc
# Add the repo for this Ubuntu codename:
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt install -y postgresql-17 postgresql-client-17 postgresql-contrib-17
# postgresql-contrib installs useful extensions (pg_stat_statements, pgcrypto, etc.)
```

On Debian/Ubuntu the package **automatically runs `initdb` and starts a cluster** for you (named `main`), and wraps everything in the Debian multi-cluster tooling (`pg_lsclusters`, `pg_ctlcluster`, `pg_createcluster`). On RHEL/Rocky/Alma you must initialise the cluster manually:

```bash
# --- RHEL / Rocky / Alma 9: PostgreSQL 17 ---
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql        # disable the older built-in module stream
sudo dnf install -y postgresql17-server postgresql17-contrib
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb   # initialise the data directory (cluster)
sudo systemctl enable --now postgresql-17           # start at boot + now
```

### 2.3 The data directory and `initdb` — what a "cluster" is **[B/I]**

The single most important file-system concept in PostgreSQL administration is the **data directory** (a.k.a. `PGDATA` or "the cluster"). A PostgreSQL **cluster** is *one running server process tree plus the one directory tree it owns* — and that directory holds **all the databases** for that server (a confusing use of "cluster": it is *not* a group of machines). Everything the server persists lives under `PGDATA`: the actual table data, the indexes, the WAL, the config files (on RHEL layout) and the control file.

`initdb` is the program that **creates** a fresh data directory: it lays out the on-disk structure, creates the `postgres`, `template0` and `template1` databases, sets the superuser, the encoding (use `UTF8`), the locale, and the **data checksums** option. You run it once per cluster.

```bash
# Manually init a cluster (when not using distro tooling). Run as the `postgres` OS user.
sudo -u postgres /usr/lib/postgresql/17/bin/initdb \
  --pgdata=/var/lib/postgresql/17/main \  # where the data lives
  --encoding=UTF8 \                       # ALWAYS UTF8 in 2026
  --locale=C.UTF-8 \                      # collation/locale; C.UTF-8 is fast + predictable
  --data-checksums                        # detect silent disk corruption — ON in prod (default ON in PG18 ⚡)
```

> **⚡ Version note:** **PostgreSQL 18** makes **data checksums ON by default** at `initdb`. On PG 17 and earlier you *must* pass `--data-checksums` explicitly — and you cannot turn them on later without `pg_checksums --enable` on a stopped cluster. Always enable them in production: the small CPU cost buys early detection of silent storage corruption.

Typical data-directory contents (know these names; you will reference them during incidents):

| Path under `PGDATA` | What it is |
|---|---|
| `base/` | The actual table & index data files, one subdir per database |
| `pg_wal/` | Write-Ahead Log segments (16 MB each) — the durability journal |
| `global/` | Cluster-wide tables (roles, databases catalog) |
| `pg_xact/` | Transaction commit status (clog) |
| `postgresql.conf` | Main config (Debian relocates this to `/etc/postgresql/...`) |
| `pg_hba.conf` | Host-Based Authentication rules (who may connect, how) |
| `postmaster.pid` | PID + socket info of the running server |
| `PG_VERSION` | The major version this directory belongs to |

### 2.4 Installing MySQL / MariaDB from packages **[B]**

MySQL and MariaDB are *forks of a common ancestor* and remain mostly compatible at the SQL level but **diverge in administration** (config defaults, replication, some tools). Pick one and be consistent. Oracle's **MySYQL 8.4 LTS** and **MariaDB 11.4 LTS** are the production-grade lines in 2026.

```bash
# --- MySQL 8.4 LTS on Ubuntu via Oracle's APT repo ---
curl -fsSL https://repo.mysql.com/RPM-GPG-KEY-mysql-2023 | sudo gpg --dearmor -o /usr/share/keyrings/mysql.gpg
# (Oracle ships a .deb config package; on Ubuntu the simplest path is:)
sudo apt install -y mysql-server     # distro MySQL, or use the mysql-apt-config .deb for 8.4
sudo systemctl enable --now mysql

# CRITICAL first step: run the hardening wizard (set root password, remove anon users,
# disallow remote root, drop the test database, reload privileges):
sudo mysql_secure_installation
```

```bash
# --- MariaDB 11.4 on Debian/Ubuntu ---
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo mariadb-secure-installation     # MariaDB's equivalent hardening wizard
```

MySQL/MariaDB use a single **data directory** too (default `/var/lib/mysql`), with the dominant storage engine **InnoDB** (ACID, row-level locking, crash recovery via the **redo log**). The config file is `my.cnf` (`/etc/mysql/my.cnf` and `/etc/mysql/mysql.conf.d/` on Debian).

### 2.5 Databases in Docker (dev & test) **[B/I]**

For local development and CI, a containerised database is ideal: disposable, version-pinned, no host pollution. **The one rule that matters: mount a volume for the data directory**, or treat the container as truly disposable.

```bash
# PostgreSQL 17 in Docker with a NAMED VOLUME so data survives container removal.
docker run -d --name pg17 \
  -e POSTGRES_PASSWORD='dev_only_change_me' \  # sets the postgres superuser password
  -e POSTGRES_DB=appdb \                       # auto-creates this database
  -e POSTGRES_USER=appuser \                   # creates this role (instead of `postgres`)
  -p 127.0.0.1:5432:5432 \                     # bind to LOCALHOST ONLY — never 0.0.0.0 in dev
  -v pgdata17:/var/lib/postgresql/data \       # persistent named volume
  postgres:17

# MySQL 8.4 in Docker
docker run -d --name mysql84 \
  -e MYSQL_ROOT_PASSWORD='dev_only_change_me' \
  -e MYSQL_DATABASE=appdb \
  -p 127.0.0.1:3306:3306 \
  -v mysqldata:/var/lib/mysql \
  mysql:8.4
```

A `docker compose` file is the better way to manage this for a project (see the **[Docker guide](DOCKER_GUIDE.md)**). **Do not run a stateful production database in a plain single Docker container** — for production containers use a Kubernetes **operator** (CloudNativePG, Percona, MongoDB Operator) that handles failover, backups and persistent storage (§15).

### 2.6 Service management with systemd **[B/I]**

On Linux, the database runs as a **systemd service**. Mastering these commands is table stakes (see the **[Linux Server Administration guide](LINUX_SERVER_ADMIN_GUIDE.md)** for systemd depth):

```bash
sudo systemctl status postgresql@17-main   # is it running? (Debian per-cluster unit)
sudo systemctl start  postgresql           # start
sudo systemctl stop   postgresql           # stop (graceful: waits for connections)
sudo systemctl restart postgresql          # full restart (needed for some params, e.g. shared_buffers)
sudo systemctl reload  postgresql          # re-read config WITHOUT dropping connections (most params)
sudo systemctl enable  postgresql          # start automatically at boot — ESSENTIAL in prod
journalctl -u postgresql@17-main -f        # tail the service logs

# MySQL/MariaDB:
sudo systemctl status mysql      # or: mariadb
sudo systemctl restart mysql
```

The **reload vs restart** distinction is operationally vital: a *reload* (`SIGHUP`) re-reads config and applies the many parameters that can change live, with **zero downtime**; a *restart* drops all connections and is needed only for the handful of parameters that allocate fixed resources at startup (e.g. `shared_buffers`, `max_connections`). Knowing which a change needs lets you avoid unnecessary outages (§3.2).

### 2.7 First connection & client tools **[B]**

After install you connect with the engine's **command-line client** — your primary DBA tool. The first-connect dance differs by engine because of default authentication.

```bash
# --- PostgreSQL: connect as the bootstrap superuser via "peer" auth ---
# Out of the box, the `postgres` OS user maps to the `postgres` DB superuser over the local socket.
sudo -u postgres psql
# Inside psql, useful meta-commands (backslash commands — not SQL):
#   \l            list databases
#   \du           list roles/users
#   \c appdb      connect to database appdb
#   \dt           list tables in current schema
#   \d tablename  describe a table
#   \conninfo     show how you're connected
#   \timing on    show query timings
#   \x            toggle expanded (vertical) output — great for wide rows
#   \q            quit

# Connect remotely / with a full connection string (libpq URI):
psql "postgresql://appuser:secret@db.internal:5432/appdb?sslmode=require"
```

```bash
# --- MySQL / MariaDB: connect as root ---
sudo mysql                        # uses auth_socket (Debian) — root via the unix socket
mysql -u appuser -p -h db.internal appdb   # prompt for password (-p with no value = secure prompt)
# Inside the mysql client:
#   SHOW DATABASES;
#   USE appdb;
#   SHOW TABLES;
#   DESCRIBE users;        -- or:  SHOW CREATE TABLE users\G   (\G = vertical output)
#   STATUS;                -- connection/server summary
#   \q  (or: exit)
```

GUI/IDE clients exist (**pgAdmin**, **DBeaver**, **TablePlus**, **DataGrip**, **MySQL Workbench**, **mongosh**/**Compass**, SSMS for SQL Server) and are fine for browsing, but **a competent DBA lives in the CLI** because it works over SSH on a headless server, scripts cleanly, and is always available during an incident.

---

## 3. Configuration Fundamentals & Connection Pooling

### 3.1 Where config lives and how it loads **[I]**

Every engine has a **main text config file** read at startup. Knowing where it is, how changes take effect, and which knobs matter is the core of day-to-day tuning.

| Engine | Main config file(s) | View current settings | Apply a change |
|---|---|---|---|
| PostgreSQL | `postgresql.conf` (+ `conf.d/`, `postgresql.auto.conf` for `ALTER SYSTEM`) | `SHOW name;` / `SELECT * FROM pg_settings;` | `SELECT pg_reload_conf();` or restart |
| MySQL/MariaDB | `my.cnf` / `mysqld.cnf` (+ `conf.d/`) | `SHOW VARIABLES LIKE 'name';` | `SET GLOBAL name=...;` (live) or restart |
| MongoDB | `mongod.conf` (YAML) | `db.adminCommand({getParameter:...})` | `setParameter` or restart |
| Redis | `redis.conf` | `CONFIG GET name` | `CONFIG SET name value` (live) + `CONFIG REWRITE` |

In PostgreSQL, a setting's `pg_settings.context` tells you *what it takes to change it*: `internal` (compile-time), `postmaster` (**restart**), `sighup` (**reload**), `superuser`/`user` (per-session). Find it with:

```sql
-- Which params need a restart vs a reload?
SELECT name, setting, unit, context, short_desc
FROM pg_settings
WHERE name IN ('shared_buffers','work_mem','max_connections','wal_level')
ORDER BY name;
-- context = 'postmaster' → restart;  'sighup' → reload;  'user' → just re-run SET in your session.
```

> **Best practice:** prefer `ALTER SYSTEM SET shared_buffers = '8GB';` (writes to `postgresql.auto.conf`) over hand-editing `postgresql.conf` for production changes — it's auditable, survives package upgrades, and is the same mechanism config-management tools use. Then `SELECT pg_reload_conf();` (or restart for `postmaster` params). Keep the hand-written file for documented baselines, the auto file for live tweaks.

### 3.2 PostgreSQL: the parameters that actually matter **[I/A]**

You can ignore 95 % of `postgresql.conf`. These are the load-bearing ones. The numbers below are *starting points* for a dedicated database server with a given RAM size; always measure and adjust.

```conf
# ===== postgresql.conf — the parameters a DBA tunes first =====

# --- MEMORY ---
# shared_buffers: PostgreSQL's own page cache, allocated once at startup (needs RESTART).
#   Rule of thumb: ~25% of total RAM on a dedicated server (the OS cache holds the rest).
#   Too small = constant disk reads; too big = competes with OS cache, diminishing returns.
shared_buffers = 8GB                 # on a 32 GB machine

# effective_cache_size: NOT an allocation — a HINT to the planner about how much memory
#   (shared_buffers + OS page cache) is available for caching. Set to ~50-75% of RAM.
#   Influences whether the planner favours index scans. Costs nothing to set high-ish.
effective_cache_size = 24GB

# work_mem: memory PER SORT/HASH OPERATION, PER QUERY. THE classic footgun:
#   a complex query can use several multiples of this, and many concurrent queries multiply it.
#   Total worst case ≈ work_mem × max_connections × (sorts per query). Set conservatively.
work_mem = 32MB                      # raise for analytics, keep low for high-concurrency OLTP

# maintenance_work_mem: memory for VACUUM, CREATE INDEX, ALTER TABLE. Can be generous
#   because few run at once. Speeds up index builds and vacuum dramatically.
maintenance_work_mem = 1GB

# --- CONNECTIONS ---
# max_connections: hard cap on concurrent connections (needs RESTART). Each connection is a
#   PROCESS using memory. DON'T set this to thousands — use a POOLER instead (§3.5).
max_connections = 200

# --- WRITE-AHEAD LOG & DURABILITY ---
wal_level = replica                  # 'replica' enables physical replication & PITR; 'logical' for logical repl
max_wal_size = 4GB                   # how much WAL accumulates between checkpoints (bigger = fewer checkpoints)
min_wal_size = 1GB
checkpoint_completion_target = 0.9   # spread checkpoint I/O over 90% of the interval (smooths I/O spikes)
synchronous_commit = on              # 'on' = durable; 'off' trades durability for throughput (RPO risk!)
wal_compression = on                 # compress full-page images in WAL — saves disk & replication bandwidth

# --- PLANNER / STORAGE ---
random_page_cost = 1.1               # ~1.1 for SSD/NVMe (default 4.0 assumed spinning disk!) — important on SSD
effective_io_concurrency = 200       # number of concurrent I/O ops the storage can handle (SSD/NVMe: high)
default_statistics_target = 100      # sampling for ANALYZE; raise for skewed columns

# --- AUTOVACUUM (keep dead rows in check — see §10) ---
autovacuum = on                      # NEVER turn this off
autovacuum_max_workers = 5
autovacuum_vacuum_cost_limit = 2000  # let autovacuum work faster on busy systems

# --- LOGGING (for monitoring & slow-query hunting — see §9/§10) ---
logging_collector = on
log_min_duration_statement = 1000    # log any statement slower than 1000 ms — your slow-query log
log_checkpoints = on
log_connections = on
log_lock_waits = on                  # log when a session waits a long time on a lock
log_line_prefix = '%m [%p] %u@%d %a %h '  # timestamp, pid, user@db, app, host — make logs parseable
```

**Why these and not others:** `shared_buffers` and `effective_cache_size` shape the memory and the planner's caching assumptions; `work_mem` is the single most common cause of out-of-memory incidents (set too high × high concurrency); `random_page_cost` defaulting to a spinning-disk value on SSD hardware causes the planner to wrongly avoid index scans (a famous easy win); `synchronous_commit` is your RPO dial; `log_min_duration_statement` creates the slow-query log you cannot tune without.

### 3.3 MySQL / MariaDB: the parameters that matter **[I/A]**

```ini
# ===== my.cnf — the parameters a MySQL/MariaDB DBA tunes first =====
[mysqld]

# --- InnoDB BUFFER POOL: the single most important MySQL setting ---
# innodb_buffer_pool_size: InnoDB's in-memory cache of data + index pages. On a DEDICATED
#   server, 60-75% of RAM. This is the MySQL analogue of shared_buffers but bigger (MySQL
#   relies less on the OS cache for InnoDB). Too small = disk-bound; this is the #1 knob.
innodb_buffer_pool_size = 24G        # on a 32 GB dedicated server
innodb_buffer_pool_instances = 8     # split into instances to reduce contention (large pools)

# --- InnoDB DURABILITY (your RPO dial) ---
# innodb_flush_log_at_trx_commit:
#   1 = full ACID, flush+fsync redo log every commit (default; safest, RPO≈0)
#   2 = write each commit but fsync once/sec (survives mysqld crash, loses ~1s on OS crash)
#   0 = flush once/sec regardless (fastest, riskiest)
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1                      # fsync the binary log every commit (durable replication/PITR)
innodb_flush_method = O_DIRECT       # bypass OS cache for InnoDB files (avoids double-caching)
innodb_redo_log_capacity = 4G        # MySQL 8.0.30+: total redo log size (replaces innodb_log_file_size) ⚡

# --- CONNECTIONS ---
max_connections = 200                # again: use a pooler (ProxySQL) rather than thousands
thread_cache_size = 64               # reuse threads instead of creating per connection

# --- BINARY LOG (replication + PITR — see §6/§7) ---
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW                  # ROW is the safe, replication-correct default (not STATEMENT)
binlog_expire_logs_seconds = 604800  # keep binlogs 7 days (for PITR & replica catch-up)
server_id = 1                        # unique per server in a replication topology — REQUIRED for replication
gtid_mode = ON                       # GTIDs: global transaction IDs — modern, easier failover
enforce_gtid_consistency = ON

# --- SLOW QUERY LOG (your tuning starting point — see §10) ---
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1                  # log queries slower than 1 second
log_queries_not_using_indexes = 1    # also catch full-table-scan queries
```

> **⚡ Version note:** MySQL 8.0.30 replaced the old `innodb_log_file_size` + `innodb_log_files_in_group` pair with a single dynamic `innodb_redo_log_capacity`. MySQL 8.4 LTS removed several long-deprecated options and changed some defaults (e.g. `binlog_transaction_dependency_tracking` and the default authentication plugin handling) — re-check your `my.cnf` after upgrading.

### 3.4 Why connection pooling is non-negotiable **[I/A]**

Here is a problem that surprises every new DBA. In **PostgreSQL each connection is a full OS process** (forked from the postmaster); in MySQL it's a thread. Either way, each idle connection consumes memory and scheduling overhead, and there's a hard ceiling (`max_connections`). Web apps, especially serverless and many-replica deployments, tend to open **far more connections than the database can handle** — every app instance keeps a pool, and 50 app pods × 20 connections each = 1000 connections, crushing a server sized for 200.

A **connection pooler** sits between the application and the database and **multiplexes** many client connections onto a small number of real database connections. The app connects to the pooler (which is cheap to connect to); the pooler hands the app one of its already-open backend connections for the duration of a query/transaction, then reuses it for someone else. This decouples "how many clients want to talk" from "how many real backends exist", dramatically reducing memory and context-switch overhead and protecting `max_connections`.

| Pooler | For | Notable |
|---|---|---|
| **PgBouncer** | PostgreSQL | Tiny, single-process, the standard. Modes: `session`, `transaction` (most common), `statement` |
| **Pgpool-II** | PostgreSQL | Heavier; also does load balancing & some failover (often Patroni is preferred for HA) |
| **Supabase Supavisor** | PostgreSQL | Cloud-scale, multi-tenant pooler (used by Supabase) |
| **ProxySQL** | MySQL | Pooling + read/write split + query routing/caching |
| **MySQL Router** | MySQL InnoDB Cluster | Official routing/pooling for InnoDB Cluster |

**Pooling mode matters enormously.** PgBouncer's **transaction** mode (return the backend to the pool at the end of each transaction) gives the best multiplexing but **breaks features that span transactions** on a single session: session-level `SET`, prepared statements (historically), advisory locks, `LISTEN/NOTIFY`, temp tables. **Session** mode (one backend per client connection until it disconnects) is safe for everything but pools less aggressively. Pick transaction mode unless your app needs session features — and verify your driver/ORM is configured for it.

### 3.5 PgBouncer in practice **[A]**

```ini
# ===== /etc/pgbouncer/pgbouncer.ini =====
[databases]
# Map a virtual db name the app connects to → the real backend.
appdb = host=127.0.0.1 port=5432 dbname=appdb

[pgbouncer]
listen_addr = 127.0.0.1          # pooler listens here (apps connect to PgBouncer, not Postgres directly)
listen_port = 6432
auth_type = scram-sha-256        # match Postgres's password method (NEVER md5/trust in prod)
auth_file = /etc/pgbouncer/userlist.txt   # "username" "SCRAM-hash"

pool_mode = transaction          # best multiplexing for typical web apps (see caveats §3.4)
max_client_conn = 2000           # how many app connections PgBouncer accepts
default_pool_size = 25           # how many REAL backend connections per (user,db) pair
reserve_pool_size = 5            # extra backends for bursts
server_idle_timeout = 600        # close idle backends after 10 min

# Apps now connect to: postgresql://appuser:...@pgbouncer-host:6432/appdb
# 2000 app connections → as few as 25 real Postgres backends. That's the whole point.
```

The arithmetic to internalise: with `default_pool_size = 25`, a fleet of app servers asking for thousands of connections is served by **25 real Postgres processes**, keeping you well under `max_connections = 200` with headroom for replicas, monitoring and admin sessions. Right-size the pool: too small and queries queue at the pooler; too big and you lose the protection. Start near `(number of CPU cores × 2-4)` for the backend pool and tune by watching wait events.

---

## 4. Authentication, Users / Roles & Authorization

### 4.1 The two layers: who can connect, and what they can do **[I]**

Database access control has **two distinct layers** that beginners conflate:

1. **Authentication** — *can this connection prove who it is, from where, and may it connect at all?* Handled server-side by `pg_hba.conf` (Postgres) or the `user@host` + plugin model (MySQL). This is the front door.
2. **Authorization** — *given that you're authenticated, what objects may you read/write and what may you do?* Handled by **roles/users**, **privileges** and **`GRANT`/`REVOKE`**. This is which rooms you can enter.

A secure setup needs both: tight authentication (only the app host, over TLS, with a strong credential) **and** least-privilege authorization (the app role can only touch its own schema, not `DROP DATABASE`).

### 4.2 PostgreSQL roles, the GRANT model & least privilege **[I/A]**

PostgreSQL unifies "users" and "groups" into one concept: **roles**. A role with `LOGIN` is effectively a user; a role without `LOGIN` used to group privileges is effectively a group. Roles can be **members of** other roles, inheriting their privileges. Special attributes: `SUPERUSER` (bypasses all checks — give to almost nobody), `CREATEDB`, `CREATEROLE`, `REPLICATION` (for standbys), `BYPASSRLS`.

The cardinal principle is **least privilege**: every role gets the minimum it needs, nothing more. The classic safe pattern is a *role hierarchy*: privilege-holding group roles + login roles that are members of them, so you grant to the group and assign people/apps to groups.

```sql
-- ===== A least-privilege setup for an application database =====

-- 1. Create the database and a dedicated OWNER role (NOT a superuser).
CREATE ROLE appowner LOGIN PASSWORD 'CHANGE_ME_strong';   -- owns the schema/objects
CREATE DATABASE appdb OWNER appowner;
\c appdb

-- 2. Lock down the public schema (PG 15+ already revokes CREATE from PUBLIC by default ⚡).
REVOKE ALL ON SCHEMA public FROM PUBLIC;          -- nobody gets implicit rights
REVOKE ALL ON DATABASE appdb FROM PUBLIC;         -- not even CONNECT by default

-- 3. Group roles describing INTENT, not people.
CREATE ROLE app_readwrite NOLOGIN;   -- for the application's normal traffic
CREATE ROLE app_readonly  NOLOGIN;   -- for analytics/replicas/reporting

-- 4. Grant the groups exactly what they need.
GRANT CONNECT ON DATABASE appdb TO app_readwrite, app_readonly;
GRANT USAGE  ON SCHEMA public  TO app_readwrite, app_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT SELECT                          ON ALL TABLES IN SCHEMA public TO app_readonly;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;  -- for serial/identity columns

-- 5. CRUCIAL: default privileges so FUTURE tables get the grants automatically.
--    Without this, every new table the owner creates is invisible to the app roles!
ALTER DEFAULT PRIVILEGES FOR ROLE appowner IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES FOR ROLE appowner IN SCHEMA public
  GRANT SELECT ON TABLES TO app_readonly;

-- 6. Login roles for the actual app + a reporting user, assigned to the groups.
CREATE ROLE app_svc      LOGIN PASSWORD 'CHANGE_ME_app'  IN ROLE app_readwrite;
CREATE ROLE report_svc   LOGIN PASSWORD 'CHANGE_ME_rpt'  IN ROLE app_readonly;
-- Now the app connects as app_svc and can ONLY do CRUD; it cannot DROP, ALTER, or read other schemas.
```

**Row-Level Security (RLS)** and **column privileges** let you go finer: RLS attaches `USING`/`WITH CHECK` policies so a role only sees its own rows (multi-tenant SaaS, see the **[PostgreSQL guide](POSTGRESQL_GUIDE.md)** for policy syntax); column-level `GRANT SELECT (col1, col2)` hides sensitive columns. Use them when the app can't be trusted to filter, or for defence-in-depth.

```sql
-- Row-Level Security skeleton (tenant isolation):
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::int);   -- only rows for the set tenant
-- The app sets the tenant per connection:  SET app.tenant_id = '42';
```

### 4.3 MySQL / MariaDB users, hosts & privileges **[I/A]**

MySQL identity is **`'user'@'host'`** — the *same username from a different host is a different account*. The `host` part is an access-control feature: `'app'@'10.0.%'` permits the app user only from the `10.0.0.0/16` private range; `'app'@'%'` (any host) is usually a mistake. Privileges are granted globally (`*.*`), per-database (`appdb.*`), per-table or per-column.

```sql
-- ===== Least-privilege MySQL app user, restricted by source host =====

-- Create the app account, locked to the private subnet, with a strong password & modern plugin.
CREATE USER 'app_svc'@'10.0.%'
  IDENTIFIED WITH caching_sha2_password BY 'CHANGE_ME_strong'   -- 8.0 default, stronger than mysql_native
  PASSWORD EXPIRE INTERVAL 90 DAY                                -- rotation policy
  FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;                  -- lock after 5 bad tries for 1 day

-- Grant ONLY CRUD on the app schema — no DROP, no GRANT, no admin.
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app_svc'@'10.0.%';

-- A read-only reporting account.
CREATE USER 'report_svc'@'10.0.%' IDENTIFIED WITH caching_sha2_password BY 'CHANGE_ME_rpt';
GRANT SELECT ON appdb.* TO 'report_svc'@'10.0.%';

-- Roles (MySQL 8.0+/MariaDB 10.0.5+) group privileges like Postgres group roles:
CREATE ROLE 'readwrite';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'readwrite';
GRANT 'readwrite' TO 'app_svc'@'10.0.%';
SET DEFAULT ROLE 'readwrite' TO 'app_svc'@'10.0.%';   -- activate it automatically on login

FLUSH PRIVILEGES;        -- reload the grant tables (needed after low-level changes)
SHOW GRANTS FOR 'app_svc'@'10.0.%';   -- audit what an account can actually do
```

> **⚡ Version note:** MySQL 8.0 made **`caching_sha2_password`** the default authentication plugin (stronger SHA-256-based challenge), replacing `mysql_native_password` — which MySQL 8.4 has now **disabled by default** and is deprecated for removal. Ensure your client drivers support `caching_sha2_password` (they do, in current versions) and TLS, since it transmits the password securely only over an encrypted/secure channel.

### 4.4 Password policy, rotation & secrets **[I/A]**

The database is only as secure as the credentials. Best practices:

- **Strong, unique passwords** per account; never the default, never shared between environments.
- **Rotation** — periodic forced rotation (`PASSWORD EXPIRE INTERVAL`, validity periods), and *immediate* rotation when someone leaves or a secret leaks.
- **Strength enforcement** — MySQL's `validate_password` component / MariaDB's `simple_password_check`; Postgres has `passwordcheck` and `credcheck` extensions.
- **Never hard-code credentials** in app code or config committed to git. Use a **secrets manager**: HashiCorp Vault (can even issue *dynamic, short-lived* database credentials), AWS Secrets Manager, GCP Secret Manager, or at minimum environment variables injected at deploy.
- **Prefer non-password auth where possible**: client-certificate auth, IAM-token auth (RDS/Cloud SQL can authenticate using cloud IAM with rotating tokens — no static password at all), Kerberos/GSSAPI in enterprises, SCRAM with TLS as the floor.

A common modern pattern: the app authenticates to Vault using its workload identity, Vault hands it a **dynamic Postgres role** valid for one hour, and rotates/revokes it automatically. No long-lived database password ever sits on disk.

---

## 5. Connections, Networking & Access Control

### 5.1 The cardinal rule: never expose a database to the internet **[I/A]**

The single most important network fact in this entire guide: **a production database should never be directly reachable from the public internet.** Databases are constantly scanned and attacked; an exposed Postgres/MySQL/Mongo/Redis with a weak or default credential is compromised within minutes (whole ransomware campaigns target exposed MongoDB and Redis instances). The database belongs on a **private network**, reachable only by the application servers, a bastion, and the admin's VPN. Read the **[Networking guide](NETWORKING_GUIDE.md)** for the underlying TCP/firewall/VPC concepts.

The defence is layered:

1. **Bind to private interfaces only** — don't `listen` on `0.0.0.0` unless behind a firewall; bind to the private IP or socket.
2. **Firewall** — allow the DB port *only* from the app subnet / specific hosts (cloud security groups, `ufw`/`firewalld`/`iptables`).
3. **Network isolation** — put the DB in a private subnet/VPC with no public IP; reach it through a **bastion host** or VPN/SSH tunnel for admin.
4. **TLS** in transit, strong auth, least privilege (§4) as defence-in-depth in case the network controls fail.

### 5.2 PostgreSQL: listen addresses & `pg_hba.conf` **[I/A]**

Two files control PostgreSQL network access. `postgresql.conf` decides *which interfaces and port* the server listens on; `pg_hba.conf` (Host-Based Authentication) decides *which clients may connect to which databases as which users, from which IPs, using which auth method* — evaluated **top-to-bottom, first match wins**.

```conf
# ===== postgresql.conf — listening =====
listen_addresses = 'localhost,10.0.1.5'   # loopback + the PRIVATE NIC only. NEVER '*' on an exposed host.
port = 5432
ssl = on                                  # enable TLS for client connections (§5.4)
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file  = '/etc/postgresql/ssl/server.key'
```

```conf
# ===== pg_hba.conf — first match wins, evaluated top→bottom =====
# TYPE   DATABASE   USER         ADDRESS            METHOD
local    all        postgres                        peer              # local socket: OS user 'postgres' → role 'postgres'
host     all        all          127.0.0.1/32       scram-sha-256     # local TCP needs a SCRAM password
hostssl  appdb      app_svc      10.0.0.0/16        scram-sha-256     # app subnet, TLS REQUIRED, SCRAM password
hostssl  appdb      report_svc   10.0.2.10/32       scram-sha-256     # one reporting host only
host     replication repl_user   10.0.1.0/24        scram-sha-256     # standbys may stream WAL
# (No 'host all all 0.0.0.0/0 trust' — that line is how databases get owned. The implicit final rule REJECTS.)
```

Key `pg_hba.conf` methods: `scram-sha-256` (modern password — use this), `cert` (require a valid client TLS certificate), `peer` (local socket OS-user match), `reject` (deny), and the forbidden `trust` (no auth — only ever for a locked-down local socket in special cases) and deprecated `md5`. Use **`hostssl`** rather than `host` for any password rule so credentials never cross the wire unencrypted. After editing, `SELECT pg_reload_conf();` — no restart needed.

### 5.3 MySQL: bind-address, skip-networking, host grants **[I/A]**

```ini
# ===== my.cnf =====
[mysqld]
bind-address = 10.0.1.5        # private NIC only (default in modern installs is 127.0.0.1 — good)
# bind-address = 127.0.0.1     # if app is on the same host, loopback only is even safer
port = 3306
require_secure_transport = ON  # REFUSE any non-TLS connection — strong control
# skip-networking             # extreme: disable TCP entirely, socket-only (single-host setups)
```

MySQL's per-account `@host` (§4.3) is itself an access-control layer: even if the port were reachable, `'app'@'10.0.%'` only authenticates from the private range. Combine `bind-address` + `require_secure_transport` + host-scoped grants + a firewall.

### 5.4 TLS / SSL for client connections **[I/A]**

Without TLS, credentials and data cross the network in cleartext, readable by anyone who can sniff the wire (a compromised switch, a malicious cloud tenant, a misconfigured network). **Encrypt every client connection in production.** The server presents a certificate; clients can be told to merely *encrypt* (`require`), to *verify the CA* (`verify-ca`), or to *verify the CA and that the hostname matches* (`verify-full` — the only mode that prevents man-in-the-middle, use it).

```bash
# Postgres client SSL modes, weakest → strongest:
psql "host=db.internal dbname=appdb user=app_svc sslmode=require"      # encrypt only (no MITM protection)
psql "host=db.internal dbname=appdb user=app_svc sslmode=verify-full \
      sslrootcert=/etc/ssl/certs/ca.crt"                              # encrypt + verify CA + hostname (USE THIS)

# MySQL client:
mysql --ssl-mode=VERIFY_IDENTITY --ssl-ca=/etc/ssl/certs/ca.crt -u app_svc -p -h db.internal
```

For internal databases, an internal/private CA (or the cloud provider's managed certs) is fine — you don't need a public CA. **mTLS** (the client also presents a cert, verified by the server via `pg_hba.conf` `cert` method) raises the bar further and is common in zero-trust environments.

### 5.5 Sockets vs TCP, bastions & tunnels **[I/A]**

When the application is on the **same host** as the database, a **Unix domain socket** (`/var/run/postgresql/.s.PGSQL.5432`, `/var/run/mysqld/mysqld.sock`) is faster and more secure than TCP — no network stack, filesystem-permission controlled. Use it for co-located components and local admin.

For remote administration of a database with no public IP, the standard is an **SSH tunnel through a bastion host**: a hardened jump box that *is* reachable (on SSH only), through which you forward the database port to your laptop.

```bash
# Tunnel local port 5433 → db.internal:5432 via the bastion. Then connect psql to localhost:5433.
ssh -L 5433:db.internal:5432 admin@bastion.example.com
psql "host=127.0.0.1 port=5433 dbname=appdb user=app_svc sslmode=require"
```

Cloud equivalents: AWS **SSM Session Manager** / RDS Proxy, GCP **Cloud SQL Auth Proxy** (handles auth + TLS automatically), Azure Bastion. These give audited, credential-free access without opening ports.

---

## 6. Backups & Recovery — The Most Important DBA Skill

### 6.1 Why this is *the* skill, and the cardinal rule **[I]**

If you learn one thing from this guide, learn backups. Every other competency — performance, HA, scaling — makes the database *better*; backups are what stand between the company and **permanent, unrecoverable data loss**. Disks fail, ransomware encrypts, a deploy script runs `TRUNCATE` against prod, a developer fat-fingers `DELETE FROM users` without a `WHERE`. When (not if) one of these happens, your backup is the only thing that brings the company back.

> **The cardinal rule: a backup you have never restored is not a backup — it is a hope.** Backups silently rot: a misconfigured `pg_dump` excludes a schema; the cron job's disk fills and writes zero-byte files; the WAL archive has a gap; the storage was never actually durable. The only proof a backup works is a **test restore**. Schedule regular automated restore tests (restore last night's backup into a scratch instance and run a checksum/row-count/smoke test). Untested backups fail *exactly* when you need them.

Two more rules: follow **3-2-1** (≥3 copies, on ≥2 different media/storage types, with ≥1 off-site/another region) so a single fire/region-outage/account-compromise can't destroy all copies; and keep **at least one copy immutable/offline** (write-once object storage with object-lock, or air-gapped) because ransomware and a compromised admin account will try to delete your backups too.

### 6.2 Logical vs physical backups — the fundamental split **[I/A]**

There are two categories of database backup, and a mature shop uses **both**.

**Logical backups** export the *data and schema as statements/data* (effectively "the SQL to recreate everything"). Examples: `pg_dump`, `mysqldump`/`mydumper`, `mongodump`. They are **portable** (restore into a different major version or even different hardware/OS), **selective** (dump one table or schema), and human-inspectable. But they are **slow to produce and especially slow to restore** for large databases (the restore re-executes every insert and rebuilds every index), and they capture a *point* in time, not a continuous stream.

**Physical backups** copy the *actual data files / data directory at the block level*. Examples: `pg_basebackup`, Percona **XtraBackup** (MySQL), filesystem/LVM/cloud-disk **snapshots**. They are **fast to restore** (it's a file copy, no re-execution), **version- and platform-specific**, and — crucially — they pair with **WAL/binlog archiving** to enable **Point-In-Time Recovery**. This is what you use for large production databases where restore *time* (RTO) matters.

| | **Logical** (`pg_dump`, `mysqldump`) | **Physical** (`pg_basebackup`, XtraBackup, snapshots) |
|---|---|---|
| What it copies | SQL statements / data export | Raw data-directory files (block-level) |
| Restore speed | Slow (re-runs inserts, rebuilds indexes) | Fast (file copy) |
| Portability | High (cross-version, cross-platform) | Low (same major version & arch) |
| Selective | Yes (single table/schema) | No (whole cluster) |
| Enables PITR | No (point snapshot only) | Yes (with WAL/binlog archiving) |
| Best for | Small DBs, migrations, dev refresh, logical export | Large DBs, fast RTO, PITR, replicas |

### 6.3 PostgreSQL logical backups: `pg_dump` / `pg_dumpall` **[I]**

```bash
# pg_dump dumps ONE database. Use the CUSTOM format (-Fc): compressed, allows selective/parallel restore.
pg_dump -h db.internal -U appowner -Fc -f /backups/appdb_$(date +%F).dump appdb

# Parallel dump (directory format -Fd) for big DBs — multiple worker processes:
pg_dump -h db.internal -U appowner -Fd -j 4 -f /backups/appdb_$(date +%F).dir appdb

# pg_dumpall captures CLUSTER-WIDE objects pg_dump misses: ROLES, TABLESPACES, GRANTS.
# Run it alongside per-db dumps so you can recreate users on restore:
pg_dumpall -h db.internal -U postgres --globals-only -f /backups/globals_$(date +%F).sql

# --- RESTORE (custom/dir format uses pg_restore) ---
createdb -h db.internal -U postgres appdb_restored
pg_restore -h db.internal -U postgres -d appdb_restored -j 4 /backups/appdb_2026-06-25.dump
# Restore a SINGLE table from the dump:
pg_restore -h db.internal -U postgres -d appdb_restored -t users /backups/appdb_2026-06-25.dump
```

### 6.4 MySQL logical backups: `mysqldump` / `mydumper` **[I]**

```bash
# Consistent dump of a transactional (InnoDB) database WITHOUT locking the whole server:
mysqldump --single-transaction \   # take a consistent snapshot in one transaction (InnoDB) — no global lock
          --routines --triggers --events \   # include stored programs (mysqldump omits these by default!)
          --source-data=2 \        # record the binlog position as a COMMENT (for PITR / setting up replicas)
          -u backup_user -p appdb > /backups/appdb_$(date +%F).sql

# Restore:
mysql -u root -p appdb < /backups/appdb_2026-06-25.sql

# For LARGE MySQL databases, mysqldump is single-threaded and slow. Use mydumper/myloader (parallel):
mydumper -h db.internal -u backup_user -p '...' -B appdb -o /backups/appdb.dir --threads 4 --compress
myloader -h db.internal -u root      -p '...' -B appdb -d /backups/appdb.dir --threads 4
```

> **Best practice:** always pass `--single-transaction` for InnoDB (avoids the `FLUSH TABLES WITH READ LOCK` that freezes writes), and **don't forget `--routines --triggers --events`** — `mysqldump` silently omits stored procedures, triggers and events by default, a classic "my restore is missing things" surprise.

### 6.5 Physical backups & Point-In-Time Recovery (PITR) — the heart of serious DBA **[A]**

This is the technique that lets you **restore to any second in the past** — e.g. "to 14:32:59, just before the bad `DELETE` ran". It is the gold standard for production and the difference between losing a day of data and losing a few seconds.

**How it works (Postgres terms, MySQL is analogous with the binlog).** A database's **Write-Ahead Log (WAL)** records *every change* before it touches the data files (that's how it guarantees durability). PITR combines:

1. A **base backup** — a physical copy of the data directory at some moment (a "starting point"), via `pg_basebackup`.
2. **Continuous WAL archiving** — every WAL segment is copied to safe storage as it fills (the `archive_command`). This is an *unbroken stream of every change since the base backup*.

To recover, you **restore the base backup**, then **replay the archived WAL forward** up to your chosen `recovery_target_time`. Because WAL is continuous, you can stop at any instant — hence "point in time". Your **RPO ≈ how recent your last archived WAL segment is** (seconds to a minute), and your **RTO ≈ base-restore time + WAL-replay time**.

```conf
# ===== postgresql.conf — enable continuous WAL archiving =====
wal_level = replica                 # (or 'logical') — required for archiving/replication
archive_mode = on
# archive_command runs for each completed WAL segment. It must return 0 ONLY on success,
# and must NOT overwrite an existing file (the %f/%p are filename/path placeholders).
archive_command = 'test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
# In production use a real tool (pgBackRest / WAL-G) that ships WAL to S3/GCS with compression & encryption.
```

```bash
# 1. Take a base backup (this is also how you seed a standby — see §7).
pg_basebackup -h db.internal -U repl_user -D /backups/base_$(date +%F) \
  -Ft -z \                  # tar format, gzip-compressed
  -X stream \               # stream the WAL generated DURING the backup so it's self-contained
  -c fast \                 # trigger an immediate checkpoint (start sooner)
  -P                        # progress

# 2. RECOVER to a point in time:
#    - stop postgres, replace the data dir with the restored base backup,
#    - tell it where to fetch archived WAL and when to stop:
cat >> $PGDATA/postgresql.conf <<'EOF'
restore_command = 'cp /wal_archive/%f %p'        # how to fetch archived WAL during recovery
recovery_target_time = '2026-06-25 14:32:59+00'  # STOP replay just before the disaster
recovery_target_action = 'promote'               # become a normal primary when target is reached
EOF
touch $PGDATA/recovery.signal     # PG 12+: this file triggers recovery mode on next start
# Start postgres: it restores the base, replays WAL to 14:32:59, then promotes. Data after that point is gone (intentionally).
```

> **⚡ Version note:** Since PostgreSQL 12 the old `recovery.conf` is gone — recovery settings live in `postgresql.conf`/`postgresql.auto.conf` and recovery is triggered by an empty **`recovery.signal`** (or `standby.signal`) file. Don't follow pre-12 tutorials that edit `recovery.conf`.

**Don't roll your own.** For real PostgreSQL backup/PITR use **pgBackRest** (the de-facto standard: parallel, incremental, compressed, encrypted, S3/GCS/Azure native, with `restore`/`verify`/`expire` commands and a clean retention policy) or **WAL-G**. For MySQL, **Percona XtraBackup** does the equivalent hot physical backup, and binlog archiving plus `mysqlbinlog --start-position` provides PITR.

```bash
# pgBackRest: a full backup, then PITR restore — what production actually runs.
pgbackrest --stanza=appdb backup --type=full              # full backup (also: incr, diff)
pgbackrest --stanza=appdb backup --type=incr              # fast incremental between fulls
# Restore to a point in time:
pgbackrest --stanza=appdb --type=time \
  --target="2026-06-25 14:32:59+00" --delta restore       # --delta only copies changed files (faster)
```

### 6.6 Filesystem / cloud snapshots **[A]**

Cloud block storage (EBS, Persistent Disk, Azure Disk) and LVM/ZFS offer **point-in-time disk snapshots**. These are fast, but for a *running* database a raw snapshot may be **crash-consistent** but not *application-consistent* unless you either (a) freeze writes briefly, (b) use the database's snapshot-safe mode (`pg_backup_start`/`pg_backup_stop` to mark a consistent window), or (c) rely on the engine's crash recovery to clean up on restore. Snapshots pair well with WAL/binlog archiving (snapshot = base, archive = continuous changes). They're convenient on managed services (RDS "snapshots") but **verify restores** all the same.

### 6.7 Backup scheduling, retention & a sample policy **[I/A]**

A backup *strategy* is a schedule + retention + offsite + verification. Derive it from your RPO/RTO.

| Component | Example policy |
|---|---|
| Continuous WAL/binlog archiving | Always on → RPO ≈ seconds |
| Full physical backup | Nightly (or weekly full + nightly incr with pgBackRest) |
| Logical dump | Weekly (portability / cross-version safety net) |
| Retention | 7 daily, 4 weekly, 12 monthly (GFS — Grandfather-Father-Son) |
| Off-site | Replicate backups to a second region / immutable object store |
| Verification | Automated weekly test-restore + monthly full DR drill |
| Encryption | All backups encrypted at rest; keys in a KMS |

```bash
# A minimal cron for logical dumps (real shops use pgBackRest's own scheduler/retention).
# /etc/cron.d/pg-backup  — nightly at 02:30, keep 14 days, fail loudly.
30 2 * * * postgres pg_dump -Fc appdb -f /backups/appdb_$(date +\%F).dump \
  && find /backups -name 'appdb_*.dump' -mtime +14 -delete \
  || echo "PG BACKUP FAILED" | mail -s "ALERT: backup failed" dba@example.com
```

---

## 7. High Availability & Replication

### 7.1 Why replication exists **[A]**

A single database server is a **single point of failure**: if its disk dies, its kernel panics, or its data centre loses power, your application is *down* and possibly your data is *gone*. **Replication** maintains one or more **copies (replicas/standbys)** of the database on other servers, kept continuously in sync with the **primary**. This buys two distinct things people often conflate:

- **High Availability (HA)** — if the primary fails, a standby can be **promoted** to take over, restoring service quickly (low RTO). This is about *uptime*.
- **Read scaling** — read-only queries can be served by replicas, offloading the primary (§8). This is about *throughput*.

Replication also underpins disaster recovery (a standby in another region survives a regional outage) and zero/low-downtime maintenance (fail over to a standby, patch the old primary).

### 7.2 Physical/streaming vs logical replication **[A]**

Two mechanisms, different trade-offs:

**Physical / streaming replication** ships the **WAL** (the byte-level change log) from primary to standby, which *replays* it to stay an exact block-for-block copy. It replicates **the entire cluster**, is low-overhead, and gives a standby you can promote or read from. But the standby is the **same major version**, read-only, and an exact copy (you can't replicate just one table or transform data). This is the default for HA in Postgres (`primary` → `hot standby`) and MySQL's classic/GTID replication is conceptually similar (ships the binlog).

**Logical replication** decodes the WAL/binlog into **logical change events** (INSERT/UPDATE/DELETE on specific tables) and applies them on the subscriber. It can replicate **selected tables**, **across major versions** (the killer feature for near-zero-downtime upgrades, §12), to a **writable** target, and into a *different schema*. The cost: more overhead, some limitations (DDL isn't replicated automatically, large transactions, conflict handling). Postgres has built-in logical replication (`CREATE PUBLICATION` / `CREATE SUBSCRIPTION`); MySQL achieves similar with binlog + tools.

| | **Physical / streaming** | **Logical** |
|---|---|---|
| Granularity | Whole cluster | Selected tables/rows |
| Cross-version | No (same major) | Yes |
| Target writable | No (read-only standby) | Yes |
| Overhead | Low | Higher |
| Main use | HA, read replicas, DR | Major upgrades, selective replication, data integration |

### 7.3 Synchronous vs asynchronous — the RPO knob **[A]**

When the primary commits a write, does it **wait** for a replica to confirm it received/wrote that change?

- **Asynchronous** (the common default): the primary commits and replies to the client *immediately*, shipping WAL to replicas in the background. **Fast**, but if the primary dies before a write reached the replica, that write is **lost** on failover → **RPO > 0** (you can lose the last few seconds). Replicas can also lag.
- **Synchronous**: the primary **waits** for at least one (or a quorum of) replica(s) to confirm before telling the client "committed". **RPO = 0** (no committed transaction is ever lost on failover), at the cost of **higher write latency** (a network round-trip per commit) and reduced availability if synchronous replicas are unreachable (the primary may stall).

```conf
# ===== Postgres synchronous replication (primary's postgresql.conf) =====
synchronous_standby_names = 'ANY 1 (standby1, standby2)'   # wait for ANY 1 of these to confirm
synchronous_commit = on        # commit waits for the sync standby's WAL flush (RPO=0 for that standby)
# 'remote_apply' is even stricter (wait for replay so reads on the standby see it); 'remote_write' is looser.
```

The pragmatic real-world pattern: **synchronous to one nearby standby (RPO 0, low latency) + asynchronous to a distant DR standby (geo-redundant, doesn't slow commits).** Choose sync only where the business truly needs RPO 0 (payments, ledgers); many systems accept seconds of RPO for the latency/availability win of async.

### 7.4 Setting up a Postgres streaming standby **[A]**

```bash
# On the PRIMARY: a replication role + a pg_hba rule allowing the standby (see §5.2).
psql -c "CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'CHANGE_ME';"
# pg_hba.conf:  host replication repl_user 10.0.1.0/24 scram-sha-256
# postgresql.conf:  wal_level=replica  max_wal_senders=10  (defaults are fine)

# On the STANDBY: clone the primary's data directory and configure it to follow.
sudo systemctl stop postgresql
sudo -u postgres rm -rf /var/lib/postgresql/17/main/*
sudo -u postgres pg_basebackup \
  -h primary.internal -U repl_user \
  -D /var/lib/postgresql/17/main \
  -Fp -Xs -P -R          # -R writes standby.signal + primary_conninfo automatically (the magic flag)
# -R created these for us in postgresql.auto.conf:
#   primary_conninfo = 'host=primary.internal user=repl_user password=... sslmode=require'
#   plus an empty standby.signal file → start in standby (read-only, hot-standby) mode.
sudo systemctl start postgresql

# Verify on PRIMARY: see connected standbys and their lag.
psql -x -c "SELECT client_addr, state, sync_state,
            pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
            FROM pg_stat_replication;"
```

### 7.5 Automatic failover, quorum & split-brain **[A]**

Replication gives you a standby; **failover** is the act of promoting it when the primary dies, and **automatic** failover is what keeps RTO low (no human paged at 3 a.m.). Doing it safely is *hard* because of two dangers:

- **Split-brain**: if the network partitions and the old primary is still alive (just unreachable) while a standby gets promoted, you now have **two primaries accepting writes** → divergent, corrupted, unmergeable data. This is the nightmare scenario.
- **Premature failover** (flapping): promoting on a transient blip causes needless churn and risk.

The solution is a **cluster manager** that uses **consensus/quorum** (an odd number of voting members so a majority can always be established) and **fencing** (forcibly demoting/killing the old primary — STONITH, "shoot the other node in the head") to guarantee only one primary exists.

| Tool | Engine | Notes |
|---|---|---|
| **Patroni** | PostgreSQL | The modern standard. Uses a DCS (etcd/Consul/ZooKeeper) for leader election; automatic failover, REST API; pairs with HAProxy/PgBouncer for routing |
| **repmgr** | PostgreSQL | Older, simpler; replication management + failover (uses a witness for quorum) |
| **pg_auto_failover** | PostgreSQL | Citus/Microsoft's monitor-based approach |
| **Orchestrator** | MySQL | Topology management + automated failover; widely used (GitHub) |
| **MySQL InnoDB Cluster** | MySQL | Group Replication + MySQL Router + MySQL Shell — official, quorum-based (Paxos-like) |
| **Galera Cluster** | MySQL/MariaDB | Synchronous multi-primary; built-in quorum |

**Patroni** is the canonical Postgres answer: each node runs a Patroni agent that races to hold a leader lock in **etcd**; only the lock-holder is primary; if it fails to renew the lock (crash/partition) another node wins the lock and promotes — and because the old primary can't hold the lock, split-brain is prevented. A load balancer (HAProxy) routes writes to whoever Patroni's REST API reports as leader. **Read scaling** then routes read-only traffic across the standbys (§8), often via a separate read endpoint.

> **Operational caution:** automatic failover is powerful and dangerous. Test it relentlessly in staging (kill the primary, pull its network, fill its disk) and ensure your **app reconnects** cleanly to the new primary. A failover that promotes a standby but leaves the app pinned to a dead IP is just a slower outage.

---

## 8. Scaling: Read Replicas, Sharding & Partitioning

### 8.1 Vertical vs horizontal — and "scale up first" **[A]**

When the database can't keep up, there are two directions:

- **Vertical scaling (scale up)** — give the *one* server more CPU, RAM, faster NVMe, more IOPS. Simple, no app changes, no consistency headaches. The practical default first move — modern single servers are *enormous* (hundreds of cores, terabytes of RAM) and handle far more than people expect. The limit: you eventually hit the biggest available machine, and a bigger machine is a bigger single point of failure and a bigger blast radius.
- **Horizontal scaling (scale out)** — spread load across *many* servers. Two flavours: **read replicas** (easy, for read-heavy workloads) and **sharding** (hard, for when writes/data exceed one machine).

**The DBA's mantra: scale up, then add read replicas, then cache, and only shard when you genuinely must.** Sharding adds enormous operational and application complexity; defer it as long as a beefier box + replicas + a cache tier suffice.

### 8.2 Read replicas & read/write splitting **[A]**

The most common and cost-effective scale-out: spin up streaming replicas (§7) and route **read-only** traffic to them, keeping **writes** on the primary. This linearly scales read capacity, which is what most web apps are bottlenecked on. The catch is **replication lag** (async replicas are slightly behind), which causes **read-after-write anomalies**: a user writes, then immediately reads from a lagging replica and doesn't see their own write. Handle it by routing reads that must be fresh to the primary (or using `synchronous_commit = remote_apply`, or per-request "read your writes" pinning).

Routing happens at the app (a primary DSN + a replica DSN, the ORM picks), or in a proxy that understands SQL:

```sql
-- ProxySQL (MySQL) query rules: send writes to the primary hostgroup, reads to the replica hostgroup.
-- (configured in ProxySQL's admin interface)
INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup, apply)
VALUES (1, '^SELECT.*FOR UPDATE', 10, 1),   -- locking selects → PRIMARY (hostgroup 10)
       (2, '^SELECT',            20, 1);    -- plain reads → REPLICAS (hostgroup 20)
LOAD MYSQL QUERY RULES TO RUNTIME; SAVE MYSQL QUERY RULES TO DISK;
```

For analytics specifically, route the heavy OLAP queries to a **dedicated replica** (or an analytics warehouse) so they never compete with OLTP traffic on the primary (§1.4).

### 8.3 Partitioning — splitting a big table within one server **[A]**

**Partitioning** divides one logically-single large table into many physical **partitions** (sub-tables) by a key — by **range** (e.g. one partition per month of `created_at`), **list** (by region/tenant), or **hash**. It is *not* the same as sharding (still one server), and it's a maintenance/performance technique:

- **Partition pruning**: a query with `WHERE created_at >= '2026-06-01'` only scans the relevant partition(s), not the whole table.
- **Cheap data lifecycle**: dropping old data is `DROP TABLE old_partition` (instant, no bloat) instead of a giant slow `DELETE` (which leaves dead rows for vacuum).
- **Smaller indexes** per partition, faster maintenance, parallelisable vacuum/analyze.

```sql
-- PostgreSQL declarative range partitioning by month (schema depth lives in the POSTGRESQL guide).
CREATE TABLE events (id bigint, created_at timestamptz NOT NULL, payload jsonb)
  PARTITION BY RANGE (created_at);
CREATE TABLE events_2026_06 PARTITION OF events
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE events_2026_07 PARTITION OF events
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
-- Dropping June later is instant + bloat-free:  DROP TABLE events_2026_06;
-- Use pg_partman to AUTOMATE creating/retiring time partitions.
```

### 8.4 Sharding — splitting data across many servers **[A]**

**Sharding** distributes rows across **multiple independent database servers (shards)** by a **shard key** (e.g. `user_id % N`, or by tenant, or by a hash). Each shard holds a subset of the data and handles its own reads/writes, so you scale writes and total data size beyond one machine. This is how the largest systems scale — and it is the **hardest** thing in this guide.

The costs you take on: choosing a good shard key (a bad one creates **hotspots** — one shard overloaded); **cross-shard queries and joins** become slow/complex or impossible; **cross-shard transactions** lose simple ACID (you need two-phase commit or saga patterns); **rebalancing** when you add shards is painful; every operational task (backup, schema migration, monitoring) now happens N times. **Only shard when vertical scaling + replicas + caching are genuinely exhausted.**

| Approach | What it is |
|---|---|
| **App-level sharding** | The app computes the shard from the key and connects to the right server. Max control, max app complexity |
| **Citus** (Postgres extension) | Turns Postgres into a distributed/sharded database with a coordinator + workers; transparent-ish |
| **Vitess** (MySQL) | The proven MySQL sharding layer (built at YouTube, runs Slack/GitHub); routing, resharding, online schema change |
| **MongoDB sharded cluster** | Native sharding via `mongos` routers + config servers (§13) — document DBs make this more first-class |
| **NewSQL / distributed SQL** | CockroachDB, YugabyteDB, TiDB, Spanner — databases built sharded-from-scratch; you get horizontal scale + SQL + ACID without bolting it on |

If you're greenfield and *know* you'll need horizontal write scale, a **distributed SQL** database (CockroachDB/Yugabyte/Spanner) is often a better choice than sharding Postgres/MySQL by hand.

### 8.5 The caching tier **[A]**

Before (and alongside) scaling the database, put a **cache** in front of it. A read that hits **Redis** (§13) returns in microseconds and never touches the database, absorbing the bulk of read traffic for hot data (sessions, rendered pages, computed aggregates, lookup tables). This is frequently the **highest-leverage scaling move** — far cheaper than more database servers. See the **[Redis guide](REDIS_GUIDE.md)** for cache patterns (cache-aside, write-through, TTLs, invalidation). The DBA's role: provision/operate the cache, watch its hit ratio and memory, and ensure cache failures degrade gracefully (the DB must survive a cold cache without melting — beware the **thundering herd** when a popular key expires).

---

## 9. Monitoring & Observability

### 9.1 Why monitor, and the golden signals **[I]**

You cannot operate what you cannot see. Monitoring exists so you learn the database is sick **before users do**, can **diagnose** incidents from data instead of guesses, and can **plan capacity** from trends rather than surprises. The goal is the classic trio: **metrics** (numeric time series — connections, lag, cache hit ratio), **logs** (slow queries, errors, lock waits), and **traces** (request flow through the system).

The database "golden signals" every DBA watches:

| Signal | What it tells you | Alert when |
|---|---|---|
| **Connections** (active/idle/total) | Are you near `max_connections`? Connection storm? | > ~80 % of max |
| **Cache hit ratio** | Fraction of reads served from RAM vs disk | Postgres < ~99 %, MySQL buffer-pool low |
| **Replication lag** | How far behind the replicas are | > a few seconds (workload-dependent) |
| **Locks / blocked queries** | Contention; a long transaction blocking others | Any long-held lock / lock wait |
| **Slow queries** | Queries exceeding the threshold | Spikes in count or duration |
| **Disk usage & IOPS / latency** | Running out of space; I/O saturation | Disk > ~80 %; rising I/O wait |
| **Transaction-ID wraparound** (Postgres) | Age of oldest unfrozen XID — *can shut the DB down* | `age(datfrozenxid)` high — see §10 |
| **Bloat** (Postgres) | Dead tuples wasting space, slowing scans | High dead-tuple ratio |
| **Errors / restarts / crashes** | The server flapping | Any unexpected restart |

**Disk-full is the classic, avoidable, catastrophic outage** — a full data disk crashes the database hard and can corrupt it. Always alert on disk well before 100 %, and remember WAL/binlogs and the slow-query log can fill a disk fast.

### 9.2 PostgreSQL: the `pg_stat_*` views **[I/A]**

Postgres exposes its internals through **statistics views** you query with plain SQL — the DBA's dashboard before any external tool.

```sql
-- Connections by state — are you near the limit / leaking idle-in-transaction?
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
-- 'idle in transaction' connections are dangerous: they hold locks & block vacuum. Hunt long ones:
SELECT pid, usename, state, now()-xact_start AS xact_age, query
FROM pg_stat_activity
WHERE state = 'idle in transaction' AND now()-xact_start > interval '5 min';

-- Cache hit ratio (want > 0.99 on an OLTP DB):
SELECT sum(blks_hit)::float / nullif(sum(blks_hit)+sum(blks_read),0) AS cache_hit_ratio
FROM pg_stat_database;

-- Replication lag, from the PRIMARY:
SELECT client_addr, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Bloat / dead tuples needing vacuum:
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Transaction-ID wraparound watch (act long before this gets near 2^31):
SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database ORDER BY xid_age DESC;

-- Blocked queries (who is blocking whom):
SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

**`pg_stat_statements`** is the single most valuable extension for tuning: it records normalised query texts with aggregate call counts, total/mean time and rows — your top-queries-by-total-time report (§10). Enable it once and never look back:

```sql
-- In postgresql.conf:  shared_preload_libraries = 'pg_stat_statements'  (needs restart), then:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- Top 10 queries by total time consumed across the whole server:
SELECT round(total_exec_time::numeric,1) AS total_ms, calls,
       round(mean_exec_time::numeric,2) AS mean_ms, query
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

### 9.3 MySQL: Performance Schema & sys schema **[I/A]**

MySQL's equivalent is the **Performance Schema** (low-level instrumentation) with the friendlier **`sys` schema** views on top, plus `SHOW ENGINE INNODB STATUS` for the InnoDB internals.

```sql
-- Top statements by total latency (the sys schema makes Performance Schema readable):
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;
-- Statements doing full table scans:
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
-- Current connections / threads:
SHOW STATUS LIKE 'Threads_connected'; SHOW STATUS LIKE 'Threads_running';
-- InnoDB buffer pool hit ratio (reads served from memory):
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';   -- compute 1 - reads/read_requests
-- Replication health on a replica:
SHOW REPLICA STATUS\G   -- watch Seconds_Behind_Source, Replica_IO_Running, Replica_SQL_Running
-- InnoDB internals (locks, deadlocks, I/O):
SHOW ENGINE INNODB STATUS\G
```

> **⚡ Version note:** MySQL 8.0.22+ renamed the replication commands and columns to inclusive terminology: `SHOW REPLICA STATUS` (was `SHOW SLAVE STATUS`), `START REPLICA`, `CHANGE REPLICATION SOURCE TO`, `Seconds_Behind_Source`. Old `SLAVE`/`MASTER` syntax still works as deprecated aliases.

### 9.4 External stacks: Prometheus, exporters, PMM, pgwatch **[I/A]**

Querying views by hand is for incidents; **continuous** monitoring needs a stack that scrapes, stores, graphs and alerts. The 2026-standard pattern: a **Prometheus exporter** per database exposes metrics; **Prometheus** scrapes and stores them; **Grafana** dashboards them; **Alertmanager** pages on thresholds.

| Tool | Role |
|---|---|
| **postgres_exporter** / **mysqld_exporter** | Expose engine metrics to Prometheus |
| **Prometheus + Grafana + Alertmanager** | Scrape, store, dashboard, alert (the common open-source stack) |
| **Percona Monitoring & Management (PMM)** | Turnkey monitoring for MySQL/Postgres/Mongo — exporters + dashboards + query analytics in one |
| **pgwatch** | Postgres-focused monitoring with prebuilt dashboards |
| **Datadog / New Relic / Cloud-native (CloudWatch, Cloud Monitoring)** | Commercial/managed APM with DB integrations |

**Alerting** is the point — a dashboard nobody looks at saves no one. Set **threshold-based alerts** on the golden signals (disk > 80 %, connections > 80 % of max, replica lag > 30 s, any unexpected restart, wraparound XID age high) routed to your on-call (PagerDuty/Opsgenie). Tune thresholds to avoid **alert fatigue** (too many false pages and people ignore the real one). Pair metrics with **log aggregation** (ship Postgres/MySQL logs to Loki/ELK/cloud logging) so the slow-query and error logs are searchable during an incident.

---

## 10. Performance Tuning & Maintenance

### 10.1 The tuning methodology **[A]**

Tuning is **measure → find the bottleneck → fix the biggest one → measure again** — never guess-and-tweak random config. Most database performance problems are **a handful of bad queries**, not the server config. So the order of operations is:

1. **Find the expensive queries** (`pg_stat_statements` / slow-query log / `sys.statement_analysis`) — sort by *total* time (frequency × duration), because a fast query run a million times costs more than a slow one run twice.
2. **Understand why** with `EXPLAIN (ANALYZE, BUFFERS)` — is it a sequential scan that should be an index scan? A bad join order? Missing statistics? (Query-plan reading depth is in the **[PostgreSQL](POSTGRESQL_GUIDE.md)** / engine guides.)
3. **Fix at the right layer** — add/adjust an **index**, rewrite the query, fix statistics (`ANALYZE`), or *only then* touch config.
4. **Tune config** (memory, parallelism) once queries are sane.
5. **Re-measure** and watch for regressions.

```sql
-- The core diagnostic. ANALYZE actually RUNS the query (careful on writes!); BUFFERS shows cache vs disk.
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42 AND status = 'open';
-- Red flags: "Seq Scan" on a big table with a selective filter (wants an index);
--   actual rows ≫ estimated rows (stale statistics → run ANALYZE);
--   huge "Buffers: read=" (cache misses → undersized shared_buffers/buffer_pool or too much data).
```

### 10.2 Indexing for operations **[A]**

The engine guides teach index *design*; the DBA cares about index *operations*. Key operational facts:

- **Build indexes without locking writes.** A plain `CREATE INDEX` takes an `ACCESS EXCLUSIVE`-ish lock that blocks writes for the whole build — unacceptable on a busy prod table. Use **`CREATE INDEX CONCURRENTLY`** (Postgres) / online DDL (MySQL `ALGORITHM=INPLACE`) which builds without blocking writes (slower, can't run in a transaction).
- **Find unused indexes** — they cost write performance and disk for nothing. Drop them.
- **Find missing indexes** — from the queries doing sequential scans / not using indexes.
- **Watch index bloat** (Postgres) — rebuild bloated indexes with `REINDEX CONCURRENTLY`.

```sql
-- Postgres: build an index online (no write downtime):
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);

-- Find indexes that are never used (drop candidates):
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY relname;

-- Rebuild a bloated index without blocking (PG 12+):
REINDEX INDEX CONCURRENTLY idx_orders_customer;
```

```sql
-- MySQL online index add (doesn't block reads/writes for InnoDB):
ALTER TABLE orders ADD INDEX idx_customer (customer_id), ALGORITHM=INPLACE, LOCK=NONE;
```

### 10.3 PostgreSQL VACUUM, autovacuum & bloat — the signature PG ops topic **[A]**

PostgreSQL's **MVCC** (Multi-Version Concurrency Control) means an `UPDATE` or `DELETE` does **not** overwrite/remove the old row immediately — it writes a new version and marks the old one **dead** (so concurrent transactions still see a consistent snapshot). Those dead rows ("dead tuples") accumulate as **bloat**: wasted space that slows scans and inflates tables/indexes. **VACUUM** reclaims dead tuples (making the space reusable); **autovacuum** runs VACUUM automatically in the background. This is unique among the major engines and a constant source of incidents, so understand it deeply.

Two things VACUUM does that you must respect:

1. **Reclaim dead tuples** so space is reused and bloat is bounded. **`VACUUM FULL`** rewrites the table to *fully* compact it but takes an `ACCESS EXCLUSIVE` lock (blocks everything) and needs free disk — use sparingly; prefer keeping autovacuum healthy, or `pg_repack` for online compaction.
2. **Freeze old transaction IDs** to prevent **transaction-ID wraparound** — a catastrophic condition where, if the oldest unfrozen XID gets too old, **Postgres shuts down writes to protect data**. Autovacuum's "anti-wraparound" runs handle this; **never disable autovacuum**, and alert on `age(datfrozenxid)` (§9.2).

```sql
-- Manual maintenance (autovacuum normally handles this — manual is for catch-up or specific tables):
VACUUM (VERBOSE, ANALYZE) orders;     -- reclaim dead tuples AND refresh planner statistics
ANALYZE orders;                       -- just refresh statistics (the planner's input — keep these fresh!)

-- Tune autovacuum to be MORE aggressive on a hot, frequently-updated table:
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,   -- vacuum after 2% of rows change (default 20% is too lazy for big tables)
  autovacuum_vacuum_cost_limit = 2000      -- let it work faster
);
```

> **The classic autovacuum incident:** a long-running transaction (or an abandoned `idle in transaction` connection, or an old replication slot) holds back the "oldest visible XID", so autovacuum **can't remove dead tuples newer than that** — bloat balloons and wraparound age climbs even though autovacuum is "running". The fix is to find and kill the long transaction / drop the stale replication slot, not to vacuum harder. This is why §9.2 hunts `idle in transaction`.

### 10.4 MySQL/InnoDB maintenance **[A]**

InnoDB updates rows in place (no Postgres-style dead-tuple bloat), but it still fragments over time and its statistics drift. Maintenance:

```sql
ANALYZE TABLE orders;     -- refresh index statistics for the optimizer
OPTIMIZE TABLE orders;    -- rebuild/defragment the table & indexes (for InnoDB it's an online rebuild)
-- Watch the InnoDB buffer pool hit ratio and history-list length (long uncommitted transactions
-- inflate the undo log, the rough MySQL analogue of PG's dead-tuple problem). See SHOW ENGINE INNODB STATUS.
```

### 10.5 Lock contention **[A]**

Locks enforce consistency but **contention** (many sessions fighting for the same rows/tables) tanks throughput and causes timeouts/deadlocks. The DBA's job: detect long lock waits (§9), find the **blocking** transaction (often a long-running or `idle in transaction` one), and resolve it (kill it, or fix the app to keep transactions short). General rules: **keep transactions short**, acquire locks in a **consistent order** (to avoid deadlocks), do bulk operations in **batches** rather than one giant statement that locks a table for minutes, and use online-DDL tools (`pg_repack`, `gh-ost`/`pt-online-schema-change` for MySQL) for big schema changes on hot tables.

```sql
-- Postgres: terminate a specific blocking backend (last resort during an incident):
SELECT pg_terminate_backend(12345);   -- 12345 = the offending pid from pg_stat_activity
-- MySQL: KILL the offending thread id from SHOW PROCESSLIST / sys.processlist:
KILL 12345;
```

### 10.6 Parameter tuning, sane defaults & autotuning tools **[A]**

Once queries are sane, tune the memory/IO/parallelism parameters from §3 to the hardware and workload. Don't tune blind: tools like **PGTune** (web) and **pgtune**-style scripts generate a sensible `postgresql.conf` baseline from RAM/CPU/workload-type; MySQL has similar calculators. Treat their output as a **starting point**, then iterate from real metrics. The discipline: **change one thing at a time, measure, keep a record** so you can attribute and revert.

---

## 11. Security & Hardening

### 11.1 The security model **[A]**

Database security is **defence in depth**: assume any single control can fail and stack several so a breach of one doesn't expose the data. The layers, roughly outside-in: network isolation (§5) → authentication (§4) → authorization/least-privilege (§4) → encryption (in transit & at rest) → auditing → patching → secrets management → application-side defences (SQL-injection) → compliance controls. A useful framing: protect against the **network attacker** (firewall, TLS), the **insider/over-privileged app** (least privilege, RLS), the **stolen disk/backup** (encryption at rest), and the **stolen credential** (rotation, MFA on admin paths, short-lived creds).

### 11.2 Encryption in transit and at rest **[A]**

- **In transit** — TLS on every client connection and every replication link (§5.4). Non-negotiable in production. Without it, credentials and data are sniffable.
- **At rest** — encrypt the data files and backups so a stolen disk, snapshot or backup tape is useless. Options: **filesystem/volume encryption** (LUKS on Linux, cloud-provider disk encryption — the easiest, transparent to the DB), or **Transparent Data Encryption (TDE)** built into the engine (SQL Server and MySQL Enterprise/Percona have TDE; community Postgres relies on filesystem-level encryption, though TDE is on the roadmap and exists in forks). **Always encrypt backups** — they're the most commonly stolen copy of your data.

### 11.3 Auditing **[A]**

You must be able to answer "who did what, when". Engine auditing records connections, privilege changes and (optionally) statements:

- **PostgreSQL**: **pgAudit** extension — fine-grained logging of SELECT/DML/DDL by class, with object-level rules. Plus `log_connections`/`log_disconnections` and DDL logging.
- **MySQL**: the **Audit Log** plugin (Enterprise) / MariaDB **server_audit** plugin / Percona audit plugin.
- **SQL Server**: **SQL Server Audit** (server- and database-level audits to file/Event Log).

```ini
# Postgres pgAudit (shared_preload_libraries = 'pgaudit'), then:
# Log all DDL and writes, plus role/privilege changes, to the server log for the SIEM to ingest:
pgaudit.log = 'ddl, write, role'
pgaudit.log_catalog = off          # reduce noise from catalog reads
```

Ship audit logs **off the database host** to a SIEM/central log store so an attacker who compromises the box can't erase their tracks, and so logs are tamper-evident.

### 11.4 Patching, CIS benchmarks & hardening checklist **[A]**

Unpatched databases are a top breach vector. Subscribe to your engine's security announcements and apply **minor version updates** promptly (they're backward-compatible and usually low-risk — §12). Use the **CIS Benchmarks** (free hardening checklists for PostgreSQL, MySQL, MongoDB, SQL Server) as your baseline.

A condensed hardening checklist:

- [ ] Not exposed to the internet; firewalled to app subnet only (§5).
- [ ] TLS required for all client & replication connections.
- [ ] No `trust`/anonymous/default-password accounts; strong SCRAM/`caching_sha2_password`.
- [ ] Least-privilege roles; the app role is **not** a superuser and cannot DDL in prod.
- [ ] Superuser/root used only for admin, never by apps; admin access via bastion + MFA.
- [ ] Data files and backups encrypted at rest; backups immutable/off-site.
- [ ] Auditing on, logs shipped off-host to a SIEM.
- [ ] Minor versions patched promptly; CIS benchmark applied.
- [ ] Default ports optionally changed (minor obscurity), test/sample DBs removed.
- [ ] Secrets in a manager, rotated; ideally short-lived/dynamic credentials.

### 11.5 SQL injection — the DB-side perspective **[A]**

SQL injection is primarily an **application** vulnerability (untrusted input concatenated into SQL), and the real fix is **parameterised queries / prepared statements** in the app code (covered in the language/ORM guides — see **[Prisma](PRISMA_ORM_GUIDE.md)**, **[Go ent](GO_ENT_ORM_GUIDE.md)**). But the DBA provides **defence in depth** so a successful injection does limited damage:

- **Least privilege** (§4): if the app role can only `SELECT/INSERT/UPDATE/DELETE` its own tables, an injection can't `DROP TABLE`, read other schemas, or `COPY ... TO PROGRAM` (RCE). This single control turns many catastrophic injections into minor ones.
- **Disable dangerous capabilities** for app roles (Postgres `COPY ... TO/FROM PROGRAM`, `pg_read_file`, untrusted languages; MySQL `LOAD_FILE`, `INTO OUTFILE`, `--secure-file-priv`).
- **Row-Level Security** to bound what any one session can ever see.
- **Auditing/alerting** on anomalous query patterns.

### 11.6 Data masking & compliance basics **[A]**

**Data masking / anonymisation** replaces sensitive values (PII, card numbers) with realistic-but-fake data, primarily so **non-production environments** (dev, staging, analytics) don't contain real customer data — a major compliance and breach-surface reduction. Tools: PostgreSQL **anon** extension, MySQL Enterprise Data Masking, or masking during the backup-to-staging refresh pipeline.

A DBA needs working literacy in the big compliance regimes (your legal/security team owns them, but the controls land on you):

| Regime | Domain | DB controls it drives |
|---|---|---|
| **PCI DSS** | Payment-card data | Encryption at rest/in transit, strict access control, audit logging, network segmentation, no storing CVV |
| **HIPAA** | US healthcare (PHI) | Encryption, audit trails, access controls, BAAs, breach notification |
| **GDPR / CCPA** | EU/CA personal data | Data minimisation, right-to-erasure (deletes that actually delete, incl. backups policy), data-residency, breach reporting |
| **SOC 2** | Service-org trust | Documented controls, access reviews, change management, monitoring (much of this guide) |

The recurring theme: **encrypt, restrict access, audit everything, and be able to find and delete a person's data.** "Right to be forgotten" is operationally tricky because data hides in **replicas, backups and logs** — have a documented policy for it.

---

## 12. Upgrades & Migrations

### 12.1 Minor vs major upgrades **[A]**

Two very different beasts:

- **Minor upgrades** (e.g. 17.2 → 17.5, 8.4.1 → 8.4.3) are **bug/security fixes, same on-disk format**. They're low-risk: install the new package and **restart**; no data conversion. Apply them **promptly** — they're how security patches reach you. Still test in staging and have a rollback (the old package), but these are routine.
- **Major upgrades** (e.g. 16 → 17, 8.0 → 8.4) often **change the on-disk format** and behaviour, so you can't just swap binaries — you must **migrate the data** and re-test the application. Higher risk, more planning, real downtime budget.

### 12.2 PostgreSQL major upgrades **[A]**

Three strategies, trading downtime for complexity:

| Method | Downtime | How |
|---|---|---|
| **`pg_dump` + restore** | High (size-dependent) | Dump from old, restore into new. Simplest, safest, slowest. Good for small DBs |
| **`pg_upgrade`** | Low | Converts the cluster in place (or with `--link`, near-instant by hard-linking files). Fast even for huge DBs |
| **Logical replication** | Near-zero | Stand up new-version cluster, logically replicate from old, cut over when caught up. Best for big/HA systems |

```bash
# pg_upgrade with --link: near-instant major upgrade (16 → 17) by hard-linking data files.
# Both versions' binaries must be installed. Run as the postgres user, both clusters STOPPED.
/usr/lib/postgresql/17/bin/pg_upgrade \
  --old-datadir=/var/lib/postgresql/16/main \
  --new-datadir=/var/lib/postgresql/17/main \
  --old-bindir=/usr/lib/postgresql/16/bin \
  --new-bindir=/usr/lib/postgresql/17/bin \
  --link \              # hard-link instead of copy → fast, but you CAN'T fall back to the old cluster after
  --check               # ALWAYS run with --check FIRST (dry run, reports incompatibilities)
# After: run the generated analyze script to rebuild statistics (the new cluster starts with none):
#   ./analyze_new_cluster.sh   (or vacuumdb --all --analyze-in-stages)
```

> **⚡ Version note:** `pg_upgrade --link` does **not** copy data, so once you start the new cluster you **cannot revert** to the old one (they share files). For a fallback path, omit `--link` (full copy, needs 2× disk) or use the logical-replication method where the old cluster stays intact until cutover. Always **take a backup before any major upgrade.**

The **logical-replication / blue-green** approach is the modern near-zero-downtime path: build the new cluster, create a publication on the old and a subscription on the new, let it catch up, run the app against the new one in read-only validation, then flip writes over in a brief maintenance window. It also lets you upgrade **across** major versions safely and roll back by flipping back.

### 12.3 MySQL major upgrades **[A]**

- **In-place**: install 8.4 binaries over 8.0 data dir; `mysqld` runs the data-dictionary upgrade on first start (8.0+ made this largely automatic; older versions needed `mysql_upgrade`). Test heavily; 8.4 LTS removed deprecated features (re-check your config and SQL).
- **Logical**: `mysqldump`/`mydumper` from old, load into new — portable, slower.
- **Replication-based cutover**: replicate old → new (new-version replica), promote the new version. Near-zero downtime, the production-grade approach.

### 12.4 Schema migrations in production — the everyday "migration" **[A]**

Most "migrations" a DBA reviews aren't version upgrades — they're **schema changes** that ship with application releases (add a column, add an index, change a type). On a busy production database these are dangerous because some DDL takes **blocking locks** that can freeze the table (and thus the app) for the duration. The DBA's job is to make schema changes **safe and online**.

Principles:

- **Expand/contract (backward-compatible) migrations**: never break the currently-running app version. Add the new column/table first (expand), deploy code that writes both old+new, backfill, switch reads, then later drop the old (contract). This decouples deploy from migration and enables rollback.
- **Avoid long table locks**: `CREATE INDEX CONCURRENTLY`; add a column **without** a volatile default on old engines (modern PG/MySQL add nullable/constant-default columns instantly via metadata-only changes); backfill data in **batches**, not one giant `UPDATE`.
- **Use online-schema-change tools** for big MySQL changes: **gh-ost** or **pt-online-schema-change** (build a shadow table, copy in chunks, swap) avoid the long `ALTER` lock. Postgres uses **pg_repack** and concurrent operations.
- **Migrations run by tools** — Prisma Migrate, Flyway, Liquibase, ent, Rails/Django/Alembic — version your schema in code. See the **[Prisma](PRISMA_ORM_GUIDE.md)** and **[Go ent](GO_ENT_ORM_GUIDE.md)** guides. The DBA **reviews** these for production safety (locks, lock_timeout, batch sizes) before they run.
- **Always set a `lock_timeout`/`statement_timeout`** on migrations so a blocked DDL fails fast instead of piling up a queue of blocked queries behind it.

```sql
-- A safe Postgres migration pattern: bounded lock wait, online index, batched backfill.
SET lock_timeout = '3s';                 -- don't wait forever for the lock; fail fast if blocked
ALTER TABLE users ADD COLUMN email_verified boolean;   -- nullable add = instant metadata change (PG 11+)
CREATE INDEX CONCURRENTLY idx_users_email_verified ON users (email_verified);  -- no write lock
-- Backfill in batches in the app/migration runner (not one statement that locks millions of rows):
-- UPDATE users SET email_verified = false WHERE id BETWEEN $1 AND $2;   (loop over ranges)
```

**Test every migration against production-like data and always have a rollback plan** (the contract step, or a tested down-migration). A migration that runs in 10 ms on an empty dev table can lock a 500-million-row prod table for an hour.

---

## 13. Operating MongoDB & Redis as Servers

This library teaches **using** MongoDB and Redis (data model, queries, commands) in the **[MongoDB](MONGODB_GUIDE.md)** and **[Redis](REDIS_GUIDE.md)** guides. Here we cover only how **operating them as servers** differs from the relational ops above.

### 13.1 MongoDB replica sets **[I/A]**

A production MongoDB is never a standalone `mongod` — it's a **replica set**: a group of `mongod` nodes (typically 3) where one is **PRIMARY** (takes writes) and the others are **SECONDARIES** that replicate via the **oplog** (a capped collection recording every write operation). The members **hold an election** (Raft-like consensus) to choose a primary; if the primary fails, a secondary is automatically elected — built-in HA with no external tool. An **arbiter** (a voting-only member with no data) can break ties in a 2-data-node setup, though 3 full data nodes is preferred.

```javascript
// Initialise a 3-node replica set (run in mongosh on one node). Members must reach each other on 27017.
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1.internal:27017" },
    { _id: 1, host: "mongo2.internal:27017" },
    { _id: 2, host: "mongo3.internal:27017" }
  ]
})
rs.status()    // PRIMARY/SECONDARY roles, oplog lag, election state
rs.conf()      // membership, priorities, votes
// Read preference lets you offload reads to secondaries (eventual consistency, like a read replica):
//   db.coll.find().readPref("secondaryPreferred")
```

Ops notes: size the **oplog** so a secondary can be offline for your maintenance window and still catch up (too-small oplog → full resync). **Write concern** is Mongo's RPO knob: `w: "majority"` waits for a majority of nodes (durable, RPO≈0) vs `w: 1` (primary only, faster, riskier); `readConcern` similarly controls read consistency. Backups: **mongodump/mongorestore** (logical) for small sets; filesystem snapshots or **Ops Manager / Atlas continuous backups** with oplog for PITR on large ones.

### 13.2 MongoDB sharded clusters **[A]**

When data/throughput exceeds one replica set, MongoDB **shards** natively. A sharded cluster has three components: **shards** (each itself a replica set, holding a subset of data), **config servers** (a replica set storing cluster metadata/chunk locations), and **`mongos`** routers (stateless query routers the app connects to, which route operations to the right shard by **shard key**). Choosing the shard key well (high cardinality, even distribution, aligned with query patterns) is as critical and unforgiving as in relational sharding (§8.4) — a bad key creates hotspots and jumbo chunks. The balancer redistributes chunks as data grows.

### 13.3 Redis: persistence, the durability question **[I/A]**

Redis keeps data **in RAM**, which is why it's microsecond-fast — and why **persistence is a deliberate choice**, not assumed. The first ops question is: **is this Redis a cache (data is reconstructable, durability optional) or a data store (must survive restarts)?** That answer drives everything.

Two persistence mechanisms, often combined:

- **RDB (snapshotting)**: periodically forks and dumps the whole dataset to a compact `dump.rdb` file. Fast restart, small files, but you **lose everything since the last snapshot** on a crash (minutes of data → high RPO).
- **AOF (Append-Only File)**: logs every write command; on restart, replays them. Much better durability (with `appendfsync everysec` you lose ≤1 s; `always` loses nothing but is slow). Larger files, slower restart; AOF is periodically rewritten/compacted.

```conf
# ===== redis.conf — persistence choices =====
# Pure cache (max speed, OK to lose all data on restart):
#   save ""            # disable RDB snapshots
#   appendonly no      # disable AOF
# Durable data store (recommended for a system of record): AOF + RDB together.
appendonly yes
appendfsync everysec        # fsync once per second — lose ≤1s on crash (good balance)
save 900 1                  # also snapshot RDB: after 900s if ≥1 key changed
save 300 100                #               after 300s if ≥100 keys changed

# ===== Memory management (critical for a cache) =====
maxmemory 4gb               # hard cap; without this Redis can OOM-kill the host
maxmemory-policy allkeys-lru  # when full, EVICT least-recently-used keys (cache behaviour)
# For a DATA STORE you DON'T want silent eviction — use 'noeviction' (writes error when full) and alert on memory.
```

> **⚡ Version & licensing note:** In 2024 Redis Inc. changed Redis's license (to RSALv2/SSPLv1, then Redis 8 added AGPL), prompting the Linux Foundation to fork the last BSD version as **Valkey** (backed by AWS, Google, Oracle and others). **Valkey 8.x** is a drop-in Redis replacement that many distros and clouds have adopted as the default "Redis". Operationally they're equivalent today; pick based on your support/licensing needs. Redis 7.4 introduced hash-field TTLs; Redis 8 folded in the modules (Search, JSON, etc.).

### 13.4 Redis HA: Sentinel vs Cluster **[I/A]**

- **Redis Sentinel**: a set of sentinel processes that **monitor** a primary + replicas, and on primary failure **elect** a new primary and reconfigure replicas/clients — automatic failover for a single (non-sharded) dataset. Clients ask Sentinel for the current primary's address. Good for HA when one node holds all the data.
- **Redis Cluster**: **shards** the keyspace across multiple primaries (16384 hash slots distributed across nodes), each primary with its own replicas, with built-in failover. Use it when the dataset or throughput exceeds one node. Adds constraints (multi-key operations must touch keys in the same slot — use **hash tags**).

```bash
# Set up replication (a replica follows a primary), the building block for Sentinel HA:
redis-cli -h replica1 REPLICAOF primary.internal 6379
# Sentinel watches and fails over automatically (sentinel.conf):
#   sentinel monitor mymaster 10.0.1.5 6379 2     # 2 = quorum of sentinels needed to declare failure
```

---

## 14. Microsoft SQL Server Basics for DBAs

SQL Server is the dominant database in Microsoft/enterprise shops, and its administration differs enough from Postgres/MySQL to warrant its own orientation. (For T-SQL/query depth, consult SQL Server-specific docs; this is the ops orientation.) Runs on Windows and Linux; see the **[Windows Server Administration guide](WINDOWS_SERVER_ADMIN_GUIDE.md)**.

### 14.1 Editions & tools **[I]**

| Edition | Use |
|---|---|
| **Express** | Free, capped (10 GB/db, limited RAM/CPU) — small apps, dev |
| **Developer** | Free, **full Enterprise features**, non-production only — the DBA's lab |
| **Standard** | Production, mid-tier (basic Always On, capped memory) |
| **Enterprise** | Full feature set (advanced HA, partitioning, TDE, etc.) — costly |
| **Azure SQL Database / Managed Instance** | Managed cloud (PaaS) — the managed path |

Core tools: **SSMS** (SQL Server Management Studio, the rich Windows GUI), **Azure Data Studio** (cross-platform, lighter), and **`sqlcmd`** (CLI, scriptable, what you use over a remote session).

```bash
# sqlcmd: connect and run a query (works on Windows & Linux). -E = Windows auth; -U/-P = SQL auth.
sqlcmd -S localhost -U sa -P 'StrongPassword!' -Q "SELECT name, state_desc FROM sys.databases;"
```

### 14.2 Logins vs users — the two-level security model **[I]**

SQL Server splits identity into two levels, which trips up newcomers:

- A **login** is a **server-level** principal — it authenticates you *to the instance*. Two kinds: **Windows authentication** (AD account/group — preferred in enterprises, no password in the app) and **SQL authentication** (username/password stored in SQL Server).
- A **user** is a **database-level** principal mapped to a login — it authorizes you *within a specific database*. Permissions (and **roles** like `db_datareader`, `db_datawriter`, `db_owner`) are granted to users per database.

So "can you connect" (login) and "what can you do in `SalesDB`" (user + role) are separate — least privilege means a login mapped to a `db_datareader`/`db_datawriter` user, never `sysadmin`/`db_owner` for apps.

### 14.3 Backup models: full / differential / log **[I/A]**

SQL Server's backup story is built around the **recovery model**, which determines whether you can do point-in-time recovery:

- **FULL recovery model** + **transaction-log backups** = **PITR** (the SQL Server analogue of WAL/binlog archiving). The transaction log accumulates until you back it up — *you must take regular log backups or the log file grows forever*.
- **SIMPLE recovery model** = log auto-truncates, **no PITR** (only restore to the last full/diff). Fine for data you can rebuild.

The three backup types compose:

| Backup | What | Restore role |
|---|---|---|
| **FULL** | Entire database | The base |
| **DIFFERENTIAL** | Changes since last FULL | Reduces restore time (one diff vs many logs) |
| **LOG** (transaction log) | All log records since last log backup | Enables PITR; restore in sequence |

```sql
-- A classic SQL Server backup chain (FULL recovery model):
BACKUP DATABASE SalesDB TO DISK = 'X:\bak\SalesDB_full.bak' WITH COMPRESSION, CHECKSUM;
BACKUP DATABASE SalesDB TO DISK = 'X:\bak\SalesDB_diff.bak' WITH DIFFERENTIAL, COMPRESSION;
BACKUP LOG SalesDB      TO DISK = 'X:\bak\SalesDB_log.trn'  WITH COMPRESSION;   -- run frequently (e.g. 15 min)
-- PITR restore: full, then latest diff, then logs in order, stopping at a point in time:
RESTORE DATABASE SalesDB FROM DISK='...full.bak' WITH NORECOVERY;
RESTORE DATABASE SalesDB FROM DISK='...diff.bak' WITH NORECOVERY;
RESTORE LOG      SalesDB FROM DISK='...log.trn'  WITH STOPAT = '2026-06-25T14:32:59', RECOVERY;
```

### 14.4 Always On Availability Groups & Agent jobs **[I/A]**

**Always On Availability Groups (AGs)** are SQL Server's flagship HA/DR feature (Enterprise; Standard has Basic AGs): a group of databases replicated to one or more **secondary replicas**, with **automatic failover** (synchronous-commit replicas) and **readable secondaries** for offloading reads/backups — conceptually like Postgres streaming replication + Patroni, integrated and Windows-Failover-Cluster-aware. Synchronous-commit gives RPO 0 within a region; async secondaries provide cross-region DR.

**SQL Server Agent** is the built-in **job scheduler** — the DBA automates backups, index maintenance, integrity checks (`DBCC CHECKDB`) and alerts as **Agent jobs** on schedules. The community-standard maintenance toolkit is **Ola Hallengren's scripts** (backups, integrity, index/stats maintenance) — nearly every SQL Server shop runs them. Always schedule regular **`DBCC CHECKDB`** to detect corruption early.

---

## 15. Automation & Running Databases in Containers / Cloud

### 15.1 Cattle, not pets — automate everything **[A]**

The old model treated a database server as a **pet**: hand-built, hand-configured, irreplaceable, nursed back to health when sick. Modern operations treat infrastructure as **cattle**: identically provisioned from code, disposable, replaced rather than repaired. For databases (which are *stateful*, so you can't be quite as cavalier) this means: the **server config, users, extensions and topology are defined in code**, a fresh replica can be stood up automatically, and recovery is a documented, automated procedure rather than tribal knowledge. The benefits — reproducibility, auditability, fast recovery, no "works on the one server we're afraid to touch" — are exactly what reliability needs.

### 15.2 Infrastructure as Code & configuration management **[A]**

- **Provisioning**: **Terraform**/OpenTofu defines the database instances, networks, parameter groups, replicas and backups declaratively (especially powerful with managed services — an RDS instance + Multi-AZ + automated backups in ~20 lines). **Pulumi**/CloudFormation are alternatives.
- **Configuration**: **Ansible** (or Chef/Puppet/Salt) installs the engine, lays down `postgresql.conf`/`my.cnf`, manages `pg_hba.conf`, creates roles — so a rebuilt host is byte-identical. Pair with the **[Linux Server Administration guide](LINUX_SERVER_ADMIN_GUIDE.md)**.
- **Schema** is code too (migration tools, §12.4), versioned in git and applied through CI/CD (see the **[GitHub Actions CI/CD guide](GITHUB_ACTIONS_CICD_GUIDE.md)** if present).

```hcl
# Terraform: a managed Postgres with automated backups & Multi-AZ HA in a few lines (AWS RDS).
resource "aws_db_instance" "app" {
  engine                  = "postgres"
  engine_version          = "17"
  instance_class          = "db.r6g.xlarge"
  allocated_storage       = 200
  storage_encrypted       = true          # encryption at rest (§11.2)
  multi_az                = true           # synchronous standby + automatic failover (HA)
  backup_retention_period = 14            # automated daily backups + PITR, 14 days
  deletion_protection     = true           # don't let a `terraform destroy` nuke prod data
  publicly_accessible     = false          # NEVER expose to the internet (§5.1)
}
```

### 15.3 Kubernetes operators **[A]**

Running databases on **Kubernetes** used to be discouraged (stateful workloads on a system designed for stateless pods), but **operators** changed that. An **operator** encodes DBA knowledge as a controller: you declare a `PostgresCluster` custom resource (size, replicas, backup schedule, Postgres version) and the operator provisions the pods, persistent volumes, streaming replication, **automated failover**, **continuous backups to S3**, and rolling minor upgrades — the toil from §2-§7, automated and reconciled.

| Operator | Engine |
|---|---|
| **CloudNativePG**, **Crunchy PGO**, **Zalando postgres-operator** | PostgreSQL (CNPG is the popular modern choice) |
| **Percona Operators** | MySQL/Postgres/MongoDB |
| **MongoDB Community/Enterprise Operator** | MongoDB |
| **Vitess Operator** | Sharded MySQL |

Operators need **persistent volumes** (CSI storage with the right IOPS) and careful resource/anti-affinity settings (don't co-locate all replicas on one node). They're excellent for platform teams running many databases as a self-service internal product; for a single database, a managed service is often simpler.

### 15.4 The managed-service operational reality **[A]**

On managed services (§1.5) the *tasks* in this guide map to provider features: backups → automated snapshots + PITR (still test restores!); HA → Multi-AZ checkbox; replicas → "create read replica"; monitoring → Performance Insights/Enhanced Monitoring + CloudWatch; parameters → "parameter groups"; upgrades → maintenance windows. **You still own the strategy**: setting the backup retention to meet your RPO, sizing instances, designing the replica/failover topology, defining alerts and thresholds, reviewing schema migrations, and responding to incidents. Managed = less toil, same responsibility.

---

## 16. Production Operations & Company-Level Practices

### 16.1 On-call & runbooks **[A]**

At company scale, someone is **on-call** for the databases — reachable, with the access and authority to fix a 3 a.m. outage. What makes on-call survivable is **runbooks**: written, step-by-step procedures for the situations you can anticipate, so the on-call engineer (who may be sleepy, junior, or unfamiliar with this system) doesn't have to improvise under pressure. A good runbook library covers: "disk is filling up", "replica lag is climbing", "primary is down — how to fail over", "connections exhausted", "a bad deploy needs PITR rollback", "how to restore a single table". Each runbook: symptoms → diagnosis steps → remediation → escalation path → verification. **Test runbooks in game-days** (deliberately break staging and run the book) so they're correct when it counts.

### 16.2 Change management & schema-change review **[A]**

In production, **uncoordinated changes cause outages**. Mature shops gate database changes through **change management**: schema migrations and config changes are **reviewed** (a DBA or senior engineer checks for blocking locks, missing `lock_timeout`, un-indexed foreign keys, batch sizes, rollback plans — §12.4), tested against production-like data, scheduled into maintenance windows when risky, and have an explicit **rollback**. The DBA is the **gatekeeper and advisor** for schema changes — not to slow developers down, but to catch the migration that would lock a 500M-row table during peak traffic. Embed this as a CI check (lint migrations, e.g. with `squawk` for Postgres) plus human review for risky ones.

### 16.3 The DBA–developer relationship **[A]**

The healthiest pattern is **partnership, not gatekeeping-by-default**. Developers own their data model and queries; the DBA brings operational expertise (what scales, what locks, what to index, how to migrate safely) and provides **paved roads** — vetted patterns, migration tooling, query-review, dashboards, and self-service guardrails — so developers move fast *safely*. Anti-patterns to avoid: the DBA as a bottleneck who must approve every query (doesn't scale), or no DBA input at all (outages waiting to happen). Teach developers to read `EXPLAIN`, to write expand/contract migrations, and to set timeouts; review the risky changes, automate the routine ones.

### 16.4 Capacity planning **[A]**

**Capacity planning** is using monitoring **trends** to provision *ahead* of need — so you scale up before the disk fills or the CPU saturates, not during the incident. Track growth of: data size (and thus backup time and disk), connections, query rate, CPU/memory/IOPS utilisation, and replication headroom. Project forward (linear or seasonal), and set thresholds that trigger a *planned* scale-up (add storage, bigger instance, another replica) with lead time. Watch for **non-linear cliffs** (a table crossing a size where a key query's plan flips to a seq scan; running out of connection slots; transaction-ID wraparound horizon). Capacity planning is how you avoid both outages (under-provisioned) and waste (over-provisioned).

### 16.5 Incident response & postmortems **[A]**

When the database breaks, a calm **incident response** process limits damage: **detect** (alert fires), **declare** (open an incident, assign a commander), **mitigate** (restore service first — fail over, kill a runaway query, roll back the deploy — *stabilise before you root-cause*), **communicate** (status updates to stakeholders), **resolve**, then **learn**. The learning step is the **blameless postmortem**: a written analysis of what happened, the timeline, the contributing causes (not "who to blame" — focus on the systemic gaps), and concrete **action items** to prevent recurrence (a missing alert, a runbook gap, a config change, a guardrail). The goal is an organisation that gets *more* reliable after each incident. **Document everything** — topology diagrams, runbooks, escalation paths, the "why" behind config choices — because the cost of undocumented tribal knowledge is paid during the next 3 a.m. incident.

### 16.6 The company-level operations checklist **[A]**

| Area | "Are we doing this?" |
|---|---|
| Backups | Automated, off-site, immutable, **restore-tested**, meets RPO/RTO |
| HA | Standby + tested automatic failover; app reconnects cleanly |
| Monitoring | Golden signals dashboarded; alerts routed to on-call; no alert fatigue |
| Security | Not internet-exposed, TLS, least privilege, encrypted at rest, audited, patched |
| Change mgmt | Migrations reviewed, tested, with rollback; schema in code |
| On-call | Runbooks written & game-day-tested; clear escalation |
| Capacity | Growth tracked; scale-up triggered with lead time |
| DR | Cross-region copy; documented, **drilled** disaster-recovery procedure |
| Docs | Topology, runbooks, postmortems, config rationale — all written down |

---

## 17. Gotchas & Best Practices

A consolidated list of the traps that bite real DBAs and the practices that prevent them.

**Backups & recovery**
- **An untested backup is not a backup.** Schedule automated restore tests; do a full DR drill regularly. The first time you restore should *never* be during a real disaster.
- **WAL/binlog/transaction-log can fill the disk.** Archiving that fails silently → a gap in PITR; SQL Server FULL model without log backups → log grows forever. Monitor and alert.
- **`pg_dump` doesn't capture roles/grants** — also run `pg_dumpall --globals-only`. `mysqldump` omits routines/triggers/events without flags. Test that your restore actually has everything.
- **3-2-1 + immutability.** Ransomware and a compromised admin will try to delete your backups; keep an offline/object-locked copy.

**Configuration & memory**
- **`work_mem` is per-operation, per-query** — a high value × high concurrency × multiple sorts per query = OOM. The #1 Postgres memory footgun.
- **`random_page_cost = 4` on SSD** makes the planner avoid index scans. Set ~1.1 on SSD/NVMe.
- **Don't set `max_connections` to thousands** — each is a process/thread. Use a pooler (PgBouncer/ProxySQL).
- **Know reload vs restart** — don't cause an outage changing a parameter that only needed a reload (or fail to apply one that needed a restart).

**Security & networking**
- **Never expose the database to the internet.** Firewall to the app subnet; TLS everywhere; no `trust`/default-password accounts.
- **The app role must not be a superuser.** Least privilege turns a SQL-injection catastrophe into a minor incident.
- **Encrypt backups**, not just the live data — backups are the most-stolen copy.

**PostgreSQL-specific**
- **Never disable autovacuum.** Transaction-ID **wraparound** can force the database to stop accepting writes. Monitor `age(datfrozenxid)`.
- **`idle in transaction` connections** hold locks, block vacuum (causing bloat + wraparound risk), and block other transactions. Hunt and kill long ones; set `idle_in_transaction_session_timeout`.
- **Stale replication slots** retain WAL forever (disk fills) and hold back vacuum. Drop slots for standbys that are gone.
- **`pg_upgrade --link` is irreversible** once you start the new cluster. Back up first; use the no-link or logical path if you need a fallback.

**MySQL-specific**
- **`binlog_format = ROW`** (not STATEMENT) for replication correctness; set a unique `server_id`; use GTIDs.
- **`mysqldump --single-transaction`** for InnoDB (avoid the global read lock); don't forget `--routines --triggers --events`.
- **MySQL 8.4 removed deprecated features** and changed defaults (auth plugin, etc.) — re-test config & drivers after upgrading.

**Schema migrations**
- **Big DDL takes locks.** `CREATE INDEX CONCURRENTLY` / online DDL / gh-ost / pt-online-schema-change. Always set a `lock_timeout`.
- **Expand/contract** migrations (backward-compatible) decouple deploy from migration and enable rollback.
- **A migration that's instant on dev can lock a huge prod table for an hour.** Test against production-scale data.

**Operations**
- **Disk-full crashes databases hard** — alert at ~80 %, not 100 %.
- **Stabilise before root-causing** during an incident — restore service, then investigate.
- **Blameless postmortems with action items** make you more reliable each time.
- **Document the topology, runbooks and config rationale** — tribal knowledge fails at 3 a.m.

---

## 18. Study Path & Build-to-Learn Projects

### Suggested learning order

1. **Foundations (§1-§2)** — Learn what a DBA does, RPO/RTO, OLTP vs OLAP, and the managed-vs-self-managed tradeoff. Install PostgreSQL and MySQL **from packages on a Linux VM** so you see the data directory, service, config files and logs. Connect with `psql` and the `mysql` CLI and get comfortable in them. (Pair with the **[Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md)** guide.)
2. **Config & access (§3-§5)** — Tune `shared_buffers`/`work_mem`/`innodb_buffer_pool_size`, stand up **PgBouncer**, build least-privilege roles, lock down `pg_hba.conf`/`bind-address`, and enable TLS. Practise the reload-vs-restart distinction.
3. **Backups & recovery (§6)** — The core skill. Do logical dumps and restores; set up **WAL archiving + `pg_basebackup`** (or **pgBackRest**) and perform a real **Point-In-Time Recovery** to "just before a bad DELETE". Restore at least once — prove your backup works.
4. **HA & replication (§7-§8)** — Build a streaming standby by hand, then automate failover with **Patroni** (+ etcd + HAProxy). Add a read replica and route reads to it. Read about sharding; partition a large table.
5. **Observability & tuning (§9-§10)** — Wire up **postgres_exporter + Prometheus + Grafana**; learn `pg_stat_statements` and the slow-query log; read `EXPLAIN (ANALYZE, BUFFERS)`; create and fix a deliberate autovacuum/bloat problem.
6. **Security, upgrades, scale (§11-§16)** — Harden against the CIS benchmark; do a major upgrade with `pg_upgrade` and again with logical replication; run a database under a **Kubernetes operator (CloudNativePG)** or provision one with **Terraform** on a managed service; write a runbook and run a game-day.
7. **Breadth (§13-§14)** — Stand up a **MongoDB replica set** and a **Redis Sentinel** setup; spin up **SQL Server Developer Edition** and practise the full/diff/log backup chain and an Always On AG.

### Build-to-learn projects

- **Project 1 — The recoverable database.** Install Postgres, load a sample dataset, configure pgBackRest with WAL archiving, run nightly fulls. Then *deliberately* run `DELETE FROM orders;`, and recover to the second before it via PITR. Script the whole restore as a runbook. *(Cements §6 — the most important skill.)*
- **Project 2 — Highly-available cluster.** Build a 3-node Patroni + etcd + HAProxy Postgres cluster. Kill the primary (`systemctl stop`, then pull its network) and watch automatic failover; verify your app reconnects. Add a read replica and split reads/writes. *(Cements §7-§8.)*
- **Project 3 — Observability & tuning.** Stand up postgres_exporter + Prometheus + Grafana + Alertmanager. Generate load (pgbench), create a missing-index and an autovacuum-bloat scenario, find them via `pg_stat_statements` and `pg_stat_user_tables`, fix them online (`CREATE INDEX CONCURRENTLY`), and confirm the dashboards/alerts react. *(Cements §9-§10.)*
- **Project 4 — Zero-downtime major upgrade.** Upgrade Postgres N→N+1 using logical replication (build the new cluster, replicate, validate, cut over), then do it again with `pg_upgrade --link`. Compare downtime. *(Cements §12.)*
- **Project 5 — Connection-pool stress test.** Point hundreds of app connections (or pgbench `-c 500`) at Postgres directly and watch it struggle near `max_connections`; insert PgBouncer in transaction mode and watch the same load served by 25 backends. *(Cements §3.4-§3.5.)*
- **Project 6 — Polyglot ops.** Stand up a MongoDB replica set (test failover by killing the primary), a Redis Sentinel + replica (test failover, compare RDB vs AOF on restart after a crash), and SQL Server Developer Edition (full/diff/log backup chain + a PITR restore). *(Cements §13-§14.)*
- **Project 7 — Infra as cattle.** Define a managed Postgres (Multi-AZ, encrypted, 14-day PITR) in Terraform, or run CloudNativePG on a local k8s (kind/minikube) with S3-compatible backups (MinIO). Destroy and recreate it from code. *(Cements §15.)*

Work the projects in order; each builds the muscle for the next, and together they take you from "I can run a query" to "I can run the database servers a company depends on."

---

> **Cross-references:** **[PostgreSQL](POSTGRESQL_GUIDE.md)** · **[MongoDB](MONGODB_GUIDE.md)** · **[SQLite3](SQLITE3_GUIDE.md)** · **[Redis](REDIS_GUIDE.md)** · **[Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)** · **[Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md)** · **[Windows Server Administration](WINDOWS_SERVER_ADMIN_GUIDE.md)** · **[Networking](NETWORKING_GUIDE.md)** · **[Docker](DOCKER_GUIDE.md)** · **[Nginx](NGINX_GUIDE.md)** · **[Prisma ORM](PRISMA_ORM_GUIDE.md)** · **[Go ent ORM](GO_ENT_ORM_GUIDE.md)** · **[GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md)**
