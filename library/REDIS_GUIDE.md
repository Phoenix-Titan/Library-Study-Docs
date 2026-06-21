# Redis — Complete Offline Reference

> **Who this is for:** Backend and full-stack developers (beginner to advanced) who want to understand Redis deeply — every core data type, every major feature (caching, queues, pub/sub, streams, transactions, scripting, clustering, vector search) — and use it correctly in production. Every concept is shown with **`redis-cli` commands** *and* **client code** in **Node.js** (`ioredis` and `node-redis`/`redis` v4+) and **Go** (`github.com/redis/go-redis/v9`). You can read this end-to-end with no internet access and walk away able to build real systems.
>
> **Version note:** This guide targets **Redis 7.x and Redis 8.x**, current as of **2026**.
>
> - **Redis 8 (GA mid-2025)** is a *unified* release: the modules that used to ship separately as **Redis Stack** — **RediSearch** (search + vectors), **RedisJSON**, **Redis Time Series**, and **probabilistic types** (Bloom/Cuckoo/Count-Min/Top-K/t-digest) — are now **bundled into core Redis**. You no longer need a separate "Redis Stack" image for JSON, search, or vector similarity. Redis 8 also added a new high-performance keyspace engine and major latency improvements.
> - **Licensing history (important):** Through Redis 7.2 the server was **BSD-3** (true open source). In **March 2024** Redis Ltd. relicensed to a **dual RSALv2 / SSPLv1** model (source-available, *not* OSI open source). In **May 2025**, with Redis 8, they **added AGPLv3** as a third option, so Redis 8 is once again available under an OSI-approved copyleft license (alongside RSALv2/SSPL). The 2024 relicensing triggered the **Valkey** fork (a Linux Foundation project, BSD-3 licensed, drop-in compatible, backed by AWS, Google, Oracle, and others). **Valkey 7.2 / 8.x** is the fully-open alternative; almost everything in this guide applies to it verbatim (Valkey lacks the bundled JSON/Search/Vector modules but has its own module ecosystem). Pick Redis 8 (AGPL) or Valkey depending on your license posture.
>
> Fast-moving details (module bundling, licensing, cluster internals) are flagged with **⚡ Version note**. When in doubt, confirm against `redis.io/docs` and the `COMMAND DOCS` output of your server.

---

## Table of Contents

1. [What Redis Is, When to Use It, Install & RESP](#1-what-redis-is-when-to-use-it-install--resp)
2. [Data Types in Depth](#2-data-types-in-depth)
3. [Key Management — TTL, SCAN, Naming](#3-key-management--ttl-scan-naming)
4. [Pub/Sub](#4-pubsub)
5. [Streams & Consumer Groups](#5-streams--consumer-groups)
6. [Transactions & Optimistic Locking](#6-transactions--optimistic-locking)
7. [Lua Scripting & Functions](#7-lua-scripting--functions)
8. [Persistence — RDB vs AOF](#8-persistence--rdb-vs-aof)
9. [Caching Patterns & Eviction](#9-caching-patterns--eviction)
10. [Pipelining, Performance & Connection Pooling](#10-pipelining-performance--connection-pooling)
11. [Distributed Locking & Rate Limiting](#11-distributed-locking--rate-limiting)
12. [Queues — Lists vs Pub/Sub vs Streams](#12-queues--lists-vs-pubsub-vs-streams)
13. [High Availability — Replication, Sentinel, Cluster](#13-high-availability--replication-sentinel-cluster)
14. [Security — ACLs, AUTH, TLS](#14-security--acls-auth-tls)
15. [Vector Search & Redis as a Vector DB](#15-vector-search--redis-as-a-vector-db)
16. [Observability & Operations](#16-observability--operations)
17. [Real-World Recipes](#17-real-world-recipes)
18. [Gotchas & Best Practices](#18-gotchas--best-practices)
19. [Study Path](#19-study-path)

---

## 1. What Redis Is, When to Use It, Install & RESP

### What Redis is

**Redis** (REmote DIctionary Server) is an **in-memory data structure store**. Unlike a traditional database that keeps data on disk and caches some in RAM, Redis keeps the **entire dataset in RAM** and (optionally) persists to disk for durability. This is why it is *fast*: typical operations complete in **microseconds**, and a single instance can serve **hundreds of thousands to millions of operations per second**.

The crucial mental model: **Redis is not a key→string store, it is a key→data-structure store.** A value can be a string, a list, a set, a sorted set, a hash, a stream, a bitmap, a HyperLogLog, a geospatial index, a JSON document, or a search/vector index. Redis gives you **server-side commands that operate on these structures atomically**, so you push computation to the data instead of round-tripping it to your app.

Redis is **single-threaded for command execution** (one command runs to completion before the next starts — this is what makes individual commands atomic), though modern Redis uses extra threads for I/O, background saving, and some commands. Do not confuse "single-threaded" with "slow": it removes lock contention and makes reasoning about atomicity trivial.

### When to use Redis (and when not)

**Great fits:**

| Use case | Why Redis |
|---|---|
| **Caching** | Microsecond reads, TTLs, eviction policies — the #1 use case |
| **Session store** | Fast, shared across app instances, auto-expiring |
| **Rate limiting** | Atomic counters with expiry |
| **Leaderboards / ranking** | Sorted sets give O(log N) ranked queries |
| **Real-time feeds / pub-sub** | Pub/Sub & Streams for fan-out and event delivery |
| **Job / task queues** | Lists (`BLPOP`) or Streams (consumer groups) |
| **Distributed locks** | `SET NX PX` + tokens |
| **Counting & analytics** | `INCR`, HyperLogLog (cardinality), bitmaps (presence) |
| **Geospatial** | Radius/box queries with `GEOSEARCH` |
| **Vector search / RAG** | Vector similarity for AI apps (Redis 8 / RediSearch) |

**Poor fits / be careful:**

- **Primary store for data larger than RAM** — Redis keeps everything in memory. If your dataset is hundreds of GB+ and mostly cold, a disk database is cheaper.
- **Complex relational queries / joins / ad-hoc analytics** — use Postgres.
- **Strong multi-key ACID transactions across a cluster** — Redis transactions are limited (no rollback on logic errors; cluster transactions must touch one slot).
- **Durability-critical financial ledgers** — Redis *can* be durable (AOF `always`) but at a throughput cost; many shops use it as a cache fronting a durable store.

### Install

#### Docker (recommended for dev — same everywhere)

```bash
# Redis 8 — JSON, Search, Vectors, Time Series, Bloom all bundled in core.
docker run -d --name redis -p 6379:6379 redis:8

# Persist data to a named volume so it survives container removal:
docker run -d --name redis -p 6379:6379 \
  -v redis-data:/data \
  redis:8 \
  redis-server --appendonly yes        # enable AOF persistence

# Open a redis-cli inside the running container:
docker exec -it redis redis-cli
# 127.0.0.1:6379> PING
# PONG
```

⚡ **Version note:** Before Redis 8 you needed the `redis/redis-stack` image to get JSON/Search/Vectors and a separate `redis/redis-stack-server`. On Redis 8 the plain `redis:8` image includes them. For the fully-open fork: `docker run -d -p 6379:6379 valkey/valkey:8`.

#### Native install

```bash
# macOS (Homebrew):
brew install redis
brew services start redis          # run as a background service

# Debian / Ubuntu (official APT repo gives you the latest):
sudo apt-get install -y redis      # distro package (may be older)
# or build from source for the newest version:
#   wget https://download.redis.io/redis-stable.tar.gz
#   tar xzf redis-stable.tar.gz && cd redis-stable && make && sudo make install

# Start a server manually with a config file:
redis-server /etc/redis/redis.conf

# Check it's alive:
redis-cli PING        # → PONG
```

### `redis-cli` basics

`redis-cli` is the interactive client and the tool you'll live in while learning.

```bash
# Connect (defaults: 127.0.0.1:6379, no auth, db 0):
redis-cli

# Connect to a remote/secured server:
redis-cli -h myhost -p 6380 -a 'password' --tls

# Use the new RESP3 protocol (richer replies, see below):
redis-cli -3

# Run a single command and exit (great for scripts):
redis-cli SET greeting "hello"
redis-cli GET greeting        # → "hello"

# Select a logical database (0–15 by default). NOTE: SELECT/multiple DBs
# do NOT exist in Cluster mode — avoid relying on numbered DBs.
redis-cli -n 1

# Helpful flags:
redis-cli --scan --pattern 'user:*'   # safely iterate keys (uses SCAN, not KEYS)
redis-cli --stat                      # live throughput / memory stats every second
redis-cli --latency                   # measure round-trip latency
redis-cli --bigkeys                   # sample the keyspace for large keys
redis-cli --memkeys                   # sample keys by memory usage
```

Inside the interactive prompt:

```redis
127.0.0.1:6379> SET name "Ada"
OK
127.0.0.1:6379> GET name
"Ada"
127.0.0.1:6379> TYPE name
string
127.0.0.1:6379> DEL name
(integer) 1
127.0.0.1:6379> HELP SET        # built-in command help
127.0.0.1:6379> COMMAND DOCS GET  # full machine-readable docs
```

### The RESP protocol (what's on the wire)

Redis speaks **RESP** (REdis Serialization Protocol) — a simple, line-based, human-readable text protocol over TCP. Understanding it demystifies Redis and helps you debug.

- **RESP2** (classic): replies are typed by their **first byte**:
  - `+` simple string → `+OK\r\n`
  - `-` error → `-ERR unknown command\r\n`
  - `:` integer → `:1000\r\n`
  - `$` bulk string (binary-safe, length-prefixed) → `$5\r\nhello\r\n`
  - `*` array → `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`
- **RESP3** (Redis 6+, default on by HELLO): adds types like **maps** (`%`), **sets** (`~`), **doubles** (`,`), **booleans** (`#`), **big numbers**, **verbatim strings**, and **push** messages (used for pub/sub & client-side caching). Clients negotiate it with `HELLO 3`.

A request is *always* an array of bulk strings. You can literally type Redis over `nc`:

```bash
# Send "SET foo bar" by hand over a raw TCP connection:
printf '*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n' | nc localhost 6379
# Server replies: +OK
# (*3 = array of 3 elements; each $N = bulk string of N bytes)
```

You almost never write RESP by hand — clients do it for you — but knowing the framing explains *pipelining* (just send many request arrays back-to-back) and why bulk strings are *binary-safe* (you can store JPEGs, protobufs, anything).

### Connecting from code — the canonical clients

**Node.js — `ioredis`** (feature-rich, popular, great cluster/sentinel support):

```js
// npm install ioredis
import Redis from "ioredis";

// Connection string or options object. ioredis auto-reconnects.
const redis = new Redis("redis://localhost:6379");
// const redis = new Redis({ host: "localhost", port: 6379, password: "secret" });

await redis.set("greeting", "hello");
console.log(await redis.get("greeting")); // → "hello"

await redis.quit(); // graceful close
```

**Node.js — `node-redis` (the official `redis` package, v4+)** — promise-based, you must `connect()`:

```js
// npm install redis
import { createClient } from "redis";

const client = createClient({ url: "redis://localhost:6379" });
client.on("error", (err) => console.error("Redis error:", err));
await client.connect(); // v4+ requires an explicit connect

await client.set("greeting", "hello");
console.log(await client.get("greeting")); // → "hello"

await client.quit();
```

**Go — `github.com/redis/go-redis/v9`** (the official, maintained client):

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

	// NewClient returns a client backed by a connection POOL (safe for concurrent use).
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // default DB
	})
	defer rdb.Close()

	if err := rdb.Set(ctx, "greeting", "hello", 0).Err(); err != nil {
		panic(err)
	}
	val, err := rdb.Get(ctx, "greeting").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println(val) // → hello
}
```

> Throughout the guide, assume `redis` (ioredis), `client` (node-redis), and `rdb` + `ctx` (go-redis) are already set up as above.

---

## 2. Data Types in Depth

Redis values are typed. Pick the right type and many problems become one-liners. Below: each type, when to use it, a full command reference, and code.

### 2.1 Strings

The simplest type. A string value is **binary-safe** and can be up to **512 MB**. "String" is a misnomer — it stores text, numbers (for atomic counters), serialized JSON/protobuf, or raw bytes.

**Use cases:** caching serialized objects, counters, feature flags, simple key/value, bitmaps (a string viewed as bits).

```redis
SET user:1:name "Ada Lovelace"
GET user:1:name                 # "Ada Lovelace"

# SET options (combine them — this is the workhorse command):
SET session:abc "{...}" EX 3600        # expire in 3600s
SET lock:job "owner1" NX PX 5000       # set only if Not eXists, expire 5000ms
SET config:flag "on" XX                # set only if it already eXists
SET counter 10 GET                      # set new value, return OLD value (Redis 6.2+)
SET k v KEEPTTL                         # overwrite value but keep existing TTL

# Counters are atomic (no read-modify-write race):
SET views 0
INCR views                  # 1
INCRBY views 10             # 11
DECR views                  # 10
INCRBYFLOAT price 3.50      # works on floats

# Multiple keys at once (saves round trips):
MSET a 1 b 2 c 3
MGET a b c                  # 1) "1" 2) "2" 3) "3"

# Substring & length:
APPEND log "line1\n"        # returns new length
STRLEN user:1:name          # byte length
GETRANGE user:1:name 0 2    # "Ada"
SETRANGE user:1:name 4 "B." # overwrite from offset 4
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
// ioredis — cache a JSON object with TTL, the cache-aside pattern (see §9):
await redis.set("user:1", JSON.stringify({ id: 1, name: "Ada" }), "EX", 3600);
const raw = await redis.get("user:1");
const user = raw ? JSON.parse(raw) : null;

// Atomic counter:
const views = await redis.incr("page:home:views");
```

```go
// go-redis — same idea:
import "encoding/json"
import "time"

b, _ := json.Marshal(map[string]any{"id": 1, "name": "Ada"})
rdb.Set(ctx, "user:1", b, time.Hour)

views, _ := rdb.Incr(ctx, "page:home:views").Result()
_ = views
```

### 2.2 Lists

Ordered collections of strings, implemented as a **linked list / quicklist** — O(1) push/pop at the ends, O(n) random access. Think: queues, stacks, recent-items, simple logs.

**Use cases:** job queues (`LPUSH`/`BRPOP`), activity timelines (capped with `LTRIM`), inter-process messaging.

```redis
RPUSH mylist a b c          # push to tail → [a, b, c]
LPUSH mylist z              # push to head → [z, a, b, c]
LRANGE mylist 0 -1          # read all (0 to -1 = whole list)
LPOP mylist                 # remove & return head ("z")
RPOP mylist 2               # pop 2 from tail (Redis 6.2+)
LLEN mylist                 # length
LINDEX mylist 0             # element at index
LSET mylist 0 "new"         # set by index
LINSERT mylist BEFORE b "x" # insert relative to a value
LREM mylist 1 a             # remove 1 occurrence of "a"
LTRIM mylist 0 99           # keep only indices 0..99 (cap a list cheaply)

# Blocking pops — wait up to N seconds for an element (great for queues):
BLPOP myqueue 5             # block ≤5s; 0 = block forever
BRPOP myqueue 0

# Atomically move between lists (reliable queue handoff):
LMOVE src dst LEFT RIGHT    # pop src head → push dst tail
BLMOVE src dst LEFT RIGHT 0 # blocking version (replaces deprecated RPOPLPUSH)
```

| Command | Purpose |
|---|---|
| `LPUSH` / `RPUSH` / `LPUSHX` / `RPUSHX` | Push head/tail (`X` = only if list exists) |
| `LPOP` / `RPOP [count]` | Pop head/tail |
| `BLPOP` / `BRPOP` | **Blocking** pop (queue consumer) |
| `LMOVE` / `BLMOVE` | Atomic move between lists (reliable queues) |
| `LRANGE` / `LINDEX` / `LLEN` | Read range / by index / length |
| `LSET` / `LINSERT` / `LREM` / `LTRIM` | Modify / insert / remove / cap |

```js
// ioredis — a minimal reliable worker loop using a blocking pop:
async function worker() {
  while (true) {
    // BRPOP returns [key, value] or null on timeout.
    const res = await redis.brpop("jobs", 5);
    if (!res) continue;          // timed out → loop again
    const [, job] = res;
    await handle(JSON.parse(job));
  }
}
```

```go
// go-redis — producer + blocking consumer:
rdb.LPush(ctx, "jobs", `{"id":1}`)

res, err := rdb.BRPop(ctx, 5*time.Second, "jobs").Result()
// res = ["jobs", "{\"id\":1}"]; err == redis.Nil on timeout.
if err == redis.Nil {
	// no job within timeout
}
```

### 2.3 Sets

Unordered collections of **unique** strings, backed by a hash table (O(1) add/remove/membership). Plus fast **set algebra** (union/intersection/difference) server-side.

**Use cases:** tags, unique visitors, "who liked this", relationships, deduplication, allow/deny lists.

```redis
SADD tags:post:1 redis db nosql
SADD tags:post:1 redis          # duplicate ignored, returns 0
SMEMBERS tags:post:1            # all members (O(n) — avoid on huge sets)
SISMEMBER tags:post:1 redis     # 1 if member, else 0
SMISMEMBER tags:post:1 redis go # multi membership (Redis 6.2+)
SCARD tags:post:1               # count
SREM tags:post:1 nosql          # remove
SRANDMEMBER tags:post:1 2       # 2 random members (no removal)
SPOP tags:post:1                # remove & return a random member

# Set algebra (push computation to the server):
SINTER online:gamers online:premium     # users that are BOTH
SUNION followers:a followers:b           # combined audience
SDIFF all banned                         # everyone except banned
SINTERCARD 2 a b LIMIT 100               # count of intersection, capped (6.2+)
SINTERSTORE result a b                    # store the intersection in a key

# Iterate large sets safely (cursor-based, non-blocking):
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
// ioredis — track unique daily visitors, then count them:
await redis.sadd("visitors:2026-06-21", "user:42", "user:99");
const uniqueToday = await redis.scard("visitors:2026-06-21");
// Mutual followers:
const mutuals = await redis.sinter("followers:alice", "following:alice");
```

### 2.4 Sorted Sets (ZSET)

The crown jewel. Like a Set, but every member has an associated **double score**, and members are kept **sorted by score** (ties broken lexicographically). O(log N) inserts and ranked lookups. This single structure powers leaderboards, priority queues, time-series windows, rate limiters, and secondary indexes.

**Use cases:** leaderboards, top-N, delayed/priority queues (score = timestamp), sliding-window rate limiting, "trending" with time-decay.

```redis
ZADD leaderboard 100 alice 250 bob 175 carol
ZADD leaderboard GT 300 alice    # only update if greater (GT/LT, Redis 6.2+)
ZADD leaderboard NX 50 dave      # add only if not present
ZADD leaderboard INCR 10 bob     # increment bob's score by 10, return new score

ZSCORE leaderboard bob           # bob's score
ZINCRBY leaderboard 5 carol      # add 5 to carol
ZCARD leaderboard                # member count
ZRANK leaderboard alice          # 0-based rank (ascending)
ZREVRANK leaderboard alice       # rank descending (0 = top)

# Ranked queries — the leaderboard bread and butter:
ZRANGE leaderboard 0 9 REV WITHSCORES        # top 10 (Redis 6.2+ unified syntax)
ZRANGE leaderboard 0 -1 WITHSCORES           # everyone, ascending
ZREVRANGE leaderboard 0 2 WITHSCORES         # top 3 (legacy form)

# Query by score range:
ZRANGEBYSCORE leaderboard 100 200            # scores in [100,200]
ZRANGEBYSCORE leaderboard "(100" 200         # exclusive lower bound
ZRANGEBYSCORE leaderboard -inf +inf LIMIT 0 5
ZCOUNT leaderboard 100 200                    # count in range

# Query by lexicographic range (when ALL scores are equal — a sorted index):
ZRANGEBYLEX names "[a" "[c"                    # members a..c

ZREM leaderboard dave                          # remove member
ZREMRANGEBYRANK leaderboard 0 -11              # keep only top 10 (trim)
ZREMRANGEBYSCORE leaderboard -inf 1700000000   # drop old entries (time window)
ZPOPMAX leaderboard                            # remove & return highest
ZPOPMIN leaderboard 3                          # remove & return 3 lowest
BZPOPMIN q 0                                    # blocking pop (priority queue!)

# Combine sorted sets:
ZUNIONSTORE dest 2 z1 z2 WEIGHTS 1 2
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
| `ZREMRANGEBYRANK/SCORE/LEX` | Bulk trim | O(log N + M) |
| `ZUNIONSTORE` / `ZINTERSTORE` / `ZDIFFSTORE` | Combine sets | varies |

```js
// ioredis — leaderboard helpers:
await redis.zadd("leaderboard", 100, "alice", 250, "bob");
await redis.zincrby("leaderboard", 50, "alice");        // alice now 150

// Top 10 with scores, highest first:
const top = await redis.zrange("leaderboard", 0, 9, "REV", "WITHSCORES");
// → ["bob","250","alice","150", ...] (flat array; pair them up)

// A player's rank (1-based for display):
const rank = await redis.zrevrank("leaderboard", "alice");
const displayRank = rank === null ? null : rank + 1;
```

```go
// go-redis — leaderboard:
rdb.ZAdd(ctx, "leaderboard",
	redis.Z{Score: 100, Member: "alice"},
	redis.Z{Score: 250, Member: "bob"},
)
// Top 10 descending, with scores → []redis.Z:
top, _ := rdb.ZRevRangeWithScores(ctx, "leaderboard", 0, 9).Result()
for i, z := range top {
	fmt.Printf("#%d %v = %.0f\n", i+1, z.Member, z.Score)
}
```

### 2.5 Hashes

A hash maps field→value within a single key — like a small object/record stored under one key. Memory-efficient for many small fields (Redis uses a compact "listpack" encoding below a threshold).

**Use cases:** storing objects (user profile, product), grouping related fields, counters per field, partial updates without re-serializing the whole object.

```redis
HSET user:1 name "Ada" age 36 city "London"
HGET user:1 name                 # "Ada"
HMGET user:1 name age            # multiple fields
HGETALL user:1                   # all fields+values (O(n) — careful on huge hashes)
HKEYS user:1 / HVALS user:1      # just fields / just values
HLEN user:1                      # field count
HEXISTS user:1 email             # 0/1
HDEL user:1 city                 # delete field(s)
HINCRBY user:1 age 1             # atomic per-field counter
HINCRBYFLOAT cart:1 total 9.99
HSETNX user:1 verified true      # set field only if absent
HRANDFIELD user:1 2 WITHVALUES   # random fields (6.2+)
HSCAN user:1 0 COUNT 50          # safe iteration

# Per-FIELD TTL — Redis 7.4+ (huge feature: expire individual hash fields):
HEXPIRE user:1 60 FIELDS 1 sessiontoken    # field expires in 60s
HTTL user:1 FIELDS 1 sessiontoken          # remaining TTL on a field
HPERSIST user:1 FIELDS 1 sessiontoken      # remove field TTL
```

⚡ **Version note:** **Per-field TTL** (`HEXPIRE`, `HPEXPIRE`, `HTTL`, `HPERSIST`, etc.) arrived in **Redis 7.4**. Before that, TTL only applied to whole keys. This makes hashes usable as a true session/cache store where individual fields expire independently.

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
await redis.hincrby("user:1", "age", 1);            // birthday!
const profile = await redis.hgetall("user:1");      // → { name, age, city }
```

```go
// go-redis — HSet accepts a map or pairs; HGetAll returns map[string]string:
rdb.HSet(ctx, "user:1", map[string]any{"name": "Ada", "age": 36})
profile, _ := rdb.HGetAll(ctx, "user:1").Result()
rdb.HIncrBy(ctx, "user:1", "age", 1)
```

### 2.6 Bitmaps

Not a separate type — a **string** treated as a bit array (up to 2^32 bits = 512 MB). Extremely memory-efficient for boolean state across a large id space (e.g. "did user N do X today?" for millions of users in a few MB).

**Use cases:** daily active users, feature-flag-per-user, A/B bucketing, presence, real-time analytics.

```redis
SETBIT active:2026-06-21 42 1     # mark user id 42 active
GETBIT active:2026-06-21 42       # 1
BITCOUNT active:2026-06-21        # how many users active (population count)
BITCOUNT active:2026-06-21 0 0 BYTE   # count within a byte range
BITPOS active:2026-06-21 1        # first set bit

# Combine days with bitwise ops (store result in a destination key):
BITOP AND result active:mon active:tue     # users active BOTH days
BITOP OR weekly active:mon active:tue ...   # active any day
BITOP XOR / NOT ...

# Multi-bitfield ops on counters packed into a string:
BITFIELD stats SET u8 0 255 GET u8 0 INCRBY u8 0 10
```

```js
// ioredis — count daily active users cheaply:
await redis.setbit("dau:2026-06-21", 1000042, 1);
const dau = await redis.bitcount("dau:2026-06-21");
```

### 2.7 HyperLogLog

A **probabilistic** structure for counting **unique elements (cardinality)** using a fixed ~12 KB regardless of how many items you add — with a standard error of ~0.81%. You cannot list the members, only estimate the count. Use it when "roughly how many uniques" is enough and you can't afford a giant set.

**Use cases:** unique visitors per page/day, unique search terms, dedup counting at massive scale.

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

> **Set vs HyperLogLog:** a Set of 10M ids costs hundreds of MB and gives exact counts + membership; an HLL costs 12 KB and gives an approximate count only. Choose based on whether you need exactness/membership.

### 2.8 Geospatial

Geo commands are sugar over a **sorted set** (it geohash-encodes lon/lat into the score). Store points, then query by radius or bounding box and get distances.

**Use cases:** "stores near me", ride-hailing driver matching, geofencing, delivery zones.

```redis
GEOADD places -122.27 37.80 "oakland" -122.42 37.77 "sf"
GEOPOS places "sf"                  # → coordinates
GEODIST places "sf" "oakland" km    # distance in km

# Modern unified search (Redis 6.2+, replaces GEORADIUS):
GEOSEARCH places FROMMEMBER "sf" BYRADIUS 20 km ASC WITHCOORD WITHDIST
GEOSEARCH places FROMLONLAT -122.4 37.7 BYBOX 40 40 km ASC COUNT 10
GEOSEARCHSTORE dest places FROMMEMBER "sf" BYRADIUS 50 km   # store results
```

```js
// ioredis — find places within 20km of a point:
await redis.geoadd("places", -122.42, 37.77, "sf");
const near = await redis.geosearch(
  "places", "FROMLONLAT", -122.4, 37.7, "BYRADIUS", 20, "km", "ASC", "WITHDIST"
);
// → [["sf","1.83"], ...] (member + distance)
```

### 2.9 Streams

An **append-only log** (like a mini Kafka topic inside Redis): each entry has an auto-generated, monotonically increasing **ID** (`<ms>-<seq>`) and a set of field/value pairs. Supports fan-out reads, blocking reads, capped length, and — crucially — **consumer groups** for at-least-once, load-balanced processing with acknowledgements. (Full treatment in §5.)

**Use cases:** event sourcing, durable job queues, activity logs, IoT/telemetry ingestion, message bus.

```redis
XADD events * type "signup" user "42"     # * = server-generated ID
XADD events MAXLEN ~ 10000 * k v          # cap length (~ = approximate, cheap)
XLEN events
XRANGE events - +                          # all entries (- = min, + = max)
XRANGE events - + COUNT 10
XREAD COUNT 5 STREAMS events 0             # read from the beginning
XREAD BLOCK 0 STREAMS events $             # block for NEW entries after now
```

### 2.10 JSON (RedisJSON)

Store, query, and atomically update **native JSON documents** with JSONPath. Values are stored in a parsed binary tree, so you update a nested field without reading/rewriting the whole document.

⚡ **Version note:** RedisJSON (the `JSON.*` commands) is a **module**. On **Redis 8** it ships in core — just use `redis:8`. On Redis 7.x you need the **Redis Stack** image (`redis/redis-stack`). The fully-open **Valkey** does not bundle it (use a community module or store JSON as a string).

**Use cases:** document storage with partial updates, config trees, API response caching with field-level access, combined with RediSearch for indexed JSON queries.

```redis
JSON.SET user:1 $ '{"name":"Ada","age":36,"roles":["admin"],"addr":{"city":"London"}}'
JSON.GET user:1 $.name              # ["Ada"]
JSON.GET user:1 $.addr.city         # ["London"]
JSON.SET user:1 $.age 37            # update a nested field in place
JSON.NUMINCRBY user:1 $.age 1       # atomic numeric increment on a path
JSON.ARRAPPEND user:1 $.roles '"editor"'   # push to a JSON array
JSON.ARRLEN user:1 $.roles
JSON.STRAPPEND user:1 $.name '" Lovelace"'
JSON.DEL user:1 $.addr.city         # delete a path
JSON.OBJKEYS user:1 $.addr
JSON.TYPE user:1 $.roles            # array
JSON.MGET user:1 user:2 $.name      # field across multiple docs
```

```js
// node-redis v4+ exposes JSON via client.json.*:
await client.json.set("user:1", "$", { name: "Ada", age: 36, roles: ["admin"] });
const name = await client.json.get("user:1", { path: "$.name" }); // ["Ada"]
await client.json.numIncrBy("user:1", "$.age", 1);
await client.json.arrAppend("user:1", "$.roles", "editor");

// ioredis has no built-in JSON helpers; use sendCommand / call:
await redis.call("JSON.SET", "user:2", "$", JSON.stringify({ name: "Bob" }));
const v = await redis.call("JSON.GET", "user:2", "$.name");
```

```go
// go-redis v9 has JSON helpers (redis.JSONGet/JSONSet) when the module is present:
rdb.JSONSet(ctx, "user:1", "$", `{"name":"Ada","age":36}`)
name, _ := rdb.JSONGet(ctx, "user:1", "$.name").Result()  // ["Ada"]
rdb.JSONNumIncrBy(ctx, "user:1", "$.age", 1)
// Or fall back to raw commands: rdb.Do(ctx, "JSON.SET", key, path, val)
```

### Type quick-reference

| Type | Stores | Killer command(s) | Classic use |
|---|---|---|---|
| String | bytes/number | `SET`, `INCR` | cache, counter |
| List | ordered seq | `LPUSH`/`BRPOP`, `LMOVE` | queue, timeline |
| Set | unique members | `SADD`, `SINTER` | tags, uniques |
| Sorted Set | scored members | `ZADD`, `ZRANGE REV` | leaderboard, priority q |
| Hash | field→value | `HSET`, `HINCRBY` | objects, records |
| Bitmap | bits in a string | `SETBIT`, `BITCOUNT` | DAU, flags |
| HyperLogLog | approx cardinality | `PFADD`, `PFCOUNT` | unique counting at scale |
| Geo | lon/lat points | `GEOADD`, `GEOSEARCH` | nearby search |
| Stream | append-only log | `XADD`, `XREADGROUP` | events, durable queues |
| JSON | JSON docs | `JSON.SET`, `JSON.GET` | documents, configs |

---

## 3. Key Management — TTL, SCAN, Naming

### Existence, type, deletion, rename

```redis
EXISTS user:1            # 1 if exists (EXISTS k1 k2 k3 counts how many exist)
TYPE user:1             # string | list | set | zset | hash | stream | ...
DEL user:1 user:2       # delete keys synchronously (blocks on huge values)
UNLINK user:1           # delete ASYNchronously (frees memory in background) — prefer for big keys
RENAME old new          # rename (overwrites new); errors if old missing
RENAMENX old new        # rename only if new doesn't exist
COPY src dst [REPLACE]  # copy a key (6.2+)
RANDOMKEY               # a random key (debug)
DBSIZE                  # number of keys in the current DB
FLUSHDB / FLUSHALL      # ⚠ wipe current DB / ALL DBs (ASYNC option available)
OBJECT ENCODING user:1  # internal encoding (listpack, intset, skiplist, ...)
OBJECT IDLETIME user:1  # seconds since last access (LRU info)
OBJECT FREQ user:1      # access frequency (when maxmemory-policy is LFU)
```

### TTL / expiration

Any key can carry a **time to live**; when it elapses, Redis deletes the key. Expiry is how caches stay fresh and sessions auto-clean.

```redis
SET s "v" EX 60          # set with 60s TTL in one shot (preferred)
EXPIRE s 120             # (re)set TTL to 120s
EXPIRE s 120 NX          # only if no TTL yet (NX/XX/GT/LT, Redis 7.0+)
PEXPIRE s 5000           # TTL in milliseconds
EXPIREAT s 1893456000    # expire at an absolute UNIX time (seconds)
PEXPIREAT s 1893456000000
TTL s                    # remaining seconds (-1 = no TTL, -2 = key gone)
PTTL s                   # remaining ms
PERSIST s                # remove TTL → key becomes permanent
EXPIRETIME s             # absolute expiry time in seconds (7.0+)
```

**How expiry actually works:** Redis uses **lazy** expiration (a key is checked when accessed) *plus* a **background sampler** that periodically scans random expiring keys. So an expired key may still occupy memory briefly until accessed or sampled — fine for almost all uses, but don't assume memory frees the *instant* a TTL hits.

> **Gotcha:** Most write commands **do not** reset a key's TTL. `SET` without `KEEPTTL` **does clear** it. `INCR`, `LPUSH`, `HSET`, `APPEND`, etc. keep the existing TTL. So `SET k v` removes the expiry unless you add `KEEPTTL` or re-`EXPIRE`.

### `SCAN` vs `KEYS` — never run `KEYS` in production

```redis
KEYS user:*             # ⚠ O(N) over the WHOLE keyspace, BLOCKS the server. Dev only!
```

`KEYS` walks every key and blocks the single thread — on a big instance it can freeze Redis for seconds. Use the **cursor-based `SCAN`** family, which returns a few keys per call and never blocks:

```redis
# SCAN returns [next-cursor, [keys...]]. Repeat until cursor == 0.
SCAN 0 MATCH user:* COUNT 100
SCAN 17 MATCH user:* COUNT 100   # feed back the returned cursor
# Type-specific variants iterate inside one big key:
HSCAN bighash 0 COUNT 100
SSCAN bigset  0 COUNT 100
ZSCAN bigzset 0 COUNT 100
# SCAN ... TYPE string   # filter by value type (6.0+)
```

```js
// ioredis — iterate the whole keyspace safely with the built-in stream:
const stream = redis.scanStream({ match: "session:*", count: 100 });
stream.on("data", (keys) => {
  if (keys.length) redis.unlink(...keys); // delete in batches, async
});
stream.on("end", () => console.log("done"));
```

```go
// go-redis — SCAN iterator hides the cursor loop for you:
iter := rdb.Scan(ctx, 0, "session:*", 100).Iterator()
for iter.Next(ctx) {
	key := iter.Val()
	rdb.Unlink(ctx, key)
}
if err := iter.Err(); err != nil {
	panic(err)
}
```

> `SCAN` guarantees every key present for the whole scan is returned **at least once**, but may return duplicates and reflects concurrent changes — so dedupe if needed and treat it as a best-effort snapshot.

### Key naming conventions

Redis has a flat namespace, so **encode hierarchy in the key with colons**. A good scheme makes keys self-describing, greppable, and easy to scope/expire.

```
object-type:id:field          e.g.  user:1000:profile
object-type:id:subtype:id     e.g.  post:42:comments
feature:scope:window          e.g.  ratelimit:ip:1.2.3.4:2026-06-21
app:env:...                   e.g.  myapp:prod:cache:user:1000
```

Best practices:

- **Prefix by app/feature** so multiple apps can share an instance without collisions.
- Keep keys **short but readable** (key strings live in RAM too; millions of long keys add up).
- Put the **variable part last** so `SCAN feature:*` works.
- Use a **consistent separator** (`:` is the de-facto standard).
- For Cluster, remember keys with the **same `{hashtag}`** land on the same slot — use `user:{1000}:profile` and `user:{1000}:sessions` when you need multi-key ops on related keys.

| Command | Purpose |
|---|---|
| `EXISTS` / `TYPE` / `OBJECT ENCODING` | Inspect a key |
| `DEL` (sync) / `UNLINK` (async) | Delete (prefer `UNLINK` for big values) |
| `RENAME` / `RENAMENX` / `COPY` | Move/duplicate |
| `EXPIRE` / `PEXPIRE` / `EXPIREAT` / `PERSIST` | Manage TTL |
| `TTL` / `PTTL` / `EXPIRETIME` | Inspect TTL |
| `SCAN` / `HSCAN` / `SSCAN` / `ZSCAN` | Safe iteration (use instead of `KEYS`) |

---

## 4. Pub/Sub

**Publish/Subscribe** is a **fire-and-forget**, fan-out messaging system. Publishers send messages to **channels**; subscribers listening on those channels receive them. There is **no persistence and no acknowledgement**: if no one is subscribed when you publish, the message is gone. A subscriber that disconnects misses everything sent while away.

**Use cases:** real-time notifications, cache invalidation broadcasts, chat where missing offline messages is acceptable, live dashboards. For anything that needs durability or delivery guarantees, use **Streams** (§5) instead.

```redis
# Terminal 1 — subscribe (this connection can now only do (un)subscribe commands):
SUBSCRIBE news sports
PSUBSCRIBE news.*           # pattern subscribe (glob-style)

# Terminal 2 — publish:
PUBLISH news "Redis 8 released"     # returns count of receivers
PUBLISH news.tech "module update"

# Introspection:
PUBSUB CHANNELS              # active channels with subscribers
PUBSUB NUMSUB news          # subscriber count per channel
PUBSUB NUMPAT               # number of pattern subscriptions
```

⚡ **Version note:** In **Redis Cluster**, classic `PUBLISH` is broadcast to all nodes (works but is chatty). **Sharded Pub/Sub** (`SPUBLISH` / `SSUBSCRIBE` / `SUNSUBSCRIBE`, Redis 7.0+) keeps messages on the shard owning the channel's slot — use it for scalable pub/sub in a cluster.

```js
// ioredis — a SUBSCRIBER must be a SEPARATE connection from one issuing other commands.
const sub = new Redis();           // dedicated subscriber connection
const pub = new Redis();           // publisher / normal commands

await sub.subscribe("news");
sub.on("message", (channel, message) => {
  console.log(`[${channel}] ${message}`);
});

// Pattern subscription:
await sub.psubscribe("news.*");
sub.on("pmessage", (pattern, channel, message) => {
  console.log(`(${pattern}) ${channel}: ${message}`);
});

await pub.publish("news", "hello world");
```

```go
// go-redis — Subscribe returns a *PubSub; range over its Channel():
sub := rdb.Subscribe(ctx, "news")
defer sub.Close()

ch := sub.Channel() // Go channel of *redis.Message
go func() {
	for msg := range ch {
		fmt.Printf("[%s] %s\n", msg.Channel, msg.Payload)
	}
}()

rdb.Publish(ctx, "news", "hello world")
```

> **Why a dedicated subscriber connection?** Once a connection enters subscribe mode (RESP2) it can only (un)subscribe — it can't run `GET`/`SET`. RESP3 relaxes this via push messages, but keeping a separate connection is the safe, portable habit.

---

## 5. Streams & Consumer Groups

Streams turn Redis into a **durable, ordered, replayable log** with Kafka-like semantics. Unlike Pub/Sub, stream entries **persist** (subject to length/trim policy) and consumers can **resume**, **acknowledge**, and **share work** within a group.

### Producing & basic reading

```redis
# XADD appends an entry. * lets the server assign a sorted ID (<ms>-<seq>).
XADD orders * item "book" qty 2
XADD orders * item "pen"  qty 5

# Cap the stream so it doesn't grow forever:
XADD orders MAXLEN ~ 100000 * item "x"   # ~ = approximate trim (much cheaper)
XADD orders MINID ~ 1700000000000-0 * ...# trim by minimum ID (time-based)
XTRIM orders MAXLEN 50000                 # explicit trim

XLEN orders                  # entry count
XRANGE orders - +            # all entries (oldest→newest)
XREVRANGE orders + - COUNT 1 # newest entry
XDEL orders 1700000000000-0  # delete a specific entry

# Tailing the stream (like `tail -f`): block for entries after the last seen ID.
XREAD COUNT 10 BLOCK 5000 STREAMS orders $   # $ = only entries added after now
XREAD STREAMS orders 0                         # everything from the start
```

### Consumer groups — load-balanced, at-least-once processing

A **consumer group** lets multiple workers cooperatively process a stream: each entry is delivered to **exactly one consumer in the group**, tracked in a **Pending Entries List (PEL)** until **acknowledged**. If a worker dies mid-processing, the entry stays pending and another worker can **claim** it — giving **at-least-once** delivery.

```redis
# Create a group. $ = start consuming only NEW messages; 0 = from the beginning.
# MKSTREAM creates the stream if it doesn't exist yet.
XGROUP CREATE orders workers $ MKSTREAM

# A consumer reads as many NEW (never-delivered) messages as it can.
# > means "messages never delivered to any consumer in this group".
XREADGROUP GROUP workers worker-1 COUNT 10 BLOCK 5000 STREAMS orders >

# After successfully processing, ACK so it leaves the PEL:
XACK orders workers 1700000000000-0

# Inspect what's still pending (unacked):
XPENDING orders workers                          # summary
XPENDING orders workers - + 10                    # detailed, up to 10 entries
XPENDING orders workers - + 10 worker-1           # for one consumer

# Recover from a crashed consumer — claim messages idle > 60s:
XAUTOCLAIM orders workers worker-2 60000 0        # (Redis 6.2+, preferred)
XCLAIM orders workers worker-2 60000 1700000000000-0   # claim specific IDs

# Re-read a consumer's OWN pending (not-yet-acked) entries (0 instead of >):
XREADGROUP GROUP workers worker-1 STREAMS orders 0

# Housekeeping:
XINFO STREAM orders                # stream metadata
XINFO GROUPS orders                # groups + lag + pending counts
XINFO CONSUMERS orders workers     # per-consumer pending/idle
XGROUP DELCONSUMER orders workers worker-1
XGROUP SETID orders workers 0      # rewind the group
```

| Command | Purpose |
|---|---|
| `XADD [MAXLEN/MINID] key * f v` | Append (with optional trim) |
| `XLEN` / `XRANGE` / `XREVRANGE` / `XDEL` / `XTRIM` | Inspect / range / delete / trim |
| `XREAD [BLOCK] STREAMS key id` | Read/tail without a group |
| `XGROUP CREATE/SETID/DELCONSUMER/DESTROY` | Manage groups |
| `XREADGROUP GROUP g c STREAMS key >` | Consume new (or `0` for own pending) |
| `XACK` | Acknowledge processed entries |
| `XPENDING` | Inspect the Pending Entries List |
| `XCLAIM` / `XAUTOCLAIM` | Reassign stalled entries |
| `XINFO STREAM/GROUPS/CONSUMERS` | Introspection |

```js
// ioredis — a robust consumer-group worker.
const GROUP = "workers";
const CONSUMER = `worker-${process.pid}`;

async function ensureGroup() {
  try {
    await redis.xgroup("CREATE", "orders", GROUP, "$", "MKSTREAM");
  } catch (e) {
    if (!String(e.message).includes("BUSYGROUP")) throw e; // group already exists → fine
  }
}

async function run() {
  await ensureGroup();
  while (true) {
    // Read new messages; block up to 5s if idle.
    const res = await redis.xreadgroup(
      "GROUP", GROUP, CONSUMER, "COUNT", 10, "BLOCK", 5000, "STREAMS", "orders", ">"
    );
    if (!res) continue; // timed out
    const [, entries] = res[0]; // res = [[stream, [[id, [f,v,...]], ...]]]
    for (const [id, fields] of entries) {
      try {
        await process(fields);          // your business logic
        await redis.xack("orders", GROUP, id); // ack only on success
      } catch (err) {
        console.error("processing failed, will be reclaimed later:", err);
        // do NOT ack → stays pending → XAUTOCLAIM picks it up
      }
    }
  }
}
```

```go
// go-redis — consumer group worker:
func runWorker(ctx context.Context, rdb *redis.Client) error {
	const group, consumer = "workers", "worker-1"
	// Ignore BUSYGROUP (group already exists).
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
					continue // leave pending for reclaim
				}
				rdb.XAck(ctx, "orders", group, msg.ID)
			}
		}
	}
}
```

### Streams vs Kafka

| Aspect | Redis Streams | Apache Kafka |
|---|---|---|
| Storage | In-memory (+ persistence), trimmed by length/age | Disk log, long retention by design |
| Scale | Single shard per stream (cluster by sharding keys) | Native partitioned topics, huge throughput |
| Delivery | At-least-once via groups + ACK + claim | At-least-once / exactly-once (with txns) |
| Ordering | Per stream | Per partition |
| Ops weight | Lightweight (you already run Redis) | Heavier (brokers, ZooKeeper/KRaft) |
| Best for | Real-time jobs, moderate volume, low latency | High-throughput event pipelines, long retention, replay |

**Rule of thumb:** if you already run Redis and need a durable queue/event log at moderate scale with millisecond latency, Streams are perfect. If you need massive throughput, multi-day retention, and a rich ecosystem (Connect, ksqlDB), reach for Kafka.

---

## 6. Transactions & Optimistic Locking

Redis transactions group commands so they execute **atomically and in isolation** — no other client's commands interleave between them. They use four commands: `MULTI`, `EXEC`, `DISCARD`, `WATCH`.

> **Important — Redis transactions are NOT SQL transactions.** There is **no rollback**: if a queued command fails *at runtime* (e.g. wrong type), the other commands **still execute**. `EXEC` does not undo anything. What you *do* get: commands are queued and then run as one isolated batch.

```redis
MULTI                 # start queuing
SET account:a 100
DECRBY account:a 30
INCRBY account:b 30
EXEC                  # run all queued commands atomically → array of replies
# (DISCARD instead of EXEC throws the queued commands away)
```

Two error modes:

- **Compile-time error** (e.g. a typo'd command, wrong arity) → the *whole* transaction is aborted at `EXEC` (Redis 2.6.5+ detects this when queuing).
- **Runtime error** (e.g. `INCR` on a list) → only that command errors; **the rest still run**. You must check each reply.

### Optimistic locking with `WATCH`

`WATCH` makes `EXEC` **conditional**: if any watched key changes between `WATCH` and `EXEC`, the transaction **aborts** (`EXEC` returns nil), and you retry. This is **compare-and-set / optimistic concurrency** — perfect for "read, compute, conditionally write" without a blocking lock.

```redis
WATCH inventory:item:1          # watch the key(s) you read
# (now read the value in your app)
# GET inventory:item:1  → e.g. "5"
# decide: enough stock? then:
MULTI
DECR inventory:item:1
EXEC
# If another client modified inventory:item:1 after WATCH, EXEC returns (nil) → retry.
```

```js
// ioredis — atomic "decrement stock if available" with WATCH + retry loop.
async function buyOne(itemKey) {
  for (let attempt = 0; attempt < 5; attempt++) {
    await redis.watch(itemKey);                 // start watching
    const stock = parseInt((await redis.get(itemKey)) || "0", 10);
    if (stock <= 0) {
      await redis.unwatch();                     // nothing to do; release watch
      return { ok: false, reason: "out of stock" };
    }
    // Queue the conditional write:
    const result = await redis
      .multi()
      .decr(itemKey)
      .exec();
    // exec() returns null if a watched key changed → retry.
    if (result !== null) return { ok: true, remaining: stock - 1 };
  }
  return { ok: false, reason: "too much contention" };
}
```

```go
// go-redis — the idiomatic Watch helper handles the loop & retry for you:
func buyOne(ctx context.Context, rdb *redis.Client, key string) error {
	txf := func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		if n <= 0 {
			return fmt.Errorf("out of stock")
		}
		// Pipelined writes run only if no watched key changed:
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Decr(ctx, key)
			return nil
		})
		return err
	}
	// Retry on optimistic-lock conflict (redis.TxFailedErr):
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

> For complex read-compute-write atomicity, **Lua scripts (§7) are usually simpler and faster** than `WATCH`/`MULTI` because the whole script runs atomically server-side with no retry loop. Reach for `WATCH` when the logic must live in the app, Lua when it can live in the script.

---

## 7. Lua Scripting & Functions

Redis can run **Lua scripts server-side, atomically**: while a script runs, **no other command executes** (same single-threaded guarantee as a single command). This lets you do read-modify-write logic — check-then-set, conditional updates, multi-key atomic ops — in **one round trip** with no race and no `WATCH` retry loop.

### `EVAL` / `EVALSHA`

```redis
# EVAL script numkeys key1 key2 ... arg1 arg2 ...
# KEYS[] and ARGV[] arrays are how you pass parameters (1-indexed in Lua).
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey hello

# A real example: set a key only if its current value matches (compare-and-set).
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then \
        return redis.call('SET', KEYS[1], ARGV[2]) \
      else return 0 end" 1 mykey oldval newval

# To avoid re-sending the script body every time, load it once and call by SHA:
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "a5260dd66ce02462c5b5231c727b3f7772c0bcc5"  (the SHA1)
EVALSHA a5260dd66ce02462c5b5231c727b3f7772c0bcc5 1 mykey
# If you get NOSCRIPT (server restarted/flushed), reload with SCRIPT LOAD and retry.
SCRIPT EXISTS a5260dd...      # check if cached
SCRIPT FLUSH                  # clear the script cache
```

**Rules & gotchas:**

- **Always pass keys via `KEYS[]`, not hardcoded** — Redis (and Cluster) needs to know which keys you touch (for routing & correctness).
- A script is **atomic** — keep it **fast**; a long script blocks the whole server.
- Scripts must be **deterministic** in older versions (no random/time-dependent writes); modern Redis allows non-deterministic reads via `redis.setresp`/effects replication, but keep scripts pure when you can.
- Return values map Lua→RESP: Lua number→integer, string→bulk string, table→array, `false`→nil.

### Atomic rate limiter in Lua (a canonical example)

```lua
-- rate_limit.lua — fixed-window counter. Returns 1 if allowed, 0 if over limit.
-- KEYS[1] = counter key, ARGV[1] = limit, ARGV[2] = window seconds
local current = redis.call('INCR', KEYS[1])
if current == 1 then
  -- first hit in this window → set the window expiry
  redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
  return 0   -- over the limit
end
return 1     -- allowed
```

```js
// ioredis — register the script once with defineCommand, then call it like a method.
redis.defineCommand("rateLimit", {
  numberOfKeys: 1,
  lua: `
    local current = redis.call('INCR', KEYS[1])
    if current == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
    if current > tonumber(ARGV[1]) then return 0 end
    return 1`,
});

// allowed === 1 means the request is within the limit:
const allowed = await redis.rateLimit(`rl:${userId}`, 100, 60); // 100 req / 60s
```

```go
// go-redis — redis.NewScript handles EVALSHA-with-EVAL-fallback automatically.
var rateLimit = redis.NewScript(`
  local current = redis.call('INCR', KEYS[1])
  if current == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
  if current > tonumber(ARGV[1]) then return 0 end
  return 1`)

func allow(ctx context.Context, rdb *redis.Client, userID string) (bool, error) {
	// Run() tries EVALSHA, falls back to EVAL on NOSCRIPT.
	n, err := rateLimit.Run(ctx, rdb, []string{"rl:" + userID}, 100, 60).Int()
	return n == 1, err
}
```

### Functions (Redis 7.0+)

**Redis Functions** are the modern, managed evolution of scripting: you load a **library** of named functions that persist with the dataset (they survive restarts and replicate to replicas — unlike the ephemeral `EVAL` script cache). Good for shipping reusable, versioned server-side logic.

⚡ **Version note:** Functions require **Redis 7.0+**. They use the `FUNCTION` command family and (by default) the Lua engine.

```lua
-- mylib.lua — a function library.
#!lua name=mylib

redis.register_function('my_incr', function(keys, args)
  return redis.call('INCRBY', keys[1], args[1])
end)
```

```redis
# Load the library (REPLACE to update an existing one):
FUNCTION LOAD REPLACE "$(cat mylib.lua)"
FCALL my_incr 1 counter 5      # call it: KEYS=[counter], ARGS=[5]
FUNCTION LIST                   # list loaded libraries
FUNCTION DUMP / FUNCTION RESTORE  # backup/restore libraries
FUNCTION FLUSH                  # remove all functions
```

| Command | Purpose |
|---|---|
| `EVAL` / `EVALSHA` | Run a Lua script (inline / by SHA) |
| `EVAL_RO` / `EVALSHA_RO` | Read-only variants (safe on replicas) |
| `SCRIPT LOAD` / `SCRIPT EXISTS` / `SCRIPT FLUSH` | Manage the script cache |
| `FUNCTION LOAD/LIST/DELETE/FLUSH/DUMP/RESTORE` | Manage function libraries |
| `FCALL` / `FCALL_RO` | Call a registered function |

---

## 8. Persistence — RDB vs AOF

Redis is in-memory, but it can persist to disk so data survives restarts/crashes. Two mechanisms, often used **together**.

### RDB (snapshots)

A **point-in-time binary snapshot** of the whole dataset, written compactly to `dump.rdb`. Redis `fork()`s a child process that writes the snapshot while the parent keeps serving — so saving is non-blocking (aside from the fork's copy-on-write memory cost).

```conf
# redis.conf — save a snapshot if BOTH a time and change threshold are met.
# "after 900s if ≥1 key changed", "after 300s if ≥100 changed", "after 60s if ≥10000".
save 900 1
save 300 100
save 60 10000
save ""                 # (empty) disables automatic RDB

dbfilename dump.rdb
dir /data
rdbcompression yes
rdbchecksum yes
```

```redis
SAVE        # ⚠ synchronous snapshot — BLOCKS the server. Avoid in prod.
BGSAVE      # background snapshot via fork (preferred)
LASTSAVE    # unix time of the last successful save
```

- **Pros:** compact, fast to load, great for **backups** and replication bootstrap, minimal runtime overhead.
- **Cons:** you can **lose data since the last snapshot** (e.g. up to several minutes) if Redis crashes between saves.

### AOF (Append-Only File)

Logs **every write command** to a file; on restart, Redis replays it to reconstruct the dataset. Far more durable, larger file, periodically **rewritten/compacted** to stay small.

```conf
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"      # AOF lives in a multi-file dir (Redis 7+)

# fsync policy — the durability/performance tradeoff:
# always   → fsync every write  (safest, slowest; ~no data loss)
# everysec → fsync once per second (default; lose ≤1s on crash) ← recommended
# no       → let the OS decide (fastest, least safe)
appendfsync everysec

# Auto-rewrite (compaction) when the AOF grows past these thresholds:
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

```redis
BGREWRITEAOF      # manually trigger an AOF rewrite/compaction
```

⚡ **Version note:** Redis 7 changed AOF to a **multi-part layout** (a base RDB-format file + incremental AOF files in `appendonlydir/`) — more efficient rewrites. Don't hand-edit these files.

### Durability tradeoffs & recommendation

| | RDB only | AOF (everysec) | Both (recommended) |
|---|---|---|---|
| Worst-case loss on crash | minutes | ≤ 1 second | ≤ 1 second |
| Restart load speed | fast | slower (replay) | fast (loads AOF) |
| File size | small | larger | both |
| Backup convenience | excellent | awkward | use RDB for backups |
| Runtime overhead | low (periodic fork) | slightly higher | moderate |

**Default recommendation:** run **both** — AOF (`everysec`) for low data-loss on crash, plus periodic RDB snapshots for fast restarts and easy off-box backups. For a pure cache where loss is acceptable, you can disable persistence entirely (`save ""` + `appendonly no`) for max speed. For strict durability, `appendfsync always` (slower) — but a single Redis is still a single point of failure; combine with replication (§13).

> **Gotcha:** RDB saving and AOF rewrites use `fork()`, which (via copy-on-write) can transiently use up to ~2× memory under heavy write load. Provision headroom and set `vm.overcommit_memory = 1` on Linux to avoid fork failures.

---

## 9. Caching Patterns & Eviction

Caching is Redis's #1 job. The pattern you pick decides consistency, latency, and failure behavior.

### Cache-aside (lazy loading) — the default

The app checks the cache; on a **miss**, it loads from the database, stores in cache with a TTL, and returns. Simple, resilient (cache down ≠ app down), only caches what's used. Downside: first request per key is slow, and cache can be briefly stale after a DB write (mitigate by deleting/updating the cache key on write).

```js
// ioredis — cache-aside read-through helper:
async function getUser(id) {
  const key = `user:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);           // HIT

  const user = await db.users.findById(id);        // MISS → load from DB
  if (user) {
    // Cache for 1h. Add slight jitter to TTL to avoid synchronized expiry (see stampede).
    const ttl = 3600 + Math.floor(Math.random() * 120);
    await redis.set(key, JSON.stringify(user), "EX", ttl);
  }
  return user;
}

// On update, invalidate (delete) so the next read repopulates:
async function updateUser(id, patch) {
  const user = await db.users.update(id, patch);
  await redis.del(`user:${id}`);                   // or set the new value directly
  return user;
}
```

```go
// go-redis — cache-aside:
func getUser(ctx context.Context, rdb *redis.Client, id string) (*User, error) {
	key := "user:" + id
	if b, err := rdb.Get(ctx, key).Bytes(); err == nil {
		var u User
		json.Unmarshal(b, &u)
		return &u, nil // cache hit
	} else if err != redis.Nil {
		return nil, err // real error (not just a miss)
	}
	u, err := loadFromDB(id) // miss
	if err != nil {
		return nil, err
	}
	b, _ := json.Marshal(u)
	rdb.Set(ctx, key, b, time.Hour)
	return u, nil
}
```

### Write-through

Writes go **through the cache to the DB synchronously** — cache and DB stay consistent, reads are always warm, but every write pays the cache+DB latency and you cache data that may never be read.

```js
async function writeThrough(id, value) {
  await db.users.update(id, value);                 // write DB first
  await redis.set(`user:${id}`, JSON.stringify(value), "EX", 3600); // then cache
}
```

### Write-behind (write-back)

Writes hit the cache immediately and are flushed to the DB **asynchronously** (batched). Lowest write latency and DB load, but risk of data loss if Redis dies before flush, and added complexity (you build the flush pipeline, often with a Stream/list as the buffer).

```js
// Buffer writes in a stream; a separate worker batches them to the DB.
await redis.set(`user:${id}`, JSON.stringify(value)); // serve reads instantly
await redis.xadd("db:writeback", "*", "table", "users", "id", id, "data", JSON.stringify(value));
// A consumer-group worker (see §5) drains "db:writeback" → bulk-upserts to the DB.
```

### TTL strategies

- **Absolute TTL:** expire N seconds after write (`SET ... EX`). Simple, bounds staleness.
- **Refresh-ahead:** proactively reload hot keys before they expire (background job watches `TTL` / popularity).
- **Sliding TTL:** extend TTL on each access (e.g. sessions) — re-`EXPIRE` on read, or use `GETEX key EX 1800`.
- **TTL jitter:** add randomness to TTLs so many keys don't expire at the same instant (prevents stampedes).

```redis
GETEX session:abc EX 1800     # read AND slide the TTL to 30 min (Redis 6.2+)
GETEX session:abc PERSIST     # read and remove TTL
GETDEL onetime:token          # read and delete atomically (one-time tokens)
```

### Cache stampede / dogpile prevention

A **stampede** (a.k.a. dogpile / thundering herd) happens when a hot key expires and **many concurrent requests miss simultaneously**, all hitting the DB at once. Defenses:

1. **TTL jitter** (above) — desynchronize expirations.
2. **Locking / single-flight** — first miss takes a short lock and recomputes; others wait or serve stale.
3. **Probabilistic early expiration** (XFetch) — recompute slightly *before* expiry, randomly, so one request refreshes while others still get the cached value.

```js
// ioredis — single-flight cache fill with a short lock (others briefly retry).
async function getWithLock(key, loader, ttl = 3600) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  // Try to acquire a short lock so only ONE caller hits the DB.
  const gotLock = await redis.set(lockKey, "1", "NX", "PX", 5000);
  if (!gotLock) {
    // Someone else is filling it — wait briefly then re-read the cache.
    await new Promise((r) => setTimeout(r, 50));
    return getWithLock(key, loader, ttl);
  }
  try {
    const value = await loader();                       // the single DB hit
    await redis.set(key, JSON.stringify(value), "EX", ttl + Math.floor(Math.random() * 60));
    return value;
  } finally {
    await redis.del(lockKey);                           // release lock
  }
}
```

### Eviction policies (`maxmemory-policy`)

When Redis hits `maxmemory`, it evicts keys per the configured policy. For a **cache**, set `maxmemory` and pick an eviction policy; for a **data store**, you may prefer `noeviction` (writes error rather than silently dropping data).

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru     # evict least-recently-used across all keys
```

| Policy | Evicts | Use when |
|---|---|---|
| `noeviction` | nothing — writes return errors | Redis is a datastore, not a cache |
| `allkeys-lru` | least-recently-used, any key | general cache (most common) |
| `allkeys-lfu` | least-**frequently**-used, any key | cache with stable hot set (often best) |
| `volatile-lru` | LRU among keys **with a TTL** | mixed cache + persistent keys |
| `volatile-lfu` | LFU among keys with a TTL | same, frequency-based |
| `allkeys-random` | random key | uniform access, cheap |
| `volatile-random` | random among keys with TTL | rare |
| `volatile-ttl` | shortest remaining TTL first | evict soon-to-die keys first |

⚡ **Note:** **LFU** (frequency-based, since Redis 4) often outperforms LRU for caches because it resists one-off scans polluting the cache. Inspect with `OBJECT FREQ key` (LFU mode) or `OBJECT IDLETIME key` (LRU mode).

---

## 10. Pipelining, Performance & Connection Pooling

### Pipelining — batch round trips

Each command normally costs a network round trip (RTT). **Pipelining** sends many commands at once without waiting for each reply, then reads all replies together — turning N RTTs into ~1. This is *not* a transaction (commands aren't atomic/isolated), just a latency optimization. It can be a **10–100×** speedup for bulk ops over a network.

```redis
# In redis-cli you can pipe a file of commands:
cat commands.txt | redis-cli --pipe
```

```js
// ioredis — pipeline() batches commands; exec() returns [[err, result], ...].
const pipe = redis.pipeline();
for (let i = 0; i < 1000; i++) pipe.set(`k:${i}`, i);
const results = await pipe.exec(); // one network round trip for all 1000
// ioredis also auto-pipelines concurrent commands when enableAutoPipelining: true.
```

```go
// go-redis — Pipeline() then Exec(); or the Pipelined() helper:
pipe := rdb.Pipeline()
cmds := make([]*redis.StatusCmd, 1000)
for i := 0; i < 1000; i++ {
	cmds[i] = pipe.Set(ctx, fmt.Sprintf("k:%d", i), i, 0)
}
_, err := pipe.Exec(ctx) // single round trip
_ = err
```

> **Pipeline vs MULTI/EXEC:** pipelining = fewer round trips, *no* atomicity. `MULTI/EXEC` = atomicity/isolation. Use a pipeline for bulk speed, a transaction (or Lua) for atomic groups. You can pipeline a `MULTI/EXEC` too.

### Connection pooling

Opening a TCP connection per request is wasteful. Reuse connections via a **pool**.

- **go-redis:** `redis.NewClient` is **already a pool** (default `PoolSize = 10 × NumCPU`) and is safe for concurrent goroutines — share **one** client app-wide. Tune `PoolSize`, `MinIdleConns`, `PoolTimeout`.
- **ioredis / node-redis:** Node is single-threaded, so a **single client** multiplexes commands over one connection efficiently. Share one client instance; for blocking commands (`BLPOP`, `SUBSCRIBE`) or heavy parallelism use a few dedicated extra connections rather than one per request.

```go
// go-redis — pool tuning:
rdb := redis.NewClient(&redis.Options{
	Addr:         "localhost:6379",
	PoolSize:     50,               // max sockets
	MinIdleConns: 10,               // keep warm connections ready
	PoolTimeout:  4 * time.Second,  // wait for a free conn before erroring
	ReadTimeout:  3 * time.Second,
})
```

### Performance tips

- **Pipeline** bulk operations; use **`MGET`/`MSET`/`HMGET`** to fetch many values in one call.
- **Avoid O(N) commands on huge keys** (`KEYS`, `SMEMBERS`, `HGETALL`, `LRANGE 0 -1` on giant collections) — they block the single thread. Use `SCAN`/`HSCAN`/`ZRANGE` with limits.
- **Watch for "big keys" and "hot keys"** — a 1 GB list or a single key taking 90% of traffic causes latency spikes and uneven cluster load. Find them with `redis-cli --bigkeys` / `--hotkeys`.
- **Keep values reasonable** — prefer many small keys or a hash over one enormous value when you only need part of it.
- **Disable transparent huge pages** on Linux (`madvise`/never) — THP causes latency spikes during fork.
- **Use `UNLINK` not `DEL`** for large keys (async free).
- **Measure** with `redis-cli --latency`, `--latency-history`, `LATENCY DOCTOR`, and `SLOWLOG` (§16).

---

## 11. Distributed Locking & Rate Limiting

### Simple lock with `SET NX PX` + a token

A single-instance lock is just `SET key token NX PX ttl`: set only if absent, with an auto-expiring TTL so a crashed holder doesn't deadlock. The **random token** lets you release **only your own** lock (avoid deleting a lock someone else acquired after yours expired) — and the release **must be atomic** (Lua), since check-then-delete has a race.

```js
// ioredis — acquire/release a lock safely.
import { randomUUID } from "crypto";

async function acquireLock(resource, ttlMs = 10000) {
  const token = randomUUID();
  // NX = only if not exists; PX = expiry in ms. Returns "OK" or null.
  const ok = await redis.set(`lock:${resource}`, token, "PX", ttlMs, "NX");
  return ok ? token : null;
}

// Release ONLY if we still own it (atomic compare-and-delete via Lua):
const RELEASE = `
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  else return 0 end`;

async function releaseLock(resource, token) {
  return redis.eval(RELEASE, 1, `lock:${resource}`, token); // 1 = freed, 0 = not ours
}

// Usage:
const token = await acquireLock("order:42");
if (token) {
  try { /* critical section */ } finally { await releaseLock("order:42", token); }
}
```

```go
// go-redis — same pattern:
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

### Redlock — multi-node locking, and its caveats

A single Redis lock breaks if that instance fails or fails over (the lock may be lost on the new primary because replication is async). **Redlock** is an algorithm to acquire a lock across **N independent Redis masters** (no replication between them): you try to lock a majority (N/2+1) within a small time bound; if you get the majority before the TTL mostly elapses, you hold the lock. Libraries: `redlock` (Node), `github.com/go-redsync/redsync/v4` (Go).

```js
// npm install redlock — across several INDEPENDENT redis nodes:
import Redlock from "redlock";
import Redis from "ioredis";

const redlock = new Redlock(
  [new Redis("redis://a:6379"), new Redis("redis://b:6379"), new Redis("redis://c:6379")],
  { retryCount: 3, retryDelay: 200 }
);

// using() auto-extends the lock while your work runs, and releases at the end:
await redlock.using(["resource:order:42"], 5000, async (signal) => {
  // critical section; check signal.aborted for safety on long work
  if (signal.aborted) throw signal.error;
  await doWork();
});
```

> **⚠ Redlock caveats (read this):** Redlock is **controversial**. Martin Kleppmann's well-known critique argues it is unsafe under clock skew, GC/STW pauses, and network delays — a process can *believe* it holds the lock after the TTL expired. The mitigation is a **fencing token** (a monotonically increasing number the lock service returns, which the protected resource checks and rejects stale writers) — but Redis doesn't provide fencing tokens natively. **Bottom line:** for *efficiency* (avoid duplicate work, mostly-mutual-exclusion) Redis locks are fine. For *correctness* (must NEVER have two holders, e.g. money) prefer a system with a real consensus/linearizable store (ZooKeeper, etcd) and fencing tokens. Don't bet financial correctness on Redlock.

### Rate limiting

#### Fixed window (simple, cheap, but bursty at boundaries)

Covered in §7's Lua example: `INCR` a per-window key, set TTL on first hit, reject above the limit. Boundary problem: a client can do `limit` requests at the end of one window and `limit` more at the start of the next.

#### Sliding-window log with a sorted set (precise)

Store each request's timestamp as a ZSET member; drop entries older than the window; count what remains. Accurate, slightly more memory.

```js
// ioredis — sliding-window rate limiter via ZSET, atomic in one Lua script.
redis.defineCommand("slidingLimit", {
  numberOfKeys: 1,
  lua: `
    local key    = KEYS[1]
    local now    = tonumber(ARGV[1])   -- current time (ms)
    local window = tonumber(ARGV[2])   -- window size (ms)
    local limit  = tonumber(ARGV[3])   -- max requests in window
    -- 1) drop timestamps older than the window:
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    -- 2) count requests currently in the window:
    local count = redis.call('ZCARD', key)
    if count >= limit then return 0 end          -- over limit → reject
    -- 3) record this request (score = member = now) and bound the key's life:
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('PEXPIRE', key, window)
    return 1                                       -- allowed
  `,
});

const ok = await redis.slidingLimit(`rl:${userId}`, Date.now(), 60000, 100); // 100/min
```

#### Token bucket (smooth, allows controlled bursts)

A bucket refills tokens at a steady rate up to a capacity; each request spends a token. Stored as a hash `{tokens, ts}`, refilled lazily in a Lua script.

```lua
-- token_bucket.lua  KEYS[1]=bucket  ARGV: rate(tokens/sec) capacity now(sec) cost
local rate     = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local cost     = tonumber(ARGV[4])

local data   = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or capacity   -- new bucket starts full
local ts     = tonumber(data[2]) or now

-- refill based on elapsed time, capped at capacity:
local elapsed = math.max(0, now - ts)
tokens = math.min(capacity, tokens + elapsed * rate)

local allowed = tokens >= cost
if allowed then tokens = tokens - cost end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], math.ceil(capacity / rate) * 2)  -- self-clean idle buckets
return allowed and 1 or 0
```

```go
// go-redis — load the bucket script and use it:
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

| Approach | Delivery | Durable? | Multiple consumers | Use when |
|---|---|---|---|---|
| **List** (`LPUSH`+`BRPOP`) | one consumer per message | yes (persists) | competing (one gets each) | simple work queue |
| **List** (`LMOVE`/`BLMOVE`) | reliable handoff (processing list) | yes | competing | at-least-once via "in-flight" list |
| **Pub/Sub** | broadcast (all subscribers) | **no** | fan-out | live notifications, ephemeral |
| **Stream + group** | one consumer per message, **acked** | yes | competing + replay | durable jobs, at-least-once, retries |

**Decision guide:**
- Need fan-out to all listeners, OK to lose messages → **Pub/Sub**.
- Simple background jobs, one worker each, light needs → **List queue** (`BRPOP`).
- Need acknowledgement, retries, multiple workers, replay, crash recovery → **Streams + consumer groups** (the right answer for most real job queues today).

### A reliable list-based job queue (with in-flight recovery)

```js
// ioredis — producer:
await redis.lpush("jobs:pending", JSON.stringify({ id: 1, task: "email" }));

// Consumer using BLMOVE so an in-flight item isn't lost if the worker crashes:
async function consume() {
  while (true) {
    // Atomically move a job from pending → a per-worker "processing" list.
    const job = await redis.blmove("jobs:pending", "jobs:processing", "RIGHT", "LEFT", 5);
    if (!job) continue;
    try {
      await handle(JSON.parse(job));
      await redis.lrem("jobs:processing", 1, job); // done → remove from in-flight
    } catch (e) {
      // leave it in "jobs:processing"; a reaper requeues stale items.
    }
  }
}
// A separate reaper periodically moves long-stuck items from jobs:processing back to jobs:pending.
```

For most teams, prefer a battle-tested library built on these primitives: **BullMQ** (Node, Streams-based), **Asynq** or **River** (Go), **Sidekiq** (Ruby), **Celery + Redis** (Python). They give retries, scheduling, dead-letter queues, and dashboards for free.

---

## 13. High Availability — Replication, Sentinel, Cluster

A single Redis is a single point of failure and is bounded by one machine's RAM. Three building blocks address availability and scale.

### Replication (primary/replica)

A **replica** asynchronously copies a **primary's** data and serves **reads** (offloading the primary) and acts as a hot standby. Replication is **asynchronous** — a replica may lag slightly, so reads from a replica can be **stale**, and a failover can lose the last few writes.

```conf
# On the replica's config (or at runtime with REPLICAOF):
replicaof 192.168.1.10 6379
replica-read-only yes
```

```redis
REPLICAOF 192.168.1.10 6379   # start replicating (runtime)
REPLICAOF NO ONE              # promote this replica to a standalone primary
INFO replication              # role, connected replicas, offsets, lag
WAIT 1 100                    # block until ≥1 replica acked, ≤100ms (tunable durability)
```

### Sentinel — automatic failover for a primary/replica setup

**Redis Sentinel** is a separate process (run **3+** for quorum) that **monitors** the primary and replicas, **detects** failure, **elects a new primary**, reconfigures replicas, and **notifies clients**. Clients connect to Sentinels to discover the *current* primary address (which changes after failover). Sentinel does **not** shard data — it provides HA for a single logical dataset.

```conf
# sentinel.conf — monitor a master named "mymaster"; need a quorum of 2 to act.
port 26379
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel down-after-milliseconds mymaster 5000    # mark down after 5s unreachable
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

```bash
redis-sentinel /etc/redis/sentinel.conf
# or: redis-server /etc/redis/sentinel.conf --sentinel
```

```js
// ioredis — connect VIA sentinels; it auto-discovers and follows the current master.
const redis = new Redis({
  sentinels: [
    { host: "sentinel-1", port: 26379 },
    { host: "sentinel-2", port: 26379 },
    { host: "sentinel-3", port: 26379 },
  ],
  name: "mymaster",      // the monitored master's name
  role: "master",        // or "slave" to read from replicas
});
```

```go
// go-redis — failover client (Sentinel-aware):
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
	MasterName:    "mymaster",
	SentinelAddrs: []string{"sentinel-1:26379", "sentinel-2:26379", "sentinel-3:26379"},
})
```

### Redis Cluster — horizontal scale + HA via sharding

**Cluster** splits the keyspace into **16384 hash slots** distributed across multiple primaries (each typically with replicas). A key's slot is `CRC16(key) mod 16384`; the client routes commands to the node owning that slot. This scales **memory and throughput beyond one machine** and provides HA (replicas auto-promote on primary failure). Trade-offs: **multi-key ops only work when keys share a slot** (use `{hashtags}`), no numbered DBs, and some commands behave differently.

```redis
# Create a 3-primary, 3-replica cluster from 6 running nodes:
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1

CLUSTER INFO          # cluster_state, slots assigned/ok
CLUSTER NODES         # node ids, roles, slot ranges
CLUSTER SLOTS         # slot→node mapping
CLUSTER KEYSLOT user:1000   # which slot a key hashes to
CLUSTER SHARDS        # (7.0+) shard topology

# Resharding (move slots between nodes) — online, no downtime:
redis-cli --cluster reshard 127.0.0.1:7000
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000   # add then reshard onto it
redis-cli --cluster rebalance 127.0.0.1:7000
```

**Hash tags** force related keys onto the same slot so multi-key commands / transactions / Lua work:

```redis
# Both keys hash by the part inside {…} → same slot → MGET/MSET/EXEC across them is allowed:
MSET user:{1000}:profile "..." user:{1000}:sessions "..."
```

```js
// ioredis — Cluster client (give it a few seed nodes; it learns the topology):
import { Cluster } from "ioredis";
const cluster = new Cluster([
  { host: "127.0.0.1", port: 7000 },
  { host: "127.0.0.1", port: 7001 },
]);
await cluster.set("user:{1000}:name", "Ada"); // routed to the right slot/node
```

```go
// go-redis — Cluster client:
rdb := redis.NewClusterClient(&redis.ClusterOptions{
	Addrs: []string{"127.0.0.1:7000", "127.0.0.1:7001", "127.0.0.1:7002"},
})
```

**Client-side considerations:** handle **MOVED** (slot moved permanently → update map) and **ASK** (slot migrating → redirect this one request) redirections — good clients do this for you. Reads can be served by replicas with `READONLY`. Expect occasional **CROSSSLOT** errors when a multi-key op spans slots without a shared hash tag.

| Tool | Solves | Sharding | Auto-failover |
|---|---|---|---|
| Replication | read scaling, hot standby | no | no (manual) |
| Sentinel | HA for one dataset | no | yes |
| Cluster | scale + HA | yes (16384 slots) | yes |

---

## 14. Security — ACLs, AUTH, TLS

Redis is **fast and trusting by default** — never expose it raw to the internet. Lock it down in layers.

### Protected mode & binding

```conf
bind 127.0.0.1 ::1        # only listen on loopback unless you mean otherwise
protected-mode yes        # refuse external connections when no auth/bind is set
port 6379                 # consider 0 to disable the plain port and use TLS only
```

> **⚠ The classic breach:** a Redis bound to `0.0.0.0` with no password is trivially hijacked (attackers use it to write SSH keys/cron jobs). Always bind to private interfaces + require auth.

### AUTH & the legacy password

```conf
requirepass "a-long-random-password"    # legacy single shared password
```

```redis
AUTH a-long-random-password         # authenticate (legacy)
AUTH alice herpassword              # ACL user + password (preferred)
```

### ACLs (Redis 6+) — fine-grained users

ACLs let you create **named users** with specific allowed **commands**, **key patterns**, **pub/sub channels**, and passwords — least privilege instead of one all-powerful password.

```redis
# A read-only cache user, limited to keys under cache:* :
ACL SETUSER cacheuser on >s3cret ~cache:* +@read +get +mget -@dangerous

# A worker that can use the jobs stream only:
ACL SETUSER worker on >wpass ~jobs:* +@stream +@read +@write

# Inspect & manage:
ACL LIST                 # all rules
ACL WHOAMI               # current user
ACL CAT                  # command categories (@read, @write, @admin, @dangerous, ...)
ACL GETUSER cacheuser
ACL DELUSER cacheuser
ACL SAVE                 # persist ACLs to aclfile (if configured)
```

```conf
# Persist ACLs in a file instead of redis.conf:
aclfile /etc/redis/users.acl
```

Syntax cheatsheet: `on/off` (enable user), `>pass` / `<pass` (add/remove password), `~pattern` (key access), `&channel` (pub/sub), `+cmd`/`-cmd` (allow/deny command), `+@category`/`-@category` (command groups), `allcommands`/`nocommands`, `allkeys`/`resetkeys`.

```js
// Authenticate with an ACL user from clients:
const redis = new Redis({ username: "cacheuser", password: "s3cret" });
```

```go
rdb := redis.NewClient(&redis.Options{
	Addr: "localhost:6379", Username: "cacheuser", Password: "s3cret",
})
```

### TLS (encryption in transit)

```conf
tls-port 6379
port 0                                  # disable the non-TLS port
tls-cert-file /etc/redis/redis.crt
tls-key-file  /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes                    # require client certs (mutual TLS)
```

```bash
redis-cli --tls --cert client.crt --key client.key --cacert ca.crt -h myhost
```

```js
const redis = new Redis({ host: "myhost", port: 6379, tls: { /* ca, cert, key */ } });
```

### Command renaming / disabling (defense in depth)

Hide or rename dangerous commands so a compromised client can't `FLUSHALL`, `CONFIG`, `DEBUG`, etc. (ACLs are the modern, preferred mechanism — `-@dangerous`, `-flushall`.)

```conf
rename-command FLUSHALL ""                 # disable entirely
rename-command CONFIG "CONFIG_a8f3k2"      # obscure it
```

**Security checklist:** bind to private NICs, `requirepass`/ACL users, TLS for remote traffic, drop/deny `FLUSHALL`/`CONFIG`/`DEBUG`/`KEYS` for app users, run as non-root, keep Redis patched, never put it on a public IP, use a firewall/security group.

---

## 15. Vector Search & Redis as a Vector DB

In 2026, Redis is a popular **vector database** for AI/RAG apps: store text/image **embeddings** alongside your data and run **k-nearest-neighbor (KNN)** similarity search with optional metadata filtering — all in the same low-latency store you already use for caching. This is powered by **RediSearch** (the `FT.*` commands).

⚡ **Version note:** Vector search needs the **RediSearch** module. On **Redis 8** it's **in core** (`redis:8`). On Redis 7.x use the **Redis Stack** image. **Valkey** doesn't bundle RediSearch — use a Valkey search module or a dedicated vector DB. Redis supports **FLAT** (exact, brute-force) and **HNSW** (approximate, fast) index types, distance metrics **L2 / IP / COSINE**, and pre-filtering by tags/numbers.

### Create an index and store vectors (on JSON or hashes)

```redis
# Create an index over HASH keys prefixed "doc:". The vector field uses HNSW,
# 3 dimensions (use your model's real dim, e.g. 1536 for OpenAI text-embedding-3-small),
# FLOAT32 elements, COSINE distance. We also index a TAG field for filtering.
FT.CREATE idx:docs
  ON HASH PREFIX 1 doc:
  SCHEMA
    title TEXT
    category TAG
    embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 3 DISTANCE_METRIC COSINE

# Store a document. The vector must be the raw FLOAT32 bytes (your client packs this).
HSET doc:1 title "Intro to Redis" category "db" embedding "<float32-bytes>"
HSET doc:2 title "AI with Redis"   category "ai" embedding "<float32-bytes>"

# KNN query: top 3 nearest to a query vector, COSINE distance returned as __vec_score.
# $vec is a parameter holding the query embedding's raw bytes.
FT.SEARCH idx:docs "*=>[KNN 3 @embedding $vec AS vec_score]"
  PARAMS 2 vec "<float32-bytes>"
  SORTBY vec_score
  RETURN 2 title vec_score
  DIALECT 2

# Hybrid search: filter by category=ai FIRST, then KNN among those (pre-filtering):
FT.SEARCH idx:docs "(@category:{ai})=>[KNN 3 @embedding $vec AS vec_score]"
  PARAMS 2 vec "<float32-bytes>" DIALECT 2
```

### Node.js (node-redis v4+) — real packing & querying

```js
// node-redis has first-class FT.* helpers. Pack JS numbers into a Float32 Buffer.
import { createClient, SchemaFieldTypes, VectorAlgorithms } from "redis";

const client = createClient();
await client.connect();

// Helper: turn an embedding array into raw FLOAT32 little-endian bytes.
const toBytes = (arr) => Buffer.from(new Float32Array(arr).buffer);

// 1) Create the index (idempotent: ignore "Index already exists").
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

// Create index:
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

> **When to use Redis for vectors:** great when you want **one system** for cache + app data + vectors, **low latency**, and **metadata pre-filtering** for RAG. For billions of vectors or specialized ANN features you might still pick a dedicated vector DB — but for most RAG/semantic-search workloads at small-to-medium scale, Redis is an excellent, simple choice.

---

## 16. Observability & Operations

### `INFO` — the dashboard in one command

```redis
INFO                 # everything, grouped into sections
INFO server          # version, uptime, os, process id
INFO clients         # connected_clients, blocked_clients, maxclients
INFO memory          # used_memory, maxmemory, fragmentation_ratio, evicted
INFO stats           # ops/sec, keyspace hits/misses, expired/evicted keys
INFO replication     # role, replicas, offsets, lag
INFO persistence     # rdb/aof state, last save, loading
INFO keyspace        # per-DB key counts and avg TTL
```

Key health metrics to watch:

- **Hit ratio:** `keyspace_hits / (keyspace_hits + keyspace_misses)` — low ratio = cache not helping.
- **`used_memory` vs `maxmemory`** and **`evicted_keys`** — rising evictions = under-provisioned or wrong policy.
- **`mem_fragmentation_ratio`** — well above 1.5 may indicate fragmentation (consider `activedefrag yes`).
- **`connected_clients` / `blocked_clients`** — connection leaks or stuck blocking commands.
- **`instantaneous_ops_per_sec`** and replication **`master_repl_offset`** lag.

### SLOWLOG — find slow commands

```redis
CONFIG SET slowlog-log-slower-than 10000   # log commands slower than 10ms (µs units)
CONFIG SET slowlog-max-len 128             # keep last 128 entries
SLOWLOG GET 10                              # last 10 slow entries (cmd, args, µs, time)
SLOWLOG LEN
SLOWLOG RESET
```

### MONITOR — see every command live (debug only)

```redis
MONITOR     # streams EVERY command the server processes — ⚠ huge perf hit; never leave on in prod
```

### LATENCY — latency spike diagnostics

```redis
CONFIG SET latency-monitor-threshold 100   # record events ≥100ms
LATENCY LATEST       # most recent latency spike per event
LATENCY HISTORY fork # history for a specific event (e.g. fork, expire, command)
LATENCY DOCTOR       # human-readable analysis + advice
LATENCY RESET
```

### Memory analysis

```redis
MEMORY USAGE user:1            # bytes a key consumes (incl. overhead)
MEMORY DOCTOR                  # advice on memory issues
MEMORY STATS                   # detailed breakdown
DEBUG OBJECT user:1            # encoding, serialized length, etc. (debug)
```

```bash
# Sampling tools from the CLI:
redis-cli --bigkeys     # biggest key per type
redis-cli --memkeys     # keys by memory
redis-cli --hotkeys     # most-accessed keys (needs LFU policy)
redis-cli --latency-history   # rolling latency
redis-cli INFO stats | grep -E 'hits|misses|evicted|expired'
```

### Keyspace notifications (react to key events)

Redis can publish events when keys change/expire — useful for cache-invalidation hooks, expired-session cleanup, etc.

```redis
CONFIG SET notify-keyspace-events KEA      # enable all key event notifications
# Then subscribe to the event channels:
PSUBSCRIBE '__keyevent@0__:expired'        # fires when a key in DB 0 expires
PSUBSCRIBE '__keyspace@0__:user:1'         # all events on key user:1
```

---

## 17. Real-World Recipes

### Session store (Express + connect-redis)

```js
// npm install express express-session connect-redis ioredis
import session from "express-session";
import { RedisStore } from "connect-redis";
import Redis from "ioredis";

const redis = new Redis();
app.use(session({
  store: new RedisStore({ client: redis, prefix: "sess:", ttl: 86400 }), // 1 day
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, maxAge: 86400_000 },
}));
// Sessions now live in Redis (sess:<id>), shared across all app instances, auto-expiring.
```

### Leaderboard with rank + neighbors (ZSET)

```js
// Submit a score and read a player's rank plus the 2 players above/below them.
async function submitScore(game, player, score) {
  // GT = only raise the score, never lower it on a worse run:
  await redis.zadd(`lb:${game}`, "GT", score, player);
}

async function aroundPlayer(game, player, window = 2) {
  const key = `lb:${game}`;
  const rank = await redis.zrevrank(key, player);     // 0 = top
  if (rank === null) return null;
  const start = Math.max(0, rank - window);
  const end = rank + window;
  const rows = await redis.zrange(key, start, end, "REV", "WITHSCORES");
  // pair up [member, score, member, score, ...] and attach absolute ranks:
  const out = [];
  for (let i = 0; i < rows.length; i += 2) {
    out.push({ rank: start + i / 2 + 1, player: rows[i], score: Number(rows[i + 1]) });
  }
  return out;
}
```

### Per-IP rate limiter middleware (Express)

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

### Response cache with single-flight (Express)

```js
// Cache GET responses for 60s, preventing stampedes via the §9 getWithLock helper.
function cacheGet(ttl = 60) {
  return async (req, res, next) => {
    if (req.method !== "GET") return next();
    const key = `httpcache:${req.originalUrl}`;
    try {
      const body = await getWithLock(key, async () => {
        // "loader" actually runs the route by capturing res.json once:
        return await fetchFreshPayloadFor(req); // your data-producing function
      }, ttl);
      res.set("X-Cache", "HIT-OR-FILLED").json(body);
    } catch (e) { next(e); }
  };
}
```

### Real-time feed / fan-out (Streams + WebSocket)

```js
// Producer: append events to a per-topic stream (capped).
await redis.xadd("feed:global", "MAXLEN", "~", 1000, "*", "type", "post", "id", postId);

// Consumer: tail the stream and push to connected WebSocket clients.
let lastId = "$"; // start from "now"
async function pump(wss) {
  while (true) {
    const res = await redis.xread("BLOCK", 0, "COUNT", 50, "STREAMS", "feed:global", lastId);
    if (!res) continue;
    const [, entries] = res[0];
    for (const [id, fields] of entries) {
      lastId = id;                                    // advance cursor
      const msg = JSON.stringify(Object.fromEntries(chunk(fields, 2)));
      wss.clients.forEach((c) => c.readyState === 1 && c.send(msg));
    }
  }
}
const chunk = (a, n) => a.reduce((r, _, i) => (i % n ? r : [...r, a.slice(i, i + n)]), []);
```

---

## 18. Gotchas & Best Practices

**Do:**
- ✅ Set a **`maxmemory`** + **eviction policy** on caches (default `noeviction` makes writes fail when full).
- ✅ Always give cache keys a **TTL** — unbounded keys are a silent memory leak.
- ✅ Use **`SCAN`**, never `KEYS`, in production. Use **`UNLINK`** for big deletes.
- ✅ Use a **key naming convention** (`app:type:id:field`) and **hash tags** for cluster-related keys.
- ✅ **Pipeline** bulk ops; use **`MGET`/`MSET`/`HMGET`** to cut round trips.
- ✅ Prefer **Lua scripts** for atomic read-modify-write over `WATCH` retry loops.
- ✅ Run **AOF (everysec) + periodic RDB**; back up RDBs off-box.
- ✅ Use **HA**: replicas + **Sentinel** (single dataset) or **Cluster** (sharded).
- ✅ **Secure it:** bind to private NICs, ACL users, TLS, drop dangerous commands. Never on a public IP.
- ✅ Add **TTL jitter** and **single-flight** locking to prevent cache stampedes.
- ✅ Share **one client/pool** app-wide (go-redis is already a pool); dedicate separate connections for `SUBSCRIBE`/blocking commands.

**Don't / common bugs:**
- ❌ Running `KEYS *`, `SMEMBERS`/`HGETALL`/`LRANGE 0 -1` on huge keys — they **block** the single thread.
- ❌ Expecting **transactions to roll back** — they don't; a runtime error doesn't undo other queued commands.
- ❌ Treating **Redlock** as correctness-grade for money — without fencing tokens it can allow two holders.
- ❌ Relying on **Pub/Sub** for messages that must not be lost — it has no persistence; use Streams.
- ❌ Forgetting that `SET k v` (without `KEEPTTL`) **clears the TTL**.
- ❌ Reading from a **replica** and assuming it's fresh — replication is async (stale reads, possible lost writes on failover).
- ❌ Using **numbered DBs** in Cluster (unsupported) or for "namespacing" (prefer key prefixes).
- ❌ Multi-key commands across **different slots** in Cluster without a shared `{hashtag}` → `CROSSSLOT` error.
- ❌ Leaving **`MONITOR`** running in prod — it cripples throughput.
- ❌ Storing one **giant value** when you only ever need part of it — use a hash or many keys.
- ❌ Exposing Redis with **no auth on `0.0.0.0`** — the #1 way Redis gets pwned.

**Tricks:**
- 🔹 `GETDEL` for one-time tokens; `GETEX` for sliding-TTL reads.
- 🔹 `SET k v NX GET` reads the old value and conditionally sets in one atomic op.
- 🔹 `OBJECT ENCODING k` to confirm Redis is using a compact encoding (listpack/intset) for small structures.
- 🔹 `redis-cli --bigkeys` / `--hotkeys` / `LATENCY DOCTOR` / `MEMORY DOCTOR` for instant diagnostics.
- 🔹 `WAIT numreplicas timeout` after critical writes for stronger (synchronous) durability.
- 🔹 `COPY src dst` + rename for atomic-ish "blue/green" key swaps.
- 🔹 `CLIENT NO-EVICT on` / `CLIENT NO-TOUCH on` for special clients; `CLIENT LIST`/`CLIENT KILL` to manage connections.
- 🔹 Enable **client-side caching** (RESP3 tracking / `CLIENT TRACKING`) to cache hot reads in the app and get invalidations pushed.

---

## 19. Study Path

Learn in this order for fastest offline mastery:

1. **Fundamentals** — install via Docker, live in `redis-cli`, understand the key→data-structure model and RESP (§1).
2. **Data types** — Strings, Hashes, Lists, Sets, Sorted Sets first; they cover 90% of real work. Then Bitmaps/HLL/Geo/Streams/JSON as needed (§2).
3. **Keys & TTL** — expiration, `SCAN` vs `KEYS`, naming conventions (§3). Internalize "always TTL your cache".
4. **Caching** — cache-aside, eviction policies, stampede prevention. This is the #1 production use (§9).
5. **Atomicity** — pipelining vs transactions vs Lua; write a rate limiter and a lock (§6, §7, §10, §11).
6. **Messaging** — Pub/Sub, then Streams + consumer groups; build a job queue (§4, §5, §12).
7. **Persistence & HA** — RDB vs AOF, replication, Sentinel, then Cluster (§8, §13).
8. **Security & ops** — ACLs/TLS, `INFO`/`SLOWLOG`/`LATENCY`/memory tools (§14, §16).
9. **Advanced/2026** — vector search for RAG; explore Redis 8 bundled modules and Valkey (§15).

### Build to learn (do these projects)

1. **Cache layer for an API** — wrap a Postgres-backed REST API with cache-aside, TTL + jitter, and single-flight stampede protection. Measure the hit ratio with `INFO stats`.
2. **Real-time leaderboard service** — ZSET-backed top-N + "rank around me" + live updates over WebSocket; add a per-IP sliding-window rate limiter.
3. **Durable job queue** — Streams + consumer groups with ACK, retries, dead-letter handling, and `XAUTOCLAIM`-based crash recovery; run multiple workers and kill one to watch recovery.
4. **Semantic search / RAG box** — embed documents, index with `FT.CREATE ... VECTOR HNSW`, and serve KNN + tag-filtered queries over HTTP (Redis 8 or Redis Stack).
5. **HA playground** — stand up a 3-node Sentinel setup *and* a 6-node Cluster in Docker Compose; kill primaries and observe failover/resharding from a client.

> Those projects cover the bread and butter of what teams actually use Redis for: caching, ranking/real-time, queues, AI search, and running it reliably. Keep `SET/GET/EXPIRE`, `INCR`, `HSET/HGETALL`, `ZADD/ZRANGE REV`, `LPUSH/BRPOP`, `XADD/XREADGROUP/XACK`, and `SCAN` in muscle memory — they're 80% of daily Redis.
