# Redis — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Backend and full-stack developers going from "I have never run a Redis command" to "I can design, secure, and operate Redis in production" — entirely offline. Every concept is explained in **prose first** (what it is, *why* it works that way, when and why to use it, the key parameters, best practices, and security notes), and only *then* shown with heavily-commented, runnable code in **`redis-cli`**, **Node.js** (`ioredis` and the official `node-redis`/`redis` v4+), and **Go** (`github.com/redis/go-redis/v9`). Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Redis 7.x and Redis 8.x**, current as of **2026**.
>
> - **Redis 8 (GA mid-2025)** is a *unified* release: the modules that used to ship separately as **Redis Stack** — **RediSearch** (search + vectors), **RedisJSON**, **Redis Time Series**, and **probabilistic types** (Bloom/Cuckoo/Count-Min/Top-K/t-digest) — are now **bundled into core Redis**. You no longer need a separate "Redis Stack" image for JSON, search, or vector similarity. Redis 8 also added a new high-performance keyspace engine and major latency improvements.
> - **Licensing history (important):** Through Redis 7.2 the server was **BSD-3** (true open source). In **March 2024** Redis Ltd. relicensed to a **dual RSALv2 / SSPLv1** model (source-available, *not* OSI open source). In **May 2025**, with Redis 8, they **added AGPLv3** as a third option, so Redis 8 is once again available under an OSI-approved copyleft license (alongside RSALv2/SSPL). The 2024 relicensing triggered the **Valkey** fork (a Linux Foundation project, BSD-3 licensed, drop-in compatible, backed by AWS, Google, Oracle, and others). **Valkey 7.2 / 8.x** is the fully-open alternative; almost everything in this guide applies to it verbatim (Valkey lacks the bundled JSON/Search/Vector modules but has its own module ecosystem). Pick Redis 8 (AGPL) or Valkey depending on your license posture.
>
> Fast-moving details (module bundling, licensing, cluster internals) are flagged with **⚡ Version note**. When in doubt, confirm against `redis.io/docs` and the `COMMAND DOCS` output of your server.
>
> **Cross-references:** Redis almost never lives alone. This guide assumes you may also be using the library's **`POSTGRESQL_GUIDE.md`** (the durable system of record Redis usually caches in front of), **`NODEJS_GUIDE.md`** (the Node runtime, event loop, and async model the `ioredis`/`node-redis` examples build on), and **`GO_GUIDE.md`** (goroutines, `context`, error handling, and pooling that the `go-redis` examples rely on). Where a topic is better covered there — relational modeling, the Node event loop, Go concurrency — this guide points you to it instead of duplicating.

---

## Table of Contents

1. [What Redis Is, When to Use It, Install & RESP](#1-what-redis-is-when-to-use-it-install--resp) **[B]**
2. [Data Types in Depth](#2-data-types-in-depth) **[B/I]**
3. [Key Management — TTL, SCAN, Naming](#3-key-management--ttl-scan-naming) **[B/I]**
4. [Pub/Sub](#4-pubsub) **[I]**
5. [Streams & Consumer Groups](#5-streams--consumer-groups) **[I/A]**
6. [Transactions & Optimistic Locking](#6-transactions--optimistic-locking) **[I]**
7. [Lua Scripting & Functions](#7-lua-scripting--functions) **[I/A]**
8. [Persistence — RDB vs AOF](#8-persistence--rdb-vs-aof) **[I]**
9. [Caching Patterns & Eviction](#9-caching-patterns--eviction) **[I]**
10. [Pipelining, Performance & Connection Pooling](#10-pipelining-performance--connection-pooling) **[I/A]**
11. [Distributed Locking & Rate Limiting](#11-distributed-locking--rate-limiting) **[A]**
12. [Queues — Lists vs Pub/Sub vs Streams](#12-queues--lists-vs-pubsub-vs-streams) **[I/A]**
13. [High Availability — Replication, Sentinel, Cluster](#13-high-availability--replication-sentinel-cluster) **[A]**
14. [Security — ACLs, AUTH, TLS](#14-security--acls-auth-tls) **[I/A]**
15. [Vector Search & Redis as a Vector DB](#15-vector-search--redis-as-a-vector-db) **[A]**
16. [Observability & Operations](#16-observability--operations) **[I/A]**
17. [Real-World Recipes](#17-real-world-recipes) **[I/A]**
18. [Gotchas & Best Practices](#18-gotchas--best-practices) **[I/A]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. What Redis Is, When to Use It, Install & RESP

### 1.1 What Redis is — the mental model that makes everything click **[B]**

**Redis** (REmote DIctionary Server) is an **in-memory data structure store**. To understand *why* it behaves the way it does, contrast it with a traditional database. A relational database like PostgreSQL keeps its data on **disk** and uses RAM only as a cache for the hot pages; every query may touch the disk, and the engine spends enormous effort on indexes, query planners, and concurrency control to make disk access tolerable. Redis flips this: it keeps the **entire dataset in RAM** and treats disk as a *backup* medium, not a working medium. That single design decision is the source of almost everything that follows — the speed, the simplicity, the limits, and the failure modes.

Because the data lives in RAM, a typical operation completes in **microseconds**, and a single modest instance serves **hundreds of thousands to millions of operations per second**. There is no disk seek on the read path, no page cache to warm, no query planner to outsmart.

The crucial second idea — and the one beginners most often miss — is that **Redis is not a key→string store, it is a key→data-structure store.** In a plain key/value cache (like memcached) a value is an opaque blob; you can only `GET` and `SET` the whole thing. In Redis a value can be a **string, a list, a hash, a set, a sorted set, a stream, a bitmap, a HyperLogLog, a geospatial index, a JSON document, or a search/vector index**, and Redis ships **server-side commands that operate on those structures atomically**. So instead of fetching a list to your application, appending one element, and writing the whole list back (three round trips, a race condition, and wasted bandwidth), you send a single `LPUSH` and the server mutates the structure in place. This is the deepest reason to learn Redis well: **picking the right data structure turns a hard concurrency problem into a one-line command.**

#### Why single-threaded — and why that is a *feature*

Redis executes commands on a **single thread**: one command runs to completion before the next one starts. New developers hear "single-threaded" and assume "slow," but the opposite is true here, and the reasoning is worth internalizing:

- **Atomicity is free.** Because only one command runs at a time, *every individual command is automatically atomic* — there is no possibility of two clients' `INCR`s interleaving and losing an update. You get correctness without locks. This is the foundation that makes counters, locks, and rate limiters trivial later.
- **No lock contention, no context-switching overhead.** A multi-threaded store spends cycles acquiring/releasing mutexes and ping-ponging cache lines between cores. Redis avoids all of it. For in-memory work, where each operation is already microseconds, lock overhead would dominate.
- **Predictable performance.** With one thread, latency is easy to reason about: a single slow O(N) command (say `KEYS *` on a huge keyspace) blocks *everything* until it finishes. This is the flip side and the source of the most common Redis production incident (§3, §10) — but it also means there are no mysterious concurrency Heisenbugs.

"Single-threaded" refers specifically to **command execution**. Modern Redis (6+) uses **extra threads for network I/O**, and always used background processes/threads for snapshotting (`fork`), lazy freeing (`UNLINK`), and a few specific commands. The model to keep in your head: *one logical timeline of commands, executed in arrival order, each atomic.*

### 1.2 When to use Redis — and when not to **[B]**

Choosing Redis well means knowing where its in-memory, structure-server nature is a superpower and where it is a liability.

**Great fits** — these all exploit microsecond latency, atomic structure operations, or TTLs:

| Use case | Why Redis is the right tool |
|---|---|
| **Caching** | Microsecond reads, built-in TTLs, automatic eviction — the #1 use case by far (§9) |
| **Session store** | Fast, shared across all app instances, auto-expiring; survives app restarts (§17) |
| **Rate limiting** | Atomic counters with expiry; sliding windows via sorted sets (§11) |
| **Leaderboards / ranking** | Sorted sets give O(log N) ranked inserts and queries (§2.4) |
| **Real-time feeds / pub-sub** | Pub/Sub for fan-out, Streams for durable event delivery (§4, §5) |
| **Job / task queues** | Lists (`BLPOP`) for simple, Streams (consumer groups) for durable (§12) |
| **Distributed locks** | `SET NX PX` + a fencing token for mutual exclusion across processes (§11) |
| **Counting & analytics** | `INCR`, HyperLogLog (cardinality), bitmaps (presence) at huge scale (§2.6–2.7) |
| **Geospatial** | Radius/box "near me" queries with `GEOSEARCH` (§2.8) |
| **Vector search / RAG** | Vector similarity for AI apps, co-located with your cache (§15) |

**Poor fits / proceed carefully** — these fight the in-memory, single-node, no-rollback nature:

- **Primary store for data much larger than RAM.** Everything lives in memory; if your dataset is hundreds of GB and mostly cold, RAM is far more expensive than disk and a disk-first database wins. (Redis-on-flash exists in some commercial editions, but the open server is RAM-bound.)
- **Complex relational queries, joins, ad-hoc analytics.** Redis has no general query language and no joins. For anything relational, use PostgreSQL (see **`POSTGRESQL_GUIDE.md`**) and let Redis *cache* the results.
- **Strong multi-key ACID across a cluster.** Redis transactions are limited (no rollback on logic errors — §6), and in Cluster a transaction must touch keys in a single hash slot.
- **Durability-critical ledgers (money).** Redis *can* be durable (`appendfsync always`), but at a throughput cost, and a single instance is still a single point of failure. Most shops use Redis as a fast layer in front of a durable system of record, not as the ledger itself.

> **The recurring architecture:** Redis is rarely your *only* datastore. The canonical pattern is **PostgreSQL (or another durable DB) as the system of record, Redis as the speed layer** — caching reads, holding sessions, brokering jobs, and counting things. Design with that division of responsibility in mind.

### 1.3 Install **[B]**

You will almost always develop against Docker (identical everywhere, disposable) and deploy with a managed service or a tuned native install. Here are both.

#### Docker (recommended for development)

```bash
# Redis 8 — JSON, Search, Vectors, Time Series, and Bloom are all bundled in core.
docker run -d --name redis -p 6379:6379 redis:8

# Persist data to a NAMED VOLUME so it survives container removal,
# and enable AOF so writes are durable across restarts (see §8):
docker run -d --name redis -p 6379:6379 \
  -v redis-data:/data \
  redis:8 \
  redis-server --appendonly yes

# Open an interactive redis-cli INSIDE the running container:
docker exec -it redis redis-cli
# 127.0.0.1:6379> PING
# PONG
```

⚡ **Version note:** Before Redis 8 you needed the `redis/redis-stack` image to get JSON/Search/Vectors and a separate `redis/redis-stack-server`. On Redis 8 the plain `redis:8` image includes them. For the fully-open fork: `docker run -d -p 6379:6379 valkey/valkey:8`.

#### Native install

```bash
# macOS (Homebrew):
brew install redis
brew services start redis          # run as a background launchd service

# Debian / Ubuntu:
sudo apt-get install -y redis      # distro package (may lag the latest release)
# or build the newest from source:
#   wget https://download.redis.io/redis-stable.tar.gz
#   tar xzf redis-stable.tar.gz && cd redis-stable && make && sudo make install

# Start a server manually with a config file (the config is where you tune everything):
redis-server /etc/redis/redis.conf

# Confirm it is alive:
redis-cli PING        # → PONG
```

> **Best practice:** in production, do not run the default zero-config server. Start from a copy of the shipped `redis.conf`, set `maxmemory` + an eviction policy (§9), choose a persistence strategy (§8), and lock down security (§14) *before* anything connects.

### 1.4 `redis-cli` — the tool you will live in while learning **[B]**

`redis-cli` is the interactive client. Learning its flags pays off immediately because the same flags double as production diagnostics.

```bash
# Connect (defaults: 127.0.0.1:6379, no auth, logical DB 0):
redis-cli

# Connect to a remote / secured server:
redis-cli -h myhost -p 6380 -a 'password' --tls
# (Passing -a on the command line leaks the password to `ps`/shell history;
#  prefer the REDISCLI_AUTH env var or an interactive AUTH — see §14.)

# Use the newer RESP3 protocol (richer reply types — see §1.6):
redis-cli -3

# Run a single command and exit (great for scripts and one-offs):
redis-cli SET greeting "hello"
redis-cli GET greeting        # → "hello"

# Select a logical database (0–15 by default). IMPORTANT: numbered DBs do NOT exist
# in Cluster mode — never design around them; use key prefixes instead (§3).
redis-cli -n 1

# Diagnostic flags you will reuse constantly:
redis-cli --scan --pattern 'user:*'   # safely iterate keys (uses SCAN, never KEYS)
redis-cli --stat                      # live throughput / memory stats, refreshed each second
redis-cli --latency                   # measure round-trip latency to the server
redis-cli --bigkeys                   # sample the keyspace for the largest key of each type
redis-cli --memkeys                   # sample keys by memory usage
redis-cli --hotkeys                   # most-accessed keys (requires an LFU eviction policy)
```

Inside the interactive prompt, these are the commands you reach for first:

```redis
127.0.0.1:6379> SET name "Ada"
OK
127.0.0.1:6379> GET name
"Ada"
127.0.0.1:6379> TYPE name          # what data structure is stored here?
string
127.0.0.1:6379> DEL name
(integer) 1
127.0.0.1:6379> HELP SET           # built-in, offline command help
127.0.0.1:6379> COMMAND DOCS GET   # full machine-readable docs for a command
```

### 1.5 Logical databases (the numbered DBs) **[B]**

A single Redis instance exposes **16 numbered logical databases** (0–15 by default) selected with `SELECT n` or `redis-cli -n n`. They are *not* separate processes or separate memory pools — they are just namespaces sharing the same instance, the same `maxmemory`, and the same single thread. Historically people used them to separate "cache" from "sessions," but this is now considered an anti-pattern: **prefer key prefixes** (`cache:...`, `sess:...`) instead, because (a) numbered DBs **do not exist in Cluster mode**, so designing around them blocks you from scaling horizontally later, and (b) prefixes are self-documenting and greppable. Treat numbered DBs as a convenience for throwaway experiments in `redis-cli`, not an architecture.

### 1.6 The RESP protocol — what is actually on the wire **[I]**

Redis speaks **RESP** (REdis Serialization Protocol) — a simple, line-based, mostly human-readable text protocol over TCP. You will never write it by hand in real code (clients do it for you), but understanding it *demystifies* Redis and explains two important features (pipelining and binary-safety) that otherwise look like magic.

- **RESP2** (the classic protocol): every reply is typed by its **first byte**:
  - `+` simple string → `+OK\r\n`
  - `-` error → `-ERR unknown command\r\n`
  - `:` integer → `:1000\r\n`
  - `$` bulk string (binary-safe, length-prefixed) → `$5\r\nhello\r\n`
  - `*` array → `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`
- **RESP3** (Redis 6+, negotiated with `HELLO 3`): adds real types — **maps** (`%`), **sets** (`~`), **doubles** (`,`), **booleans** (`#`), **big numbers**, **verbatim strings**, and crucially **push** messages (used for Pub/Sub and client-side caching). RESP3 lets a connection receive out-of-band pushes while still issuing normal commands, which is why client-side caching and Pub/Sub-on-the-same-connection became practical.

A **request is always an array of bulk strings** — that uniformity is what makes the protocol trivial to implement and pipeline. You can literally type Redis over `netcat`:

```bash
# Send "SET foo bar" by hand over a raw TCP connection:
printf '*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n' | nc localhost 6379
# Server replies: +OK
#   *3   = array of 3 elements (the command + 2 args)
#   $3   = the next bulk string is 3 bytes long
#   SET  = those 3 bytes
```

Two consequences worth remembering:

- **Pipelining is "just" sending many request arrays back-to-back** without waiting for replies in between — the protocol imposes no per-command handshake (§10).
- **Bulk strings are binary-safe** (length-prefixed, not delimiter-terminated), so a Redis value can hold a JPEG, a protobuf, a gzip blob — anything, up to 512 MB.

### 1.7 Connecting from code — the canonical clients **[B]**

Three clients are used throughout this guide. Set them up once; later sections assume they exist.

**Node.js — `ioredis`** (feature-rich, hugely popular, excellent Cluster/Sentinel support, auto-pipelining). See **`NODEJS_GUIDE.md`** for the async/`await` and event-loop model these examples rely on.

```js
// npm install ioredis
import Redis from "ioredis";

// Accepts a connection string OR an options object. ioredis auto-reconnects on drop.
const redis = new Redis("redis://localhost:6379");
// const redis = new Redis({ host: "localhost", port: 6379, password: "secret" });

await redis.set("greeting", "hello");
console.log(await redis.get("greeting")); // → "hello"

await redis.quit(); // graceful close (flush pending + send QUIT). Use disconnect() to drop now.
```

**Node.js — `node-redis` (the official `redis` package, v4+)** — promise-based, and you must explicitly `connect()`. It has first-class helpers for JSON and Search (handy in §2.10 and §15).

```js
// npm install redis
import { createClient } from "redis";

const client = createClient({ url: "redis://localhost:6379" });
client.on("error", (err) => console.error("Redis error:", err)); // ALWAYS attach this
await client.connect(); // v4+ requires an explicit connect

await client.set("greeting", "hello");
console.log(await client.get("greeting")); // → "hello"

await client.quit();
```

**Go — `github.com/redis/go-redis/v9`** (the official, maintained client). It is **already a connection pool** and is safe for concurrent goroutines — see **`GO_GUIDE.md`** for the `context`, goroutine, and error-handling idioms below.

```go
// go get github.com/redis/go-redis/v9
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()

	// NewClient returns a client backed by a connection POOL.
	// Share ONE *redis.Client app-wide; it is safe for concurrent use by many goroutines.
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // default logical DB
	})
	defer rdb.Close()

	// Every command takes a context (for timeouts/cancellation) and returns a typed *Cmd.
	if err := rdb.Set(ctx, "greeting", "hello", 0).Err(); err != nil {
		panic(err)
	}
	val, err := rdb.Get(ctx, "greeting").Result()
	if err != nil {
		panic(err) // NB: a MISSING key returns redis.Nil, which is an error here — handle it (§9).
	}
	fmt.Println(val) // → hello
}
```

> Throughout the guide, assume `redis` (ioredis), `client` (node-redis), and `rdb` + `ctx` (go-redis) are already set up exactly as above.

---

## 2. Data Types in Depth

Redis values are **typed**, and the single most valuable skill in Redis is picking the right type for the job — it is the difference between a one-line atomic command and a buggy read-modify-write loop. For each type below: *what it is*, *why/when to use it*, the full command reference, and code in all three clients.

### 2.1 Strings **[B]**

**What it is.** The simplest type: a single value associated with a key. A string is **binary-safe** (it stores raw bytes, so text, numbers, JSON, protobuf, or a JPEG are all fine) and can be up to **512 MB**. "String" is a slight misnomer — think "a single value."

**Why/when.** Use strings for caching a serialized object, atomic counters, feature flags, single key/value pairs, and — viewed as bits — bitmaps (§2.6). The killer feature beyond plain get/set is the family of **atomic numeric operations** (`INCR`, `INCRBY`, `INCRBYFLOAT`): because Redis is single-threaded, an `INCR` is a complete read-modify-write with **no race**, which is why counters and rate limiters start here.

**Key parameters.** `SET` is the workhorse and its options compose:
- `EX seconds` / `PX ms` / `EXAT ts` / `PXAT ts-ms` — set a TTL atomically with the value (always prefer this over a separate `EXPIRE`).
- `NX` (set only if **N**ot e**X**ists) / `XX` (only if it already e**X**ists) — the basis of locks and "create-once" semantics.
- `GET` — return the *old* value while setting the new one (atomic get-and-set, Redis 6.2+).
- `KEEPTTL` — overwrite the value but keep the existing TTL (a plain `SET` otherwise **clears** the TTL — a classic bug, see §3).

**Best practices.** Always TTL cache strings. Prefer `SET ... EX` to `SETEX`/`SETNX` (the dedicated legacy commands still work but `SET` with options supersedes them). Don't store one giant 100 MB string you only ever need part of — a hash or many keys is usually better (§10).

**Security.** Treat any string you `JSON.parse`/`json.Unmarshal` as untrusted input; a poisoned cache entry becomes app input. Never put secrets in keys (key names appear in `MONITOR`, `SLOWLOG`, and `--bigkeys` output).

```redis
SET user:1:name "Ada Lovelace"
GET user:1:name                 # "Ada Lovelace"

# SET options — combine them; this single command replaces several legacy ones:
SET session:abc "{...}" EX 3600        # expire in 3600s
SET lock:job "owner1" NX PX 5000       # set only if absent, expire in 5000ms (a lock! §11)
SET config:flag "on" XX                # set only if the key already exists
SET counter 10 GET                      # set new value, RETURN the old value (Redis 6.2+)
SET k v KEEPTTL                         # overwrite value but preserve existing TTL

# Atomic counters (no read-modify-write race — the single thread guarantees it):
SET views 0
INCR views                  # 1
INCRBY views 10             # 11
DECR views                  # 10
INCRBYFLOAT price 3.50      # floats too (note: floats can't be DECR'd; add a negative)

# Multiple keys in one round trip (saves network latency):
MSET a 1 b 2 c 3
MGET a b c                  # 1) "1" 2) "2" 3) "3"

# Substring & length operations:
APPEND log "line1\n"        # append; returns the new total length
STRLEN user:1:name          # byte length
GETRANGE user:1:name 0 2    # "Ada" (inclusive byte range)
SETRANGE user:1:name 4 "B." # overwrite starting at byte offset 4
```

| Command | Purpose | Big-O |
|---|---|---|
| `SET k v [EX/PX/EXAT/PXAT] [NX/XX] [GET] [KEEPTTL]` | Set value (+ options) | O(1) |
| `GET k` / `MGET k...` / `MSET k v...` | Read / multi-read / multi-write | O(1) / O(n) |
| `GETSET` / `SET ... GET` | Set and return old value | O(1) |
| `INCR` / `DECR` / `INCRBY` / `INCRBYFLOAT` | Atomic numeric ops | O(1) |
| `APPEND` / `STRLEN` | Append / length | O(1) amortized / O(1) |
| `GETRANGE` / `SETRANGE` | Substring read/write | O(n) |
| `SETEX k ttl v` / `PSETEX` / `SETNX` | Legacy combined forms (prefer `SET` options) | O(1) |

```js
// ioredis — cache a JSON object with a TTL (this is the cache-aside pattern, §9):
await redis.set("user:1", JSON.stringify({ id: 1, name: "Ada" }), "EX", 3600);
const raw = await redis.get("user:1");
const user = raw ? JSON.parse(raw) : null; // a missing key returns null in ioredis

// Atomic counter — safe even under massive concurrency:
const views = await redis.incr("page:home:views");
```

```go
// go-redis — same idea, with Go's explicit error handling:
import (
	"encoding/json"
	"time"
)

b, _ := json.Marshal(map[string]any{"id": 1, "name": "Ada"})
rdb.Set(ctx, "user:1", b, time.Hour) // the 4th arg is the TTL; 0 means "no expiry"

views, _ := rdb.Incr(ctx, "page:home:views").Result()
_ = views
```

### 2.2 Lists **[B]**

**What it is.** An ordered collection of strings, implemented internally as a **quicklist** (a linked list of compact "listpack" nodes). The performance profile follows directly from that: **O(1) push/pop at either end**, but **O(N) random access** by index (you have to walk the list). Think of it as a deque (double-ended queue).

**Why/when.** Lists shine wherever you push at one end and pop at the other: **job queues** (`LPUSH` to enqueue, `BRPOP` to dequeue), **activity timelines / recent-items** (push new items, `LTRIM` to cap the length), and simple inter-process messaging. The **blocking pops** (`BLPOP`/`BRPOP`) are what make a list a real queue — a consumer parks efficiently waiting for work instead of busy-polling.

**Key parameters.** `BLPOP key [key...] timeout` blocks up to `timeout` seconds (0 = forever) and returns the first available element. `LMOVE src dst LEFT|RIGHT LEFT|RIGHT` atomically pops from one list and pushes to another — the foundation of a *reliable* queue (you move the in-flight job to a "processing" list so a crash doesn't lose it).

**Best practices.** Cap timelines with `LTRIM` so they don't grow unbounded (a list is a silent memory leak otherwise). Use `LMOVE`/`BLMOVE` (not the deprecated `RPOPLPUSH`) for at-least-once handoff. Avoid `LRANGE 0 -1` on a million-element list — it is O(N) and blocks the single thread (§10).

**Security.** A queue consumer executes whatever the producer enqueued — validate/deserialize defensively, and put a hard cap on queue length to resist a flood that exhausts memory.

```redis
RPUSH mylist a b c          # push to the tail → [a, b, c]
LPUSH mylist z              # push to the head → [z, a, b, c]
LRANGE mylist 0 -1          # read all (index 0 to -1 = whole list)
LPOP mylist                 # remove & return the head ("z")
RPOP mylist 2               # pop 2 from the tail (count arg is Redis 6.2+)
LLEN mylist                 # current length
LINDEX mylist 0             # element at an index (O(N))
LSET mylist 0 "new"         # set by index
LINSERT mylist BEFORE b "x" # insert relative to the first matching value
LREM mylist 1 a             # remove 1 occurrence of "a" (from head)
LTRIM mylist 0 99           # keep only indices 0..99 — cheaply cap a list

# Blocking pops — park the consumer until an element arrives (the heart of a queue):
BLPOP myqueue 5             # block ≤5s for an element; 0 = block forever
BRPOP myqueue 0

# Atomically move between lists — reliable queue handoff (crash-safe):
LMOVE src dst LEFT RIGHT    # pop src's head → push to dst's tail
BLMOVE src dst LEFT RIGHT 0 # blocking version (replaces the deprecated RPOPLPUSH)
```

| Command | Purpose |
|---|---|
| `LPUSH` / `RPUSH` / `LPUSHX` / `RPUSHX` | Push head/tail (`X` = only if the list already exists) |
| `LPOP` / `RPOP [count]` | Pop head/tail |
| `BLPOP` / `BRPOP` | **Blocking** pop — the queue consumer's tool |
| `LMOVE` / `BLMOVE` | Atomic move between lists (reliable queues) |
| `LRANGE` / `LINDEX` / `LLEN` | Read range / by index / length |
| `LSET` / `LINSERT` / `LREM` / `LTRIM` | Modify / insert / remove / cap |

```js
// ioredis — a minimal reliable worker loop using a blocking pop:
async function worker() {
  while (true) {
    // BRPOP returns [key, value] when an element arrives, or null on timeout.
    const res = await redis.brpop("jobs", 5);
    if (!res) continue;          // timed out → loop again (lets us check shutdown flags)
    const [, job] = res;         // res = ["jobs", "<payload>"]
    await handle(JSON.parse(job));
  }
}
```

```go
// go-redis — producer + blocking consumer:
rdb.LPush(ctx, "jobs", `{"id":1}`)

res, err := rdb.BRPop(ctx, 5*time.Second, "jobs").Result()
// res = ["jobs", "{\"id\":1}"]; err == redis.Nil on timeout (NOT a real failure).
if err == redis.Nil {
	// no job within the timeout — loop and try again
}
```

### 2.3 Sets **[B]**

**What it is.** An unordered collection of **unique** strings, backed by a hash table, giving **O(1) add, remove, and membership test**. On top of that, Redis offers fast **set algebra** — union, intersection, difference — computed *server-side*.

**Why/when.** Reach for a set whenever uniqueness or membership is the question: **tags**, **unique visitors**, "**who liked this**," relationships (followers/following), deduplication, and allow/deny lists. The set-algebra commands are the superpower: "users who are both online AND premium" is a single `SINTER`, not a join you write in your app.

**Key parameters.** `SINTERCARD numkeys key... LIMIT n` (6.2+) returns the *size* of an intersection capped at `n` — perfect when you only need "do at least N match?" without materializing the whole result. The `...STORE` variants write the result into a destination key so you can chain operations.

**Best practices.** `SMEMBERS` on a huge set is O(N) and blocks the thread — use `SSCAN` to iterate large sets incrementally (§3). For pure *counting* of uniques at massive scale where approximate is fine, a HyperLogLog (§2.7) costs 12 KB versus a set's hundreds of MB.

**Security.** Same as lists — set members come from somewhere; if they're user-supplied (e.g. tags), bound their count and length.

```redis
SADD tags:post:1 redis db nosql
SADD tags:post:1 redis          # duplicate ignored → returns 0 (count actually added)
SMEMBERS tags:post:1            # all members (O(N) — avoid on huge sets; use SSCAN)
SISMEMBER tags:post:1 redis     # 1 if member, else 0
SMISMEMBER tags:post:1 redis go # multi-membership in one call (Redis 6.2+)
SCARD tags:post:1               # cardinality (count)
SREM tags:post:1 nosql          # remove member(s)
SRANDMEMBER tags:post:1 2       # 2 random members WITHOUT removing them
SPOP tags:post:1                # remove AND return a random member

# Set algebra — push the computation to the server, not your app:
SINTER online:gamers online:premium     # users in BOTH sets
SUNION followers:a followers:b           # combined audience
SDIFF all banned                         # everyone except banned users
SINTERCARD 2 a b LIMIT 100               # size of the intersection, capped (6.2+)
SINTERSTORE result a b                    # store the intersection into a key

# Iterate a large set safely (cursor-based, never blocks):
SSCAN bigset 0 MATCH "user:*" COUNT 100
```

| Command | Purpose |
|---|---|
| `SADD` / `SREM` / `SCARD` | Add / remove / count |
| `SISMEMBER` / `SMISMEMBER` | Membership (single / multi) |
| `SMEMBERS` / `SSCAN` | All members / safe cursor iteration |
| `SRANDMEMBER` / `SPOP` | Random read / random remove |
| `SINTER` / `SUNION` / `SDIFF` (+ `STORE`, `CARD`) | Set algebra |

```js
// ioredis — track unique daily visitors, then count and intersect them:
await redis.sadd("visitors:2026-06-21", "user:42", "user:99");
const uniqueToday = await redis.scard("visitors:2026-06-21");
// Mutual followers = intersection of followers and following:
const mutuals = await redis.sinter("followers:alice", "following:alice");
```

```go
// go-redis — set membership and algebra:
rdb.SAdd(ctx, "visitors:2026-06-21", "user:42", "user:99")
n, _ := rdb.SCard(ctx, "visitors:2026-06-21").Result()
mutuals, _ := rdb.SInter(ctx, "followers:alice", "following:alice").Result() // []string
_ = n
_ = mutuals
```

### 2.4 Sorted Sets (ZSET) **[I]**

**What it is.** The crown jewel of Redis. A sorted set is like a Set, but **every member carries an associated `double` score**, and members are kept **permanently ordered by score** (ties broken lexicographically by member). Internally it is a skip list plus a hash table, giving **O(log N) inserts and ranked lookups** and O(1) score reads. This one structure powers leaderboards, priority queues, time-windowed data, rate limiters, and secondary indexes.

**Why/when.** Any time you need "things ordered by a number you control": **leaderboards** (score = points), **top-N / trending**, **priority queues** (score = priority, pop the min/max), **delayed jobs** (score = run-at timestamp; poll for due items), **sliding-window rate limiting** (score = request timestamp, §11), and **time-series windows** (score = event time; trim old). Because the score is a double, you can encode time, priority, money (carefully — floats!), or composite values.

**Key parameters.**
- `ZADD` flags: `NX` (only add new), `XX` (only update existing), `GT`/`LT` (only update if greater/less — *essential* for leaderboards so a worse run never lowers a high score), `INCR` (increment the score and return the new value, like `ZINCRBY`).
- `ZRANGE` is the **unified range query** (6.2+): `[REV]` for descending, `BYSCORE`/`BYLEX` to range by score or lexicographically, `LIMIT offset count` for pagination, `WITHSCORES` to get scores back. It supersedes the older `ZREVRANGE`/`ZRANGEBYSCORE`/`ZRANGEBYLEX` (which you'll still see widely).
- Range bounds: `-inf`/`+inf`, exclusive bounds with a leading `(` (e.g. `(100`), lexicographic bounds with `[`/`(` (e.g. `[a`).

**Best practices.** Use `GT` when submitting scores so retries/old runs can't regress a leaderboard. Trim time-window sets with `ZREMRANGEBYSCORE ... -inf <cutoff>`. For "rank around me" pages, get the rank then `ZRANGE` a window around it (§17). Beware float precision if scores must be exact integers beyond 2^53 — encode as fixed-point or use a different scheme.

```redis
ZADD leaderboard 100 alice 250 bob 175 carol
ZADD leaderboard GT 300 alice    # update ONLY if 300 > current (GT/LT, Redis 6.2+)
ZADD leaderboard NX 50 dave      # add only if not already present
ZADD leaderboard INCR 10 bob     # increment bob by 10, return his NEW score

ZSCORE leaderboard bob           # bob's score
ZINCRBY leaderboard 5 carol      # add 5 to carol's score
ZCARD leaderboard                # member count
ZRANK leaderboard alice          # 0-based rank, ascending (lowest score = rank 0)
ZREVRANK leaderboard alice       # rank descending (0 = the top scorer)

# Ranked queries — the leaderboard bread and butter:
ZRANGE leaderboard 0 9 REV WITHSCORES        # top 10 (unified syntax, Redis 6.2+)
ZRANGE leaderboard 0 -1 WITHSCORES           # everyone, ascending
ZREVRANGE leaderboard 0 2 WITHSCORES         # top 3 (legacy form, still common)

# Query by SCORE range:
ZRANGEBYSCORE leaderboard 100 200            # scores in [100, 200]
ZRANGEBYSCORE leaderboard "(100" 200         # exclusive lower bound → (100, 200]
ZRANGEBYSCORE leaderboard -inf +inf LIMIT 0 5
ZCOUNT leaderboard 100 200                    # count members in a score range

# Query by LEXICOGRAPHIC range (only meaningful when ALL scores are equal — a sorted index):
ZRANGEBYLEX names "[a" "[c"                    # members from "a" to "c" inclusive

ZREM leaderboard dave                          # remove a member
ZREMRANGEBYRANK leaderboard 0 -11              # keep only the top 10 (trim the rest)
ZREMRANGEBYSCORE leaderboard -inf 1700000000   # drop entries older than a timestamp
ZPOPMAX leaderboard                            # remove & return the highest (priority q)
ZPOPMIN leaderboard 3                          # remove & return the 3 lowest
BZPOPMIN q 0                                    # BLOCKING pop of the min — a priority queue!

# Combine sorted sets (with optional weighting):
ZUNIONSTORE dest 2 z1 z2 WEIGHTS 1 2           # union; z2 scores doubled
ZINTERSTORE dest 2 z1 z2
ZDIFF 2 z1 z2 WITHSCORES                        # (6.2+)
```

| Command | Purpose | Big-O |
|---|---|---|
| `ZADD [NX/XX/GT/LT/INCR]` | Add/update with score | O(log N) |
| `ZSCORE` / `ZMSCORE` / `ZINCRBY` | Get score / multi / increment | O(1) / O(log N) |
| `ZRANK` / `ZREVRANK` | Rank ascending/descending | O(log N) |
| `ZRANGE ... [REV] [BYSCORE/BYLEX] [LIMIT] [WITHSCORES]` | The unified range query | O(log N + M) |
| `ZRANGEBYSCORE` / `ZRANGEBYLEX` | Range by score / lex (legacy but common) | O(log N + M) |
| `ZCOUNT` / `ZCARD` / `ZLEXCOUNT` | Counting | O(log N) / O(1) |
| `ZPOPMIN` / `ZPOPMAX` / `BZPOPMIN` / `BZPOPMAX` | Pop extremes (priority queue) | O(log N) |
| `ZREMRANGEBYRANK/SCORE/LEX` | Bulk trim by rank/score/lex | O(log N + M) |
| `ZUNIONSTORE` / `ZINTERSTORE` / `ZDIFFSTORE` | Combine sets | varies |

```js
// ioredis — leaderboard helpers:
await redis.zadd("leaderboard", 100, "alice", 250, "bob");
await redis.zincrby("leaderboard", 50, "alice");        // alice now 150

// Top 10 with scores, highest first (returns a FLAT array; pair them up):
const top = await redis.zrange("leaderboard", 0, 9, "REV", "WITHSCORES");
// → ["bob","250","alice","150", ...]

// A player's rank, converted to 1-based for display:
const rank = await redis.zrevrank("leaderboard", "alice");
const displayRank = rank === null ? null : rank + 1;
```

```go
// go-redis — leaderboard; ZAdd takes redis.Z structs, ...WithScores returns []redis.Z:
rdb.ZAdd(ctx, "leaderboard",
	redis.Z{Score: 100, Member: "alice"},
	redis.Z{Score: 250, Member: "bob"},
)
// Top 10 descending, WITH scores → []redis.Z (typed, no flat-array unpacking):
top, _ := rdb.ZRevRangeWithScores(ctx, "leaderboard", 0, 9).Result()
for i, z := range top {
	fmt.Printf("#%d %v = %.0f\n", i+1, z.Member, z.Score)
}
```

### 2.5 Hashes **[B]**

**What it is.** A hash maps **field → value** under a single key — essentially a small object/record stored at one key. Below a configurable size threshold Redis uses a compact "listpack" encoding, making small hashes extremely memory-efficient (far cheaper than one key per field).

**Why/when.** Use a hash to store an **object** (user profile, product, cart) when you want to read or update **individual fields without re-serializing the whole thing**. With a JSON string you must `GET`, parse, mutate, re-serialize, and `SET` (and risk clobbering a concurrent writer); with a hash you `HSET user:1 age 37` and touch only that field, atomically. Hashes also give you **per-field counters** (`HINCRBY`).

**Key parameters.** `HSET key field value [field value ...]` sets one or many fields. `HINCRBY`/`HINCRBYFLOAT` do atomic per-field arithmetic. **Per-field TTL** (`HEXPIRE`, `HTTL`, `HPERSIST` — Redis 7.4+) lets individual fields expire independently, which finally makes a hash a viable session/cache store.

**Best practices.** Hashes are ideal for many small objects; they keep your keyspace tidy (one key per entity instead of `user:1:name`, `user:1:age`, ...). Avoid `HGETALL` on a hash with thousands of fields (O(N) block); use `HSCAN` or fetch specific fields with `HMGET`. Keep the listpack encoding by keeping fields small/few (tune `hash-max-listpack-entries`/`-value`).

**Security.** When you mirror a DB row into a hash, don't blindly copy sensitive columns (password hashes, tokens) into the cache; cache only what the read path needs.

```redis
HSET user:1 name "Ada" age 36 city "London"
HGET user:1 name                 # "Ada"
HMGET user:1 name age            # fetch several fields at once
HGETALL user:1                   # ALL fields+values (O(N) — careful on huge hashes)
HKEYS user:1                     # just the field names
HVALS user:1                     # just the values
HLEN user:1                      # field count
HEXISTS user:1 email             # 0 / 1
HDEL user:1 city                 # delete field(s)
HINCRBY user:1 age 1             # atomic per-field integer counter
HINCRBYFLOAT cart:1 total 9.99   # atomic per-field float counter
HSETNX user:1 verified true      # set a field only if it is absent
HRANDFIELD user:1 2 WITHVALUES   # random fields (6.2+)
HSCAN user:1 0 COUNT 50          # safe incremental iteration of a big hash

# Per-FIELD TTL — Redis 7.4+ (a major feature: expire individual fields, not just keys):
HEXPIRE user:1 60 FIELDS 1 sessiontoken    # this ONE field expires in 60s
HTTL user:1 FIELDS 1 sessiontoken          # remaining TTL on that field
HPERSIST user:1 FIELDS 1 sessiontoken      # remove the field's TTL
```

⚡ **Version note:** **Per-field TTL** (`HEXPIRE`, `HPEXPIRE`, `HTTL`, `HPERSIST`, `HEXPIREAT`, etc.) arrived in **Redis 7.4**. Before that, TTL applied only to whole keys, so you couldn't expire one field of a hash. This makes hashes usable as a true session/cache store where individual fields expire on their own schedule.

| Command | Purpose |
|---|---|
| `HSET` / `HSETNX` / `HMSET`(deprecated) | Set field(s) |
| `HGET` / `HMGET` / `HGETALL` | Read field(s) / all |
| `HDEL` / `HEXISTS` / `HLEN` | Delete / check / count |
| `HINCRBY` / `HINCRBYFLOAT` | Per-field atomic counters |
| `HKEYS` / `HVALS` / `HSCAN` | Fields / values / iterate |
| `HEXPIRE` / `HTTL` / `HPERSIST` (7.4+) | Per-field expiry |

```js
// ioredis — store and partially update a profile:
await redis.hset("user:1", { name: "Ada", age: 36, city: "London" });
const name = await redis.hget("user:1", "name");
await redis.hincrby("user:1", "age", 1);            // birthday — touch ONLY age, atomically
const profile = await redis.hgetall("user:1");      // → { name, age, city } (all strings)
```

```go
// go-redis — HSet accepts a map or alternating pairs; HGetAll returns map[string]string:
rdb.HSet(ctx, "user:1", map[string]any{"name": "Ada", "age": 36})
profile, _ := rdb.HGetAll(ctx, "user:1").Result()
rdb.HIncrBy(ctx, "user:1", "age", 1)
_ = profile
```

### 2.6 Bitmaps **[I]**

**What it is.** Not a distinct type — a **string treated as an array of bits** (up to 2^32 bits ≈ 512 MB). You address individual bits by offset.

**Why/when.** Bitmaps are astonishingly memory-efficient for **boolean state across a large integer ID space**. "Did user N do X today?" for **millions** of users fits in a few megabytes (1 bit per user). Classic uses: **daily active users (DAU)**, feature-flag-per-user, A/B-test bucketing, presence, and real-time analytics where each user maps to a stable integer ID.

**Key parameters.** `SETBIT key offset 0|1`, `GETBIT key offset`. `BITCOUNT` is the population count (how many bits are set), optionally over a byte/bit range. `BITOP AND|OR|XOR|NOT dest src...` combines bitmaps server-side — e.g. AND of Monday and Tuesday's DAU bitmaps = users active *both* days. `BITFIELD` packs multiple small integer counters into one string and operates on them atomically.

**Best practices.** Bitmaps require a dense, stable integer ID space (user 5,000,000 sets bit 5,000,000); sparse or string IDs waste space — use a set or HLL instead. `BITCOUNT` over a huge bitmap is O(N) in bytes; it's fast but not free.

```redis
SETBIT active:2026-06-21 42 1     # mark user id 42 active today
GETBIT active:2026-06-21 42       # 1
BITCOUNT active:2026-06-21        # how many users active (population count)
BITCOUNT active:2026-06-21 0 0 BYTE   # count within a byte range
BITPOS active:2026-06-21 1        # offset of the first set bit

# Combine days with bitwise ops, storing the result in a destination key:
BITOP AND result active:mon active:tue     # users active on BOTH days
BITOP OR  weekly active:mon active:tue      # active on ANY day
BITOP XOR / NOT ...

# Pack multiple counters into one string and operate atomically:
BITFIELD stats SET u8 0 255 GET u8 0 INCRBY u8 0 10
```

```js
// ioredis — count daily active users cheaply across millions of IDs:
await redis.setbit("dau:2026-06-21", 1000042, 1);
const dau = await redis.bitcount("dau:2026-06-21");
```

```go
// go-redis — bitmap DAU:
rdb.SetBit(ctx, "dau:2026-06-21", 1000042, 1)
dau, _ := rdb.BitCount(ctx, "dau:2026-06-21", nil).Result() // nil = count whole key
_ = dau
```

### 2.7 HyperLogLog **[I]**

**What it is.** A **probabilistic** data structure that estimates the number of **unique elements (cardinality)** of a set using a **fixed ~12 KB** of memory, *no matter how many items you add*, with a standard error of about **0.81%**. The trade-off: you **cannot list the members**, only estimate the count.

**Why/when.** When the question is "**roughly how many uniques?**" at a scale where storing the actual members would be prohibitive: unique visitors per page/day, unique search terms, unique IPs, dedup counting across billions of events. An HLL counting 50 million uniques still costs 12 KB; the equivalent Set would cost hundreds of MB.

**Key parameters.** `PFADD key element...` adds elements (returns 1 if the estimate likely changed). `PFCOUNT key...` returns the estimated cardinality (and the estimated cardinality of the *union* when given several keys). `PFMERGE dest src...` merges HLLs into one — so you can keep per-day HLLs and merge them for a weekly unique count without double-counting.

**Best practices.** Use HLLs precisely when approximate-but-mergeable counting is acceptable and exactness/membership is not needed. Keep per-bucket HLLs (per day, per page) and `PFMERGE` on demand — merging is exact at the structure level, so weekly = merge of 7 dailies.

```redis
PFADD visitors:home "user:1" "user:2" "user:3"   # add elements
PFCOUNT visitors:home                            # estimated unique count
PFADD visitors:about "user:2" "user:9"
PFMERGE visitors:all visitors:home visitors:about  # union into one HLL
PFCOUNT visitors:all                              # estimated union cardinality
```

```js
// ioredis — unique visitors per page, mergeable across pages/days:
await redis.pfadd("uv:2026-06-21:home", "u1", "u2");
const approxUnique = await redis.pfcount("uv:2026-06-21:home");
```

```go
// go-redis — HLL:
rdb.PFAdd(ctx, "uv:2026-06-21:home", "u1", "u2")
approx, _ := rdb.PFCount(ctx, "uv:2026-06-21:home").Result()
_ = approx
```

> **Set vs HyperLogLog — the decision in one line:** a Set of 10M IDs costs hundreds of MB and gives **exact** counts *plus* membership tests; an HLL costs **12 KB** and gives an **approximate count only**. Choose by whether you need exactness/membership or just a scalable estimate.

### 2.8 Geospatial **[I]**

**What it is.** Geo commands are syntactic sugar over a **sorted set**: each location's longitude/latitude is geohash-encoded into the ZSET score, so "points near here" becomes a score-range query. You store named points, then query by radius or bounding box and get back distances.

**Why/when.** "**Stores near me**," ride-hailing **driver matching**, **geofencing**, delivery-zone checks — any "find things within X of a coordinate" feature.

**Key parameters.** `GEOADD key lon lat member`. `GEOSEARCH` (6.2+, the modern unified command that replaces `GEORADIUS`/`GEORADIUSBYMEMBER`): `FROMMEMBER m` or `FROMLONLAT lon lat` to set the center, `BYRADIUS r unit` or `BYBOX w h unit` for the shape, `ASC`/`DESC` to sort by distance, `COUNT n` to cap, and `WITHCOORD`/`WITHDIST`/`WITHHASH` for extra columns. `GEOSEARCHSTORE` writes the matching members into a new sorted set.

**Best practices.** Because geo *is* a ZSET, you can mix in normal ZSET ops, and all the ZSET caveats apply (e.g. it lives on one shard in Cluster). Always specify a `COUNT` for radius queries on dense areas, and prefer `GEOSEARCH` over the deprecated `GEORADIUS` family.

```redis
GEOADD places -122.27 37.80 "oakland" -122.42 37.77 "sf"
GEOPOS places "sf"                  # → [longitude, latitude]
GEODIST places "sf" "oakland" km    # distance in km between two members

# Modern unified search (Redis 6.2+, replaces GEORADIUS):
GEOSEARCH places FROMMEMBER "sf" BYRADIUS 20 km ASC WITHCOORD WITHDIST
GEOSEARCH places FROMLONLAT -122.4 37.7 BYBOX 40 40 km ASC COUNT 10
GEOSEARCHSTORE dest places FROMMEMBER "sf" BYRADIUS 50 km   # store results in a ZSET
```

```js
// ioredis — find places within 20km of a point, with distances:
await redis.geoadd("places", -122.42, 37.77, "sf");
const near = await redis.geosearch(
  "places", "FROMLONLAT", -122.4, 37.7, "BYRADIUS", 20, "km", "ASC", "WITHDIST"
);
// → [["sf","1.83"], ...]  (member + distance)
```

```go
// go-redis — typed geo search:
rdb.GeoAdd(ctx, "places", &redis.GeoLocation{Name: "sf", Longitude: -122.42, Latitude: 37.77})
near, _ := rdb.GeoSearchLocation(ctx, "places", &redis.GeoSearchLocationQuery{
	GeoSearchQuery: redis.GeoSearchQuery{
		Longitude: -122.4, Latitude: 37.7,
		Radius: 20, RadiusUnit: "km", Sort: "ASC",
	},
	WithDist: true,
}).Result()
_ = near
```

### 2.9 Streams **[I/A]**

**What it is.** An **append-only log** — effectively a mini Kafka topic living inside Redis. Each entry has an auto-generated, monotonically increasing **ID** of the form `<ms>-<seq>` and a set of field/value pairs. Streams support fan-out reads, blocking tails, capped length, and — the headline feature — **consumer groups** that give **at-least-once, load-balanced** processing with explicit acknowledgements. (Full treatment in §5.)

**Why/when.** **Event sourcing**, **durable job queues**, activity logs, IoT/telemetry ingestion, and any internal message bus where (unlike Pub/Sub) you need entries to **persist** and consumers to **resume after a crash**.

**Key parameters.** `XADD key * field value...` appends (`*` = let the server assign the ID). `MAXLEN ~ n` / `MINID ~ id` cap the stream by length or minimum ID (the `~` makes trimming approximate and therefore cheap). `XREAD [BLOCK ms] STREAMS key id` tails the stream (`$` = only new entries; `0` = from the start).

```redis
XADD events * type "signup" user "42"     # * = server-generated, sorted ID
XADD events MAXLEN ~ 10000 * k v          # cap length (~ = approximate trim, much cheaper)
XLEN events
XRANGE events - +                          # all entries (- = smallest ID, + = largest)
XRANGE events - + COUNT 10
XREAD COUNT 5 STREAMS events 0             # read from the beginning
XREAD BLOCK 0 STREAMS events $             # block forever for entries added AFTER now
```

### 2.10 JSON (RedisJSON) **[I]**

**What it is.** Native **JSON documents** stored in a parsed binary tree, queried and mutated with **JSONPath**. Because the document is stored parsed (not as an opaque string), you can read or update a deeply nested field **without fetching and rewriting the whole document**.

**Why/when.** Document storage with **partial updates**, config trees, caching API responses while still accessing individual fields, and — combined with RediSearch (§15) — indexed queries over JSON. It's the natural fit when your data is genuinely document-shaped and you frequently touch sub-fields.

⚡ **Version note:** RedisJSON (the `JSON.*` commands) is a **module**. On **Redis 8** it ships in core — just use `redis:8`. On Redis 7.x you need the **Redis Stack** image (`redis/redis-stack`). The fully-open **Valkey** does not bundle it (store JSON as a string, or use a community module).

**Key parameters.** `JSON.SET key path value` sets a value at a JSONPath (`$` is the root). `JSON.GET key path` reads (returns a JSON array of matches). `JSON.NUMINCRBY`, `JSON.ARRAPPEND`, `JSON.STRAPPEND`, `JSON.DEL` mutate in place at a path.

**Best practices.** Don't reach for JSON reflexively — a plain hash is cheaper and simpler for flat objects. JSON earns its keep when documents are nested *and* you update sub-fields often, or when you'll index them with `FT.CREATE ... ON JSON`.

```redis
JSON.SET user:1 $ '{"name":"Ada","age":36,"roles":["admin"],"addr":{"city":"London"}}'
JSON.GET user:1 $.name              # ["Ada"]
JSON.GET user:1 $.addr.city         # ["London"]
JSON.SET user:1 $.age 37            # update a nested field IN PLACE (no full rewrite)
JSON.NUMINCRBY user:1 $.age 1       # atomic numeric increment at a path
JSON.ARRAPPEND user:1 $.roles '"editor"'   # push onto a JSON array
JSON.ARRLEN user:1 $.roles
JSON.STRAPPEND user:1 $.name '" Lovelace"'
JSON.DEL user:1 $.addr.city         # delete a path
JSON.OBJKEYS user:1 $.addr
JSON.TYPE user:1 $.roles            # "array"
JSON.MGET user:1 user:2 $.name      # one field across multiple documents
```

```js
// node-redis v4+ exposes JSON via client.json.* (first-class helpers):
await client.json.set("user:1", "$", { name: "Ada", age: 36, roles: ["admin"] });
const name = await client.json.get("user:1", { path: "$.name" }); // ["Ada"]
await client.json.numIncrBy("user:1", "$.age", 1);
await client.json.arrAppend("user:1", "$.roles", "editor");

// ioredis has no built-in JSON helpers; use .call() to send raw module commands:
await redis.call("JSON.SET", "user:2", "$", JSON.stringify({ name: "Bob" }));
const v = await redis.call("JSON.GET", "user:2", "$.name");
```

```go
// go-redis v9 has JSON helpers (JSONGet/JSONSet/...) when the module is present:
rdb.JSONSet(ctx, "user:1", "$", `{"name":"Ada","age":36}`)
name, _ := rdb.JSONGet(ctx, "user:1", "$.name").Result()  // ["Ada"]
rdb.JSONNumIncrBy(ctx, "user:1", "$.age", 1)
// Or fall back to raw commands for anything not wrapped:
//   rdb.Do(ctx, "JSON.SET", key, path, val)
```

### Type quick-reference

| Type | Stores | Killer command(s) | Classic use |
|---|---|---|---|
| String | bytes / number | `SET`, `INCR` | cache, counter, flag |
| List | ordered sequence | `LPUSH`/`BRPOP`, `LMOVE` | queue, timeline |
| Set | unique members | `SADD`, `SINTER` | tags, uniques, relationships |
| Sorted Set | scored members | `ZADD`, `ZRANGE REV` | leaderboard, priority/delay queue |
| Hash | field→value | `HSET`, `HINCRBY` | objects/records |
| Bitmap | bits in a string | `SETBIT`, `BITCOUNT` | DAU, per-user flags |
| HyperLogLog | approx cardinality | `PFADD`, `PFCOUNT` | unique counting at scale |
| Geo | lon/lat points | `GEOADD`, `GEOSEARCH` | nearby search |
| Stream | append-only log | `XADD`, `XREADGROUP` | events, durable queues |
| JSON | JSON documents | `JSON.SET`, `JSON.GET` | nested documents, configs |

---

## 3. Key Management — TTL, SCAN, Naming

Mastering keys — how to inspect, expire, iterate, and name them — is what separates a toy Redis setup from a maintainable one. This section is short on glamour and long on the things that cause (or prevent) production incidents.

### 3.1 Existence, type, deletion, rename **[B]**

These are the everyday key-housekeeping commands. The one that matters most operationally is the **`DEL` vs `UNLINK`** distinction: `DEL` frees the value *synchronously* on the single thread, so deleting a multi-gigabyte structure **blocks the whole server** for the duration; `UNLINK` unlinks the key immediately and frees the memory **in a background thread**. Prefer `UNLINK` for anything that might be large.

```redis
EXISTS user:1            # 1 if it exists (EXISTS k1 k2 k3 returns how many of them exist)
TYPE user:1             # string | list | set | zset | hash | stream | ...
DEL user:1 user:2       # delete keys SYNCHRONOUSLY (blocks on huge values)
UNLINK user:1           # delete ASYNCHRONOUSLY (frees memory in the background) — prefer for big keys
RENAME old new          # rename (overwrites new); errors if `old` is missing
RENAMENX old new        # rename only if `new` does not already exist
COPY src dst [REPLACE]  # copy a key (6.2+)
RANDOMKEY               # a random key (debugging)
DBSIZE                  # number of keys in the current DB
FLUSHDB / FLUSHALL      # ⚠ wipe the current DB / ALL DBs (an ASYNC option exists)
OBJECT ENCODING user:1  # internal encoding (listpack, intset, skiplist, embstr, ...)
OBJECT IDLETIME user:1  # seconds since last access (LRU info)
OBJECT FREQ user:1      # access-frequency counter (when maxmemory-policy is LFU)
```

> **Security gotcha:** `FLUSHALL`/`FLUSHDB`/`CONFIG`/`DEBUG`/`KEYS` are exactly the commands an attacker or a buggy client can use to wipe or DoS your instance. Disable or rename them for application users via ACLs (§14).

### 3.2 TTL / expiration — how caches stay fresh **[B]**

**What it is.** Any key can carry a **time-to-live (TTL)**; when it elapses, Redis deletes the key. TTL is the mechanism behind cache freshness, auto-expiring sessions, one-time tokens, and self-cleaning rate-limit windows.

**How expiry *actually* works (and why it matters).** Redis uses **two** strategies together: **lazy expiration** (when a client touches a key, Redis checks whether it expired and, if so, deletes it then) *plus* an **active background sampler** that periodically scans a sample of keys-with-TTL and evicts the expired ones. The consequence: an expired key may briefly continue to occupy memory until it's accessed or sampled. This is fine for almost everything, but **do not assume memory is freed the instant a TTL hits** — under memory pressure, lean on `maxmemory` + eviction (§9), not on TTL alone.

```redis
SET s "v" EX 60          # set value AND TTL in one atomic command (preferred)
EXPIRE s 120             # (re)set TTL to 120 seconds
EXPIRE s 120 NX          # only if there is no TTL yet (NX/XX/GT/LT flags, Redis 7.0+)
PEXPIRE s 5000           # TTL in milliseconds
EXPIREAT s 1893456000    # expire at an ABSOLUTE Unix time (seconds)
PEXPIREAT s 1893456000000
TTL s                    # remaining seconds (-1 = key exists but no TTL, -2 = key gone)
PTTL s                   # remaining milliseconds
PERSIST s                # remove the TTL → the key becomes permanent
EXPIRETIME s             # the absolute expiry time in seconds (7.0+)
```

> **The #1 TTL bug:** most write commands (`INCR`, `LPUSH`, `HSET`, `APPEND`, `SADD`, ...) **preserve** an existing TTL, but a plain **`SET k v` *clears* the TTL** unless you add `KEEPTTL`. So overwriting a cache entry with `SET` silently makes it permanent. Either re-`EXPIRE` after `SET`, use `SET ... EX`, or use `SET ... KEEPTTL`.

### 3.3 `SCAN` vs `KEYS` — never run `KEYS` in production **[B/I]**

This is the single most common Redis production incident, so it gets its own section.

```redis
KEYS user:*             # ⚠ O(N) over the ENTIRE keyspace, BLOCKS the single thread. DEV ONLY!
```

**Why `KEYS` is dangerous.** `KEYS` walks **every key in the instance** and builds the full reply before returning — and it does so on the one command thread. On a database with millions of keys it can **freeze Redis for seconds**, stalling every other client (remember: single thread, §1). It looks harmless in a `redis-cli` on a tiny dev DB and then takes down production.

**The safe alternative: the `SCAN` family.** `SCAN` is **cursor-based**: each call returns a small batch of keys plus a cursor; you repeat until the cursor comes back `0`. It never blocks for long, so it's safe to run against production.

```redis
# SCAN returns [next-cursor, [keys...]]. Repeat until the cursor is 0.
SCAN 0 MATCH user:* COUNT 100
SCAN 17 MATCH user:* COUNT 100   # feed the returned cursor back in
# Type-specific variants iterate WITHIN one big collection (not the keyspace):
HSCAN bighash 0 COUNT 100
SSCAN bigset  0 COUNT 100
ZSCAN bigzset 0 COUNT 100
# SCAN ... TYPE string           # filter by value type (Redis 6.0+)
```

A subtle but important guarantee: `SCAN` promises that **every key present for the entire duration of the scan is returned at least once**, but it **may return duplicates** and reflects concurrent insertions/deletions. So **dedupe if you must**, and treat the result as a best-effort live view, not a frozen snapshot.

```js
// ioredis — iterate the whole keyspace safely with the built-in stream wrapper:
const stream = redis.scanStream({ match: "session:*", count: 100 });
stream.on("data", (keys) => {
  if (keys.length) redis.unlink(...keys); // delete in batches, asynchronously (UNLINK)
});
stream.on("end", () => console.log("done"));
```

```go
// go-redis — the SCAN iterator hides the cursor loop for you:
iter := rdb.Scan(ctx, 0, "session:*", 100).Iterator()
for iter.Next(ctx) {
	key := iter.Val()
	rdb.Unlink(ctx, key) // async free
}
if err := iter.Err(); err != nil {
	panic(err)
}
```

### 3.4 Key naming conventions **[B]**

**Why naming matters.** Redis has a **flat namespace** — there are no tables or folders. So you encode hierarchy *into the key name*, by convention, using `:` as a separator. A disciplined scheme makes keys self-describing, greppable with `SCAN`, easy to scope by prefix, and safe to share an instance across apps.

```
object-type:id:field          e.g.  user:1000:profile
object-type:id:subtype:id     e.g.  post:42:comments
feature:scope:window          e.g.  ratelimit:ip:1.2.3.4:2026-06-21
app:env:...                   e.g.  myapp:prod:cache:user:1000
```

Best practices:

- **Prefix by app/feature** so several apps can share one instance without colliding (`myapp:`, `billing:`).
- **Keep keys short but readable** — key strings live in RAM too; millions of long keys add up to real memory.
- **Put the variable part last** so a prefix scan (`SCAN feature:*`) is natural.
- **Use a consistent separator** — `:` is the de-facto standard across the ecosystem and tooling.
- **For Cluster (§13):** keys sharing the same `{hashtag}` land on the **same slot**, which is required for multi-key ops to work across them. Use `user:{1000}:profile` and `user:{1000}:sessions` when you need `MGET`/transactions/Lua across related keys.

| Command | Purpose |
|---|---|
| `EXISTS` / `TYPE` / `OBJECT ENCODING` | Inspect a key |
| `DEL` (sync) / `UNLINK` (async) | Delete (prefer `UNLINK` for big values) |
| `RENAME` / `RENAMENX` / `COPY` | Move / duplicate |
| `EXPIRE` / `PEXPIRE` / `EXPIREAT` / `PERSIST` | Manage TTL |
| `TTL` / `PTTL` / `EXPIRETIME` | Inspect TTL |
| `SCAN` / `HSCAN` / `SSCAN` / `ZSCAN` | Safe iteration (use instead of `KEYS`) |

---

## 4. Pub/Sub

### 4.1 What Pub/Sub is — and its one defining limitation **[I]**

**Publish/Subscribe** is a **fire-and-forget, fan-out** messaging system. **Publishers** send messages to named **channels**; **subscribers** listening on those channels receive them. The defining characteristic — and the thing you must understand before using it — is that there is **no persistence and no acknowledgement**. A message exists only at the instant it's published: if **no one is subscribed** at that moment, it is simply gone, and a subscriber that disconnects misses everything sent while it was away. Pub/Sub is a *broadcast bus*, not a queue.

**Why/when.** Use Pub/Sub when losing the occasional message is acceptable and you want simple, low-latency fan-out to many listeners: **real-time notifications**, **cache-invalidation broadcasts** ("key X changed, drop it from your local cache"), chat where missing offline messages is fine, and live dashboards. The moment you need durability, delivery guarantees, retries, or replay, switch to **Streams** (§5) — that's the single most important Pub/Sub-vs-Streams decision.

```redis
# Terminal 1 — subscribe (in RESP2 this connection can now ONLY do (un)subscribe commands):
SUBSCRIBE news sports
PSUBSCRIBE news.*           # pattern subscribe (glob-style matching)

# Terminal 2 — publish:
PUBLISH news "Redis 8 released"     # returns the number of receivers it reached
PUBLISH news.tech "module update"

# Introspection:
PUBSUB CHANNELS              # active channels that currently have subscribers
PUBSUB NUMSUB news          # subscriber count for given channels
PUBSUB NUMPAT               # number of active pattern subscriptions
```

⚡ **Version note:** In **Redis Cluster**, classic `PUBLISH` is **broadcast to every node** (it works but is chatty and doesn't scale). **Sharded Pub/Sub** (`SPUBLISH` / `SSUBSCRIBE` / `SUNSUBSCRIBE`, Redis 7.0+) keeps a message on the shard owning the channel's slot — use it for scalable Pub/Sub in a cluster.

### 4.2 Client code **[I]**

```js
// ioredis — a SUBSCRIBER must be a SEPARATE connection from one issuing normal commands (see below).
const sub = new Redis();           // dedicated subscriber connection
const pub = new Redis();           // publisher / normal commands

await sub.subscribe("news");
sub.on("message", (channel, message) => {
  console.log(`[${channel}] ${message}`);
});

// Pattern subscription delivers on the "pmessage" event:
await sub.psubscribe("news.*");
sub.on("pmessage", (pattern, channel, message) => {
  console.log(`(${pattern}) ${channel}: ${message}`);
});

await pub.publish("news", "hello world");
```

```go
// go-redis — Subscribe returns a *PubSub; range over its Go channel:
sub := rdb.Subscribe(ctx, "news")
defer sub.Close()

ch := sub.Channel() // a Go channel of *redis.Message
go func() {
	for msg := range ch {
		fmt.Printf("[%s] %s\n", msg.Channel, msg.Payload)
	}
}()

rdb.Publish(ctx, "news", "hello world")
```

> **Why a dedicated subscriber connection?** In RESP2, once a connection enters subscribe mode it may **only** issue (un)subscribe commands — it cannot run `GET`/`SET`. RESP3 relaxes this (subscriptions arrive as out-of-band *push* messages, so the connection stays usable), but keeping a **separate connection for subscriptions** is the safe, portable habit across both protocols and all clients.

---

## 5. Streams & Consumer Groups

Streams turn Redis into a **durable, ordered, replayable log** with Kafka-like semantics. The contrast with Pub/Sub is the whole point: stream entries **persist** (subject to your trim policy), and consumers can **resume** from where they left off, **acknowledge** what they processed, and **share work** within a group with crash recovery. This is the right tool for real job queues and event pipelines inside Redis.

### 5.1 Producing & basic reading **[I]**

```redis
# XADD appends an entry. * tells the server to assign a sorted ID (<ms>-<seq>).
XADD orders * item "book" qty 2
XADD orders * item "pen"  qty 5

# Cap the stream so it doesn't grow forever (a stream is otherwise an unbounded memory leak):
XADD orders MAXLEN ~ 100000 * item "x"    # ~ = APPROXIMATE trim (much cheaper than exact)
XADD orders MINID ~ 1700000000000-0 * ... # trim by minimum ID (time-based retention)
XTRIM orders MAXLEN 50000                  # explicit, standalone trim

XLEN orders                  # entry count
XRANGE orders - +            # all entries, oldest → newest (- = min ID, + = max ID)
XREVRANGE orders + - COUNT 1 # the single newest entry
XDEL orders 1700000000000-0  # delete a specific entry by ID

# Tailing the stream (like `tail -f`): block for entries added AFTER the last ID seen.
XREAD COUNT 10 BLOCK 5000 STREAMS orders $   # $ = only entries added after NOW
XREAD STREAMS orders 0                         # everything from the very start
```

### 5.2 Consumer groups — load-balanced, at-least-once processing **[A]**

**What a consumer group is.** A consumer group lets **multiple workers cooperatively** drain one stream. Each new entry is delivered to **exactly one consumer in the group**, and it's tracked in that group's **Pending Entries List (PEL)** until the worker **acknowledges** it with `XACK`. If a worker dies after reading but before acking, the entry **stays pending**, and another worker can **claim** it and finish the job. That combination — exactly-one-consumer delivery + pending tracking + claim — is what gives **at-least-once** delivery with crash recovery, the property a real job queue needs.

**The mental model.** Think of three cursors: `>` means "give me messages **never delivered** to anyone in this group" (normal consumption); `0` (or a specific ID) means "re-read **my own** still-pending messages" (recovery after a restart); and the group's last-delivered ID, which `XGROUP SETID` can rewind to replay history.

```redis
# Create a group. $ = start from NEW messages only; 0 = from the beginning of the stream.
# MKSTREAM creates the stream if it doesn't exist yet.
XGROUP CREATE orders workers $ MKSTREAM

# A consumer reads as many NEW (never-delivered) messages as it can.
# > means "messages never delivered to ANY consumer in this group":
XREADGROUP GROUP workers worker-1 COUNT 10 BLOCK 5000 STREAMS orders >

# After successfully processing an entry, ACK it so it leaves the PEL:
XACK orders workers 1700000000000-0

# Inspect what's still pending (unacknowledged) — your queue's "in-flight" view:
XPENDING orders workers                          # summary
XPENDING orders workers - + 10                    # detailed, up to 10 entries
XPENDING orders workers - + 10 worker-1           # for one consumer

# Recover from a crashed consumer — claim messages idle longer than 60s:
XAUTOCLAIM orders workers worker-2 60000 0        # (Redis 6.2+, the preferred recovery tool)
XCLAIM orders workers worker-2 60000 1700000000000-0   # claim specific IDs

# Re-read a consumer's OWN pending (not-yet-acked) entries (use 0 instead of >):
XREADGROUP GROUP workers worker-1 STREAMS orders 0

# Housekeeping & introspection:
XINFO STREAM orders                # stream metadata (length, first/last ID, ...)
XINFO GROUPS orders                # groups, their lag, and pending counts
XINFO CONSUMERS orders workers     # per-consumer pending count and idle time
XGROUP DELCONSUMER orders workers worker-1
XGROUP SETID orders workers 0      # rewind the group to replay from the start
```

| Command | Purpose |
|---|---|
| `XADD [MAXLEN/MINID] key * f v` | Append (with optional trim) |
| `XLEN` / `XRANGE` / `XREVRANGE` / `XDEL` / `XTRIM` | Inspect / range / delete / trim |
| `XREAD [BLOCK] STREAMS key id` | Read/tail without a group |
| `XGROUP CREATE/SETID/DELCONSUMER/DESTROY` | Manage groups |
| `XREADGROUP GROUP g c STREAMS key >` | Consume new (or `0` for own pending) |
| `XACK` | Acknowledge processed entries (removes from the PEL) |
| `XPENDING` | Inspect the Pending Entries List |
| `XCLAIM` / `XAUTOCLAIM` | Reassign stalled entries from a dead consumer |
| `XINFO STREAM/GROUPS/CONSUMERS` | Introspection |

```js
// ioredis — a robust consumer-group worker (ack-on-success, leave-pending-on-failure).
const GROUP = "workers";
const CONSUMER = `worker-${process.pid}`;

async function ensureGroup() {
  try {
    await redis.xgroup("CREATE", "orders", GROUP, "$", "MKSTREAM");
  } catch (e) {
    // BUSYGROUP just means the group already exists — that's fine.
    if (!String(e.message).includes("BUSYGROUP")) throw e;
  }
}

async function run() {
  await ensureGroup();
  while (true) {
    // Read new messages; block up to 5s if the stream is idle.
    const res = await redis.xreadgroup(
      "GROUP", GROUP, CONSUMER, "COUNT", 10, "BLOCK", 5000, "STREAMS", "orders", ">"
    );
    if (!res) continue; // timed out, no new messages
    const [, entries] = res[0]; // res = [[stream, [[id, [f,v,...]], ...]]]
    for (const [id, fields] of entries) {
      try {
        await process(fields);                 // your business logic
        await redis.xack("orders", GROUP, id); // ACK ONLY on success
      } catch (err) {
        console.error("processing failed; will be reclaimed later:", err);
        // Do NOT ack → the entry stays pending → XAUTOCLAIM reassigns it after the idle timeout.
      }
    }
  }
}
```

```go
// go-redis — consumer-group worker:
func runWorker(ctx context.Context, rdb *redis.Client) error {
	const group, consumer = "workers", "worker-1"
	// Ignore the BUSYGROUP error (group already exists).
	rdb.XGroupCreateMkStream(ctx, "orders", group, "$")

	for {
		res, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
			Group:    group,
			Consumer: consumer,
			Streams:  []string{"orders", ">"},
			Count:    10,
			Block:    5 * time.Second,
		}).Result()
		if err == redis.Nil {
			continue // timeout, no new messages
		} else if err != nil {
			return err
		}
		for _, stream := range res {
			for _, msg := range stream.Messages {
				if err := handle(msg.Values); err != nil {
					continue // leave pending → reclaimed by XAUTOCLAIM later
				}
				rdb.XAck(ctx, "orders", group, msg.ID)
			}
		}
	}
}
```

### 5.3 Streams vs Kafka **[A]**

| Aspect | Redis Streams | Apache Kafka |
|---|---|---|
| Storage | In-memory (+ persistence), trimmed by length/age | Disk log, long retention by design |
| Scale | Single shard per stream (shard by key in Cluster) | Native partitioned topics, huge throughput |
| Delivery | At-least-once via groups + ACK + claim | At-least-once / exactly-once (with transactions) |
| Ordering | Per stream | Per partition |
| Ops weight | Lightweight (you already run Redis) | Heavier (brokers, KRaft/ZooKeeper) |
| Best for | Real-time jobs, moderate volume, low latency | High-throughput pipelines, long retention, replay |

**Rule of thumb:** if you **already run Redis** and need a durable queue/event log at **moderate scale** with **millisecond latency**, Streams are perfect and save you a whole system. If you need **massive throughput**, **multi-day retention**, and a rich ecosystem (Kafka Connect, ksqlDB), reach for Kafka.

---

## 6. Transactions & Optimistic Locking

### 6.1 What a Redis transaction is — and what it is *not* **[I]**

A Redis transaction groups commands so they execute **atomically and in isolation**: once `EXEC` runs, the queued commands run **back-to-back with no other client's commands interleaved**. Four commands implement it: `MULTI` (start queuing), `EXEC` (run the queue), `DISCARD` (throw the queue away), and `WATCH` (add an optimistic-locking condition).

> **The single most important caveat — Redis transactions are NOT SQL transactions.** There is **no rollback.** If a queued command fails **at runtime** (for example, you run a list command against a string key), the **other commands still execute** and `EXEC` does **not** undo anything. What you *do* get is *grouping + isolation*, not all-or-nothing atomicity in the SQL sense. (Contrast this with PostgreSQL's `BEGIN ... ROLLBACK` in **`POSTGRESQL_GUIDE.md`** — Redis deliberately trades rollback for speed and simplicity.)

```redis
MULTI                 # begin queuing — subsequent commands reply "QUEUED" instead of running
SET account:a 100
DECRBY account:a 30
INCRBY account:b 30
EXEC                  # run all queued commands as one isolated batch → array of replies
# (DISCARD instead of EXEC throws the queued commands away without running them)
```

**The two error modes you must handle:**

- **Compile-time / queuing error** (a typo'd command name, wrong number of args) → the *whole* transaction is **aborted at `EXEC`** (Redis 2.6.5+ detects these while queuing and refuses to run any of it).
- **Runtime error** (a valid command applied to the wrong type, e.g. `INCR` on a list) → **only that one command errors**; **the rest still run**. You must inspect each reply in the `EXEC` result array yourself.

### 6.2 Optimistic locking with `WATCH` **[I]**

**What `WATCH` does.** `WATCH key...` makes the following `EXEC` **conditional**: if **any watched key is modified by anyone** between `WATCH` and `EXEC`, the transaction **aborts** — `EXEC` returns nil — and you **retry**. This is **compare-and-set / optimistic concurrency control**: instead of holding a blocking lock while you "read, decide, write," you read, decide, and let `EXEC` *verify nothing changed underneath you*. It's ideal for low-contention "read-compute-conditionally-write" logic where the work must live in your application.

**Why optimistic.** Under low contention, conflicts are rare, so you almost never retry and you never block other clients — far cheaper than a pessimistic lock. Under *high* contention, retries multiply, and a Lua script (§7) — which runs the whole logic atomically server-side with no retry loop — is usually better.

```redis
WATCH inventory:item:1          # watch the key(s) you are about to read & conditionally write
# (now read the value in your app)
# GET inventory:item:1  → e.g. "5"
# decide: enough stock? if so:
MULTI
DECR inventory:item:1
EXEC
# If ANOTHER client modified inventory:item:1 after the WATCH, EXEC returns (nil) → retry from WATCH.
```

```js
// ioredis — atomic "decrement stock if available" with WATCH + a bounded retry loop.
async function buyOne(itemKey) {
  for (let attempt = 0; attempt < 5; attempt++) {
    await redis.watch(itemKey);                 // start watching
    const stock = parseInt((await redis.get(itemKey)) || "0", 10);
    if (stock <= 0) {
      await redis.unwatch();                     // nothing to do → release the watch
      return { ok: false, reason: "out of stock" };
    }
    // Queue the conditional write:
    const result = await redis
      .multi()
      .decr(itemKey)
      .exec();
    // exec() returns null when a watched key changed → retry; otherwise success.
    if (result !== null) return { ok: true, remaining: stock - 1 };
  }
  return { ok: false, reason: "too much contention" };
}
```

```go
// go-redis — the idiomatic Watch helper runs your closure and handles the loop/retry signal.
func buyOne(ctx context.Context, rdb *redis.Client, key string) error {
	txf := func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		if n <= 0 {
			return fmt.Errorf("out of stock")
		}
		// Pipelined writes run only if no watched key changed (else TxFailedErr is returned):
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Decr(ctx, key)
			return nil
		})
		return err
	}
	// Retry on an optimistic-lock conflict (redis.TxFailedErr):
	for i := 0; i < 5; i++ {
		err := rdb.Watch(ctx, txf, key)
		if err == nil {
			return nil
		}
		if err == redis.TxFailedErr {
			continue // someone else changed the key; retry
		}
		return err
	}
	return fmt.Errorf("too much contention")
}
```

> **Transactions vs Lua — when to use which:** for complex read-compute-write atomicity, **Lua scripts (§7) are usually simpler and faster** than `WATCH`/`MULTI` because the whole script runs atomically server-side with no retry loop at all. Reach for `WATCH` when the decision logic must live in your application (e.g. it depends on data not in Redis); reach for Lua when the logic can live entirely in the script.

---

## 7. Lua Scripting & Functions

### 7.1 Why scripting exists — atomic read-modify-write in one round trip **[I/A]**

**The problem it solves.** Many useful operations are "read a value, decide something, then write" — check-then-set, conditional updates, multi-key atomic mutations. Doing this from the client means multiple round trips *and* a race window between the read and the write. `WATCH` (§6) closes the race but adds a retry loop. **Lua scripting** closes it differently and more cleanly: Redis runs your script **server-side, atomically** — while a script runs, **no other command executes** (the same single-threaded guarantee that makes one command atomic now covers your whole script). So you get read-modify-write logic with **no race, no retry loop, and one round trip.**

### 7.2 `EVAL` / `EVALSHA` **[I]**

```redis
# EVAL script numkeys key1 key2 ... arg1 arg2 ...
# Pass KEYS via KEYS[] and other args via ARGV[] (both 1-indexed in Lua).
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey hello

# A real example — set a key only if its current value matches (compare-and-set):
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then \
        return redis.call('SET', KEYS[1], ARGV[2]) \
      else return 0 end" 1 mykey oldval newval

# Avoid re-sending the whole script body each time: load it once, then call by its SHA1:
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "a5260dd66ce02462c5b5231c727b3f7772c0bcc5"  (the SHA1 of the script)
EVALSHA a5260dd66ce02462c5b5231c727b3f7772c0bcc5 1 mykey
# If you get a NOSCRIPT error (server restarted / cache flushed), SCRIPT LOAD again and retry.
SCRIPT EXISTS a5260dd...      # is this SHA cached?
SCRIPT FLUSH                  # clear the entire script cache
```

**Rules & gotchas (these prevent subtle bugs and Cluster breakage):**

- **Always pass keys via `KEYS[]`, never hardcode them in the script body.** Redis (and especially Cluster) must know which keys a script touches to route it correctly and verify all keys live in one slot. Hardcoding keys breaks Cluster and is considered a bug.
- **A script is atomic, so keep it fast.** While it runs, the whole server is blocked — a long-running script causes a latency spike for every client. No sleeping, no unbounded loops.
- **Determinism.** Older Redis required scripts to be deterministic (no `math.random`/time-dependent writes) because scripts were replicated verbatim. Modern Redis uses *effects replication* (it replicates the resulting writes, not the script), which relaxes this — but keeping scripts pure is still the safe default.
- **Return-type mapping (Lua → RESP):** Lua number → integer, string → bulk string, table → array, `false`/`nil` → nil, `true` → 1.

### 7.3 An atomic rate limiter in Lua (the canonical example) **[I]**

```lua
-- rate_limit.lua — fixed-window counter. Returns 1 if allowed, 0 if over the limit.
-- KEYS[1] = counter key, ARGV[1] = limit, ARGV[2] = window in seconds
local current = redis.call('INCR', KEYS[1])
if current == 1 then
  -- first hit of this window → set the window's expiry so the counter self-resets
  redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
  return 0   -- over the limit → reject
end
return 1     -- within the limit → allow
```

```js
// ioredis — register the script once with defineCommand, then call it like a built-in method.
redis.defineCommand("rateLimit", {
  numberOfKeys: 1,
  lua: `
    local current = redis.call('INCR', KEYS[1])
    if current == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
    if current > tonumber(ARGV[1]) then return 0 end
    return 1`,
});

// allowed === 1 means the request is within the limit:
const allowed = await redis.rateLimit(`rl:${userId}`, 100, 60); // 100 requests / 60s
```

```go
// go-redis — redis.NewScript handles EVALSHA-with-EVAL-fallback automatically.
var rateLimit = redis.NewScript(`
  local current = redis.call('INCR', KEYS[1])
  if current == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
  if current > tonumber(ARGV[1]) then return 0 end
  return 1`)

func allow(ctx context.Context, rdb *redis.Client, userID string) (bool, error) {
	// Run() tries EVALSHA first and falls back to EVAL on NOSCRIPT — no manual SHA bookkeeping.
	n, err := rateLimit.Run(ctx, rdb, []string{"rl:" + userID}, 100, 60).Int()
	return n == 1, err
}
```

### 7.4 Functions (Redis 7.0+) — the managed evolution of scripting **[A]**

**What Functions add over `EVAL`.** With `EVAL`/`EVALSHA`, scripts live only in an ephemeral cache that is **lost on restart** and **not replicated** as code — your app must be ready to re-`SCRIPT LOAD` on `NOSCRIPT`. **Redis Functions** fix this: you load a **named library** of functions that **persists with the dataset** (it survives restarts and replicates to replicas) and is callable by a stable name. Use Functions to ship **reusable, versioned, first-class server-side logic** instead of inline scripts scattered through your app.

⚡ **Version note:** Functions require **Redis 7.0+**. They use the `FUNCTION` command family and, by default, the Lua engine.

```lua
-- mylib.lua — a function library.
#!lua name=mylib

redis.register_function('my_incr', function(keys, args)
  return redis.call('INCRBY', keys[1], args[1])
end)
```

```redis
# Load the library (REPLACE updates an existing one in place):
FUNCTION LOAD REPLACE "$(cat mylib.lua)"
FCALL my_incr 1 counter 5      # call it: numkeys=1, KEYS=[counter], ARGS=[5]
FUNCTION LIST                   # list loaded libraries and their functions
FUNCTION DUMP / FUNCTION RESTORE  # back up / restore all libraries
FUNCTION FLUSH                  # remove all functions
```

| Command | Purpose |
|---|---|
| `EVAL` / `EVALSHA` | Run a Lua script (inline / by SHA) |
| `EVAL_RO` / `EVALSHA_RO` | Read-only variants (safe to run on replicas) |
| `SCRIPT LOAD` / `SCRIPT EXISTS` / `SCRIPT FLUSH` | Manage the script cache |
| `FUNCTION LOAD/LIST/DELETE/FLUSH/DUMP/RESTORE` | Manage function libraries |
| `FCALL` / `FCALL_RO` | Call a registered function |

> **Security:** scripts run with full server privileges. Never build a script body by string-concatenating untrusted input (a Lua injection); always pass user data through `ARGV[]`, which is treated as data, not code.

---

## 8. Persistence — RDB vs AOF

Redis is in-memory, but it can persist to disk so data survives restarts and crashes. There are **two** mechanisms with very different trade-offs, and they're frequently used **together**. Understanding them is how you decide your acceptable data-loss window.

### 8.1 RDB (snapshots) **[I]**

**What it is.** A **point-in-time binary snapshot** of the entire dataset, written compactly to `dump.rdb`. To avoid blocking, Redis `fork()`s a child process; the child writes the snapshot from a frozen copy-on-write view of memory while the **parent keeps serving requests**. So snapshotting is non-blocking on the command path — its only cost is the fork's copy-on-write memory overhead under heavy writes.

**Why/when.** RDB is excellent for **backups** (one compact file you can copy off-box), **fast restarts** (loading an RDB is much faster than replaying a log), and **replication bootstrap** (a new replica syncs from an RDB). The trade-off is the **data-loss window**: if Redis crashes between snapshots, you lose everything written since the last one — potentially minutes.

```conf
# redis.conf — save a snapshot when BOTH a time AND a change threshold are met.
# "after 900s if ≥1 key changed", "after 300s if ≥100 changed", "after 60s if ≥10000 changed".
save 900 1
save 300 100
save 60 10000
save ""                 # (empty string) disables automatic RDB snapshots

dbfilename dump.rdb
dir /data
rdbcompression yes
rdbchecksum yes
```

```redis
SAVE        # ⚠ SYNCHRONOUS snapshot — BLOCKS the server until done. Avoid in production.
BGSAVE      # background snapshot via fork (the one you actually use)
LASTSAVE    # Unix time of the last successful save (poll to confirm a BGSAVE finished)
```

- **Pros:** compact, fast to load, great for backups and replication, minimal steady-state overhead.
- **Cons:** you can lose all writes since the last snapshot if Redis crashes.

### 8.2 AOF (Append-Only File) **[I]**

**What it is.** The AOF **logs every write command** to a file; on restart Redis **replays** the log to reconstruct the dataset. It's far more durable than RDB (smaller loss window) but produces a larger file that must be periodically **rewritten/compacted** to stay small.

**The key knob — `appendfsync`** — is the durability/performance dial:

```conf
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"      # AOF lives in a multi-file directory (Redis 7+)

# fsync policy — how often the OS buffer is forced to disk:
# always   → fsync on every write     (safest, slowest; essentially no data loss)
# everysec → fsync once per second     (default; lose ≤1s of writes on crash) ← recommended
# no       → let the OS flush whenever  (fastest, least safe)
appendfsync everysec

# Auto-rewrite (compaction) thresholds — rewrite when the AOF grows past these:
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

```redis
BGREWRITEAOF      # manually trigger an AOF rewrite/compaction
```

⚡ **Version note:** Redis 7 changed AOF to a **multi-part layout** — a base RDB-format file plus incremental AOF files inside `appendonlydir/` — which makes rewrites more efficient. Don't hand-edit these files.

### 8.3 Durability trade-offs & the recommendation **[I]**

| | RDB only | AOF (everysec) | Both (recommended) |
|---|---|---|---|
| Worst-case loss on crash | minutes | ≤ 1 second | ≤ 1 second |
| Restart load speed | fast | slower (replay) | fast (loads AOF) |
| File size | small | larger | both |
| Backup convenience | excellent | awkward | use RDB for backups |
| Runtime overhead | low (periodic fork) | slightly higher | moderate |

**Default recommendation:** run **both** — AOF (`everysec`) for a small loss window on crash, plus periodic RDB snapshots for fast restarts and easy off-box backups. For a **pure cache** where loss is acceptable, you can disable persistence entirely (`save ""` + `appendonly no`) for maximum speed. For **strict durability**, use `appendfsync always` (slower) — but remember a *single* Redis is still a single point of failure no matter the fsync policy; combine persistence with **replication** (§13) for real durability.

> **Gotcha:** RDB saving and AOF rewrites both use `fork()`, which (via copy-on-write) can transiently use up to **~2× memory** under heavy write load, because pages modified during the save get copied. Provision RAM headroom and set `vm.overcommit_memory = 1` on Linux so the fork doesn't fail under memory pressure.

---

## 9. Caching Patterns & Eviction

Caching is Redis's number-one job, and the pattern you choose decides your consistency guarantees, latency, and behavior when things fail. The architecture context (from §1.2): **a durable store like PostgreSQL is the source of truth; Redis caches it.** Each pattern below is a different contract between those two.

### 9.1 Cache-aside (lazy loading) — the default **[I]**

**How it works.** The application checks the cache first. On a **hit**, it returns the cached value. On a **miss**, it loads from the database, stores the result in the cache with a TTL, and returns it. The database is never aware of the cache.

**Why it's the default.** It's **simple** and **resilient** — if Redis is down, your app still works (slower), because it falls back to the DB. It only caches **what's actually used** (no wasted memory on cold data). The downsides: the **first** request for each key is slow (a guaranteed miss), and the cache can be briefly **stale** after a DB write — which you mitigate by **deleting (or updating) the cache key on every write** so the next read repopulates it.

```js
// ioredis — cache-aside read-through helper:
async function getUser(id) {
  const key = `user:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);           // HIT

  const user = await db.users.findById(id);        // MISS → load from the DB
  if (user) {
    // Cache for ~1h. Add jitter so many keys don't expire at the SAME instant (see stampede).
    const ttl = 3600 + Math.floor(Math.random() * 120);
    await redis.set(key, JSON.stringify(user), "EX", ttl);
  }
  return user;
}

// On update, INVALIDATE so the next read repopulates from the DB:
async function updateUser(id, patch) {
  const user = await db.users.update(id, patch);
  await redis.del(`user:${id}`);                   // or set the new value directly
  return user;
}
```

```go
// go-redis — cache-aside; note the explicit redis.Nil handling for a miss:
func getUser(ctx context.Context, rdb *redis.Client, id string) (*User, error) {
	key := "user:" + id
	if b, err := rdb.Get(ctx, key).Bytes(); err == nil {
		var u User
		json.Unmarshal(b, &u)
		return &u, nil // cache HIT
	} else if err != redis.Nil {
		return nil, err // a REAL error (network, etc.) — not just a miss
	}
	u, err := loadFromDB(id) // MISS
	if err != nil {
		return nil, err
	}
	b, _ := json.Marshal(u)
	rdb.Set(ctx, key, b, time.Hour)
	return u, nil
}
```

> **The `redis.Nil` trap (Go):** in go-redis a **missing key is returned as the error `redis.Nil`**, not as an empty value. You must distinguish "miss" (`err == redis.Nil`, expected) from "real failure" (any other error). Treating every error as a miss hides outages; treating every error as fatal turns a normal cache miss into a 500.

### 9.2 Write-through **[I]**

**How it works.** Writes go **through the cache to the DB synchronously** — every write updates both. **Why:** the cache and DB stay consistent and reads are always warm (the data you just wrote is already cached). **Cost:** every write pays cache + DB latency, and you cache data that may never be read.

```js
async function writeThrough(id, value) {
  await db.users.update(id, value);                 // write the DB first (source of truth)
  await redis.set(`user:${id}`, JSON.stringify(value), "EX", 3600); // then update the cache
}
```

### 9.3 Write-behind (write-back) **[I/A]**

**How it works.** Writes hit the cache **immediately** and are flushed to the DB **asynchronously**, usually batched by a background worker. **Why:** lowest possible write latency and the least DB load (writes are coalesced). **Cost:** risk of data loss if Redis dies before the flush, plus the operational complexity of building and monitoring the flush pipeline (commonly a Stream or list acting as the buffer).

```js
// Buffer writes in a stream; a separate consumer-group worker (see §5) batches them to the DB.
await redis.set(`user:${id}`, JSON.stringify(value)); // serve reads instantly
await redis.xadd("db:writeback", "*", "table", "users", "id", id, "data", JSON.stringify(value));
// A worker drains "db:writeback" and bulk-upserts to the DB, then XACKs.
```

### 9.4 TTL strategies **[I]**

The TTL is your staleness budget. Different needs call for different strategies:

- **Absolute TTL:** expire N seconds after write (`SET ... EX`). Simple, bounds staleness predictably.
- **Refresh-ahead:** proactively reload hot keys *before* they expire (a background job watches popularity/`TTL`), so users never hit a cold miss on a hot key.
- **Sliding TTL:** extend the TTL on each access — perfect for **sessions** ("log out after 30 min of inactivity"). Re-`EXPIRE` on read, or do it in one command with `GETEX key EX 1800`.
- **TTL jitter:** add randomness to TTLs so a batch of keys created together don't all expire at the *same instant* (a stampede trigger — see §9.5).

```redis
GETEX session:abc EX 1800     # read the value AND slide the TTL to 30 min (Redis 6.2+)
GETEX session:abc PERSIST     # read and remove the TTL
GETDEL onetime:token          # read AND delete atomically — perfect for one-time tokens
```

### 9.5 Cache stampede / dogpile prevention **[I/A]**

**The thundering-herd problem.** A **stampede** (a.k.a. dogpile / thundering herd) happens when a **hot key expires** and, in the same instant, **many concurrent requests all miss** and all rush to the database to recompute the same value. The DB, which the cache existed to protect, gets hammered by a synchronized burst — sometimes hard enough to fall over, which then makes the cache unable to refill, a cascading failure. Three defenses, often combined:

1. **TTL jitter** (above) — desynchronize expirations so keys don't all die together. The cheapest, do-it-always defense.
2. **Locking / single-flight** — on a miss, the **first** caller takes a short lock and recomputes; everyone else briefly waits and then reads the freshly populated cache, so only **one** request hits the DB.
3. **Probabilistic early expiration (XFetch)** — recompute slightly *before* the TTL expires, at random, so one request refreshes the value while everyone else still gets a cache hit. No hard expiry cliff.

```js
// ioredis — single-flight cache fill: only ONE caller hits the DB; others briefly retry.
async function getWithLock(key, loader, ttl = 3600) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  // Acquire a SHORT lock (NX = only if absent, PX = auto-expire so a crash can't deadlock).
  const gotLock = await redis.set(lockKey, "1", "NX", "PX", 5000);
  if (!gotLock) {
    // Someone else is already filling it — wait briefly, then re-read the (now warm) cache.
    await new Promise((r) => setTimeout(r, 50));
    return getWithLock(key, loader, ttl);
  }
  try {
    const value = await loader();                       // the SINGLE database hit
    await redis.set(key, JSON.stringify(value), "EX", ttl + Math.floor(Math.random() * 60));
    return value;
  } finally {
    await redis.del(lockKey);                           // always release the lock
  }
}
```

### 9.6 Eviction policies (`maxmemory-policy`) **[I]**

**What it is.** When Redis reaches `maxmemory`, it must make room for new writes by **evicting** keys according to a configured **eviction policy**. This is fundamentally different from TTL expiry: TTL removes keys when *time* runs out; eviction removes keys when *memory* runs out. For a **cache**, you set `maxmemory` and pick an eviction policy so Redis automatically sheds the least valuable data. For a **datastore** (where every key matters), you may prefer `noeviction`, which makes writes **error** rather than silently dropping data.

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru     # evict least-recently-used across all keys
```

| Policy | Evicts | Use when |
|---|---|---|
| `noeviction` | nothing — writes return errors when full | Redis is a datastore, not a cache |
| `allkeys-lru` | least-recently-used, any key | general-purpose cache (most common) |
| `allkeys-lfu` | least-**frequently**-used, any key | cache with a stable hot set (often the best) |
| `volatile-lru` | LRU among keys that **have a TTL** | mixed cache + persistent keys in one instance |
| `volatile-lfu` | LFU among keys that have a TTL | same, frequency-based |
| `allkeys-random` | a random key | uniform access patterns, cheapest |
| `volatile-random` | random among keys with a TTL | rare |
| `volatile-ttl` | the shortest remaining TTL first | evict soon-to-die keys preferentially |

⚡ **Note:** **LFU** (frequency-based, since Redis 4) often **outperforms LRU** for caches because it resists "scan pollution" — a one-off bulk scan won't evict your genuinely hot keys just because they were touched less *recently*. Inspect access patterns with `OBJECT FREQ key` (in LFU mode) or `OBJECT IDLETIME key` (in LRU mode).

> **The most common cache misconfiguration:** leaving the default `noeviction` policy on a cache. When memory fills, writes start **failing** instead of evicting cold data — your "cache" becomes read-only and your app errors. Always set `maxmemory` + an `allkeys-*` policy on a cache (§18).

---

## 10. Pipelining, Performance & Connection Pooling

### 10.1 Pipelining — turn N round trips into one **[I]**

**The problem.** Each command normally costs one network **round trip** (RTT). Over a LAN that's maybe 0.2 ms; over the internet it can be 50 ms+. Sending 1,000 commands one at a time means waiting for 1,000 sequential RTTs — your throughput is capped by latency, not by Redis (which is idle most of that time).

**The fix.** **Pipelining** sends many commands **back-to-back without waiting** for each reply, then reads all the replies together — collapsing N RTTs into roughly **one**. (Recall from §1.6 that a request is just an array on the wire, so this is literally "send several arrays, then read several replies.") It is a pure **latency optimization** — *not* a transaction: the commands are **not atomic and not isolated** (other clients' commands can interleave). For bulk operations over a network it's commonly a **10–100× speedup**.

```redis
# In redis-cli you can pipe a file of commands in bulk:
cat commands.txt | redis-cli --pipe
```

```js
// ioredis — pipeline() batches commands; exec() returns [[err, result], ...] in order.
const pipe = redis.pipeline();
for (let i = 0; i < 1000; i++) pipe.set(`k:${i}`, i);
const results = await pipe.exec(); // ONE network round trip for all 1000 commands
// ioredis can also auto-pipeline concurrent commands when constructed with
// { enableAutoPipelining: true } — concurrent awaits get coalesced for you.
```

```go
// go-redis — Pipeline() then Exec(); or the Pipelined() closure helper:
pipe := rdb.Pipeline()
cmds := make([]*redis.StatusCmd, 1000)
for i := 0; i < 1000; i++ {
	cmds[i] = pipe.Set(ctx, fmt.Sprintf("k:%d", i), i, 0)
}
_, err := pipe.Exec(ctx) // single round trip; read each cmd's .Val()/.Err() afterward
_ = err
```

> **Pipeline vs MULTI/EXEC — don't confuse them:** **pipelining** = fewer round trips, **no** atomicity. **`MULTI`/`EXEC`** = atomicity/isolation (§6), but the commands still round-trip as a group. Use a **pipeline** for bulk *speed*, a **transaction or Lua** for atomic *groups*. (You can even pipeline a `MULTI/EXEC` to get both.)

### 10.2 Connection pooling **[I]**

Opening a fresh TCP connection per request is wasteful (handshake latency, syscall overhead, file-descriptor churn). Reuse connections via a **pool** — but the right approach differs by language because of their concurrency models:

- **go-redis:** `redis.NewClient` **is already a pool** (default `PoolSize = 10 × NumCPU`) and is safe for concurrent goroutines. **Share one client app-wide** and let it manage connections. Tune `PoolSize`, `MinIdleConns`, and `PoolTimeout` under load. (See **`GO_GUIDE.md`** §12 on goroutines for *why* one shared, pooled client is the idiomatic Go pattern.)
- **ioredis / node-redis:** Node is single-threaded with an event loop (see **`NODEJS_GUIDE.md`**), so a **single client multiplexes** many in-flight commands over **one** connection very efficiently — you don't need a pool of sockets for normal commands. **Share one client instance.** For **blocking** commands (`BLPOP`, `BRPOP`, `SUBSCRIBE`) or heavy parallel workloads, dedicate a *few extra* connections rather than one per request, because a blocking command ties up its connection.

```go
// go-redis — pool tuning for a high-concurrency service:
rdb := redis.NewClient(&redis.Options{
	Addr:         "localhost:6379",
	PoolSize:     50,               // max concurrent sockets
	MinIdleConns: 10,               // keep warm connections ready to avoid cold-start latency
	PoolTimeout:  4 * time.Second,  // how long a goroutine waits for a free conn before erroring
	ReadTimeout:  3 * time.Second,
})
```

### 10.3 Performance tips **[I/A]**

The throughline of every tip below is the single-threaded model: **anything O(N) on a big key, or anything that round-trips needlessly, hurts everyone.**

- **Pipeline bulk operations**, and use **`MGET`/`MSET`/`HMGET`** to fetch/set many values in one call.
- **Avoid O(N) commands on huge keys** — `KEYS`, `SMEMBERS`, `HGETALL`, `LRANGE 0 -1` on giant collections block the single thread for the whole scan. Use `SCAN`/`HSCAN`/`SSCAN`/`ZSCAN` or `ZRANGE` with `LIMIT`.
- **Hunt for "big keys" and "hot keys."** A single 1 GB list, or one key taking 90% of traffic, causes latency spikes and (in Cluster) uneven load. Find them with `redis-cli --bigkeys` and `--hotkeys`.
- **Keep values reasonably sized** — prefer many small keys, or a hash you can `HMGET`, over one enormous value you only ever read part of.
- **Disable Transparent Huge Pages** on Linux (`madvise`/`never`) — THP causes latency spikes during the `fork()` used by RDB/AOF (§8).
- **Use `UNLINK`, not `DEL`,** for large keys (background free instead of blocking the thread).
- **Measure, don't guess** — `redis-cli --latency` / `--latency-history`, `LATENCY DOCTOR`, and `SLOWLOG` (§16).

---

## 11. Distributed Locking & Rate Limiting

### 11.1 A correct single-instance lock: `SET NX PX` + a token **[A]**

**What a distributed lock is for.** Sometimes only one process across your whole fleet should do a thing at a time — process one job, run one cron, mutate one resource. A Redis lock gives you that mutual exclusion across machines.

**The correct pattern, and *why* each part exists:**

1. `SET key token NX PX ttl` — `NX` ensures only one acquirer wins the race; `PX ttl` sets an **auto-expiry** so a crashed holder doesn't deadlock the resource forever. Both are essential.
2. The value is a **random token unique to this acquirer** — so that when you release, you can verify you still own the lock and you're not deleting a lock that *already expired and was re-acquired by someone else*.
3. **Release must be atomic** (a Lua script): a naive "GET to check ownership, then DEL" has a race — the lock could expire and be re-taken by another process *between* your GET and your DEL, and you'd delete their lock. The Lua script does the check-and-delete as one atomic step.

```js
// ioredis — acquire/release a lock SAFELY.
import { randomUUID } from "crypto";

async function acquireLock(resource, ttlMs = 10000) {
  const token = randomUUID();
  // NX = only if not exists; PX = expiry in ms. Returns "OK" on success, null on failure.
  const ok = await redis.set(`lock:${resource}`, token, "PX", ttlMs, "NX");
  return ok ? token : null;
}

// Release ONLY if we still own it — atomic compare-and-delete via Lua (NO check-then-del race):
const RELEASE = `
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  else return 0 end`;

async function releaseLock(resource, token) {
  return redis.eval(RELEASE, 1, `lock:${resource}`, token); // 1 = freed, 0 = wasn't ours
}

// Usage:
const token = await acquireLock("order:42");
if (token) {
  try { /* critical section — keep it shorter than the TTL! */ }
  finally { await releaseLock("order:42", token); }
}
```

```go
// go-redis — same pattern, token-based release via a script:
func acquireLock(ctx context.Context, rdb *redis.Client, res string, ttl time.Duration) (string, bool) {
	token := uuid.NewString()
	ok, _ := rdb.SetNX(ctx, "lock:"+res, token, ttl).Result()
	return token, ok
}

var releaseScript = redis.NewScript(`
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1]) else return 0 end`)

func releaseLock(ctx context.Context, rdb *redis.Client, res, token string) error {
	return releaseScript.Run(ctx, rdb, []string{"lock:" + res}, token).Err()
}
```

> **Best practice:** keep the critical section **shorter than the TTL**, or extend the lock periodically (a "watchdog"). If your work outlives the TTL, the lock expires mid-work and a second holder appears — exactly what you were trying to prevent.

### 11.2 Redlock — multi-node locking, and its serious caveats **[A]**

**Why a single-instance lock isn't always enough.** The lock above lives on **one** Redis. If that instance fails or fails over to a replica, the lock can be **lost** — replication is asynchronous (§13), so the new primary may not yet have the lock key, and a second client can acquire it. For locking that must survive a node failure, the **Redlock** algorithm acquires a lock across **N independent Redis masters** (with *no* replication between them): you try to lock a **majority** (N/2 + 1) within a small time bound; if you obtain the majority before the TTL has mostly elapsed, you consider the lock held. Libraries: `redlock` (Node), `github.com/go-redsync/redsync/v4` (Go).

```js
// npm install redlock — acquire across several INDEPENDENT redis nodes:
import Redlock from "redlock";
import Redis from "ioredis";

const redlock = new Redlock(
  [new Redis("redis://a:6379"), new Redis("redis://b:6379"), new Redis("redis://c:6379")],
  { retryCount: 3, retryDelay: 200 }
);

// using() auto-extends the lock while your work runs and releases it at the end:
await redlock.using(["resource:order:42"], 5000, async (signal) => {
  // critical section; check signal.aborted on long work for safety
  if (signal.aborted) throw signal.error;
  await doWork();
});
```

> **⚠ Redlock caveats — read this before betting anything important on it.** Redlock is **controversial**. Martin Kleppmann's well-known critique argues it is unsafe under clock skew, GC/stop-the-world pauses, and network delays: a process can be paused right after acquiring the lock, the TTL expires, another process acquires it, and then the *first* process wakes up still *believing* it holds the lock — now there are **two holders**. The standard mitigation is a **fencing token**: a monotonically increasing number the lock service hands out, which the *protected resource* checks and uses to **reject stale writers**. Redis does **not** provide fencing tokens natively. **Bottom line:** for **efficiency** (avoid duplicate work, "mostly" mutual exclusion) Redis locks are fine and widely used. For **correctness** (must *never* have two holders — e.g. money movement) prefer a system built on consensus/linearizability (ZooKeeper, etcd) **with fencing tokens**. Don't bet financial correctness on Redlock.

### 11.3 Rate limiting **[A]**

Rate limiting protects your service (and downstreams) from abuse and overload. There are three classic algorithms with different precision/cost trade-offs.

#### Fixed window — simple and cheap, but bursty at boundaries

This is the §7 Lua example: `INCR` a per-window key, set its TTL on the first hit, reject above the limit. The **boundary problem**: a client can fire `limit` requests at the very *end* of one window and `limit` more at the *start* of the next — momentarily doubling the intended rate across the boundary.

#### Sliding-window log with a sorted set — precise

**How it works.** Store each request's **timestamp as a ZSET member** (score = timestamp). On each request: drop entries older than the window (`ZREMRANGEBYSCORE`), count what remains (`ZCARD`), and allow only if under the limit. This is **accurate** (a true rolling window, no boundary burst) at the cost of slightly more memory (one ZSET entry per recent request). Doing it in one Lua script keeps the read-decide-write atomic.

```js
// ioredis — sliding-window rate limiter via a ZSET, atomic in one Lua script.
redis.defineCommand("slidingLimit", {
  numberOfKeys: 1,
  lua: `
    local key    = KEYS[1]
    local now    = tonumber(ARGV[1])   -- current time (ms)
    local window = tonumber(ARGV[2])   -- window size (ms)
    local limit  = tonumber(ARGV[3])   -- max requests per window
    -- 1) drop timestamps older than the window (slide the window forward):
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    -- 2) count requests currently inside the window:
    local count = redis.call('ZCARD', key)
    if count >= limit then return 0 end          -- over the limit → reject
    -- 3) record this request and bound the key's lifetime so idle keys self-clean:
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('PEXPIRE', key, window)
    return 1                                       -- allowed
  `,
});

const ok = await redis.slidingLimit(`rl:${userId}`, Date.now(), 60000, 100); // 100 / minute
```

#### Token bucket — smooth, allows controlled bursts

**How it works.** A bucket holds up to `capacity` tokens and **refills at a steady rate**; each request spends one token (or `cost`). If tokens remain, allow; otherwise reject. This permits short **bursts** (spend the accumulated tokens) while enforcing a long-run average rate — the model most public APIs use. State is a hash `{tokens, ts}`, refilled **lazily** in Lua based on elapsed time (so there's no background refill job).

```lua
-- token_bucket.lua  KEYS[1]=bucket  ARGV: rate(tokens/sec) capacity now(sec) cost
local rate     = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local cost     = tonumber(ARGV[4])

local data   = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or capacity   -- a brand-new bucket starts full
local ts     = tonumber(data[2]) or now

-- refill based on time elapsed since last seen, capped at capacity:
local elapsed = math.max(0, now - ts)
tokens = math.min(capacity, tokens + elapsed * rate)

local allowed = tokens >= cost
if allowed then tokens = tokens - cost end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], math.ceil(capacity / rate) * 2)  -- self-clean idle buckets
return allowed and 1 or 0
```

```go
// go-redis — load the bucket script once and call it per request:
var bucket = redis.NewScript(`...token_bucket.lua contents...`)

func allowBucket(ctx context.Context, rdb *redis.Client, key string) (bool, error) {
	now := float64(time.Now().UnixMilli()) / 1000.0
	n, err := bucket.Run(ctx, rdb, []string{key},
		10,   // rate: 10 tokens/sec
		20,   // capacity: burst up to 20
		now,  // current time (seconds)
		1,    // cost per request
	).Int()
	return n == 1, err
}
```

---

## 12. Queues — Lists vs Pub/Sub vs Streams

Redis offers three ways to move work between processes, and choosing the wrong one is a common, painful mistake (e.g. using Pub/Sub for jobs that must not be lost). Here is the decision laid out explicitly.

| Approach | Delivery | Durable? | Multiple consumers | Use when |
|---|---|---|---|---|
| **List** (`LPUSH`+`BRPOP`) | one consumer per message | yes (persists) | competing (one gets each) | simple work queue |
| **List** (`LMOVE`/`BLMOVE`) | reliable handoff via a processing list | yes | competing | at-least-once via an "in-flight" list |
| **Pub/Sub** | broadcast (all subscribers) | **no** | fan-out | live notifications, ephemeral |
| **Stream + group** | one consumer per message, **acked** | yes | competing + replay | durable jobs, at-least-once, retries |

**Decision guide:**
- Need fan-out to all listeners, and it's OK to lose messages → **Pub/Sub** (§4).
- Simple background jobs, one worker per message, light requirements → **List queue** (`BRPOP`).
- Need acknowledgement, retries, multiple workers, replay, and crash recovery → **Streams + consumer groups** (§5) — the right answer for most real job queues today.

### A reliable list-based job queue (with in-flight recovery) **[I/A]**

A plain `BRPOP` queue has a gap: if a worker pops a job and then crashes before finishing, the job is **lost** (it's no longer in the list and was never completed). The fix is `LMOVE`/`BLMOVE` into a per-worker **processing list**, so an in-flight job is still recorded somewhere and a reaper can requeue it.

```js
// ioredis — producer:
await redis.lpush("jobs:pending", JSON.stringify({ id: 1, task: "email" }));

// Consumer using BLMOVE so an in-flight item survives a worker crash:
async function consume() {
  while (true) {
    // Atomically move a job from "pending" → a "processing" list (it's now recorded as in-flight).
    const job = await redis.blmove("jobs:pending", "jobs:processing", "RIGHT", "LEFT", 5);
    if (!job) continue;
    try {
      await handle(JSON.parse(job));
      await redis.lrem("jobs:processing", 1, job); // done → remove from the in-flight list
    } catch (e) {
      // leave it in "jobs:processing"; a separate reaper requeues stale items.
    }
  }
}
// A reaper periodically moves long-stuck items from jobs:processing back to jobs:pending.
```

> **Practical advice:** for most teams, prefer a **battle-tested library** built on these primitives rather than hand-rolling — they give you retries, scheduling, dead-letter queues, and dashboards for free: **BullMQ** (Node, Streams-based), **Asynq** or **River** (Go), **Sidekiq** (Ruby), **Celery + Redis** (Python).

---

## 13. High Availability — Replication, Sentinel, Cluster

A single Redis is a **single point of failure** and is bounded by **one machine's RAM**. Three building blocks address availability and scale, and they layer on top of each other: replication is the foundation, Sentinel automates failover over replication, and Cluster adds sharding *plus* failover.

### 13.1 Replication (primary/replica) **[A]**

**What it is.** A **replica** asynchronously copies a **primary's** dataset. It can serve **reads** (offloading the primary) and acts as a **hot standby** ready to be promoted. The critical property to internalize: **replication is asynchronous.** The primary acknowledges a write to your client *before* the replica has it, so a replica may **lag**, reads from a replica can be **stale**, and a failover can **lose the last few writes** that hadn't replicated yet. This is the fundamental durability/availability trade-off Redis makes for speed.

```conf
# On the replica's config (or set at runtime with REPLICAOF):
replicaof 192.168.1.10 6379
replica-read-only yes
```

```redis
REPLICAOF 192.168.1.10 6379   # start replicating from this primary (at runtime)
REPLICAOF NO ONE              # promote this replica to a standalone primary
INFO replication              # role, connected replicas, offsets, lag
WAIT 1 100                    # block until ≥1 replica has acked, or 100ms passes (tunable durability)
```

> **`WAIT` for stronger durability:** after a critical write, `WAIT numreplicas timeout` blocks until that many replicas confirm they have it (or the timeout). It makes a write *more* durable (it won't be lost if the primary dies, as long as a confirming replica survives) at the cost of latency. It is **not** full synchronous replication, but it's a useful knob for important writes.

### 13.2 Sentinel — automatic failover for a primary/replica setup **[A]**

**What it is.** **Redis Sentinel** is a separate monitoring process (run **3 or more** so they form a quorum). Sentinels **monitor** the primary and replicas, **detect** when the primary is down, **elect** a new primary from the replicas, **reconfigure** the other replicas to follow it, and **notify clients** of the change. Crucially, clients connect to the *Sentinels* to discover the **current** primary address — which changes after a failover — so the app keeps working without manual intervention. Sentinel provides **HA for a single logical dataset**; it does **not** shard data.

```conf
# sentinel.conf — monitor a master named "mymaster"; a quorum of 2 sentinels must agree to act.
port 26379
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel down-after-milliseconds mymaster 5000    # consider it down after 5s unreachable
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

```bash
redis-sentinel /etc/redis/sentinel.conf
# or equivalently: redis-server /etc/redis/sentinel.conf --sentinel
```

```js
// ioredis — connect VIA the sentinels; the client auto-discovers and FOLLOWS the current master.
const redis = new Redis({
  sentinels: [
    { host: "sentinel-1", port: 26379 },
    { host: "sentinel-2", port: 26379 },
    { host: "sentinel-3", port: 26379 },
  ],
  name: "mymaster",      // the monitored master's name (must match sentinel.conf)
  role: "master",        // or "slave" to read from replicas
});
```

```go
// go-redis — the Sentinel-aware "failover" client:
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
	MasterName:    "mymaster",
	SentinelAddrs: []string{"sentinel-1:26379", "sentinel-2:26379", "sentinel-3:26379"},
})
```

### 13.3 Redis Cluster — horizontal scale + HA via sharding **[A]**

**What it is.** **Cluster** splits the keyspace into **16,384 hash slots** distributed across multiple **primaries** (each typically backed by replicas for HA). A key's slot is `CRC16(key) mod 16384`; a cluster-aware client routes each command to the node that owns that key's slot. This is how you scale **memory and throughput beyond a single machine** *and* get automatic failover (a replica is promoted when its primary dies). The trade-offs are real and shape how you design keys:

- **Multi-key operations only work when all keys are in the same slot.** `MGET`, transactions, and Lua scripts that touch multiple keys fail with `CROSSSLOT` unless those keys share a `{hashtag}` (see below).
- **No numbered logical DBs** (only DB 0).
- **Some commands behave differently** or are restricted.

```redis
# Create a 3-primary, 3-replica cluster from 6 already-running nodes:
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1

CLUSTER INFO          # cluster_state (ok/fail), slots assigned/ok
CLUSTER NODES         # node ids, roles, and the slot ranges each owns
CLUSTER SLOTS         # slot → node mapping
CLUSTER KEYSLOT user:1000   # which slot a given key hashes to
CLUSTER SHARDS        # shard topology (Redis 7.0+)

# Resharding (move slots between nodes) — online, no downtime:
redis-cli --cluster reshard 127.0.0.1:7000
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000   # add a node, then reshard onto it
redis-cli --cluster rebalance 127.0.0.1:7000
```

**Hash tags** force related keys onto the same slot so multi-key commands / transactions / Lua work. Only the substring inside the **first `{...}`** is hashed:

```redis
# Both keys hash by the part inside {…} → same slot → MGET/MSET/EXEC across them is allowed:
MSET user:{1000}:profile "..." user:{1000}:sessions "..."
```

```js
// ioredis — Cluster client (give it a few seed nodes; it discovers the full topology):
import { Cluster } from "ioredis";
const cluster = new Cluster([
  { host: "127.0.0.1", port: 7000 },
  { host: "127.0.0.1", port: 7001 },
]);
await cluster.set("user:{1000}:name", "Ada"); // routed to the slot/node owning that key
```

```go
// go-redis — Cluster client:
rdb := redis.NewClusterClient(&redis.ClusterOptions{
	Addrs: []string{"127.0.0.1:7000", "127.0.0.1:7001", "127.0.0.1:7002"},
})
```

**Client-side considerations:** a good cluster client transparently handles **MOVED** redirections (a slot moved permanently → update the local slot map) and **ASK** redirections (a slot is mid-migration → redirect just this one request). Reads can be served from replicas with `READONLY`. Expect occasional **CROSSSLOT** errors when a multi-key op spans slots without a shared hash tag — design your keys with hash tags up front for the multi-key operations you know you'll need.

| Tool | Solves | Sharding | Auto-failover |
|---|---|---|---|
| Replication | read scaling, hot standby | no | no (manual) |
| Sentinel | HA for one dataset | no | yes |
| Cluster | scale + HA | yes (16384 slots) | yes |

---

## 14. Security — ACLs, AUTH, TLS

**The golden rule:** Redis is **fast and trusting by default** — it assumes it's on a private, trusted network. It is **never** safe to expose a raw Redis to the internet. Lock it down in **layers** (defense in depth): network isolation, authentication, least-privilege authorization, encryption in transit, and command restriction.

### 14.1 Protected mode & binding **[I]**

The first layer is **network**: make Redis listen only where it should.

```conf
bind 127.0.0.1 ::1        # listen ONLY on loopback unless you deliberately need otherwise
protected-mode yes        # refuse external connections when there's no auth/bind configured
port 6379                 # consider `port 0` to disable the plaintext port and use TLS only
```

> **⚠ The classic breach (this is how most Redis instances get pwned):** a Redis bound to `0.0.0.0` with **no password** is trivially hijacked — attackers connect and use `CONFIG SET` to write SSH `authorized_keys` or cron jobs onto the host, achieving remote code execution. **Always** bind to private interfaces, require authentication, and put Redis behind a firewall/security group. Never put it on a public IP.

### 14.2 AUTH & the legacy password **[I]**

```conf
requirepass "a-long-random-password"    # a single shared password (legacy mechanism)
```

```redis
AUTH a-long-random-password         # authenticate with the legacy single password
AUTH alice herpassword              # authenticate as an ACL user (preferred — see below)
```

> **Security:** avoid passing the password on the `redis-cli -a` command line (it leaks into `ps` output and shell history). Use the `REDISCLI_AUTH` environment variable or an interactive `AUTH`. Make the password long and random — Redis is fast enough that weak passwords are brute-forceable.

### 14.3 ACLs (Redis 6+) — fine-grained, least-privilege users **[I/A]**

**Why ACLs replace the single password.** A single `requirepass` is all-or-nothing: anyone who has it can run *every* command on *every* key, including `FLUSHALL` and `CONFIG`. **ACLs** let you define **named users**, each restricted to specific **commands**, **key patterns**, and **pub/sub channels** — the principle of **least privilege**. Your cache reader shouldn't be able to `FLUSHALL`; your queue worker shouldn't be able to read other apps' keys.

```redis
# A read-only cache user, limited to keys under cache:* and read commands only:
ACL SETUSER cacheuser on >s3cret ~cache:* +@read +get +mget -@dangerous

# A worker that may use only the jobs:* keys and stream commands:
ACL SETUSER worker on >wpass ~jobs:* +@stream +@read +@write

# Inspect & manage:
ACL LIST                 # all configured rules
ACL WHOAMI               # the current connection's user
ACL CAT                  # list command categories (@read, @write, @admin, @dangerous, ...)
ACL GETUSER cacheuser    # one user's full permission set
ACL DELUSER cacheuser
ACL SAVE                 # persist ACLs to the aclfile (if configured)
```

```conf
# Persist ACLs in a dedicated file instead of inline in redis.conf:
aclfile /etc/redis/users.acl
```

**ACL syntax cheatsheet:** `on`/`off` (enable/disable the user), `>pass` / `<pass` (add/remove a password), `~pattern` (allow these keys), `&channel` (allow these pub/sub channels), `+cmd`/`-cmd` (allow/deny a command), `+@category`/`-@category` (allow/deny a whole command group), `allcommands`/`nocommands`, `allkeys`/`resetkeys`.

```js
// Clients authenticate as an ACL user with a username + password:
const redis = new Redis({ username: "cacheuser", password: "s3cret" });
```

```go
rdb := redis.NewClient(&redis.Options{
	Addr: "localhost:6379", Username: "cacheuser", Password: "s3cret",
})
```

### 14.4 TLS (encryption in transit) **[I/A]**

**Why.** AUTH proves *who* you are but does nothing to stop a network eavesdropper from reading your data and credentials off the wire. **TLS** encrypts the connection. For any traffic that leaves a trusted host (cross-VPC, cross-datacenter, or over the internet), TLS — ideally **mutual TLS**, where the client also presents a certificate — is mandatory.

```conf
tls-port 6379
port 0                                  # disable the non-TLS port so nothing is sent in plaintext
tls-cert-file /etc/redis/redis.crt
tls-key-file  /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes                    # REQUIRE client certificates (mutual TLS)
```

```bash
redis-cli --tls --cert client.crt --key client.key --cacert ca.crt -h myhost
```

```js
const redis = new Redis({ host: "myhost", port: 6379, tls: { /* ca, cert, key */ } });
```

### 14.5 Command renaming / disabling (defense in depth) **[I]**

Even with ACLs, you can **hide or rename dangerous commands** at the server level so a compromised or buggy client can't `FLUSHALL`, `CONFIG`, `DEBUG`, etc. (ACLs with `-@dangerous`/`-flushall` are the modern, preferred mechanism, but command renaming is a useful extra layer.)

```conf
rename-command FLUSHALL ""                 # disable the command entirely
rename-command CONFIG "CONFIG_a8f3k2"      # obscure it behind an unguessable name
```

**Security checklist:** bind to private NICs; require auth (`requirepass` or, better, ACL users); use TLS for any remote traffic; drop/deny `FLUSHALL`/`CONFIG`/`DEBUG`/`KEYS` for application users; run Redis as a non-root user; keep it patched; never put it on a public IP; firewall it to only the hosts that need it. (For the application side — hashing credentials, JWTs — see the library's `GO_JWT_ARGON2_GUIDE.md` and `BETTERAUTH_GUIDE.md`.)

---

## 15. Vector Search & Redis as a Vector DB

### 15.1 Why Redis for vectors **[A]**

In 2026, Redis is a popular **vector database** for AI/RAG applications. The idea: store the **embedding** (a fixed-length float vector produced by an ML model from text/images) alongside your data, then run **k-nearest-neighbor (KNN)** similarity search to find the items whose vectors are closest to a query vector — i.e. semantically similar. The compelling reason to use Redis specifically is **co-location**: you can keep your cache, your application data, *and* your vectors in the **same low-latency store**, with **metadata pre-filtering** (filter by tags/numbers first, then do KNN among the matches). This is powered by **RediSearch** (the `FT.*` commands).

⚡ **Version note:** Vector search needs the **RediSearch** module. On **Redis 8** it's **in core** (`redis:8`). On Redis 7.x use the **Redis Stack** image. **Valkey** doesn't bundle RediSearch — use a Valkey search module or a dedicated vector DB. Redis supports **FLAT** (exact brute-force) and **HNSW** (approximate, much faster at scale) index types; distance metrics **L2 / IP / COSINE**; and pre-filtering by tag/numeric fields.

### 15.2 Create an index and store vectors **[A]**

```redis
# Create an index over HASH keys prefixed "doc:". The vector field uses HNSW,
# 3 dimensions (use your model's REAL dim — e.g. 1536 for OpenAI text-embedding-3-small),
# FLOAT32 elements, and COSINE distance. We also index a TAG field for pre-filtering.
FT.CREATE idx:docs
  ON HASH PREFIX 1 doc:
  SCHEMA
    title TEXT
    category TAG
    embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 3 DISTANCE_METRIC COSINE

# Store documents. The vector must be the raw FLOAT32 bytes (your client packs floats → bytes).
HSET doc:1 title "Intro to Redis" category "db" embedding "<float32-bytes>"
HSET doc:2 title "AI with Redis"   category "ai" embedding "<float32-bytes>"

# KNN query: top 3 nearest to a query vector. The distance is returned as the alias vec_score.
# $vec is a query PARAMETER holding the query embedding's raw bytes.
FT.SEARCH idx:docs "*=>[KNN 3 @embedding $vec AS vec_score]"
  PARAMS 2 vec "<float32-bytes>"
  SORTBY vec_score
  RETURN 2 title vec_score
  DIALECT 2

# Hybrid search: pre-filter to category=ai FIRST, then KNN among only those (the RAG sweet spot):
FT.SEARCH idx:docs "(@category:{ai})=>[KNN 3 @embedding $vec AS vec_score]"
  PARAMS 2 vec "<float32-bytes>" DIALECT 2
```

### 15.3 Node.js (node-redis v4+) — real packing & querying **[A]**

```js
// node-redis has first-class FT.* helpers. Pack JS numbers into a raw Float32 byte Buffer.
import { createClient, SchemaFieldTypes, VectorAlgorithms } from "redis";

const client = createClient();
await client.connect();

// Helper: turn an embedding array into raw FLOAT32 little-endian bytes (the wire format):
const toBytes = (arr) => Buffer.from(new Float32Array(arr).buffer);

// 1) Create the index (idempotent: ignore the "Index already exists" error on re-run).
try {
  await client.ft.create("idx:docs", {
    title: SchemaFieldTypes.TEXT,
    category: SchemaFieldTypes.TAG,
    embedding: {
      type: SchemaFieldTypes.VECTOR,
      ALGORITHM: VectorAlgorithms.HNSW,
      TYPE: "FLOAT32",
      DIM: 3,
      DISTANCE_METRIC: "COSINE",
    },
  }, { ON: "HASH", PREFIX: "doc:" });
} catch (e) { if (!String(e.message).includes("Index already exists")) throw e; }

// 2) Insert documents with their embeddings:
await client.hSet("doc:1", { title: "Intro to Redis", category: "db", embedding: toBytes([0.1, 0.2, 0.3]) });
await client.hSet("doc:2", { title: "AI with Redis",  category: "ai", embedding: toBytes([0.9, 0.8, 0.7]) });

// 3) KNN search for the 3 nearest neighbors to a query embedding:
const queryVec = toBytes([0.85, 0.82, 0.75]);
const results = await client.ft.search(
  "idx:docs",
  "*=>[KNN 3 @embedding $vec AS score]",
  {
    PARAMS: { vec: queryVec },
    SORTBY: "score",
    DIALECT: 2,
    RETURN: ["title", "score"],
  }
);
console.log(results.documents); // [{ id, value: { title, score } }, ...]
```

```go
// go-redis — pack floats to bytes and run FT.SEARCH via Do (or the FT* helpers in v9):
import (
	"encoding/binary"
	"math"
)

func floatsToBytes(fs []float32) []byte {
	b := make([]byte, 4*len(fs))
	for i, f := range fs {
		binary.LittleEndian.PutUint32(b[i*4:], math.Float32bits(f))
	}
	return b
}

// Create the index:
rdb.Do(ctx, "FT.CREATE", "idx:docs", "ON", "HASH", "PREFIX", "1", "doc:",
	"SCHEMA", "title", "TEXT", "category", "TAG",
	"embedding", "VECTOR", "HNSW", "6", "TYPE", "FLOAT32", "DIM", "3", "DISTANCE_METRIC", "COSINE")

// Insert:
rdb.HSet(ctx, "doc:1", "title", "Intro to Redis", "category", "db",
	"embedding", floatsToBytes([]float32{0.1, 0.2, 0.3}))

// KNN search:
q := floatsToBytes([]float32{0.85, 0.82, 0.75})
res, _ := rdb.Do(ctx, "FT.SEARCH", "idx:docs",
	"*=>[KNN 3 @embedding $vec AS score]",
	"PARAMS", "2", "vec", q, "SORTBY", "score", "DIALECT", "2").Result()
_ = res
```

| Command | Purpose |
|---|---|
| `FT.CREATE` | Define an index (TEXT/TAG/NUMERIC/GEO/VECTOR fields) |
| `FT.SEARCH` | Query: full-text, filters, and `[KNN k @vec $q]` vector search |
| `FT.AGGREGATE` | Group/aggregate over search results |
| `FT.INFO` / `FT.DROPINDEX` / `FT._LIST` | Inspect / drop / list indexes |

> **When to use Redis for vectors:** great when you want **one system** for cache + app data + vectors, **low latency**, and **metadata pre-filtering** for RAG. For **billions** of vectors or specialized ANN features you might still pick a dedicated vector DB — but for most RAG/semantic-search workloads at small-to-medium scale, Redis is an excellent, simple choice you may already be running.

---

## 16. Observability & Operations

You can't operate what you can't see. Redis ships a rich set of introspection commands; learning them turns "Redis feels slow" into a specific, fixable diagnosis.

### 16.1 `INFO` — the dashboard in one command **[I]**

```redis
INFO                 # everything, grouped into sections
INFO server          # version, uptime, OS, process id
INFO clients         # connected_clients, blocked_clients, maxclients
INFO memory          # used_memory, maxmemory, fragmentation_ratio, evicted
INFO stats           # ops/sec, keyspace hits/misses, expired/evicted keys
INFO replication     # role, replicas, offsets, lag
INFO persistence     # rdb/aof state, last save, whether loading
INFO keyspace        # per-DB key counts and average TTL
```

**The health metrics that actually matter, and what they tell you:**

- **Hit ratio:** `keyspace_hits / (keyspace_hits + keyspace_misses)`. A low ratio means the cache isn't helping — wrong TTLs, wrong keys cached, or too small.
- **`used_memory` vs `maxmemory`** and **`evicted_keys`:** rising evictions mean you're under-provisioned or using the wrong eviction policy (§9).
- **`mem_fragmentation_ratio`:** well above 1.5 may indicate fragmentation (consider `activedefrag yes`).
- **`connected_clients` / `blocked_clients`:** a steadily climbing count signals a connection leak or stuck blocking commands.
- **`instantaneous_ops_per_sec`** and replication **`master_repl_offset`** lag for throughput and replica health.

### 16.2 SLOWLOG — find the slow commands **[I]**

```redis
CONFIG SET slowlog-log-slower-than 10000   # log commands slower than 10ms (the unit is microseconds)
CONFIG SET slowlog-max-len 128             # keep the last 128 slow entries
SLOWLOG GET 10                              # the last 10 slow entries (command, args, µs, timestamp)
SLOWLOG LEN
SLOWLOG RESET
```

When a single-threaded server has a latency problem, it's almost always a slow command blocking the thread (an O(N) `KEYS`/`SMEMBERS`/`HGETALL` on a big key, or a heavy Lua script). SLOWLOG points right at it.

### 16.3 MONITOR — see every command live (debug only) **[I]**

```redis
MONITOR     # streams EVERY command the server processes — ⚠ a HUGE perf hit; NEVER leave it on in prod
```

### 16.4 LATENCY — latency-spike diagnostics **[I/A]**

```redis
CONFIG SET latency-monitor-threshold 100   # record latency events ≥100ms
LATENCY LATEST       # most recent latency spike per event type
LATENCY HISTORY fork # history for a specific event (e.g. fork, expire, command)
LATENCY DOCTOR       # human-readable analysis AND advice — start here
LATENCY RESET
```

### 16.5 Memory analysis **[I]**

```redis
MEMORY USAGE user:1            # bytes a key consumes, including overhead
MEMORY DOCTOR                  # advice on memory issues
MEMORY STATS                   # detailed breakdown by category
DEBUG OBJECT user:1            # encoding, serialized length, etc. (debug)
```

```bash
# Sampling tools straight from the CLI:
redis-cli --bigkeys     # the biggest key of each type
redis-cli --memkeys     # keys ranked by memory usage
redis-cli --hotkeys     # most-accessed keys (requires an LFU eviction policy)
redis-cli --latency-history   # rolling latency over time
redis-cli INFO stats | grep -E 'hits|misses|evicted|expired'
```

### 16.6 Keyspace notifications — react to key events **[I/A]**

Redis can **publish an event** when a key changes or expires, so your app can react — cache-invalidation hooks, cleanup when a session key expires, etc.

```redis
CONFIG SET notify-keyspace-events KEA      # enable ALL key-event notifications (K=keyspace, E=keyevent, A=all)
# Then subscribe to the event channels:
PSUBSCRIBE '__keyevent@0__:expired'        # fires when a key in DB 0 expires
PSUBSCRIBE '__keyspace@0__:user:1'         # all events on the key user:1
```

> **Caveat:** keyspace notifications are delivered over **Pub/Sub**, so they inherit its at-most-once, no-persistence semantics (§4) — if your subscriber is down when a key expires, you miss the event. For guaranteed delivery, drive the side effect through a Stream instead.

---

## 17. Real-World Recipes

These tie the pieces together into things you'll actually ship. Each one references the patterns it builds on.

### 17.1 Session store (Express + connect-redis) **[I]**

Sessions in Redis are shared across all app instances and auto-expire — the standard production setup. (For the Node/Express side, see **`NODEJS_GUIDE.md`**.)

```js
// npm install express express-session connect-redis ioredis
import session from "express-session";
import { RedisStore } from "connect-redis";
import Redis from "ioredis";

const redis = new Redis();
app.use(session({
  store: new RedisStore({ client: redis, prefix: "sess:", ttl: 86400 }), // 1-day TTL
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, maxAge: 86400_000 },
}));
// Sessions now live in Redis (sess:<id>), shared across every app instance, auto-expiring.
```

### 17.2 Leaderboard with rank + neighbors (ZSET) **[I]**

```js
// Submit a score, then read a player's rank plus the 2 players above/below them (§2.4).
async function submitScore(game, player, score) {
  // GT = only ever RAISE the score, never lower it on a worse run:
  await redis.zadd(`lb:${game}`, "GT", score, player);
}

async function aroundPlayer(game, player, window = 2) {
  const key = `lb:${game}`;
  const rank = await redis.zrevrank(key, player);     // 0 = top
  if (rank === null) return null;
  const start = Math.max(0, rank - window);
  const end = rank + window;
  const rows = await redis.zrange(key, start, end, "REV", "WITHSCORES");
  // Pair up the flat [member, score, member, score, ...] and attach absolute ranks:
  const out = [];
  for (let i = 0; i < rows.length; i += 2) {
    out.push({ rank: start + i / 2 + 1, player: rows[i], score: Number(rows[i + 1]) });
  }
  return out;
}
```

### 17.3 Per-IP rate limiter middleware (Express) **[I]**

```js
// Reuse the sliding-window Lua command from §11.
function rateLimit({ limit = 100, windowMs = 60000 } = {}) {
  return async (req, res, next) => {
    const key = `rl:${req.ip}`;
    const ok = await redis.slidingLimit(key, Date.now(), windowMs, limit);
    if (!ok) return res.status(429).json({ error: "Too Many Requests" });
    next();
  };
}
app.use("/api/", rateLimit({ limit: 60, windowMs: 60000 }));
```

### 17.4 Response cache with single-flight (Express) **[I/A]**

```js
// Cache GET responses for 60s, preventing stampedes via the §9 getWithLock helper.
function cacheGet(ttl = 60) {
  return async (req, res, next) => {
    if (req.method !== "GET") return next();
    const key = `httpcache:${req.originalUrl}`;
    try {
      const body = await getWithLock(key, async () => {
        // The "loader" produces fresh payload only on the single miss that wins the lock:
        return await fetchFreshPayloadFor(req); // your data-producing function
      }, ttl);
      res.set("X-Cache", "HIT-OR-FILLED").json(body);
    } catch (e) { next(e); }
  };
}
```

### 17.5 Real-time feed / fan-out (Streams + WebSocket) **[A]**

```js
// Producer: append events to a per-topic stream (capped so it can't grow unbounded).
await redis.xadd("feed:global", "MAXLEN", "~", 1000, "*", "type", "post", "id", postId);

// Consumer: tail the stream and push each new event to connected WebSocket clients.
let lastId = "$"; // start from "now" (only NEW events)
async function pump(wss) {
  while (true) {
    const res = await redis.xread("BLOCK", 0, "COUNT", 50, "STREAMS", "feed:global", lastId);
    if (!res) continue;
    const [, entries] = res[0];
    for (const [id, fields] of entries) {
      lastId = id;                                    // advance the cursor
      const msg = JSON.stringify(Object.fromEntries(chunk(fields, 2)));
      wss.clients.forEach((c) => c.readyState === 1 && c.send(msg));
    }
  }
}
const chunk = (a, n) => a.reduce((r, _, i) => (i % n ? r : [...r, a.slice(i, i + n)]), []);
// (For the WebSocket server side in Go, see GO_GORILLA_WEBSOCKETS_GUIDE.md.)
```

---

## 18. Gotchas & Best Practices

A consolidated checklist. Most of these trace back to one of three root causes: the **single-threaded model** (O(N) commands block everyone), the **in-memory model** (unbounded keys = memory leak), or the **fast-and-trusting default** (insecure out of the box).

**Do:**
- ✅ Set a **`maxmemory`** + an **eviction policy** on caches (the default `noeviction` makes writes *fail* when full).
- ✅ Always give cache keys a **TTL** — an unbounded keyspace is a silent memory leak.
- ✅ Use **`SCAN`**, never `KEYS`, in production. Use **`UNLINK`** for big deletes.
- ✅ Adopt a **key naming convention** (`app:type:id:field`) and use **hash tags** for related keys in Cluster.
- ✅ **Pipeline** bulk ops; use **`MGET`/`MSET`/`HMGET`** to cut round trips.
- ✅ Prefer **Lua scripts** for atomic read-modify-write over `WATCH` retry loops.
- ✅ Run **AOF (everysec) + periodic RDB**; back up RDBs off-box.
- ✅ Use **HA**: replicas + **Sentinel** (single dataset) or **Cluster** (sharded).
- ✅ **Secure it:** bind to private NICs, ACL users, TLS, drop dangerous commands. Never on a public IP.
- ✅ Add **TTL jitter** and **single-flight** locking to prevent cache stampedes.
- ✅ Share **one client/pool** app-wide (go-redis is already a pool); dedicate separate connections for `SUBSCRIBE`/blocking commands.

**Don't / common bugs:**
- ❌ Running `KEYS *`, or `SMEMBERS`/`HGETALL`/`LRANGE 0 -1` on huge keys — they **block** the single thread for everyone.
- ❌ Expecting **transactions to roll back** — they don't; a runtime error doesn't undo the other queued commands.
- ❌ Treating **Redlock** as correctness-grade for money — without fencing tokens it can allow two holders.
- ❌ Relying on **Pub/Sub** for messages that must not be lost — it has no persistence; use Streams.
- ❌ Forgetting that `SET k v` (without `KEEPTTL`) **clears the TTL** and silently makes the key permanent.
- ❌ Reading from a **replica** and assuming it's fresh — replication is async (stale reads, possible lost writes on failover).
- ❌ Using **numbered DBs** in Cluster (unsupported) or for "namespacing" (prefer key prefixes).
- ❌ Multi-key commands across **different slots** in Cluster without a shared `{hashtag}` → `CROSSSLOT` error.
- ❌ Leaving **`MONITOR`** running in prod — it cripples throughput.
- ❌ Storing one **giant value** when you only ever need part of it — use a hash or many keys.
- ❌ Exposing Redis with **no auth on `0.0.0.0`** — the #1 way Redis gets compromised.

**Tricks worth knowing:**
- 🔹 `GETDEL` for one-time tokens; `GETEX` for sliding-TTL reads.
- 🔹 `SET k v NX GET` reads the old value and conditionally sets it in one atomic op.
- 🔹 `OBJECT ENCODING k` confirms Redis is using a compact encoding (listpack/intset) for a small structure.
- 🔹 `redis-cli --bigkeys` / `--hotkeys` / `LATENCY DOCTOR` / `MEMORY DOCTOR` for instant diagnostics.
- 🔹 `WAIT numreplicas timeout` after critical writes for stronger (synchronous-ish) durability.
- 🔹 `COPY src dst` + rename for atomic-ish "blue/green" key swaps.
- 🔹 `CLIENT NO-EVICT on` / `CLIENT NO-TOUCH on` for special clients; `CLIENT LIST`/`CLIENT KILL` to manage connections.
- 🔹 Enable **client-side caching** (RESP3 tracking / `CLIENT TRACKING`) to cache hot reads in the app and receive invalidation pushes.

---

## 19. Study Path & Build-to-Learn Projects

### 19.1 Suggested learning order

Learn in this order for the fastest path to offline mastery:

1. **Fundamentals** — install via Docker, live in `redis-cli`, internalize the key→data-structure model and the single-threaded execution model, and skim RESP (§1).
2. **Data types** — Strings, Hashes, Lists, Sets, Sorted Sets first; they cover ~90% of real work. Then Bitmaps / HLL / Geo / Streams / JSON as you need them (§2).
3. **Keys & TTL** — expiration semantics, `SCAN` vs `KEYS`, naming conventions (§3). Burn in "always TTL your cache."
4. **Caching** — cache-aside, eviction policies, stampede prevention. This is the #1 production use (§9).
5. **Atomicity** — pipelining vs transactions vs Lua; write a rate limiter and a lock by hand (§6, §7, §10, §11).
6. **Messaging** — Pub/Sub, then Streams + consumer groups; build a real job queue (§4, §5, §12).
7. **Persistence & HA** — RDB vs AOF, replication, Sentinel, then Cluster (§8, §13).
8. **Security & ops** — ACLs/TLS, and the `INFO`/`SLOWLOG`/`LATENCY`/memory tools (§14, §16).
9. **Advanced / 2026** — vector search for RAG; explore the Redis 8 bundled modules and Valkey (§15).

### 19.2 Build to learn (do these projects)

1. **Cache layer for an API** — wrap a PostgreSQL-backed REST API (see `POSTGRESQL_GUIDE.md`) with cache-aside, TTL + jitter, and single-flight stampede protection. Measure your hit ratio with `INFO stats`.
2. **Real-time leaderboard service** — a ZSET-backed top-N plus "rank around me," with live updates over WebSocket; add a per-IP sliding-window rate limiter.
3. **Durable job queue** — Streams + consumer groups with ACK, retries, dead-letter handling, and `XAUTOCLAIM`-based crash recovery; run multiple workers and **kill one** to watch recovery happen.
4. **Semantic search / RAG box** — embed documents, index with `FT.CREATE ... VECTOR HNSW`, and serve KNN + tag-filtered queries over HTTP (Redis 8 or Redis Stack).
5. **HA playground** — stand up a 3-node Sentinel setup *and* a 6-node Cluster in Docker Compose; kill primaries and watch failover/resharding from a client.

> Those projects cover the bread and butter of what teams actually use Redis for: caching, ranking/real-time, queues, AI search, and running it reliably. Keep `SET/GET/EXPIRE`, `INCR`, `HSET/HGETALL`, `ZADD/ZRANGE REV`, `LPUSH/BRPOP`, `XADD/XREADGROUP/XACK`, and `SCAN` in muscle memory — they're 80% of daily Redis.
