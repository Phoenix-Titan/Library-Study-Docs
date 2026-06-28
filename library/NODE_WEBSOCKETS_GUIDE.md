# WebSockets in Node.js (Express, Fastify & NestJS) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Node developers going from "I have built HTTP APIs but never opened a WebSocket" all the way to "I can design, secure, and **horizontally scale a real-time service to 50,000+ concurrent connections** without dropping users on deploy." This is a **learn-offline study guide**: every concept is explained in *prose first* — what it is, the underlying logic and *why* it works that way, what it is for and when to reach for it, the key options and parameters, best practices, and explicit **security** recommendations — and only *then* shown as heavily-commented, runnable code. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. It assumes you already know the Node.js *language and runtime* — the event loop, streams, `EventEmitter`, and graceful shutdown — so if any of that is shaky, read the **[Node.js](NODEJS_GUIDE.md)** guide first. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Node.js 24 LTS** (current LTS in 2026), **`ws` v8** (the de-facto low-level WebSocket library), **Socket.IO v4** (rooms, reconnection, fallbacks, the `@socket.io/redis-adapter` backplane), **`uWebSockets.js`** (the C++-backed server for extreme scale, the path to 100k+ connections per box), **NestJS 11** (WebSocket *gateways*), **Fastify v5** with **`@fastify/websocket`**, **Express 5**, **Redis** via **`ioredis`**, and **TypeScript 5.x**. Code is **TypeScript + ESM first** with plain-JS notes where useful. Where an API moves fast or changed between majors, look for the **⚡ Version note** flag and confirm against the version your `package.json` actually resolves.
>
> **How this fits the rest of the library:** WebSockets begin life as an HTTP request, so the **[Networking](NETWORKING_GUIDE.md)** guide (TCP, TLS, the WS protocol on the wire, the C10k/C10M story) and the **[Node.js](NODEJS_GUIDE.md)** guide (event loop, streams) are the foundation. For the framework hosts see **[Fastify](FASTIFY_GUIDE.md)** and **[NestJS](NESTJS_GUIDE.md)**. The multi-instance scaling backplane is **[Redis](REDIS_GUIDE.md)**; the edge/load-balancer is **[Nginx](NGINX_GUIDE.md)**; deployment is **[Docker](DOCKER_GUIDE.md)**. Socket authentication mirrors the token concepts in **[Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md)**, and the **[Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md)** guide is the direct Go counterpart — compare the goroutine-per-connection model against Node's single-threaded event loop throughout.

---

## Table of Contents

1. [The WebSocket Protocol & When to Use It](#1-the-websocket-protocol--when-to-use-it) **[B]**
2. [The Node Real-Time Landscape — ws, Socket.IO, uWebSockets.js, Fastify, NestJS](#2-the-node-real-time-landscape--ws-socketio-uwebsocketsjs-fastify-nestjs) **[B]**
3. [Raw `ws` — Server, Client & Broadcast](#3-raw-ws--server-client--broadcast) **[B/I]**
4. [Authenticating the Socket — Tokens, Origin & CSWSH](#4-authenticating-the-socket--tokens-origin--cswsh) **[I/A]**
5. [Connection Lifecycle & Reliability — Heartbeats, Backpressure, Shutdown](#5-connection-lifecycle--reliability--heartbeats-backpressure-shutdown) **[I/A]**
6. [Designing the Message Protocol](#6-designing-the-message-protocol) **[I/A]**
7. [Express 5 + ws](#7-express-5--ws) **[I]**
8. [Fastify v5 + @fastify/websocket](#8-fastify-v5--fastifywebsocket) **[I]**
9. [NestJS 11 Gateways](#9-nestjs-11-gateways) **[I/A]**
10. [Socket.IO in Depth — Rooms, Namespaces, Acks, Middleware](#10-socketio-in-depth--rooms-namespaces-acks-middleware) **[I/A]**
11. [Scaling to 50,000+ Concurrent Connections](#11-scaling-to-50000-concurrent-connections) **[A]**
12. [Benchmarking & Load Testing to 50k](#12-benchmarking--load-testing-to-50k) **[A]**
13. [Production Hardening & Banking-Grade Security](#13-production-hardening--banking-grade-security) **[A]**
14. [Gotchas & Best Practices](#14-gotchas--best-practices) **[I/A]**
15. [Study Path & Build-to-Learn Projects](#15-study-path--build-to-learn-projects)

---

## 1. The WebSocket Protocol & When to Use It

### 1.1 The problem WebSockets solve **[B]**

Plain HTTP is a **request–response** protocol with one iron rule: *the client always speaks first.* The browser asks (`GET /messages`), the server answers, and the exchange is over. The server cannot spontaneously say "hey, something just happened." For load-a-page-submit-a-form apps that is perfect. But for anything *live* — chat, a multiplayer game, a trading or banking dashboard, collaborative editing, live cursors, a notification bell — the server constantly needs to **push** data to a client that did not ask for it at that instant. HTTP cannot do that natively.

Before WebSockets, developers faked server-push with progressively painful hacks. Knowing them is the *why* behind WebSockets:

- **Short polling** — the client asks "anything new?" on a timer (every few seconds). Simple, but wasteful: most responses are "nothing new," and each pays the full cost of a new HTTP request (headers, often a fresh TCP/TLS handshake, routing). Latency is bounded by the interval — 1 s hammers the server; 30 s feels dead.
- **Long polling** — the client requests and the server *holds it open* until it has something to say, then responds; the client immediately reconnects. Lower latency and waste than short polling, but you still pay a full HTTP round-trip per message and tie up a server slot holding the request open. A clever hack, not a real channel.
- **Server-Sent Events (SSE)** — a standardized **one-way** stream from server → client over a single long-lived HTTP response (`text/event-stream`). Genuinely good when data flows *only* server → client (live scores, notification feeds, log tailing). Simpler than WebSockets, auto-reconnects, rides normal HTTP/1.1 and HTTP/2. But it is unidirectional — the client still needs a separate HTTP request to send anything back, and over HTTP/1.1 browsers cap it at ~6 connections per host.

**WebSocket** is the real answer: a **persistent, full-duplex** connection. After a one-time handshake a single TCP connection stays open and *both sides send messages at any time*, independently, with tiny per-message overhead (about 2–14 bytes of framing versus the hundreds of bytes of headers an HTTP request re-sends every time).

### 1.2 HTTP vs SSE vs WebSocket at a glance **[B]**

| Property | HTTP (poll/long-poll) | Server-Sent Events | WebSocket |
|---|---|---|---|
| Connection lifetime | New per request (or held) | One long-lived response | Persistent until closed |
| Direction | Client → Server (server replies) | Server → Client only | **Full-duplex (both ways)** |
| Who can initiate a message | Client only | Server only | **Either side, anytime** |
| Per-message overhead | High (full HTTP headers) | Low after first | **Very low (2–14 byte frame)** |
| Latency | Round-trip / poll interval | Low | **Minimal (persistent TCP)** |
| Binary support | Awkward | No (UTF-8 text only) | **Yes (text *and* binary)** |
| Auto-reconnect built in | No | Yes (`EventSource`) | No (you implement it, or Socket.IO does) |
| Browser API | `fetch` / `XMLHttpRequest` | `EventSource` | `WebSocket` |
| Works through dumb proxies | Yes | Mostly | Needs `Upgrade` pass-through |

### 1.3 When to use which — the decision **[B]**

Choosing WebSockets when you don't need them adds real operational cost (sticky load balancing, heartbeats, a scaling backplane). Be deliberate.

**Reach for WebSockets when:** messages flow in **both** directions and timing matters — chat, multiplayer games, collaborative editing, live trading, presence/typing indicators; or the client must push *frequently* on the same logical channel the server pushes on.

**Use SSE when:** the data is essentially **server → client only** (dashboards, feeds, progress, log streaming) and you want the simplest thing that auto-reconnects. SSE rides plain HTTP, so it sails through proxies and needs no sticky-session backplane gymnastics.

**Stay on plain HTTP / polling when:** updates are infrequent or the user does not need them in real time. A "new mail" badge that refreshes on navigation does not need a socket.

### 1.4 The upgrade handshake — how a WebSocket is born **[B]**

A WebSocket connection *starts as an ordinary HTTP/1.1 GET request* and is then "upgraded" in place onto the same TCP socket. This is the single most important protocol fact to internalize, because every auth, proxy, and load-balancer decision flows from it.

The client sends a normal-looking GET with special headers:

```http
GET /chat HTTP/1.1
Host: realtime.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://app.example.com
```

- `Upgrade: websocket` + `Connection: Upgrade` ask the server to switch protocols on this connection.
- `Sec-WebSocket-Key` is a random 16-byte base64 nonce. It is **not** security — it just proves the server actually understands WebSockets (a caching proxy that blindly replays a 200 would fail the check).
- `Sec-WebSocket-Version: 13` is the only version in use today.
- `Origin` is set **by the browser** and tells you which web origin opened the socket — the cornerstone of CSWSH defense (§4).

The server computes `SHA1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11")`, base64-encodes it, and replies:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the `101`, the HTTP framing is gone — the TCP socket now carries **WebSocket frames** in both directions until either side sends a Close frame. In Node, libraries do all of this for you, but you hook into the moment *before* the `101` is sent — the **`upgrade` event** — to authenticate and authorize the connection (§4). That is the right place to reject a bad client cheaply, before you commit any per-connection memory.

### 1.5 Frames, opcodes & control messages **[B/I]**

Once upgraded, all data travels in **frames**. A frame has a tiny header (the FIN bit, an opcode, a mask bit, and a length field that is 7, 7+16, or 7+64 bits) followed by the payload. You rarely touch frames directly, but the opcodes shape the API you *do* use:

| Opcode | Name | Meaning |
|---|---|---|
| `0x0` | Continuation | A fragment continuing a previous message |
| `0x1` | Text | UTF-8 text payload → arrives as a `string` |
| `0x2` | Binary | Binary payload → arrives as a `Buffer`/`ArrayBuffer` |
| `0x8` | Close | Begin the closing handshake (optional code + reason) |
| `0x9` | Ping | Heartbeat request (control frame) |
| `0xA` | Pong | Heartbeat reply (control frame) |

Two consequences matter in practice:
- **Messages, not bytes.** Unlike a raw TCP stream, WebSockets are *message-framed*: one `send` on the client becomes exactly one `message` event on the server (libraries reassemble fragments for you). You never have to parse a length prefix yourself.
- **Ping/Pong are first-class.** The protocol has built-in heartbeats. The server sends Ping; a healthy client's stack auto-replies Pong. This is how you detect a peer whose network vanished without a TCP FIN (a laptop lid closing, a phone losing signal). You *must* use this to reap dead connections at scale (§5) — without it, dead sockets pile up and consume memory and file descriptors until you fall over.

**Client-to-server frames are masked** (XOR'd with a per-frame key) by spec; servers reject unmasked client frames. This is purely an anti-cache-poisoning measure for intermediaries — `ws` handles it transparently. Server-to-client frames are *not* masked.

### 1.6 Why Node is good at *many idle* connections **[B]**

Here is the core reason a single Node process can hold tens of thousands of WebSockets. A WebSocket connection that is just *sitting there* — open but not actively transferring — costs almost nothing in Node because Node does **not** dedicate a thread (or an OS-level blocking wait) per connection. Node runs your JavaScript on **one thread** driven by an **event loop** over an OS readiness mechanism (`epoll` on Linux, `kqueue` on BSD/macOS). Ten thousand idle sockets are ten thousand file descriptors the kernel watches; the event loop only wakes your code when one of them actually has data. Idle connections cost memory (socket buffers + the small per-connection JS object) but **zero CPU**.

Contrast the classic *thread-per-connection* server: 10,000 connections = 10,000 threads, each with a multi-megabyte stack, and the OS scheduler thrashing between them. That model hits a wall in the low thousands — the original **C10k problem** (handling 10,000 concurrent clients), which event-driven I/O was invented to solve. Go's goroutine-per-connection model (see **[Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md)**) is a middle ground: cheap user-space "threads" multiplexed onto a small thread pool. Node's single event loop is even leaner *for idle connections* but has one hard rule that dominates everything in this guide:

> **Never block the event loop.** There is exactly one thread running your JS. A synchronous CPU-heavy loop, a `JSON.parse` of a 50 MB payload, or a sync crypto call freezes *every* connection at once. The cost of a slow handler is paid by *all 50,000 users simultaneously*. Keep per-message work tiny and asynchronous; offload heavy CPU to worker threads or another service.

The journey from C10k to **C10M** (ten *million* connections) — see the **[Networking](NETWORKING_GUIDE.md)** guide — is the story of pushing this model with kernel tuning, multiple processes, and lean per-connection memory. We walk the practical 50k+ version of that story in §11.

---

## 2. The Node Real-Time Landscape — ws, Socket.IO, uWebSockets.js, Fastify, NestJS

### 2.1 The five players **[B]**

There is no single "Node WebSocket" — there is a small ecosystem, and choosing wrong early costs you a rewrite. Here is what each is and the *logic* of when to pick it.

- **`ws`** — the low-level, fast, minimal WebSocket implementation. It speaks the raw protocol and nothing more: you get `connection`, `message`, `close`, `error` and you `send`. No rooms, no reconnection, no fallbacks. It is the library *other* libraries are built on, and it is what you want when you need control and lean memory. **This is the default building block.**
- **Socket.IO** — a *framework* on top of WebSockets (with a polling fallback). It adds the features real apps need but that raw `ws` makes you build yourself: **automatic reconnection** with backoff, **rooms** and **namespaces**, **acknowledgements** (request/response over the socket), **broadcast**, connection-state recovery, and a battle-tested **Redis adapter** for multi-instance scaling. It is *not* raw WebSocket — it has its own wire protocol (Engine.IO underneath), so a Socket.IO client must talk to a Socket.IO server. You trade some bytes-on-the-wire and a slightly heavier handshake for a lot of features.
- **`uWebSockets.js`** — a Node binding over a hand-tuned C++ WebSocket/HTTP server. It is *dramatically* more memory- and CPU-efficient than `ws` (often an order of magnitude less memory per connection and far higher throughput). It is the realistic path to **100k+ connections on a single box**. The cost: a different, lower-level API, native-addon deployment concerns, and you build more yourself. Socket.IO can optionally run *on top of* it for the best of both.
- **`@fastify/websocket`** — the official Fastify plugin that mounts `ws` onto a Fastify route, so a WebSocket endpoint lives next to your HTTP routes and shares Fastify's hooks, auth, and lifecycle. Under the hood it *is* `ws`. Pick it when your app is already Fastify (see **[Fastify](FASTIFY_GUIDE.md)**).
- **NestJS gateways** — Nest's structured, decorator-driven abstraction (`@WebSocketGateway`, `@SubscribeMessage`) that plugs into Nest's DI, guards, pipes, and interceptors. You choose an underlying adapter: the **`ws` adapter** or the **Socket.IO adapter**. Pick it when your app is already Nest and you want WS handlers that look and test like the rest of your Nest code (see **[NestJS](NESTJS_GUIDE.md)**).

### 2.2 Comparison table — how to choose **[B]**

| Library | Layer | Rooms / namespaces | Auto-reconnect | Acks (req/resp) | Fallback (polling) | Relative mem/connection | Raw-WS compatible client | Best for |
|---|---|---|---|---|---|---|---|---|
| **`ws`** | Low-level | ❌ (you build it) | ❌ (client builds it) | ❌ (you build it) | ❌ | Low | ✅ | Control, leanness, embedding in a framework |
| **Socket.IO** | Framework | ✅ built-in | ✅ built-in | ✅ built-in | ✅ long-polling | Higher | ❌ (needs Socket.IO client) | Feature-rich apps, fast delivery, flaky networks |
| **`uWebSockets.js`** | Native (C++) | ✅ pub/sub topics | ❌ | ❌ | ❌ | **Lowest** | ✅ | Extreme scale (100k+/box), max throughput |
| **`@fastify/websocket`** | `ws` + Fastify | ❌ (ws under it) | ❌ | ❌ | ❌ | Low | ✅ | Fastify apps |
| **NestJS gateway** | ws *or* socket.io | depends on adapter | depends on adapter | ✅ via return value | depends on adapter | depends on adapter | depends on adapter | Nest apps, structured handlers |

### 2.3 Choosing for 50,000+ connections **[B/I]**

The scale target changes the calculus. For 50k+ concurrent connections:

- **If you need rooms/reconnection/acks and reasonable scale:** **Socket.IO + `@socket.io/redis-adapter`**, behind a load balancer with **sticky sessions**, sharded across several instances. This is the most common production answer because it gives you the features *and* a proven horizontal-scaling story. Budget more memory per connection than raw `ws`.
- **If you need maximum density per box and lean memory:** **`uWebSockets.js`** (optionally with Socket.IO bolted on for its features). This is how you hit 100k+ on one machine and keep your instance count — and cloud bill — down.
- **If you want full control and a custom protocol:** **raw `ws` + a Redis pub/sub backplane** you write yourself (§11). Most flexible, leanest with the standard `ws`, but you own reconnection, rooms, and fan-out logic.
- **Inside an existing framework:** use **`@fastify/websocket`** or a **NestJS gateway**, but know that the *scaling* concerns (sticky sessions, Redis backplane, FD limits, heartbeats) are identical regardless of the host — they live below the framework.

> **Rule of thumb:** start with `ws` or Socket.IO to learn and ship; reach for `uWebSockets.js` only when measurements (not guesses — see §12) prove you need the density. Premature `uWebSockets.js` trades developer velocity for headroom you may not use.

⚡ **Version note:** `ws` v8 requires Node 12+ and is fully stable on Node 24. Socket.IO v4 changed the default to **no fallback unless configured** in some setups and ships first-class TypeScript; the Redis adapter moved to `@socket.io/redis-adapter` (the old `socket.io-redis` is deprecated). `uWebSockets.js` is distributed from GitHub (not a normal npm tarball) and is pinned per Node ABI — check its README for the tag matching Node 24.

---

## 3. Raw `ws` — Server, Client & Broadcast

### 3.1 Why start with `ws` **[B]**

Even if you ship Socket.IO or NestJS later, learn `ws` first: it is the thinnest possible layer over the protocol, so every concept maps directly to what's on the wire — there is no magic to confuse cause and effect. It is also what `@fastify/websocket` and the Nest `ws` adapter use internally, so this knowledge transfers.

Install it:

```bash
npm install ws
npm install -D @types/ws typescript tsx   # types + a TS runner
```

### 3.2 The mental model: one connection = one handler **[B]**

This is the single most important `ws` pattern. `WebSocketServer` is an `EventEmitter`. When a client connects it emits **`connection`** with a fresh `WebSocket` object representing *that one client*. You attach `message`/`close`/`error` listeners **inside** the `connection` handler, so each socket gets its own closure of listeners and its own per-connection state. There is no global "current request" — every connection is independent and long-lived.

```ts
// server.ts — the canonical ws echo + broadcast server, heavily commented.
import { WebSocketServer, WebSocket } from 'ws';

// `port` makes ws create its OWN http.Server. In real apps you usually share
// an existing server instead (see 3.5) so HTTP and WS live on one port.
const wss = new WebSocketServer({
  port: 8080,
  // maxPayload caps a single inbound message in BYTES. CRITICAL for security:
  // without it, one client can send a multi-GB frame and OOM the process.
  maxPayload: 1024 * 1024,        // 1 MiB hard cap per message
  // perMessageDeflate compresses messages. It SAVES bandwidth but COSTS CPU
  // and memory per connection — at 50k connections it can be a memory sink,
  // so it is OFF here. Turn it on only with measured benefit (see §11).
  perMessageDeflate: false,
});

// 'connection' fires once per client. `ws` is THIS client's socket.
// `req` is the original Node http.IncomingMessage from the upgrade — your
// only access to headers, URL, and the remote IP (used for auth in §4).
wss.on('connection', (ws: WebSocket, req) => {
  const ip = req.socket.remoteAddress;
  console.log(`client connected from ${ip}; total=${wss.clients.size}`);

  // Per-connection state lives in this closure (or attach to `ws` itself).
  (ws as any).isAlive = true;     // used by the heartbeat in §5

  // 'message' fires for every inbound frame. In ws v8, `data` is a Buffer by
  // default (or an array of Buffers if fragmented). `isBinary` tells you
  // whether the client sent a text or binary frame.
  ws.on('message', (data: Buffer, isBinary: boolean) => {
    // NEVER trust this input. Validate before use (see §6). Here we echo and
    // broadcast a small text message.
    const text = data.toString('utf8');
    console.log(`recv: ${text}`);

    // Echo back to the sender only:
    ws.send(`echo: ${text}`, { binary: isBinary });

    // Broadcast to everyone EXCEPT the sender. wss.clients is a Set of all
    // connected sockets. ALWAYS check readyState — a socket mid-close throws
    // if you send to it.
    for (const client of wss.clients) {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(text, { binary: isBinary });
      }
    }
  });

  // 'pong' is the heartbeat reply — see §5 for the full reaper.
  ws.on('pong', () => { (ws as any).isAlive = true; });

  // 'close' fires once, when the connection ends (clean or not). This is your
  // ONLY chance to clean up per-connection resources (timers, room membership,
  // DB subscriptions). Forgetting this is the #1 memory leak in WS servers.
  ws.on('close', (code, reason) => {
    console.log(`closed code=${code} reason=${reason.toString()}`);
    // remove from rooms, clear intervals, etc.
  });

  // 'error' fires on protocol/socket errors. If you do NOT attach an 'error'
  // listener, an error becomes an UNCAUGHT EXCEPTION that crashes the process.
  // Always attach one, even if it only logs.
  ws.on('error', (err) => console.error('ws error:', err));

  // Greet the new client.
  ws.send(JSON.stringify({ type: 'welcome', clients: wss.clients.size }));
});

// Server-level errors (e.g. EADDRINUSE) come here, separate from per-socket.
wss.on('error', (err) => console.error('server error:', err));
console.log('ws server listening on :8080');
```

Run it with `npx tsx server.ts`.

### 3.3 The `send` API and `readyState` **[B/I]**

`ws.send(data, [options], [callback])` accepts a `string`, `Buffer`, `ArrayBuffer`, or typed array. Two things you must respect:

- **`readyState`** is one of `CONNECTING (0)`, `OPEN (1)`, `CLOSING (2)`, `CLOSED (3)`. Only send when `OPEN`. Sending in any other state throws or is silently dropped depending on state — always guard with `if (ws.readyState === WebSocket.OPEN)`.
- **The send callback** fires when the data has been *handed to the OS* (flushed from Node's internal buffer), not when the client received it. It is how you detect backpressure and write errors (§5). At scale you care deeply about this.

### 3.4 The browser & Node clients **[B]**

The **browser** client is built in — no library:

```js
// browser.js — runs in the page. The native WebSocket API.
const ws = new WebSocket('wss://realtime.example.com/chat');

ws.addEventListener('open',  () => ws.send('hello from the browser'));
ws.addEventListener('message', (ev) => console.log('server said:', ev.data));
ws.addEventListener('close', (ev) => console.log('closed', ev.code, ev.reason));
ws.addEventListener('error', () => console.log('socket error'));
// NOTE: the browser cannot set custom headers (e.g. Authorization) on this
// request — that constraint drives the auth design in §4.
```

A **Node** client (for tests, load generators, or service-to-service) uses `ws` as a client:

```ts
// client.ts — a Node WebSocket client.
import { WebSocket } from 'ws';

// Unlike the browser, the Node client CAN set headers — handy for service auth.
const ws = new WebSocket('wss://realtime.example.com/chat', {
  headers: { Authorization: `Bearer ${process.env.TOKEN}` },
});

ws.on('open',    () => ws.send('hello from node'));
ws.on('message', (data) => console.log('server:', data.toString()));
ws.on('close',   (c, r) => console.log('closed', c, r.toString()));
ws.on('error',   (e) => console.error(e));
```

### 3.5 Sharing a port with an existing HTTP server **[B/I]**

In production you almost never run WS on its own port. You attach it to your existing `http.Server` and **handle the `upgrade` event yourself**, so HTTP and WebSocket share one port (one TLS cert, one LB target, one firewall rule). This is also *where authentication belongs* (§4): you authenticate during `upgrade`, before `ws` ever emits `connection`.

```ts
// shared-server.ts — one port for HTTP and WS, with a manual upgrade path.
import http from 'node:http';
import { WebSocketServer, WebSocket } from 'ws';

const server = http.createServer((req, res) => {
  // Normal HTTP routes live here.
  res.writeHead(200, { 'content-type': 'text/plain' });
  res.end('HTTP and WebSocket share this port.\n');
});

// `noServer: true` means: do NOT listen on a port and do NOT auto-handle
// upgrades. WE decide when to "complete" the WebSocket handshake. This is the
// key to authenticating BEFORE accepting the socket.
const wss = new WebSocketServer({ noServer: true, maxPayload: 1024 * 1024 });

// The raw Node 'upgrade' event: fired for every Upgrade: websocket request,
// BEFORE any WebSocket exists. `socket` is the raw TCP socket; `head` is any
// bytes already read.
server.on('upgrade', (req, socket, head) => {
  // (Auth + Origin checks go here — see §4. If they fail:)
  //   socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n'); socket.destroy(); return;

  // Only AFTER the request passes do we complete the handshake and create the
  // WebSocket. handleUpgrade emits 'connection' on success.
  wss.handleUpgrade(req, socket, head, (ws) => {
    wss.emit('connection', ws, req);
  });
});

wss.on('connection', (ws: WebSocket, req) => {
  ws.on('message', (data) => ws.send(`echo: ${data}`));
  ws.on('error', (e) => console.error(e));
});

server.listen(8080, () => console.log('listening on :8080 (HTTP + WS)'));
```

This `noServer` + manual `upgrade` pattern is the foundation for everything secure and scalable in this guide — keep it in mind.

---

## 4. Authenticating the Socket — Tokens, Origin & CSWSH

### 4.1 Why WebSocket auth is different **[I]**

Authentication on a WebSocket is genuinely *harder* than on a normal HTTP request, for one structural reason: **the browser's `WebSocket` constructor cannot set request headers.** You can do `new WebSocket(url)` and `new WebSocket(url, protocols)` — that's it. There is no place to put `Authorization: Bearer …`. So the usual "send a bearer token in a header" pattern is unavailable to browser clients. (Node clients *can* set headers, which is why service-to-service auth is easy and browser auth is the interesting case.)

You also must decide *when* to authenticate. The right answer is **during the `upgrade` handshake, before you accept the socket** — reject unauthorized clients with a `401` and `socket.destroy()` so you never allocate per-connection memory for an attacker. Authenticating *after* `connection` (waiting for an in-band "login" message) means you hold an open, unauthenticated socket — fine for some designs, but it gives floods a cheaper target and complicates rate limiting. Prefer handshake-time auth.

### 4.2 The four ways to pass a token (and their trade-offs) **[I/A]**

| Method | How | Pros | Cons / security notes |
|---|---|---|---|
| **Query string** | `wss://host/ws?token=JWT` | Works everywhere, simplest | Token may land in **access logs / proxy logs / browser history** — use short-lived tokens; never log the raw URL |
| **Subprotocol header** | `new WebSocket(url, ['bearer', token])` | Browser *can* set this; not a normal URL | Abuses `Sec-WebSocket-Protocol`; server must echo a chosen subprotocol; awkward but log-safe |
| **Cookie** | Browser auto-sends the session cookie on the upgrade | No JS token handling; httpOnly cookie is XSS-safe | **Requires CSRF/Origin defense (CSWSH, §4.4)**; cookie must be `SameSite` aware |
| **Header (Node clients only)** | `headers: { Authorization }` | Clean, standard | **Not available in browsers** — service-to-service only |

For browser apps the pragmatic, secure choice is usually **a short-lived JWT in the query string** (issued by a normal authenticated HTTP endpoint, valid for ~30–60 s, single-use if you can), *or* an **httpOnly session cookie plus strict Origin checking**. The query-string token avoids the CSWSH problem entirely (an attacker's page cannot read your token), at the cost of log hygiene. The cookie approach is XSS-safe but **must** be paired with Origin checking because cookies are sent automatically cross-site.

### 4.3 Validating the token on `upgrade` **[I/A]**

Here is the secure handshake: parse the token, verify it, look up the user, and only then complete the upgrade — attaching the authenticated identity to the socket so every later message knows *who* sent it.

```ts
// auth-upgrade.ts — authenticate during the handshake, before accepting.
import http from 'node:http';
import { WebSocketServer, WebSocket } from 'ws';
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET!;            // from a secret manager
const ALLOWED_ORIGINS = new Set(['https://app.example.com']);

interface AuthedWS extends WebSocket {
  userId: string;
  // per-connection bookkeeping used elsewhere:
  isAlive: boolean;
}

const server = http.createServer();
const wss = new WebSocketServer({ noServer: true, maxPayload: 256 * 1024 });

server.on('upgrade', (req, socket, head) => {
  // Helper to reject cheaply and consistently.
  const reject = (code: number, msg: string) => {
    socket.write(`HTTP/1.1 ${code} ${msg}\r\nConnection: close\r\n\r\n`);
    socket.destroy();
  };

  try {
    // 1) STRICT ORIGIN CHECK FIRST — see 4.4. The browser sets Origin; a
    //    non-browser client can forge it, but combined with a token this stops
    //    Cross-Site WebSocket Hijacking from real victim browsers.
    const origin = req.headers.origin;
    if (origin && !ALLOWED_ORIGINS.has(origin)) return reject(403, 'Forbidden');

    // 2) Extract the token. We support ?token= here. WHATWG URL needs a base.
    const url = new URL(req.url ?? '', `http://${req.headers.host}`);
    const token = url.searchParams.get('token');
    if (!token) return reject(401, 'Unauthorized');

    // 3) Verify signature + expiry. jwt.verify THROWS on any failure.
    const payload = jwt.verify(token, JWT_SECRET, {
      algorithms: ['HS256'],            // pin the algorithm — never trust 'none'
    }) as { sub: string };

    // 4) Complete the handshake and stamp identity onto the socket.
    wss.handleUpgrade(req, socket, head, (ws) => {
      const authed = ws as AuthedWS;
      authed.userId = payload.sub;
      authed.isAlive = true;
      wss.emit('connection', authed, req);
    });
  } catch {
    // Expired/invalid token, bad URL, etc. Give nothing away.
    return reject(401, 'Unauthorized');
  }
});

wss.on('connection', (ws: AuthedWS) => {
  // From here on, ws.userId is trustworthy. Use it to scope rooms, authorize
  // actions, and tag logs (never log the token itself).
  ws.on('message', (data) => {
    // every message is already attributable to ws.userId
  });
  ws.on('error', (e) => console.error('socket error for', ws.userId, e));
});

server.listen(8080);
```

⚡ **Version note:** always pass `algorithms: [...]` to `jwt.verify`. Omitting it historically allowed the `alg: none` and RS/HS confusion attacks. With Argon2/JWT auth patterns, mirror the **[Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)** guide: short access-token TTLs, rotation, and verifying the *current* token, not a cached decode.

### 4.4 CSWSH — Cross-Site WebSocket Hijacking — and strict Origin checking **[I/A]**

This is the WebSocket equivalent of CSRF and it is *the* WebSocket-specific vulnerability you must understand. The setup: your auth is a **cookie** that the browser sends automatically. An attacker hosts `https://evil.com` and runs `new WebSocket('wss://your-bank.example.com/ws')` from the victim's logged-in browser. The browser **attaches your cookie** to that upgrade request — and now `evil.com` has an authenticated socket to your server, able to read and send messages as the victim.

The defenses, in order:

1. **Check the `Origin` header on every upgrade and allow only your own origins.** Unlike `fetch`, the WebSocket handshake is *not* subject to CORS preflight — the browser will happily open the socket — but it *does* send a truthful `Origin` for browser-initiated connections, and a browser cannot forge it. Rejecting unknown origins (as in §4.3 step 1) blocks the attack at the door. This is **mandatory** whenever you use cookie auth.
2. **Prefer a token you must explicitly attach** (query/subprotocol). `evil.com` cannot read your token (it's not its cookie and SOP blocks reading your storage), so it cannot forge an authenticated socket even if it opens one.
3. **Use a CSRF-style nonce** for cookie auth: issue a one-time `wsTicket` from an authenticated HTTP call, require it as `?ticket=`, and consume it server-side. Combines cookie convenience with token-style protection.

> **Banking-grade rule:** never rely on cookies alone. Use **Origin allow-listing AND an explicit token/ticket**. Treat a missing `Origin` from a context you expect to be a browser as suspicious. Default-deny: only listed origins pass.

### 4.5 Token expiry on long-lived sockets **[A]**

A WebSocket can stay open for hours; a JWT might expire in 15 minutes. What happens then? Nothing automatic — the socket stays open with a now-expired token, which for sensitive apps is a problem. Two patterns:

- **Re-auth over the socket:** the client periodically sends a fresh token in a `{type:"auth", token}` message; the server re-verifies and updates `ws.userId`/expiry. If the client fails to refresh before a hard deadline, the server closes the socket with a specific code (e.g. `4001 token expired`) so the client knows to reconnect after refreshing.
- **Hard session ceiling:** regardless of refresh, force a reconnect every N minutes/hours to re-establish identity from scratch. Good for high-security contexts where a stolen long-lived socket is unacceptable.

Track the token's `exp` server-side per connection and run it through the same heartbeat timer that reaps dead sockets (§5).

---

## 5. Connection Lifecycle & Reliability — Heartbeats, Backpressure, Shutdown

This section is where a toy server becomes a *reliable* one. Every item here is a real failure mode at scale.

### 5.1 Heartbeats — detecting dead peers **[I/A]**

The hard truth about TCP: when a client's network *vanishes* — laptop lid closes, phone goes through a tunnel, a NAT silently drops the mapping — your server often gets **no notification at all**. There is no FIN packet. The socket sits there `OPEN` forever, consuming a file descriptor, a slot in `wss.clients`, and whatever per-connection memory you allocated. At 50k connections, thousands of these "ghost" sockets accumulate and you slowly leak yourself to death.

The fix is the protocol's built-in **Ping/Pong**. The server sends a Ping frame on a timer; a live client's stack auto-replies with Pong (no app code needed on the client). If a client misses a Pong cycle, it's dead — terminate it and free the resources. The canonical `ws` reaper:

```ts
// heartbeat.ts — reap dead connections. Add to any ws server.
import { WebSocketServer, WebSocket } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  (ws as any).isAlive = true;
  // When the client replies to our Ping, mark it alive again.
  ws.on('pong', () => { (ws as any).isAlive = true; });
});

// Every 30s: anyone who hasn't ponged since the last sweep is dead.
const HEARTBEAT_MS = 30_000;
const interval = setInterval(() => {
  for (const ws of wss.clients) {
    if ((ws as any).isAlive === false) {
      // No pong since last sweep → terminate (hard close, no handshake).
      ws.terminate();          // frees the FD immediately
      continue;
    }
    (ws as any).isAlive = false; // assume dead until next pong proves otherwise
    ws.ping();                   // ask "are you there?"
  }
}, HEARTBEAT_MS);

// CRITICAL: stop the timer when the server closes, or it keeps the process up.
wss.on('close', () => clearInterval(interval));
```

The two-phase flip (`isAlive = false` then wait for `pong` to set it `true`) is the whole trick: a client has one full interval to answer before it's reaped. Tune `HEARTBEAT_MS` to balance detection speed against ping traffic — 30 s is a sane default; sub-10 s only if you must detect drops fast. **Clients should *also* heartbeat** (or simply detect missing server activity) so they can reconnect when *they* are the ones cut off.

### 5.2 Idle timeouts **[I]**

Separate from liveness is *usefulness*: a connection that is alive but has sent nothing for an hour may not be worth keeping at 50k scale. Track `lastSeen` per connection (update it in `message`), and in the same sweep close sockets idle beyond your policy with a clear close code. This is also a mild DoS defense — it bounds how long an attacker can sit on an idle, authenticated socket.

### 5.3 Client reconnection with backoff **[I/A]**

The native browser `WebSocket` does **not** reconnect — when the connection drops, it's gone, and `onclose` fires. A production client must reconnect itself, with **exponential backoff and jitter** so that when your server restarts, 50,000 clients don't all reconnect in the same millisecond and knock it over again (the "thundering herd").

```js
// reconnecting-client.js — robust browser client.
function connect(url, onMessage) {
  let attempts = 0;
  let ws;

  const open = () => {
    ws = new WebSocket(url);

    ws.addEventListener('open', () => {
      attempts = 0;                       // reset backoff on success
    });
    ws.addEventListener('message', (ev) => onMessage(ev.data));
    ws.addEventListener('close', (ev) => {
      // Don't reconnect on auth failures we shouldn't retry.
      if (ev.code === 4001 /* token expired */) { refreshTokenThen(open); return; }
      // Exponential backoff capped at 30s, plus random jitter.
      const base = Math.min(30_000, 1000 * 2 ** attempts++);
      const jitter = Math.random() * base * 0.3;
      setTimeout(open, base + jitter);
    });
    ws.addEventListener('error', () => ws.close());
  };

  open();
  return () => ws && ws.close();
}
```

Socket.IO does all of this for you — backoff, jitter, and even buffering of messages sent while disconnected — which is a big part of why teams choose it for flaky-network apps (mobile).

### 5.4 Backpressure — the slow-consumer problem **[I/A]**

This is the reliability issue that bites hardest at scale, and it is subtle. When you `ws.send()`, the data goes into Node's internal socket buffer and is written to the OS as fast as the *client's* network can drain it. If a client is **slow** (weak connection, throttled, or maliciously not reading), the data piles up in your server's memory. Broadcast a 10 KB message to 50,000 clients and 500 of them are slow — that's potentially gigabytes buffered in your single process. The event loop stays responsive, but **memory climbs until you OOM**.

`ws` exposes this as **`ws.bufferedAmount`** — the bytes queued but not yet flushed to the OS. The defensive pattern is: before/while sending, check `bufferedAmount`; if it exceeds a threshold, the client can't keep up, so you **apply a drop policy** — skip non-critical messages, coalesce updates, or disconnect the slow client to protect everyone else. For critical data you queue per-client with a bounded buffer and disconnect on overflow.

```ts
// backpressure.ts — protect the server from slow consumers.
const MAX_BUFFER = 1 * 1024 * 1024;       // 1 MiB per-connection ceiling

function safeSend(ws: WebSocket, data: string | Buffer): boolean {
  if (ws.readyState !== WebSocket.OPEN) return false;

  // If this client is already badly backed up, it's a slow consumer.
  if (ws.bufferedAmount > MAX_BUFFER) {
    // Policy choice: for a live feed, DROP this update (newer data will come).
    // For an ordered protocol you can't drop, terminate the slow client:
    ws.terminate();
    return false;
  }

  // The send callback reports flush completion AND write errors.
  ws.send(data, (err) => { if (err) ws.terminate(); });
  return true;
}
```

> **At 50k scale, broadcasting without a backpressure check is a memory bomb waiting for one bad network.** Always cap per-connection buffering and choose an explicit drop-or-disconnect policy. We expand this into per-client send queues and fan-out batching in §11.6.

### 5.5 Message size limits **[I]**

Set `maxPayload` on the server (shown in §3) — a hard byte cap per message. Without it, a single client can send a frame sized to exhaust memory before your handler ever runs. Pick the smallest value your protocol genuinely needs (often 64–256 KiB for JSON chat; larger only if you stream binary). `ws` will close a connection that exceeds the cap with code `1009 (message too big)`.

### 5.6 Graceful shutdown — draining connections **[I/A]**

When you deploy, you do not want to yank 50,000 sockets dead. Graceful shutdown: stop accepting *new* connections, tell existing clients to reconnect (they'll land on a healthy instance behind the LB), close sockets cleanly with a sane code, and exit once drained or after a timeout. Done right, a rolling deploy is invisible to users.

```ts
// graceful-shutdown.ts
import http from 'node:http';
import { WebSocketServer, WebSocket } from 'ws';

const server = http.createServer();
const wss = new WebSocketServer({ server });

let shuttingDown = false;

async function shutdown(signal: string) {
  if (shuttingDown) return;
  shuttingDown = true;
  console.log(`${signal} received — draining`);

  // 1) Stop accepting new HTTP/WS connections.
  server.close();

  // 2) Ask every client to reconnect (1001 = "going away"). A good client
  //    reconnects with backoff and the LB routes it to a live instance.
  for (const ws of wss.clients) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.close(1001, 'server shutting down');
    }
  }

  // 3) Give clients a moment to close cleanly, then force-terminate stragglers.
  const deadline = Date.now() + 10_000;
  const timer = setInterval(() => {
    if (wss.clients.size === 0 || Date.now() > deadline) {
      clearInterval(timer);
      for (const ws of wss.clients) ws.terminate();
      process.exit(0);
    }
  }, 250);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));   // what orchestrators send
process.on('SIGINT',  () => shutdown('SIGINT'));    // Ctrl-C
server.listen(8080);
```

Pair this with a load balancer that **stops sending new connections** to a draining instance (health check flips unhealthy) so the drain actually completes. With Docker/Kubernetes, set a `terminationGracePeriodSeconds` longer than your drain timeout. See **[Docker](DOCKER_GUIDE.md)** and **[Nginx](NGINX_GUIDE.md)**.

### 5.7 Close codes worth knowing **[I]**

| Code | Meaning | Use it for |
|---|---|---|
| `1000` | Normal closure | Clean, intentional close |
| `1001` | Going away | Server shutdown / page navigation |
| `1008` | Policy violation | Auth/authorization failure post-open |
| `1009` | Message too big | Exceeded `maxPayload` |
| `1011` | Internal error | Unhandled server error |
| `4000–4999` | **Application-defined** | Your own codes (e.g. `4001` token expired, `4002` rate limited) — clients branch on these |

Use the `4000–4999` private range for app semantics so your reconnect logic can decide whether to retry, refresh a token first, or stop.

---

## 6. Designing the Message Protocol

Raw `ws` gives you a pipe; *you* design what flows through it. A sloppy protocol is the source of most real-time bugs, so design it deliberately.

### 6.1 JSON vs binary **[I]**

- **JSON text frames** are the default: human-readable, trivially debuggable in browser devtools, and `JSON.parse`/`stringify` are fast for small messages. Use JSON unless you have a measured reason not to. The cost is size (field names repeated) and parse CPU on huge messages.
- **Binary** (MessagePack, CBOR, Protobuf, or a hand-rolled format) is smaller and faster to parse, and matters when you push high-frequency or large payloads (game state, market ticks, audio). Trade-off: harder to debug, needs a schema both sides agree on. For 50k clients receiving frequent small updates, shaving bytes per message reduces total bandwidth and CPU meaningfully — measure first (§12).

> Don't reach for binary prematurely. JSON at 50k connections is fine for chat/presence/notifications; binary earns its complexity only under high message rates or large payloads.

### 6.2 The message envelope **[I/A]**

Never send bare strings. Wrap every message in a typed **envelope** so both sides can dispatch on a `type` and evolve the protocol without breakage:

```ts
// protocol.ts — a typed envelope shared by client and server.
type ClientToServer =
  | { type: 'chat.send';   roomId: string; body: string;  cid: string }
  | { type: 'room.join';   roomId: string }
  | { type: 'room.leave';  roomId: string }
  | { type: 'ping';        ts: number };

type ServerToClient =
  | { type: 'chat.message'; roomId: string; from: string; body: string; ts: number }
  | { type: 'ack';          cid: string; ok: true }
  | { type: 'error';        cid?: string; code: string; message: string }
  | { type: 'presence';     roomId: string; users: string[] };
```

Key envelope ingredients:
- **`type`** — a string discriminator you switch on. The whole protocol is a tagged union.
- **`cid` (correlation id)** — a client-generated id echoed back in `ack`/`error`, giving you **request/response over a fundamentally async channel** (§6.4).
- **timestamps / sequence numbers** — for ordering, dedup, and "you missed messages while away" recovery.
- **versioning** — include a protocol version (in the handshake or per message) so old clients and new servers coexist during rollouts.

### 6.3 Rooms, channels & presence **[I/A]**

A **room** (a.k.a. channel/topic) is a named set of connections that should receive the same broadcasts — a chat room, a document, a game match. With raw `ws` you maintain the mapping yourself:

```ts
// rooms.ts — minimal room registry for raw ws (single process).
const rooms = new Map<string, Set<AuthedWS>>();

function join(roomId: string, ws: AuthedWS) {
  let set = rooms.get(roomId);
  if (!set) rooms.set(roomId, (set = new Set()));
  set.add(ws);
  (ws as any).rooms ??= new Set<string>();
  (ws as any).rooms.add(roomId);
}

function leave(roomId: string, ws: AuthedWS) {
  const set = rooms.get(roomId);
  if (!set) return;
  set.delete(ws);
  if (set.size === 0) rooms.delete(roomId);   // GC empty rooms — see gotcha §14
  (ws as any).rooms?.delete(roomId);
}

function broadcast(roomId: string, payload: object, except?: AuthedWS) {
  const set = rooms.get(roomId);
  if (!set) return;
  const data = JSON.stringify(payload);
  for (const ws of set) {
    if (ws !== except && ws.readyState === WebSocket.OPEN) safeSend(ws, data);
  }
}

// In ws.on('close'): leave EVERY room this socket was in, or you leak.
function onClose(ws: AuthedWS) {
  for (const roomId of (ws as any).rooms ?? []) leave(roomId, ws);
}
```

**Presence** ("who is online / in this room") is the set membership itself, broadcast on join/leave. At single-process scale a `Map` is enough; across instances, presence must live in Redis (§11) because no single process sees all connections.

⚡ This is exactly what **Socket.IO rooms** and **uWebSockets.js topics** give you for free — but building it once by hand makes the abstractions click.

### 6.4 Request/response (acks) over WebSocket **[I/A]**

WebSockets are message-oriented, not request-oriented — there's no built-in "reply to *this* message." You build it with the `cid`: the client sends `{type, cid, …}`, remembers the `cid` with a pending promise, and resolves it when an `ack`/`error` with the same `cid` arrives (with a timeout so a lost reply doesn't hang forever).

```ts
// rpc-client.ts — request/response over a raw ws connection (browser/Node).
const pending = new Map<string, { resolve: Function; reject: Function }>();

function request(ws: WebSocket, type: string, data: object, timeoutMs = 5000) {
  const cid = crypto.randomUUID();
  return new Promise((resolve, reject) => {
    pending.set(cid, { resolve, reject });
    const t = setTimeout(() => {
      pending.delete(cid);
      reject(new Error('request timeout'));
    }, timeoutMs);
    // wrap resolve/reject to clear the timer:
    pending.set(cid, {
      resolve: (v: unknown) => { clearTimeout(t); resolve(v); },
      reject:  (e: unknown) => { clearTimeout(t); reject(e); },
    });
    ws.send(JSON.stringify({ type, cid, ...data }));
  });
}

// on message: if it carries a cid we're waiting on, settle the promise.
function onMessage(raw: string) {
  const msg = JSON.parse(raw);
  const p = msg.cid && pending.get(msg.cid);
  if (p) { pending.delete(msg.cid); msg.type === 'error' ? p.reject(msg) : p.resolve(msg); }
}
```

Socket.IO bakes this in: `socket.emit('event', data, (ackResponse) => {…})` — the callback *is* the ack.

### 6.5 Validating inbound messages — untrusted input **[I/A]**

**Every byte from a client is hostile until proven otherwise.** A WebSocket message is exactly as untrusted as an HTTP request body — more so, because the long-lived connection invites a stream of malformed, oversized, or malicious payloads. Validate *structurally* (shape/types) before you touch any field. Use a schema validator (**Zod** shown; **Ajv**/JSON Schema works too and is what Fastify/Nest pipes use):

```ts
// validate.ts — never act on an unvalidated message.
import { z } from 'zod';

const ChatSend = z.object({
  type: z.literal('chat.send'),
  roomId: z.string().uuid(),
  body: z.string().min(1).max(4000),       // cap length to stop giant payloads
  cid: z.string().uuid(),
});

const Inbound = z.discriminatedUnion('type', [
  ChatSend,
  z.object({ type: z.literal('room.join'),  roomId: z.string().uuid() }),
  z.object({ type: z.literal('room.leave'), roomId: z.string().uuid() }),
]);

function handleMessage(ws: AuthedWS, raw: Buffer) {
  let parsed: unknown;
  try { parsed = JSON.parse(raw.toString('utf8')); }
  catch { return safeSend(ws, JSON.stringify({ type: 'error', code: 'BAD_JSON' })); }

  const result = Inbound.safeParse(parsed);   // never throws — returns success flag
  if (!result.success) {
    return safeSend(ws, JSON.stringify({ type: 'error', code: 'BAD_MESSAGE' }));
  }
  const msg = result.data;                     // now fully typed AND validated

  switch (msg.type) {
    case 'chat.send':
      // AUTHORIZATION (not just validation): may THIS user post to THIS room?
      // Validation proves the shape; authz proves the right. Do both.
      if (!userCanPost(ws.userId, msg.roomId)) {
        return safeSend(ws, JSON.stringify({ type: 'error', cid: msg.cid, code: 'FORBIDDEN' }));
      }
      broadcast(msg.roomId, { type: 'chat.message', roomId: msg.roomId,
                              from: ws.userId, body: msg.body, ts: Date.now() });
      safeSend(ws, JSON.stringify({ type: 'ack', cid: msg.cid, ok: true }));
      break;
    // room.join / room.leave …
  }
}
```

The two distinct checks — **validation** (is it well-formed?) and **authorization** (is this user allowed?) — are both mandatory. Conflating them is a classic vulnerability. Also rate-limit *per message* here (§13), since one socket can fire thousands of valid-but-abusive messages per second.

---

## 7. Express 5 + ws

### 7.1 The reality: Express does not "do" WebSockets **[I]**

Express is an HTTP request/response framework. A WebSocket is *not* a request/response — it's an upgraded TCP socket. So Express middleware does **not** run on WebSocket connections. The common helper `express-ws` papers over this, but it can hide the truth and complicate scaling. The clearest, most correct approach on **Express 5** is to run `ws` with `noServer: true` against the same `http.Server` and handle `upgrade` yourself — exactly the pattern from §3.5 and §4. You then *reuse* your Express auth logic inside the upgrade handler rather than as middleware.

⚡ **Version note:** Express 5 (finally stable, current in 2026) changed `req`/route internals and dropped some legacy behavior, but none of that affects this pattern — WS lives *beside* Express, not inside it.

```ts
// express-ws.ts — Express 5 HTTP app + ws on one port, auth reused at upgrade.
import express from 'express';
import http from 'node:http';
import { WebSocketServer, WebSocket } from 'ws';
import jwt from 'jsonwebtoken';

const app = express();
app.get('/healthz', (_req, res) => res.send('ok'));   // normal Express routes

// Build the HTTP server from the Express app so both share the port.
const server = http.createServer(app);

const wss = new WebSocketServer({ noServer: true, maxPayload: 256 * 1024 });

// Reusable auth helper — the SAME logic an Express middleware would use.
function authenticate(req: http.IncomingMessage): string | null {
  try {
    const url = new URL(req.url ?? '', `http://${req.headers.host}`);
    const token = url.searchParams.get('token');
    if (!token) return null;
    const { sub } = jwt.verify(token, process.env.JWT_SECRET!, { algorithms: ['HS256'] }) as { sub: string };
    return sub;
  } catch { return null; }
}

server.on('upgrade', (req, socket, head) => {
  // Route by path: only accept WS on /ws, 404 everything else.
  const { pathname } = new URL(req.url ?? '', `http://${req.headers.host}`);
  if (pathname !== '/ws') { socket.destroy(); return; }

  const userId = authenticate(req);
  if (!userId) { socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n'); socket.destroy(); return; }

  wss.handleUpgrade(req, socket, head, (ws) => {
    (ws as any).userId = userId;
    wss.emit('connection', ws, req);
  });
});

wss.on('connection', (ws: WebSocket) => {
  ws.on('message', (data) => ws.send(`echo: ${data}`));
  ws.on('error', (e) => console.error(e));
});

server.listen(3000, () => console.log('Express + ws on :3000'));
```

The takeaway: **keep WS handling explicit and beside Express.** Don't expect `app.use(cors())`, body parsers, or session middleware to apply to sockets — re-derive what you need (origin, token, user) in the upgrade handler. This is also why, for a WS-heavy app, **Fastify or NestJS often fit better** — they have first-class WS integration. Express is best when WS is a small bolt-on to an existing HTTP app.

---

## 8. Fastify v5 + @fastify/websocket

### 8.1 The plugin model **[I]**

`@fastify/websocket` wraps `ws` and exposes WebSocket endpoints as **Fastify routes** with `{ websocket: true }`. The win over bare Express: your WS routes share Fastify's **hooks** (`preValidation`, `onRequest`), its auth decorators, its structured Pino logging, and its lifecycle — so authentication and validation use the *same machinery* as your HTTP routes instead of a separate code path. Under the hood it's still `ws`, so everything you learned (heartbeats, backpressure, `maxPayload`) applies. See **[Fastify](FASTIFY_GUIDE.md)** for the framework itself.

```bash
npm install fastify @fastify/websocket
```

```ts
// fastify-ws.ts — Fastify v5 with @fastify/websocket, auth via a hook.
import Fastify from 'fastify';
import websocket from '@fastify/websocket';
import jwt from 'jsonwebtoken';

const app = Fastify({ logger: true });

// Register the plugin. options.maxPayload + ws options pass through to ws.
await app.register(websocket, {
  options: { maxPayload: 256 * 1024 },
});

// Authenticate BEFORE the upgrade completes using a route-level hook.
// Throwing here rejects the handshake — the socket is never accepted.
app.addHook('preValidation', async (req, reply) => {
  // Only guard WS routes; let normal HTTP routes use their own auth.
  if (!req.url.startsWith('/ws')) return;
  try {
    const token = (req.query as any)?.token;
    const { sub } = jwt.verify(token, process.env.JWT_SECRET!, { algorithms: ['HS256'] }) as { sub: string };
    (req as any).userId = sub;
  } catch {
    reply.code(401).send('unauthorized');   // rejects the upgrade
  }
});

// A WebSocket route. `socket` is the ws WebSocket; `req` is the Fastify request
// (so req.userId from the hook is available, and req.log is the Pino child).
app.get('/ws', { websocket: true }, (socket, req) => {
  const userId = (req as any).userId as string;
  req.log.info({ userId }, 'ws connected');

  socket.on('message', (data: Buffer) => {
    // validate (Zod/Ajv), then act — same protocol as §6.
    socket.send(`hello ${userId}, you said: ${data.toString()}`);
  });

  // Heartbeat + backpressure from §5 apply here unchanged.
  socket.on('error', (e) => req.log.error(e));
  socket.on('close', () => req.log.info({ userId }, 'ws closed'));
});

await app.listen({ port: 3000, host: '0.0.0.0' });
```

⚡ **Version note:** In recent `@fastify/websocket` (v10+/v11, matching Fastify v5), the handler signature is **`(socket, req)`** — `socket` is the raw `ws` WebSocket. Older versions passed `(connection, req)` where the socket was `connection.socket`. Confirm against your installed version; this is a common breaking change to trip on.

### 8.2 Why this is nicer than bare Express **[I]**

You get routing, schema validation on the *upgrade request's* query/params, hooks, and Pino logging for free, in the same style as the rest of your API. For genuinely Fastify-native apps this is the path of least resistance — but remember the scaling concerns in §11 are identical: `@fastify/websocket` is one process, so 50k+ still means multiple instances, sticky sessions, and a Redis backplane.

---

## 9. NestJS 11 Gateways

### 9.1 What a gateway is **[I]**

NestJS abstracts WebSockets behind **gateways** — classes decorated with `@WebSocketGateway()` whose methods, decorated with `@SubscribeMessage('event')`, handle named messages. The payoff is that real-time handlers become first-class Nest citizens: they get **dependency injection** (inject your services/repos), **guards** (auth), **pipes** (validation), **interceptors**, and they test like any provider. You pick the underlying transport via an **adapter**: the default `ws`-based adapter, or the **Socket.IO adapter** (`@nestjs/platform-socket.io`) when you want rooms/acks/reconnection. See **[NestJS](NESTJS_GUIDE.md)** for the framework.

```bash
# Socket.IO platform (most common with Nest):
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
# OR the raw ws platform:
npm install @nestjs/websockets @nestjs/platform-ws ws
```

### 9.2 A gateway with DI, guards, and pipes **[I/A]**

```ts
// chat.gateway.ts — NestJS 11 gateway (Socket.IO platform shown).
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  MessageBody, ConnectedSocket, OnGatewayConnection, OnGatewayDisconnect,
} from '@nestjs/websockets';
import { UseGuards, UsePipes, ValidationPipe } from '@nestjs/common';
import { Server, Socket } from 'socket.io';
import { WsAuthGuard } from './ws-auth.guard';
import { ChatService } from './chat.service';
import { SendMessageDto } from './dto/send-message.dto';

@WebSocketGateway({
  cors: { origin: ['https://app.example.com'], credentials: true }, // Origin allow-list
  // namespace: '/chat',  // optional logical separation
})
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  // The underlying server (socket.io Server here). Use it to broadcast.
  @WebSocketServer() server!: Server;

  // DI works exactly like the rest of Nest — inject any provider.
  constructor(private readonly chat: ChatService) {}

  // Lifecycle hook: runs when a client connects. Authenticate here (or in a
  // guard) and disconnect unauthorized clients.
  async handleConnection(client: Socket) {
    const userId = await this.chat.verifyToken(client.handshake.auth?.token);
    if (!userId) { client.disconnect(true); return; }
    client.data.userId = userId;                  // attach identity
    await client.join(`user:${userId}`);          // a per-user room
  }

  handleDisconnect(client: Socket) {
    // socket.io auto-leaves rooms; clean up your own state here.
  }

  // A message handler. The guard authorizes; ValidationPipe validates the DTO
  // (class-validator) — the SAME pipes/guards you use on HTTP controllers.
  @UseGuards(WsAuthGuard)
  @UsePipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
  @SubscribeMessage('chat.send')
  async onChatSend(
    @MessageBody() dto: SendMessageDto,
    @ConnectedSocket() client: Socket,
  ) {
    const userId = client.data.userId as string;
    const saved = await this.chat.persist(userId, dto.roomId, dto.body);

    // Broadcast to the room (socket.io rooms).
    this.server.to(`room:${dto.roomId}`).emit('chat.message', saved);

    // Returning a value becomes the ACK delivered to the client's callback —
    // request/response over WS, for free.
    return { ok: true, id: saved.id };
  }
}
```

```ts
// ws-auth.guard.ts — a guard that authorizes WS messages, mirroring HTTP guards.
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { WsException } from '@nestjs/websockets';

@Injectable()
export class WsAuthGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    // For WS, pull the socket out of the context.
    const client = ctx.switchToWs().getClient();
    if (!client.data?.userId) throw new WsException('unauthorized');
    return true;
  }
}
```

```ts
// send-message.dto.ts — class-validator DTO; ValidationPipe enforces it.
import { IsUUID, IsString, Length } from 'class-validator';
export class SendMessageDto {
  @IsUUID() roomId!: string;
  @IsString() @Length(1, 4000) body!: string;
}
```

### 9.3 Choosing the platform (ws vs socket.io) **[I/A]**

| | `@nestjs/platform-ws` (ws) | `@nestjs/platform-socket.io` |
|---|---|---|
| Wire protocol | Raw WebSocket | Socket.IO (Engine.IO) |
| Rooms / namespaces | Build yourself | Built-in |
| Acks (return value) | Limited | Full (return → callback) |
| Reconnection / fallback | Client builds it | Built-in |
| Scaling backplane | Custom Redis (§11) | `@socket.io/redis-adapter` |
| Memory/overhead | Lower | Higher |

**Pick socket.io platform** when you want rooms, acks, reconnection, and the ready-made Redis adapter (most apps). **Pick ws platform** when you need raw protocol compatibility or minimal overhead and will own the scaling plumbing. The 50k+ scaling rules (§11) apply to either — Nest is the *handler* layer, not the *scaling* layer.

### 9.4 The Redis adapter for Nest + Socket.IO **[A]**

To scale Nest's Socket.IO gateways across instances, install a custom `IoAdapter` backed by `@socket.io/redis-adapter` so a broadcast on one instance reaches clients on another. This is shown end-to-end in §11.4 — the mechanism is the same; you just wire it into Nest's adapter slot via `app.useWebSocketAdapter(new RedisIoAdapter(app))`.

---

## 10. Socket.IO in Depth — Rooms, Namespaces, Acks, Middleware

### 10.1 What Socket.IO actually is **[I]**

Socket.IO is **not** a WebSocket library — it's a real-time *framework* with its own protocol layered on **Engine.IO**, which itself can run over WebSocket *or* HTTP long-polling. That dual-transport design is the source of both its strengths (works on hostile networks, auto-upgrades polling → WS, reconnects automatically) and its costs (a heavier handshake, a few extra bytes per message, and the requirement that clients use the matching Socket.IO client — you cannot connect with a bare browser `WebSocket`).

```bash
npm install socket.io          # server
npm install socket.io-client   # browser/node client
```

### 10.2 Server, rooms & namespaces **[I/A]**

```ts
// socketio-server.ts — Socket.IO with auth middleware, rooms, broadcast, acks.
import { Server } from 'socket.io';
import http from 'node:http';
import jwt from 'jsonwebtoken';

const httpServer = http.createServer();
const io = new Server(httpServer, {
  cors: { origin: ['https://app.example.com'], credentials: true }, // Origin allow-list
  maxHttpBufferSize: 256 * 1024,   // == ws maxPayload: cap message size
  pingInterval: 25_000,            // built-in heartbeat (no manual reaper needed)
  pingTimeout: 20_000,             // dead if no pong within this
});

// --- Auth middleware: runs ONCE per connection, before 'connection'. ---
// Reject by calling next(err); accept by calling next(). This is THE place
// to authenticate Socket.IO connections.
io.use((socket, next) => {
  try {
    const token = socket.handshake.auth?.token;       // client sends auth:{token}
    const { sub } = jwt.verify(token, process.env.JWT_SECRET!, { algorithms: ['HS256'] }) as { sub: string };
    socket.data.userId = sub;                          // attach identity
    next();
  } catch {
    next(new Error('unauthorized'));                   // connection refused
  }
});

io.on('connection', (socket) => {
  const userId = socket.data.userId as string;

  // ROOMS: join/leave named rooms. Socket.IO tracks membership for you and
  // auto-removes on disconnect — no manual cleanup like raw ws §6.3.
  socket.on('room.join', (roomId: string, ack) => {
    socket.join(`room:${roomId}`);
    ack?.({ ok: true });                  // ACK callback — request/response
  });

  // Emitting to a room reaches every member (across instances WITH the Redis
  // adapter — see §11.4; without it, only this instance's members).
  socket.on('chat.send', (msg: { roomId: string; body: string }, ack) => {
    // validate msg here (Zod/Ajv) — untrusted input, §6.5.
    io.to(`room:${msg.roomId}`).emit('chat.message', {
      from: userId, body: msg.body, ts: Date.now(),
    });
    ack?.({ ok: true });                  // ack back to sender
  });

  socket.on('disconnect', (reason) => { /* presence update, etc. */ });
});

// NAMESPACES: logical channels with their own handlers/middleware, multiplexed
// over ONE physical connection. Good for separating concerns (e.g. /admin).
const admin = io.of('/admin');
admin.use((socket, next) => { /* stricter auth */ next(); });
admin.on('connection', (socket) => { /* admin-only events */ });

httpServer.listen(3000);
```

The client:

```ts
// socketio-client.ts
import { io } from 'socket.io-client';

const socket = io('wss://realtime.example.com', {
  auth: { token: myJwt },          // delivered to the server's io.use middleware
  transports: ['websocket'],       // skip long-polling if you don't need it
  reconnectionDelayMax: 30_000,    // backoff cap (jitter is automatic)
});

socket.on('chat.message', (m) => render(m));

// emit WITH an ack callback — request/response over the socket:
socket.emit('chat.send', { roomId, body: 'hi' }, (resp) => {
  console.log('server acked:', resp.ok);
});
```

### 10.3 Acks and middleware recap **[I]**

- **Acks**: pass a callback as the last `emit` argument; the server invokes it (or returns a value in Nest). This is built-in request/response — what you hand-rolled with `cid` in §6.4.
- **Middleware**: `io.use((socket, next) => …)` for connection-level auth; `socket.use(([event, ...args], next) => …)` for per-message middleware (rate limiting, validation, logging).

### 10.4 The cost — is Socket.IO worth it? **[I/A]**

| You want… | Verdict |
|---|---|
| Rooms, reconnection, acks, polling fallback, a proven Redis adapter | **Yes** — building these on raw `ws` is real work |
| Absolute minimum bytes/CPU per message at extreme scale | **No** — raw `ws` or `uWebSockets.js` is leaner |
| To connect from a non-Socket.IO client (IoT, another language with only a WS lib) | **No** — they can't speak the Socket.IO protocol |
| Mobile clients on flaky networks | **Yes** — reconnection + buffering shine here |

The protocol overhead is small per message but the **per-connection memory is higher than raw `ws`**, which matters at 50k+. The honest framing: Socket.IO buys you developer time and reliability features; `ws`/`uWebSockets.js` buy you density and bytes. Choose based on which is your constraint — and **measure** (§12) rather than assume.

---

## 11. Scaling to 50,000+ Concurrent Connections

This is the heart of the guide. Getting to 50k+ concurrent WebSockets is **80% operating-system and architecture, 20% code**. We work outward: tune one process to its limit, then scale horizontally with a Redis backplane and sticky sessions, then reach for `uWebSockets.js` for maximum density, then handle fan-out backpressure at scale.

### 11.1 The single-process limits and how to lift them **[A]**

A single Node process *can* hold tens of thousands of idle WebSockets — but only after you remove the default limits that exist to protect a typical web server. There are four ceilings, in the order you'll hit them:

**1. File descriptors (`ulimit -n`).** Every socket is a file descriptor. The default soft limit (often **1024**) means your process dies at ~1000 connections with `EMFILE: too many open files`. This is the #1 thing people miss. Raise it for the user/process that runs Node:

```bash
# Check current limits:
ulimit -Sn   # soft (the one that bites)
ulimit -Hn   # hard (ceiling the soft can be raised to)

# Raise for the current shell (must be <= hard limit):
ulimit -n 1048576

# Persist system-wide in /etc/security/limits.conf:
#   node    soft    nofile    1048576
#   node    hard    nofile    1048576
# And for systemd services, in the unit file:
#   [Service]
#   LimitNOFILE=1048576
```

For 50k connections per process, set the limit comfortably above 50k (e.g. 200k–1M) to leave headroom for outbound sockets (Redis, DB), pipes, and timers.

**2. The TCP accept backlog (`net.core.somaxconn`).** When 50,000 clients reconnect at once (after a deploy), connections queue in the kernel waiting to be `accept()`ed. If the backlog is the old default (**128**), excess connections are dropped and clients see resets. Raise it and pass a matching backlog to `listen()`:

```bash
# Kernel: max queued connections waiting for accept.
sysctl -w net.core.somaxconn=65535
# Max SYNs queued before accept (half-open):
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

```ts
// Pass the backlog to Node's listen() — it defaults to 511.
server.listen({ port: 8080, host: '0.0.0.0', backlog: 65535 });
```

**3. Ephemeral ports & connection tracking (mostly the LB/proxy).** Each TCP connection is identified by the 4-tuple (src IP, src port, dst IP, dst port). A *client* IP can open ~64k connections to one server IP:port before exhausting source ports — rarely a limit for real users (different IPs) but very real for **load testers** and **proxies** that originate many connections from one IP. Widen the ephemeral range and reuse TIME_WAIT sockets:

```bash
sysctl -w net.ipv4.ip_local_port_range="1024 65535"   # more source ports
sysctl -w net.ipv4.tcp_tw_reuse=1                       # reuse TIME_WAIT for new outbound
# Connection-tracking table (if a stateful firewall/NAT is in path):
sysctl -w net.netfilter.nf_conntrack_max=1048576
```

**4. Memory per connection — the real cap on density.** Once FDs are unlimited, **memory** decides how many connections one process holds. Each connection costs: the kernel socket buffers (`rmem`/`wmem`, often tens of KB each) plus the JS objects (`ws` instance, your per-connection state, room memberships, timers). Typical figures: a lean raw-`ws` connection is roughly **20–60 KB** all-in; Socket.IO is higher; `uWebSockets.js` is far lower. At 50k connections, 40 KB each is ~2 GB — so size the box and the **V8 heap** accordingly:

```bash
# Give V8 a bigger old-space heap so GC doesn't thrash near the limit.
# (Default heap is ~2GB on 64-bit; raise for high connection counts.)
node --max-old-space-size=4096 server.js     # 4 GB heap

# Shrink kernel socket buffers if connections are mostly idle (saves RAM):
sysctl -w net.ipv4.tcp_rmem="4096 16384 262144"
sysctl -w net.ipv4.tcp_wmem="4096 16384 262144"
```

> **Measure your real per-connection memory** (§12) — don't guess. Open 10k connections, read RSS, divide. Then you can compute how many fit in your box and how many boxes you need for 50k+.

A reference `sysctl` block for a dedicated WS box (put in `/etc/sysctl.d/99-websockets.conf`, apply with `sysctl --system`):

```ini
# /etc/sysctl.d/99-websockets.conf — tuning for many concurrent WebSockets.
fs.file-max = 2097152
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 120
net.netfilter.nf_conntrack_max = 1048576
```

### 11.2 Why one process is not enough — and the cross-instance problem **[A]**

Even a well-tuned single process has two hard problems at 50k+:

1. **One CPU core.** Node's event loop runs your JS on **one thread**. If your per-message work is non-trivial (validation, fan-out, serialization), one core saturates long before memory does. You must use *more cores* — via multiple processes/containers.
2. **The cross-instance broadcast problem.** The instant you run more than one process, you have the central scaling challenge of real-time systems: **a client connected to instance A and a client connected to instance B cannot see each other's messages**, because each instance only knows its own sockets. Broadcasting "user X said hi in room Y" on A never reaches Y's members who happen to be on B.

The solution to (2) is a **backplane**: a shared message bus every instance subscribes to. When instance A wants to broadcast, it *publishes* to the bus; every instance (A, B, C…) *receives* it and delivers to its own local members. **Redis Pub/Sub** is the standard backplane (see **[Redis](REDIS_GUIDE.md)**).

```
        ┌─────────┐   ┌─────────┐   ┌─────────┐
clients │ Node A  │   │ Node B  │   │ Node C  │  clients
  ↕──── │ (16k WS)│   │ (17k WS)│   │ (17k WS)│ ────↕
        └────┬────┘   └────┬────┘   └────┬────┘
             │ pub/sub     │ pub/sub     │ pub/sub
             └──────────┬──┴──────────┬──┘
                   ┌────▼──────────────▼────┐
                   │   Redis (backplane)    │   ← fan-out across instances
                   └────────────────────────┘
```

### 11.3 Custom raw-`ws` + Redis backplane **[A]**

If you run raw `ws` (no Socket.IO), you build the backplane. The pattern: one Redis **subscriber** connection per instance receives every broadcast and delivers locally; a Redis **publisher** connection sends. Critically, **use two separate Redis connections** — a connection in subscribe mode cannot issue normal commands.

```ts
// backplane.ts — raw ws fan-out across instances via Redis Pub/Sub.
import Redis from 'ioredis';
import { WebSocket } from 'ws';

const INSTANCE_ID = crypto.randomUUID();          // identify the origin instance
const pub = new Redis(process.env.REDIS_URL!);    // for PUBLISH
const sub = new Redis(process.env.REDIS_URL!);    // dedicated SUBSCRIBE connection

// Local room registry (same as §6.3): roomId -> set of local sockets.
const rooms = new Map<string, Set<WebSocket>>();

// 1) Subscribe to a channel that carries all room broadcasts.
await sub.subscribe('ws:broadcast');

// 2) On any message from Redis, deliver to LOCAL members of that room.
sub.on('message', (_channel, raw) => {
  const env = JSON.parse(raw) as { roomId: string; payload: unknown; origin: string };
  // (Optional) skip if we already delivered locally at publish time to avoid
  // double-send to our own clients — depends on your publish strategy below.
  const set = rooms.get(env.roomId);
  if (!set) return;
  const data = JSON.stringify(env.payload);
  for (const ws of set) {
    if (ws.readyState === WebSocket.OPEN) ws.send(data);
  }
});

// 3) To broadcast to a room ANYWHERE in the cluster: publish to Redis.
//    Every instance (including this one) receives it and fans out locally.
function broadcastCluster(roomId: string, payload: unknown) {
  pub.publish('ws:broadcast', JSON.stringify({ roomId, payload, origin: INSTANCE_ID }));
}

// Now a chat.send handler simply calls broadcastCluster(roomId, msg) — and
// members on every instance receive it. Presence ("who's in this room") must
// ALSO live in Redis (e.g. a SET per room) because no instance sees everyone.
```

Two design notes:
- **Presence/state must be in Redis**, not in-process, once you have multiple instances. Use a Redis Set per room (`SADD room:Y:users userX`), expire stale entries, and compute online lists from Redis.
- **Sharding/channels:** publishing every message to one `ws:broadcast` channel means every instance processes every message even if it has no members of that room. At very high message rates, use **per-room channels** (`subscribe`/`unsubscribe` as local membership changes) so an instance only receives traffic for rooms it actually hosts — this is what the Socket.IO adapter does internally.

### 11.4 Socket.IO Redis adapter (the easy path) **[A]**

With Socket.IO, the backplane is a drop-in. Install the adapter and `io.to(room).emit(...)` automatically reaches members on every instance:

```bash
npm install @socket.io/redis-adapter ioredis
```

```ts
// socketio-cluster.ts — multi-instance Socket.IO with the Redis adapter.
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import Redis from 'ioredis';
import http from 'node:http';

const httpServer = http.createServer();
const io = new Server(httpServer, { /* cors, maxHttpBufferSize, ping*… */ });

// Two connections: the adapter needs a pub and a (duplicated) sub client.
const pubClient = new Redis(process.env.REDIS_URL!);
const subClient = pubClient.duplicate();

// Wire the adapter. Now EVERY io.to(room).emit and io.emit fans out across
// all instances automatically — no custom pub/sub code.
io.adapter(createAdapter(pubClient, subClient));

io.on('connection', (socket) => {
  socket.on('chat.send', (msg) => {
    // Reaches room members on THIS and EVERY other instance:
    io.to(`room:${msg.roomId}`).emit('chat.message', msg);
  });
});

httpServer.listen(3000);
```

For NestJS, wrap this in a custom `IoAdapter` and register it with `app.useWebSocketAdapter(new RedisIoAdapter(app))`. The mechanism is identical.

⚡ **Version note:** use **`@socket.io/redis-adapter`** with **`ioredis`** (or `node-redis` v4). The legacy `socket.io-redis` package is deprecated and not compatible with Socket.IO v4. For huge fan-out you can also evaluate the **Redis Streams adapter** or a sharded-Pub/Sub setup (Redis 7 `SSUBSCRIBE`) to spread backplane load.

### 11.5 Sticky sessions — why and how **[A]**

A **sticky session** (session affinity) means a given client is always routed by the load balancer to the *same* backend instance. WebSockets need this for two reasons:

1. **Socket.IO's polling handshake spans multiple HTTP requests.** Before the connection upgrades to WebSocket, Engine.IO does a handshake over several HTTP requests that **must all hit the same instance** (they share server-side session state). Without stickiness the handshake fails with errors like "Session ID unknown." This is *the* classic Socket.IO-behind-a-load-balancer bug.
2. **Even for raw `ws`,** a connection is a long-lived TCP stream pinned to one instance for its lifetime — there's nothing to "balance" mid-connection. Stickiness matters at *connect* time so reconnections after a brief blip can resume cleanly, and so polling fallbacks work.

You implement stickiness at the load balancer by **IP-hash** (route by client IP) or a **routing cookie** (the LB sets a cookie naming the chosen backend). Cookie-based is more robust behind NATs where many clients share an IP. The Redis backplane (§11.4) is what makes stickiness *safe*: a client is pinned to one instance, but messages still reach them from anywhere because the backplane fans out across instances.

### 11.6 The Nginx config for WebSockets at scale **[A]**

Nginx (see **[Nginx](NGINX_GUIDE.md)**) is the most common WS-aware load balancer. The non-negotiable parts: pass the `Upgrade`/`Connection` headers (or the proxy strips them and the upgrade fails), use `least_conn` (balance by active connections, ideal for long-lived sockets), enable **IP-hash for stickiness**, and raise the proxy read timeout (the default 60 s would kill an idle-but-alive socket).

```nginx
# /etc/nginx/nginx.conf (relevant bits) — WebSocket-aware reverse proxy/LB.

# Raise worker connection limits to match the OS tuning in §11.1.
events {
    worker_connections 65535;   # per worker; total = workers * this
    use epoll;
}

http {
    # Map the Upgrade request header to the Connection response header.
    # If Upgrade is empty (normal HTTP) we send Connection: close.
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    upstream ws_backend {
        # Sticky by client IP so a client (and its handshake) stays on one node.
        ip_hash;
        # least_conn would balance better but conflicts with ip_hash; pick one.
        # For cookie-based stickiness use the 'sticky' directive (NGINX Plus) or
        # hash a routing cookie. ip_hash is the simplest OSS option.
        server 10.0.0.11:3000 max_fails=2 fail_timeout=10s;
        server 10.0.0.12:3000 max_fails=2 fail_timeout=10s;
        server 10.0.0.13:3000 max_fails=2 fail_timeout=10s;
        keepalive 64;            # reuse upstream connections
    }

    server {
        listen 443 ssl;
        server_name realtime.example.com;

        # TLS terminates here → wss:// to the client, plain ws to the backend.
        ssl_certificate     /etc/ssl/certs/realtime.pem;
        ssl_certificate_key /etc/ssl/private/realtime.key;
        ssl_protocols TLSv1.2 TLSv1.3;

        location /ws {
            proxy_pass http://ws_backend;

            # THE CRITICAL FOUR LINES — without these the upgrade fails.
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;

            # Preserve the real client IP for rate limiting / logging.
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Long-lived sockets: don't time out an idle-but-alive connection.
            # Set these COMFORTABLY ABOVE your heartbeat interval (§5.1).
            proxy_read_timeout  3600s;
            proxy_send_timeout  3600s;

            # Don't buffer a streaming protocol.
            proxy_buffering off;
        }
    }
}
```

> **The four `proxy_*` upgrade lines are the single most common cause of "my WebSocket works locally but not in production."** Behind a proxy that doesn't forward `Upgrade`/`Connection`, the `101` never happens and the client silently falls back or fails.

### 11.7 Node clustering vs multiple containers **[A]**

To use all CPU cores you run multiple Node processes. Two approaches:

- **`cluster` / multiple containers per box:** N worker processes each holding a share of connections. With raw `ws` you can use Node's `cluster` module with `SO_REUSEPORT` so the kernel load-balances incoming connections across workers — but a connection still lands on *one* worker, so you **still need the Redis backplane** for cross-worker broadcast (workers are just like separate instances). For Socket.IO under `cluster`, you also need sticky routing to the right worker, which is fiddly — most teams instead run **one process per container** and let the orchestrator (Kubernetes/Docker, see **[Docker](DOCKER_GUIDE.md)**) and the LB handle distribution. This is simpler to reason about and deploy.
- **Recommendation for 50k+:** prefer **multiple single-process containers behind the LB**, each tuned per §11.1, all sharing one Redis backplane, with sticky sessions at the LB. Scale out by adding containers. This is the cleanest mental model and the easiest to autoscale. To hit 50k with ~16k/process you run 3–4 containers (plus headroom).

### 11.8 `uWebSockets.js` — the path to 100k+ per box **[A]**

When per-connection memory or raw throughput is your wall, **`uWebSockets.js`** changes the math. Because the server core is hand-optimized C++, per-connection memory is *dramatically* lower (often <5 KB vs 20–60 KB for `ws`) and message throughput is far higher, so a single box can hold **100k–1M** connections. It also has **built-in pub/sub topics** (`ws.subscribe('room')` / `app.publish('room', msg)`) — rooms without your own registry — though cross-*instance* fan-out still needs Redis.

```ts
// uws-server.ts — uWebSockets.js: extreme-density WS server.
import uWS from 'uWebSockets.js';

const app = uWS.App();   // or uWS.SSLApp({ key_file_name, cert_file_name }) for wss

app.ws('/ws', {
  // Per-connection memory & limits — the knobs that enable huge density.
  maxPayloadLength: 256 * 1024,        // == ws maxPayload
  idleTimeout: 60,                     // seconds; uWS sends pings & reaps for you
  maxBackpressure: 1 * 1024 * 1024,    // per-socket backpressure ceiling (auto-drop/close)
  compression: uWS.DISABLED,           // compression costs memory; off for density

  // 'upgrade' runs at handshake time — authenticate HERE (like §4).
  upgrade: (res, req, context) => {
    const token = new URLSearchParams(req.getQuery()).get('token');
    const userId = verify(token);                  // your JWT verify
    if (!userId) { res.writeStatus('401').end(); return; }
    // Complete the upgrade, stamping identity into the socket's user data.
    res.upgrade(
      { userId },                                   // becomes ws.getUserData()
      req.getHeader('sec-websocket-key'),
      req.getHeader('sec-websocket-protocol'),
      req.getHeader('sec-websocket-extensions'),
      context,
    );
  },

  open: (ws) => {
    // Built-in pub/sub: subscribe this socket to a topic ("room").
    ws.subscribe('room:lobby');
  },

  message: (ws, message, isBinary) => {
    // message is an ArrayBuffer. Validate, then publish to a topic — uWS fans
    // out to all LOCAL subscribers extremely efficiently.
    app.publish('room:lobby', message, isBinary);
  },

  // uWS surfaces backpressure relief here so you can resume sending.
  drain: (ws) => { /* ws.getBufferedAmount() dropped — resume queued sends */ },
  close: (ws, code, msg) => { /* cleanup; uWS auto-unsubscribes topics */ },
});

app.listen(3000, (ok) => console.log(ok ? 'uWS on :3000' : 'failed'));
```

The trade-offs to accept: a **lower-level, different API** (no Express/Nest ergonomics, manual everything), it's a **native addon** (build/ABI concerns in your Docker image), and cross-instance broadcast still needs a Redis bridge you wire to `app.publish`. Reach for it when measurements prove `ws`/Socket.IO density is the bottleneck — not before.

⚡ **Version note:** `uWebSockets.js` is installed from a GitHub tag, not a semver npm range, and each build targets a specific Node ABI — pin the tag that matches **Node 24** in your `package.json` (e.g. `"uWebSockets.js": "uNetworking/uWebSockets.js#v20.x"`) and rebuild when you bump Node.

### 11.9 Fan-out & backpressure at scale **[A]**

Broadcasting to 50k clients is not "loop and `send`." Done naively it has two failure modes:

1. **Head-of-line stalls:** if you `send` to clients in a tight synchronous loop and a kernel buffer is full, you can block; more importantly, building 50k serialized copies and queuing them all at once spikes memory. **Serialize once, reuse the buffer** (don't `JSON.stringify` per client), and let each socket's own buffer drain independently.
2. **Slow consumers poison the broadcast:** one slow client backs up megabytes while fast clients wait. **Never let a slow consumer affect others.** Per-client bounded queues with a drop policy (§5.4) isolate them.

```ts
// fanout.ts — efficient, backpressure-aware broadcast to a large room.
function fanout(members: Iterable<WebSocket>, payload: object) {
  // 1) Serialize ONCE — reuse the same Buffer for every send.
  const data = Buffer.from(JSON.stringify(payload));

  for (const ws of members) {
    if (ws.readyState !== WebSocket.OPEN) continue;

    // 2) Per-client backpressure: if this client is behind, apply policy.
    if (ws.bufferedAmount > SLOW_THRESHOLD) {
      // For a live feed: DROP (a newer frame supersedes it) and count it.
      slowClientDrops.inc();
      // For ordered data you can't drop: terminate the slow client to protect
      // the room: ws.terminate();
      continue;
    }
    ws.send(data);   // each socket drains at its own pace; fast clients unaffected
  }
}
```

Other scale techniques:
- **Batching/coalescing:** at high update rates, don't send every state change. Accumulate per-room changes and flush on a short interval (e.g. every 50 ms) — one combined frame instead of 20 tiny ones. Slashes per-message overhead and CPU.
- **Per-client send queues with drop policy:** wrap each socket with a small bounded queue; on overflow drop oldest (live data) or disconnect (ordered data). Surface a `slow_consumer` metric.
- **Offload heavy work:** keep the event loop free — any non-trivial CPU (large serialization, crypto, compression) goes to a **worker thread** or another service, or it stalls *all* connections.

---

## 12. Benchmarking & Load Testing to 50k

You cannot tune what you cannot measure, and "it works on my laptop with 5 tabs" tells you nothing about 50,000 connections. This section is how to *prove* your server holds 50k and find the breaking point before users do.

### 12.1 What to measure **[A]**

| Metric | Why it matters | How |
|---|---|---|
| **Concurrent connections** | The headline number — does it actually hold 50k? | Count `wss.clients.size` / Socket.IO `engine.clientsCount` |
| **Connect rate** | Can you absorb a reconnect storm (deploy/outage)? | Connections/sec the server accepts without errors |
| **Message latency (p50/p95/p99)** | The user-felt "is it real time?" | Timestamp on send, measure round-trip at the client |
| **Memory (RSS) per connection** | Determines density and box sizing | (RSS at N conns − baseline) / N |
| **CPU per core** | One core saturating = need more processes | `top`/metrics; remember Node uses one core |
| **Event-loop lag** | The killer metric: lag = you're blocking | `perf_hooks.monitorEventLoopDelay()` |
| **Dropped/errored connections** | Hitting FD/backlog/memory limits | Server error logs, `EMFILE`, resets |

**Event-loop lag is the metric that distinguishes a healthy Node WS server from a dying one.** When lag climbs from <5 ms to hundreds of ms, you are blocking the loop — every connection feels it simultaneously. Expose it:

```ts
// loop-lag.ts — measure event-loop delay; alert if it climbs.
import { monitorEventLoopDelay } from 'node:perf_hooks';
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  // mean/p99 in ms. Healthy: low single digits. Trouble: tens-hundreds.
  console.log(`loop p99=${(h.percentile(99) / 1e6).toFixed(1)}ms mean=${(h.mean / 1e6).toFixed(1)}ms`);
  h.reset();
}, 5000);
```

### 12.2 Tools **[A]**

- **Artillery** (`artillery run`) — has first-class WebSocket and Socket.IO engines; declarative YAML scenarios, ramps, and built-in latency reporting. The easiest way to script "ramp to 50k over 5 minutes, each sends a message every 10 s."
- **k6** (with the `xk6-websockets` extension or the built-in `k6/ws`) — scriptable in JS, great metrics, good for CI gates.
- **A custom Node load client** — the most flexible and the cheapest way to generate *many* connections, because one Node process can itself open tens of thousands of `ws` clients. This is what you use to reproduce 50k locally.

### 12.3 Reproducing 50k locally **[A]**

The trick: a load *generator* faces the same OS limits as the server (FDs, ephemeral ports). To open 50k client connections from one machine you must raise the client's `ulimit -n` too, and because all clients share one source IP you'll burn through ephemeral ports — so either widen the range (§11.1), bind clients across multiple source IPs, or run the generator on a few machines. A simple, brutal Node generator:

```ts
// loadgen.ts — open N connections, hold them, optionally send periodically.
import { WebSocket } from 'ws';

const URL = process.env.URL ?? 'ws://localhost:8080/ws';
const TARGET = Number(process.env.N ?? 50_000);
const RAMP_PER_SEC = Number(process.env.RAMP ?? 2000);   // connect rate

let open = 0, failed = 0;
const sockets: WebSocket[] = [];

function spawn() {
  const ws = new WebSocket(`${URL}?token=${process.env.TOKEN}`);
  ws.on('open', () => { open++; sockets.push(ws); });
  ws.on('error', () => { failed++; });
  ws.on('close', () => { open--; });
  // Optional: each client sends a message every 10s to create message load.
  ws.on('open', () => setInterval(() => {
    if (ws.readyState === WebSocket.OPEN)
      ws.send(JSON.stringify({ type: 'ping', ts: Date.now() }));
  }, 10_000));
}

// Ramp up gradually — a reconnect-storm test would instead spawn all at once.
const timer = setInterval(() => {
  for (let i = 0; i < RAMP_PER_SEC && sockets.length + 1 <= TARGET; i++) spawn();
  console.log(`open=${open} failed=${failed} target=${TARGET}`);
  if (sockets.length >= TARGET) clearInterval(timer);
}, 1000);
```

Run with raised limits:

```bash
ulimit -n 1048576
URL=ws://localhost:8080/ws N=50000 RAMP=3000 TOKEN=$JWT npx tsx loadgen.ts
```

An Artillery scenario for the same, with latency reporting:

```yaml
# load.yml — artillery WebSocket ramp to 50k.
config:
  target: "ws://localhost:8080/ws"
  phases:
    - duration: 60        # warm up
      arrivalRate: 200    # new connections per second
    - duration: 240       # ramp toward 50k
      arrivalRate: 200
      rampTo: 1000
  ws:
    subprotocols: []
scenarios:
  - engine: ws
    flow:
      - connect: "/ws?token={{ $env.TOKEN }}"
      - loop:
          - send: '{"type":"ping","ts":{{ $timestamp }}}'
          - think: 10
        count: 30
```

### 12.4 Reading the numbers **[A]**

Watch these inflection points as you ramp:
- **Connections plateau below target + `EMFILE` in logs** → file-descriptor limit (§11.1). Raise `ulimit -n` on server *and* generator.
- **Connection resets at high connect rate** → accept backlog / `somaxconn` too low, or you're ramping faster than `accept()` keeps up.
- **RSS climbs linearly, then GC thrashes / OOM** → per-connection memory × N exceeds heap; raise `--max-old-space-size`, cut per-connection state, or add instances. Compute "(RSS − baseline)/N" to get true per-connection cost.
- **Event-loop p99 climbs with message rate** → you're doing too much per message on the one thread; batch, offload to workers, or shard across more processes.
- **Latency p99 spikes while p50 stays low** → backpressure/slow consumers or GC pauses — check `bufferedAmount` distribution and GC logs.

The goal is a clear statement like: *"One tuned container holds 16k connections at <60 ms p99 and 1.8 GB RSS with loop lag <5 ms; therefore 4 containers + Redis backplane comfortably serve 50k with headroom."* That sentence — backed by measurements — is what "handles 50k+" actually means.

---

## 13. Production Hardening & Banking-Grade Security

A long-lived, authenticated, bidirectional channel is a large attack surface. Treat a WebSocket endpoint with the same paranoia you'd treat a banking API — because for many of these apps, it *is* one. This section is the consolidated security checklist with the *why* behind each control.

### 13.1 TLS everywhere (`wss://`) **[A]**

Never use `ws://` in production. Plain WebSockets travel in cleartext — tokens, messages, everything — readable by anyone on the path, and trivially downgraded/injected. Use **`wss://`** (WebSocket over TLS). In practice you **terminate TLS at the load balancer/Nginx** (§11.6) and speak plain `ws` on the trusted internal network, or terminate in Node with `https.createServer({ key, cert })` and attach `ws` to it. Mixed content rules also force this: a page served over HTTPS *cannot* open a `ws://` socket — the browser blocks it. Use TLS 1.2+ (prefer 1.3). See **[Networking](NETWORKING_GUIDE.md)** for the TLS handshake details.

### 13.2 Strict Origin + authentication (recap, enforced) **[A]**

- **Allow-list `Origin`** on every upgrade — default-deny (§4.4). This is your CSWSH defense and it is mandatory with cookie auth.
- **Authenticate at the handshake**, reject before accepting (§4.3). Pin the JWT algorithm. Short token TTLs.
- **Re-validate long-lived sessions** (§4.5) — expire/refresh tokens over the socket; close on hard expiry.
- **Authorize every action**, not just the connection (§6.5) — "connected" ≠ "allowed to post to room Y / read account Z." Check per message.

### 13.3 Rate limiting — connections and messages **[A]**

A WebSocket invites two distinct floods, and you must limit both:

1. **Connection floods** — opening thousands of sockets to exhaust FDs/memory. Limit **concurrent connections per IP and per user**, and limit the **connect rate** per IP. Reject excess at the handshake.
2. **Message floods** — one authenticated socket firing thousands of messages/second. Limit **messages per second per connection/user**, ideally with a token bucket. Exceeding it → drop, or close with a `4002 rate limited` code.

Back the counters with **Redis** so limits are enforced **across all instances** (an attacker spreading connections over instances must still hit a shared limit). See **[Redis](REDIS_GUIDE.md)** for atomic counters.

```ts
// rate-limit.ts — per-connection message token bucket + per-IP connection cap.
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL!);

// (A) Per-IP concurrent connection cap, checked at upgrade (shared via Redis).
async function tryAcceptConnection(ip: string): Promise<boolean> {
  const key = `ws:conns:${ip}`;
  const n = await redis.incr(key);
  if (n === 1) await redis.expire(key, 3600);
  if (n > 50) { await redis.decr(key); return false; }  // cap 50 conns/IP
  return true;     // remember to DECR in ws.on('close')
}

// (B) Per-connection message rate (in-process token bucket — cheap, no Redis hit
//     per message). Refill `rate` tokens/sec up to `burst`.
class TokenBucket {
  private tokens: number;
  private last = Date.now();
  constructor(private rate: number, private burst: number) { this.tokens = burst; }
  allow(): boolean {
    const now = Date.now();
    this.tokens = Math.min(this.burst, this.tokens + ((now - this.last) / 1000) * this.rate);
    this.last = now;
    if (this.tokens < 1) return false;
    this.tokens -= 1;
    return true;
  }
}
// On connection: const bucket = new TokenBucket(20, 40);  // 20 msg/s, burst 40
// On message:    if (!bucket.allow()) { ws.close(4002, 'rate limited'); return; }
```

### 13.4 Payload size caps & validation **[A]**

- **`maxPayload` / `maxHttpBufferSize`** — hard byte cap per message (§5.5). Without it, a single frame can OOM you.
- **Validate structure before use** (§6.5) — schema-validate every inbound message; reject malformed input with an error, don't crash.
- **Cap field lengths and array sizes** in the schema (`body` ≤ 4000 chars, arrays ≤ N) so a *valid-shaped* message can't be abusively large.
- **Bound rooms and subscriptions** — cap how many rooms one socket can join; an attacker joining a million rooms is a memory DoS (§14).

### 13.5 DoS protection — slowloris, floods, slow consumers **[A]**

- **Slow-handshake (slowloris-style):** an attacker opens TCP connections and sends the upgrade headers byte-by-byte, never finishing, to tie up resources. Set a **handshake/header timeout** (`server.headersTimeout`, `server.requestTimeout`) so half-open handshakes are dropped fast. Nginx in front also absorbs much of this.
- **Connection floods:** per-IP caps (§13.3) + the OS backlog tuning + an edge LB/WAF.
- **Slow consumers:** the `bufferedAmount` drop/disconnect policy (§5.4, §11.9) — a client that won't read can't be allowed to balloon server memory.
- **Decompression bombs:** if `perMessageDeflate` is on, a tiny compressed frame can expand to a huge buffer. Cap it (`maxPayload` applies to the *decompressed* size in `ws`) or disable compression at scale.

### 13.6 Logging, secrets & observability **[A]**

- **Never log sensitive data** — no tokens, no message bodies for sensitive domains, no PII. Log *metadata*: connection id, user id (if non-sensitive), event type, sizes, latencies. A token in an access log (from `?token=`) is a credential leak — strip query strings from logs or use header/ticket auth.
- **Structured logs** (Pino/`slog`-style) with a connection/correlation id so you can trace one socket's lifetime.
- **Metrics dashboards** — export the §12 metrics (connections, connect rate, message latency, RSS, event-loop lag, slow-consumer drops, auth failures, rate-limit hits) to Prometheus/Grafana. Alert on event-loop lag and connection-count cliffs (a sudden drop = an instance died and 16k clients are reconnecting — a thundering herd).
- **Audit security events** — auth failures, Origin rejections, rate-limit trips — for incident response.

### 13.7 Graceful deploys without dropping users **[A]**

Combine §5.6 (drain) with the LB: flip the instance's health check unhealthy → LB stops new connections → close existing sockets with `1001` → clients reconnect (backoff + jitter) and the LB routes them to healthy instances → instance exits. Because the **Redis backplane** decouples message delivery from which instance a client is on, a client that moves instances mid-conversation misses nothing. Set Kubernetes `terminationGracePeriodSeconds` > drain timeout and `preStop` hooks to start draining before SIGTERM. Done right, a deploy of a 50k-connection service is invisible to users.

### 13.8 The hardening checklist **[A]**

| Control | Where | Why |
|---|---|---|
| `wss://` / TLS 1.2+ | LB or Node | Confidentiality, integrity, mixed-content |
| Origin allow-list | Upgrade | CSWSH defense |
| Auth at handshake, pinned alg, short TTL | Upgrade | No unauthenticated sockets |
| Re-auth / hard session ceiling | Per connection | Long-lived token risk |
| Authorize every action | Per message | Connected ≠ entitled |
| `maxPayload` cap | Server config | Memory DoS |
| Schema-validate inbound | Per message | Untrusted input |
| Per-IP connection cap (Redis) | Upgrade | Connection floods |
| Per-conn message rate limit | Per message | Message floods |
| `bufferedAmount` drop/disconnect | Per send | Slow-consumer memory DoS |
| `headersTimeout`/`requestTimeout` | Server config | Slowloris |
| No secrets/PII in logs | Logging | Credential/data leak |
| Metrics + alerts (loop lag, conns) | Observability | Detect attacks/failures |
| Graceful drain + LB health flip | Deploy | No dropped users |

---

## 14. Gotchas & Best Practices

The failure modes that bite real Node WebSocket services, with the fix for each.

- **Forgetting the `error` listener → process crash.** A `ws` socket with no `'error'` handler turns any socket error into an *uncaught exception* that kills the whole process — and with it, all 50k connections. **Always attach `ws.on('error', …)`** (and a server-level one), even if it only logs.
- **No heartbeat → ghost connections leak you to death.** Without ping/pong reaping (§5.1), dead-but-`OPEN` sockets accumulate FDs and memory until you fall over. **Always run the heartbeat reaper.**
- **Memory leak from unclosed handlers / un-left rooms.** Every per-connection timer, interval, room membership, or external subscription must be torn down in `ws.on('close')`. The classic leak: a `setInterval` per connection that's never cleared, or a socket left in a room `Set` after disconnect. **Do all cleanup in `close`.**
- **Unbounded rooms / registries.** A `Map<roomId, Set>` that never deletes empty rooms grows forever; a socket allowed to join unlimited rooms is a DoS. **GC empty rooms; cap rooms-per-socket.**
- **No backpressure handling → OOM on broadcast.** Broadcasting to slow clients without checking `bufferedAmount` buffers unbounded data in your process. **Cap per-connection buffering with a drop/disconnect policy** (§5.4, §11.9).
- **Blocking the event loop.** One synchronous heavy operation (huge `JSON.parse`, sync crypto, a tight CPU loop) freezes *every* connection at once. **Keep per-message work tiny; offload CPU to worker threads.** Watch event-loop lag (§12).
- **Missing sticky sessions behind a load balancer.** Especially with Socket.IO: the multi-request handshake fails with "Session ID unknown" without affinity. **Enable IP-hash or cookie stickiness** (§11.5).
- **Forgetting the Nginx `Upgrade`/`Connection` headers.** Works locally, fails in prod — the proxy strips the upgrade and the `101` never happens. **Set the four `proxy_*` lines** (§11.6).
- **Origin not checked.** Cookie-authenticated sockets without Origin allow-listing are wide open to CSWSH. **Allow-list Origin, default-deny** (§4.4).
- **File-descriptor limits.** Default `ulimit -n` 1024 caps you at ~1000 connections with `EMFILE`. **Raise it on every box and the LB** (§11.1).
- **`new WebSocket()` can't set headers in browsers.** Don't design auth around an `Authorization` header for browser clients — use query token / subprotocol / cookie+Origin (§4.2).
- **Trusting inbound messages.** A long-lived socket is a firehose of untrusted input. **Schema-validate and authorize every message** (§6.5).
- **`perMessageDeflate` on by default thinking.** It's off by default in `ws` for good reason — it costs memory/CPU per connection and can be a decompression-bomb vector. **Leave it off at scale unless measured to help.**
- **No graceful shutdown → every deploy drops 50k users.** A hard `process.exit` resets every socket. **Drain on SIGTERM** (§5.6) and coordinate with the LB.
- **One process for everything.** Node uses one core; one process can't use a 16-core box. **Run multiple instances + a Redis backplane** (§11).
- **Logging the auth token.** `?token=` lands in access/proxy logs. **Strip query strings from logs** or use ticket/header auth (§13.6).
- **Assuming numbers.** Per-connection memory and the real ceiling are *measured*, not guessed. **Load-test to 50k** (§12) before you claim it.

**Best-practice summary:** authenticate at the handshake, validate and authorize every message, heartbeat to reap dead sockets, cap payloads and rates, handle backpressure with an explicit policy, never block the event loop, scale horizontally with a Redis backplane + sticky sessions, terminate TLS at the edge, drain gracefully on deploy, and measure everything.

---

## 15. Study Path & Build-to-Learn Projects

The fastest way to internalize all of this is to build one real-time service and then *scale it in stages*, feeling each limit before you lift it.

**Stage 0 — Foundations (read first).** Skim **[Networking](NETWORKING_GUIDE.md)** (TCP/TLS, the WS protocol, C10k/C10M) and confirm the Node basics in **[Node.js](NODEJS_GUIDE.md)** (event loop, streams, graceful shutdown). Read §1–§2 of this guide.

**Stage 1 — A single-process chat + presence server (raw `ws`).**
- Build the echo/broadcast server (§3), add the typed envelope, rooms, and presence (§6), and a reconnecting browser client (§5.3).
- Add **handshake auth** with a JWT in the query string and **strict Origin checking** (§4).
- Add the **heartbeat reaper**, **backpressure-safe send**, and **graceful shutdown** (§5).
- *Goal:* a correct, secure chat with rooms, presence, and acks on one process.

**Stage 2 — Harden it.** Add **schema validation** (Zod) and **per-message authorization** (§6.5), **per-IP connection caps** and **per-connection message rate limits** (§13.3), `maxPayload`, and structured logging that never leaks tokens (§13.6). Re-validate tokens over the socket (§4.5).

**Stage 3 — Make it multi-instance (the big leap).** Run **two instances** and watch cross-instance broadcast *break* (clients on different instances can't see each other). Fix it with the **Redis backplane** — either the custom raw-`ws` pub/sub (§11.3) or, if you switched to Socket.IO, the **`@socket.io/redis-adapter`** (§11.4). Move **presence into Redis**. Put **Nginx** in front with the WebSocket config and **sticky sessions** (§11.5–§11.6).

**Stage 4 — Tune one box and load-test to 50k.** Apply the **OS tuning** (`ulimit -n`, `somaxconn`, sysctl) (§11.1). Write the **Node load generator** (or Artillery scenario) and **ramp to 50,000 connections** (§12). Measure RSS/connection, event-loop lag, p99 latency, connect rate. Find your per-process ceiling, then compute how many instances 50k needs and verify it across the cluster.

**Stage 5 — Reach for max density.** Rebuild the hot path on **`uWebSockets.js`** (§11.8) and re-run the 50k load test. Compare per-connection memory and throughput against `ws`/Socket.IO — now you can *prove* when the extra complexity is worth it. Bridge its `app.publish` to Redis for cross-instance fan-out.

**Stage 6 — Production polish.** Add **metrics + Grafana dashboards and alerts** (§13.6), a **graceful rolling-deploy** that drops no users (§13.7), Dockerize and deploy behind the LB (**[Docker](DOCKER_GUIDE.md)**, **[Nginx](NGINX_GUIDE.md)**). Run a **reconnect-storm test** (kill an instance, watch 16k clients re-home via backoff + backplane).

**Comparison capstone.** Build the *same* chat server in **Go** following **[Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md)** and load-test both to 50k. Contrast the goroutine-per-connection model against Node's single event loop: where each spends memory, how each handles a blocking handler, and what density each reaches per box. This side-by-side is the single most clarifying exercise for understanding *why* the scaling techniques in §11 exist.

**Project ideas to extend:** a collaborative document editor (ordered messages, can't drop — exercises per-client queues), a live trading/metrics dashboard (high-frequency fan-out — exercises batching and binary), a multiplayer game lobby (rooms, presence, low latency), an IoT telemetry ingest (many lean connections — the `uWebSockets.js` sweet spot).

---

*End of guide. You now have the full path from a 30-line echo server to a TLS-terminated, Redis-backed, sticky-load-balanced, rate-limited, gracefully-deploying real-time service that holds 50,000+ concurrent connections — and the measurements to prove it.*
