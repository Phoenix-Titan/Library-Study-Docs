# Server-Sent Events in Express — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Any Node.js/Express developer who has never *streamed* anything to a browser and wants to reach the point where they can build a production, banking-grade real-time feed — live notifications, an audit-log tail, a metrics dashboard — that survives reconnects, authenticates every connection, authorizes every event, and scales across many servers. You need only to be able to read TypeScript and know what an Express route handler is; this guide teaches Server-Sent Events (SSE) from the wire protocol up, using the best current library and industry standards. It is deliberately **explain-first**: every concept leads with prose — *what* it is, *why* it exists, *when* to reach for it, *how* it works, the *best practice*, and the *gotcha* — and only then shows heavily-commented code. Real-time systems fail in subtle ways (a proxy that buffers your stream, compression middleware that batches it, a token in a URL that leaks into an access log), so the "why" matters as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Node.js 22 / 24 LTS** with **Express 5.x** and **TypeScript 5.7+**. The SSE library is **`better-sse` 0.x** — the modern, TypeScript-first, actively-maintained de-facto choice (sessions, channels, auto-heartbeat, event id/retry handling). Auth uses **`jsonwebtoken` 9** for tokens and **`argon2`** (node-argon2, 0.4x) for Argon2id password hashing. The database layer is **PostgreSQL** via **`pg`** (node-postgres 8.x) with **`node-pg-migrate` 7** migrations; **`ioredis` 5** provides the cross-instance backplane; **Nginx** is the reverse proxy. SSE itself is a stable web standard (part of the HTML Living Standard) and has not changed in years — the moving parts are the libraries around it. Fast-moving details are flagged **⚡**. The author is on **Windows 11**, so shell commands are shown for PowerShell and POSIX where they differ.
>
> **This guide's place in the library:** SSE is the *one-way* half of real-time. It rides on the HTTP server you already know — Express is covered in the [Node.js](NODEJS_GUIDE.md) guide. When you need *bidirectional* messaging (chat, collaborative editing), reach for WebSockets instead — see [WebSockets in Node](NODE_WEBSOCKETS_GUIDE.md) for that, and §1 here for the exact decision. The data layer talks to [PostgreSQL](POSTGRESQL_GUIDE.md) (this guide uses raw `pg`; if you prefer an ORM, the [Prisma](PRISMA_ORM_GUIDE.md) guide slots in cleanly). To scale past one process you'll add a [Redis](REDIS_GUIDE.md) backplane behind [Nginx](NGINX_GUIDE.md), ship it in [Docker](DOCKER_GUIDE.md), and run it in CI with [GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md). The exact same system built in NestJS (with the `@Sse()` decorator and RxJS) is the sibling [Server-Sent Events in NestJS](NESTJS_SSE_GUIDE.md) guide — read whichever matches your framework. This guide assumes you can write basic Express; it teaches SSE.

---

## Table of Contents

1. [What SSE Is and When to Use It](#1-what-sse-is-and-when-to-use-it) **[B]**
2. [The event-stream Wire Protocol and EventSource](#2-the-event-stream-wire-protocol-and-eventsource) **[B]**
3. [Raw SSE in Express](#3-raw-sse-in-express) **[B]**
4. [Adopting better-sse the Production Library](#4-adopting-better-sse-the-production-library) **[I]**
5. [Channels and Targeted Delivery](#5-channels-and-targeted-delivery) **[I]**
6. [Authenticating the Stream](#6-authenticating-the-stream) **[I/A]**
7. [Persistence and Gap-Free Resume](#7-persistence-and-gap-free-resume) **[I/A]**
8. [Postgres LISTEN NOTIFY as an Event Source](#8-postgres-listen-notify-as-an-event-source) **[A]**
9. [Scaling with a Redis Backplane](#9-scaling-with-a-redis-backplane) **[A]**
10. [The Admin Dashboard UI](#10-the-admin-dashboard-ui) **[I]**
11. [Banking-Grade Security](#11-banking-grade-security) **[A]**
12. [Graceful Shutdown and Memory Management](#12-graceful-shutdown-and-memory-management) **[I/A]**
13. [Testing SSE](#13-testing-sse) **[I/A]**
14. [Gotchas and Best Practices](#14-gotchas-and-best-practices) **[A]**
15. [Study Path and Build-to-Learn Projects](#15-study-path-and-build-to-learn-projects)

---

## 1. What SSE Is and When to Use It

### 1.1 The problem SSE solves **[B]**

Plain HTTP is a question-and-answer protocol: the client asks (a request), the server answers (a response), and the connection is done. That model is perfect for "load this page" or "save this form," but it has no way to express "tell me when something happens *later*." If your server learns something new a second after the response finished — a payment cleared, an alert fired, a background job completed — it has no channel to reach the browser. The browser would have to *ask again*.

For years the workaround was **polling**: the browser asks "anything new?" every few seconds. Polling works but is wasteful — most requests return "nothing new," yet each pays the full cost of a request (headers, TLS, a round trip, a handler execution). Worse, it is *laggy*: an event that happens just after a poll waits the whole interval before the user sees it. You can shorten the interval, but then you multiply the wasted requests. Polling is a tax you pay whether or not anything is happening.

**Server-Sent Events** removes the tax. The client opens **one** long-lived HTTP response and simply *leaves it open*. The server holds that response and, whenever it has something to say, writes a small text-framed message into the still-open body and flushes it. The browser receives each message the instant it is written. One connection, opened once, carries an unbounded stream of server→client messages — no polling, no per-event overhead, no artificial latency.

The crucial word is **one-way**. SSE is a server→client pipe. The client cannot send messages back *over the SSE connection* — if it needs to talk to the server, it makes a normal HTTP request like any other. That one-way limitation is not a weakness; it is the simplification that makes SSE dramatically easier and cheaper than WebSockets for the very common case where only the server has news to push.

### 1.2 What SSE actually is, concretely **[B]**

Strip away the jargon and SSE is just three agreements:

1. **A response that never ends (until you close it).** The server sends HTTP 200 with `Content-Type: text/event-stream` and then keeps the body open, `res.write()`-ing text into it over time instead of ending it.
2. **A tiny text format for framing messages.** Each message is a few lines of UTF-8 text — `data: hello` — terminated by a blank line. That's the entire "protocol on top of HTTP." §2 covers every field.
3. **A built-in browser client that reconnects for you.** Browsers ship a native object, `EventSource`, that opens the stream, parses the frames, hands you each message as a JavaScript event, and — the killer feature — **automatically reconnects** if the connection drops, telling the server the ID of the last event it saw so you can resume without gaps.

That is the whole idea. Everything else in this guide is a consequence of those three facts.

### 1.3 SSE vs WebSockets vs long-polling **[B/I]**

Choosing the right transport is the most important decision here, because picking the heavy tool for a light job is a common and expensive mistake:

| Transport | Direction | Reconnect | Protocol | Complexity | Best for |
|---|---|---|---|---|---|
| **Polling** | client pulls | n/a | plain HTTP | trivial | Rare, non-urgent updates; the fallback of last resort. |
| **Long-polling** | server→client (faked) | manual | plain HTTP | medium | Legacy environments where SSE/WS are blocked. |
| **SSE** | **server→client only** | **automatic, built-in** | plain HTTP (`text/event-stream`) | **low** | **Live feeds, notifications, dashboards, progress, log tails — anything where only the server pushes.** |
| **WebSockets** | **full duplex** | manual (you build it) | `ws://` upgrade | high | Chat, multiplayer, collaborative editing — anything where the *client* also streams. |

Read it as a decision tree:

- **Does the client need to stream messages to the server continuously?** → **WebSockets** (see [WebSockets in Node](NODE_WEBSOCKETS_GUIDE.md)). SSE cannot do this; the client's only back-channel is ordinary HTTP requests.
- **Does only the server push, and the client just reacts (plus the occasional normal POST)?** → **SSE.** This describes the overwhelming majority of "real-time" features: a notification bell, a live order status, a deploy-log tail, a metrics graph, an admin activity feed. For all of these, SSE is less code, less infrastructure, and fewer failure modes than WebSockets.
- **Is even SSE unavailable** (an ancient proxy that strips streaming)? → **long-polling** as graceful degradation.

The reason SSE is "less" than WebSockets is that it *is just HTTP*. It flows through every proxy, load balancer, and firewall that already understands HTTP; it uses your existing cookies, auth, TLS, and logging. A WebSocket is a *protocol upgrade* — every hop must explicitly support it, and you build reconnection, heartbeats, and resume yourself. When the client doesn't need to stream, all of that is complexity bought for nothing.

> **The one-sentence rule:** *If only the server talks, use SSE; if the client also streams, use WebSockets.*

### 1.4 What we will build **[B]**

To keep every concept concrete, this guide builds one running example incrementally: a **secure admin dashboard** for a small banking-style back office. Staff log in (Argon2id-verified password → JWT in a secure cookie), and the dashboard opens an SSE stream showing, in real time: **notifications** ("a large transfer was flagged"), an **audit feed** (who did what), and **live metrics** (active sessions, events/second). By the end that stream authenticates on connect, delivers each admin only the events they're authorized to see, persists every event so a reconnecting browser replays what it missed, is driven directly by the database via Postgres `LISTEN/NOTIFY`, fans out across instances through a Redis backplane, and sits behind Nginx with banking-grade hardening. We start with the smallest possible stream and add one capability per section.

---

## 2. The event-stream Wire Protocol and EventSource

### 2.1 The frame format **[B]**

An SSE stream is UTF-8 text: a sequence of **fields**, one per line, as `field: value`. A message (the spec calls it an *event*) is a group of fields terminated by a **blank line**. There are four fields plus comments:

| Line | Meaning |
|---|---|
| `data: <text>` | The payload the client receives. Several `data:` lines are joined with `\n`. |
| `event: <name>` | Optional **event type**. Lets the client register different handlers (e.g. `notification` vs `metric`). Default is `message`. |
| `id: <string>` | Optional **event ID**. The browser remembers the last one and sends it back as `Last-Event-ID` on reconnect — the basis of gap-free resume (§7). |
| `retry: <ms>` | How long the browser waits before reconnecting after a drop. |
| `: <text>` | A **comment** (line starting with a colon). Ignored by the client; used as a **heartbeat** to keep the connection and proxies alive. |

The one framing rule you must never get wrong: **a message is terminated by a blank line** (`\n\n`). Forgetting the second newline is the single most common reason a hand-written SSE endpoint "sends nothing" — the bytes are on the wire, but without the terminator the client is still waiting for the message to end.

### 2.2 Raw frames on the wire **[B]**

```text
event: notification
id: 42
data: {"level":"warning","text":"Large transfer flagged"}

```

A named event with an ID and a JSON payload, terminated by the blank line. The client that did `addEventListener("notification", …)` receives it; `.data` is the JSON string (you `JSON.parse` it), and the browser records `42` as the last-seen ID. A `data:` value must not contain a raw blank line (that would end the message early); JSON is safe because a newline inside a JSON string is escaped as `\n`.

### 2.3 The browser EventSource API **[B]**

The reason SSE feels effortless on the front end is that browsers implement the entire client as a built-in object, **`EventSource`**: it opens the stream, parses the wire format, dispatches each message as a DOM event, and **reconnects automatically** — resuming from the last event ID — when the connection drops.

```js
// Open the stream; the browser issues a GET and holds the response open.
// withCredentials → send our auth cookie on connect AND every auto-reconnect.
const es = new EventSource("/api/stream", { withCredentials: true });

es.onmessage = (e) => console.log("default event:", e.data, e.lastEventId);

// Named events route ONLY to their named listener (NOT onmessage).
es.addEventListener("notification", (e) => {
	const n = JSON.parse(e.data);
	showToast(n.text);
});

// onerror fires on a drop — but it means "about to auto-reconnect", not "give up".
es.onerror = () => console.warn("SSE dropped; browser will retry", es.readyState);

// readyState: 0 CONNECTING, 1 OPEN, 2 CLOSED. Stop for good with es.close().
```

Three facts shape how you design the *server*:

- **Named events route to named listeners** — design your `event:` names as a small, deliberate vocabulary so one connection cleanly carries notifications, metrics, and audit frames to separate handlers.
- **`lastEventId` + automatic resume** — every frame with an `id:` is remembered; on reconnect the browser *automatically* sends `Last-Event-ID: <that id>`, and your server replays everything after it (§7). Gap-free delivery, nearly free — but only if you emit `id:` and honor the header.
- **Reconnection is the default** — the lifecycle is *open → drop → reconnect → open …* forever. Your server must be built to be reconnected to constantly.

### 2.4 The two hard limits **[B/I]**

`EventSource` has two constraints that dictate real architecture:

1. **You cannot set request headers.** The constructor takes a URL and optionally `{ withCredentials: true }` — nothing else. There is **no way to add `Authorization: Bearer …`.** This drives all of §6: you authenticate with a **cookie** (sent automatically) or a **short-lived ticket in the query string**, never a bearer header.
2. **HTTP/1.1 caps ~6 connections per domain.** Each `EventSource` holds one open for life; open six across tabs and the *seventh* request to that origin — including your normal API calls — **hangs**. The fix is **HTTP/2** (Nginx enables it trivially, §9); also never open more than one `EventSource` per tab.

---

## 3. Raw SSE in Express

### 3.1 Why start raw **[B]**

We'll adopt a library (`better-sse`) in §4, but you should first write an SSE endpoint by hand, because the library is a thin, well-designed wrapper over exactly these mechanics — and when something misbehaves in production, you debug at this level. An Express SSE handler differs from a normal one in three ways: it **sets streaming headers**, it **writes frames and flushes** (rather than sending one response and ending), and it **cleans up when the client disconnects**.

### 3.2 The headers, and why each **[B]**

```ts
import type { Request, Response } from "express";

function setSSEHeaders(res: Response): void {
	// The MIME type that makes the browser treat this as an event stream.
	res.setHeader("Content-Type", "text/event-stream");
	// Never cache a live stream, or a proxy/browser may serve a stale finite copy.
	res.setHeader("Cache-Control", "no-cache");
	// Keep the TCP connection open for the stream's life.
	res.setHeader("Connection", "keep-alive");
	// CRITICAL behind Nginx: disable proxy buffering for THIS response, or Nginx
	// holds your frames until its buffer fills and real-time delivery dies.
	res.setHeader("X-Accel-Buffering", "no");
	// Flush the headers immediately so the browser sees the stream has opened
	// before any event is written.
	res.flushHeaders();
}
```

That `X-Accel-Buffering: no` line is the most commonly-missed piece of the whole topic: everything works locally (no proxy), then you deploy behind Nginx and the stream "stops working" because Nginx is buffering it. Set it now.

### 3.3 A minimal streaming handler **[B]**

```ts
import express from "express";

const app = express();

app.get("/api/clock", (req, res) => {
	setSSEHeaders(res);

	let id = 0;
	// Write one correctly-framed event every second.
	const timer = setInterval(() => {
		id++;
		// A full frame: id + named event + JSON data + the MANDATORY blank line.
		res.write(`id: ${id}\n`);
		res.write(`event: tick\n`);
		res.write(`data: ${JSON.stringify({ time: new Date().toISOString() })}\n\n`);
		// Note: no explicit flush needed in Node — res.write pushes to the socket.
		// (This is why Node SSE is simpler than Go: the http.Flusher step is implicit.)
	}, 1000);

	// CLEAN UP when the client goes away (tab closed, network died). Without this,
	// the interval runs forever for every dead client — a classic memory/CPU leak.
	req.on("close", () => {
		clearInterval(timer);
	});
});

app.listen(8080);
```

Point a browser's `new EventSource("http://localhost:8080/api/clock")` with `addEventListener("tick", …)` at it and you have a live stream. The anatomy matches §2 exactly: `id:` then `event:` then `data:` then the blank line. Two things do quiet essential work: `res.write` with the `\n\n` terminator frames each message, and `req.on("close", …)` is your disconnect signal — the single most important line, because forgetting it leaks a timer (and whatever it references) for every client that ever connected.

### 3.4 Why raw doesn't scale — what's next **[B/I]**

The handler above owns its event source (a timer). That's fine for a clock, but a real dashboard has *many* clients who must all receive the *same* events — a notification isn't generated by a timer inside one request; it's generated elsewhere (a service handling a transfer) and must reach *every* connected admin. You also need: heartbeats so idle connections survive proxies, per-client buffering and backpressure, correct `Last-Event-ID` handling, and clean teardown. Writing all of that by hand, correctly, for every endpoint is exactly the boilerplate a good library removes — which is why we adopt `better-sse` next rather than reinventing it.

---

## 4. Adopting better-sse the Production Library

### 4.1 The library landscape, and why better-sse **[I]**

Before picking a dependency, know the field:

| Option | State | Verdict |
|---|---|---|
| **Raw `res.write`** | Always available | Great for learning and tiny cases; you re-implement heartbeats, sessions, channels, `Last-Event-ID`, and cleanup yourself. Error-prone at scale. |
| **`sse-channel`** | Old, largely unmaintained | Historical; avoid for new code. |
| **`ssestream` / ad-hoc gists** | Fragmentary | Not a complete solution. |
| **`better-sse`** | **Modern, TypeScript-first, actively maintained, framework-agnostic** | **The recommended production choice.** Sessions, channels, automatic heartbeats, event `id`/`retry`, `Last-Event-ID` handling, typed events — with a clean API and no framework lock-in. |

`better-sse` is the current de-facto standard for SSE in the Node ecosystem. It is framework-agnostic (works with Express, Fastify, Koa, or bare `http`), written in TypeScript with full types, and — importantly — it implements the fiddly-but-critical parts (heartbeats, `Last-Event-ID` parsing, per-message framing) correctly so you don't have to. It does **not** hide the protocol from you; it models exactly the two concepts you need: a **Session** (one client connection) and a **Channel** (a broadcast group of sessions).

```bash
npm install better-sse
# Auth + data + backplane deps we'll use across the guide:
npm install jsonwebtoken argon2 pg ioredis cookie-parser
npm install -D typescript @types/express @types/jsonwebtoken @types/pg @types/cookie-parser
```

### 4.2 Session — one client connection **[I]**

A **Session** wraps one request/response into an SSE connection. `createSession(req, res)` sets the correct headers, opens the stream, wires up heartbeats, parses any incoming `Last-Event-ID`, and gives you a `push` method. It's the raw handler of §3 done correctly, in one call.

```ts
import { createSession } from "better-sse";

app.get("/api/clock", async (req, res) => {
	// Creates the SSE session: sets headers, flushes, starts heartbeats, and
	// reads req's Last-Event-ID header for you. Resolves once the stream is open.
	const session = await createSession(req, res);

	let id = 0;
	const timer = setInterval(() => {
		id++;
		// push(data, eventName, eventId) — better-sse frames and writes it, JSON-
		// encoding the data automatically. No manual \n\n, no manual flush.
		session.push({ time: new Date().toISOString() }, "tick", String(id));
	}, 1000);

	// The session emits "disconnected" when the client goes away — clean up here.
	session.on("disconnected", () => clearInterval(timer));
});
```

Compare to §3.3: no header boilerplate, no manual framing, no forgotten flush, heartbeats handled, and `Last-Event-ID` already parsed into `session.lastId` for you. The `disconnected` event replaces `req.on("close")`. This is the whole value proposition — the mechanics are correct by construction, and you focus on *what* to send.

### 4.3 The session's useful surface **[I]**

A `Session` gives you everything a production handler needs:

- **`session.push(data, eventName?, eventId?)`** — send a framed event; `data` is JSON-encoded automatically.
- **`session.lastId`** — the `Last-Event-ID` the client presented on (re)connect, ready for replay (§7).
- **`session.isConnected`** — whether the stream is still open.
- **`session.state`** — a typed per-session bag for attaching your own data (e.g. the authenticated `userId`).
- **events `connected` / `disconnected`** — lifecycle hooks for setup/teardown.
- **automatic heartbeats** — better-sse sends periodic comments to keep the connection and proxies alive, so you don't hand-roll a keep-alive timer.

The single most useful field for a real app is `session.state`, because it's where the authenticated identity lives (set in §6) and what per-event authorization reads (§11).

---

## 5. Channels and Targeted Delivery

### 5.1 Why channels **[I]**

A `Session` is one client. Real broadcasting needs a **group**: "send this notification to every connected admin," or "send this only to user 42's sessions." better-sse models that as a **Channel** — a set of registered sessions you can broadcast to at once. A channel handles the fan-out and, crucially, removes sessions automatically when they disconnect, so you never leak references to dead connections (the classic memory leak of hand-rolled hubs).

```ts
import { createChannel, createSession } from "better-sse";

// One app-wide channel every dashboard session joins. Create it ONCE at boot.
const events = createChannel();

app.get("/api/stream", async (req, res) => {
	const session = await createSession(req, res);
	// Register this session with the channel; better-sse auto-deregisters it on
	// disconnect, so there is no manual cleanup and no leak of dead sessions.
	events.register(session);
});

// Anywhere else in the app — a transfer service, a webhook handler — broadcast:
function notifyAllAdmins(text: string) {
	// broadcast(data, eventName) sends to EVERY registered session at once.
	events.broadcast({ level: "warning", text }, "notification");
}
```

### 5.2 Targeted delivery — only the right user **[I]**

Broadcasting to *everyone* is rarely what a banking dashboard wants — a notification about Alice's account must not appear on Bob's screen. There are two clean patterns:

- **Per-user channels.** Keep a `Map<userId, Channel>` and broadcast to one user's channel. Simple and explicit.
- **One channel + filter on push.** Keep sessions in one channel but iterate and push only to sessions whose `state.userId` matches. Fewer objects; the filter *is* your authorization boundary.

```ts
// Pattern A: per-user channels, created lazily and cleaned up when empty.
const userChannels = new Map<number, ReturnType<typeof createChannel>>();

function channelFor(userId: number) {
	let ch = userChannels.get(userId);
	if (!ch) {
		ch = createChannel();
		userChannels.set(userId, ch);
		// When the last session leaves, drop the channel so the Map doesn't grow
		// without bound — a subtle leak if you forget it.
		ch.on("session-deregistered", () => {
			if (ch!.sessionCount === 0) userChannels.delete(userId);
		});
	}
	return ch;
}

// On connect, put the session in ITS user's channel (userId from auth — §6):
app.get("/api/stream", requireAuth, async (req, res) => {
	const session = await createSession(req, res);
	const userId = req.userId!;           // set by requireAuth middleware
	session.state.userId = userId;         // remember identity on the session
	channelFor(userId).register(session);
});

// Emit an event to exactly one user:
function notifyUser(userId: number, text: string) {
	userChannels.get(userId)?.broadcast({ level: "info", text }, "notification");
}
```

The important property: **which sessions receive an event is decided by server code from server-verified identity, never by anything the client asks for.** A client cannot request another user's channel; it only ever receives what the server routes to it. We formalize this as per-event authorization in §11 — but the channel structure is where it starts.

### 5.3 Backpressure — the detail libraries help with **[I/A]**

A slow or sleeping client whose TCP buffer fills can, in a naive hand-rolled hub, block the whole broadcast loop (as the Go SSE guide's hub shows in detail). better-sse writes to each session independently and does not let one stuck socket freeze the others, but you are still responsible for **not queueing unboundedly** for a dead-slow client. In practice: keep event payloads small, don't emit thousands/second to a single session, and for very high-frequency data (per-tick metrics) prefer coalescing (send the latest value, not every intermediate one). If a client truly can't keep up, letting its connection drop and relying on the browser's reconnect + `Last-Event-ID` replay (§7) is the correct recovery — a dropped-and-resumed client is nearly seamless to the user.

---

## 6. Authenticating the Stream

### 6.1 The core constraint, restated **[I]**

Recall the hard limit from §2.4: **`EventSource` cannot send custom headers.** Your normal API sends `Authorization: Bearer <jwt>`; the SSE connection *cannot*. So "how do I know who is on the other end, and that they're allowed?" needs a different answer than your REST endpoints. There are two industry-standard answers; a professional system uses the first for browsers and keeps the second for non-browser/cross-origin clients.

### 6.2 Argon2id login issuing a secure cookie **[I/A]**

First, authenticate the user normally and put the resulting JWT in a **secure, `HttpOnly`, `SameSite` cookie**. The browser then attaches it automatically to the `EventSource` request and every reconnect (with `withCredentials`), so the stream is authenticated with **zero client code** and **no token in a URL**. Passwords are verified with **Argon2id** — the current best-practice password hash.

```ts
import argon2 from "argon2";
import jwt from "jsonwebtoken";

const JWT_SECRET = process.env.JWT_SECRET!; // 32+ random bytes, from a secret manager

// Registration: hash with Argon2id (memory-hard; resistant to GPU cracking).
async function hashPassword(plain: string): Promise<string> {
	return argon2.hash(plain, { type: argon2.argon2id });
}

app.post("/api/login", express.json(), async (req, res) => {
	const { email, password } = req.body;
	const user = await db.findUserByEmail(email);
	// argon2.verify is constant-time and reads the params from the stored hash.
	// Always run it even if the user is missing (compare against a dummy hash) to
	// avoid a timing side-channel that reveals which emails exist.
	if (!user || !(await argon2.verify(user.passwordHash, password))) {
		return res.status(401).json({ error: "invalid credentials" });
	}

	// Short-lived access token. Pin the algorithm explicitly (see 6.4).
	const token = jwt.sign(
		{ sub: user.id, role: user.role },
		JWT_SECRET,
		{ algorithm: "HS256", expiresIn: "15m" },
	);

	// Set it as a hardened cookie. HttpOnly → JS cannot read it (XSS-safe);
	// Secure → HTTPS only; SameSite=Strict → not sent cross-site (CSRF-safe) and,
	// since the stream is same-origin, costs nothing.
	res.cookie("session", token, {
		httpOnly: true,
		secure: true,
		sameSite: "strict",
		maxAge: 15 * 60 * 1000,
		path: "/",
	});
	res.json({ ok: true });
});
```

The properties that make this banking-grade: `HttpOnly` means a successful XSS **cannot read the token** (the number-one way tokens are stolen); `Secure` keeps it off plain HTTP; `SameSite=Strict` blocks CSRF and is free because the dashboard and stream are the same origin (behind Nginx, §9).

### 6.3 The auth middleware **[I/A]**

One middleware verifies the cookie and attaches the identity; both normal endpoints and the stream use it.

```ts
import type { Request, Response, NextFunction } from "express";

// Augment Express's Request type with our fields (TypeScript).
declare global {
	namespace Express {
		interface Request { userId?: number; role?: string; }
	}
}

function requireAuth(req: Request, res: Response, next: NextFunction) {
	const token = req.cookies?.session;         // requires cookie-parser middleware
	if (!token) return res.status(401).json({ error: "no session" });
	try {
		const claims = verifyJwt(token);          // rigorous verification — see 6.4
		req.userId = claims.sub;
		req.role = claims.role;
		next();
	} catch {
		res.status(401).json({ error: "invalid session" });
	}
}
```

### 6.4 Verifying the JWT rigorously **[A]**

The infamous JWT vulnerability is **algorithm confusion**: an attacker sets the token's `alg` to `none`, or swaps `RS256` for `HS256` to trick the server into verifying an RSA-signed token with the public key as an HMAC secret. `jsonwebtoken` defends against this only if you **pin the accepted algorithms** — never trust the token's own `alg` header.

```ts
interface Claims { sub: number; role: string; }

function verifyJwt(token: string): Claims {
	// The `algorithms` allow-list is the line that stops alg-confusion attacks.
	// Without it, jsonwebtoken would honor whatever alg the token claims.
	const decoded = jwt.verify(token, JWT_SECRET, {
		algorithms: ["HS256"],   // ONLY HS256 is acceptable
	}) as jwt.JwtPayload;
	return { sub: Number(decoded.sub), role: String(decoded.role) };
}
```

This is the same discipline the WebSocket and auth guides apply; SSE reuses it unchanged. The only SSE-specific twist is *how the token reaches the endpoint* (cookie/ticket), not how it's validated once there.

### 6.5 The ticket alternative — for cross-origin or non-browser clients **[I/A]**

When a cookie isn't available (a cross-origin dashboard, a mobile/native client), use a **ticket**: the client, already authenticated by its normal bearer token, makes a quick `POST /api/sse-ticket`; the server returns a **single-use, short-lived (≈30s) token**; the client opens `new EventSource("/api/stream?ticket=" + t)`. The stream validates and immediately burns the ticket.

The ticket must be short-lived and single-use because **URLs leak** — into access logs, proxy logs, browser history, `Referer` headers. A 30-second single-use ticket that's already consumed by the time anyone reads the log is nearly worthless to an attacker; a long-lived JWT in a URL is a credential sitting in a dozen log files.

```ts
import { randomBytes } from "crypto";

// Issue: called via a NORMAL bearer-authenticated request.
app.post("/api/sse-ticket", requireBearerAuth, async (req, res) => {
	const ticket = randomBytes(32).toString("base64url");
	// Store ticket→userId in Redis with a 30s TTL; single-use via GETDEL on redeem.
	await redis.set(`sse_ticket:${ticket}`, String(req.userId), "EX", 30);
	res.json({ ticket });
});

// Redeem on connect: GETDEL is atomic → true single use even under a race.
async function authFromTicket(req: Request, res: Response, next: NextFunction) {
	const ticket = String(req.query.ticket ?? "");
	if (!ticket) return res.status(401).json({ error: "no ticket" });
	const userId = await redis.getdel(`sse_ticket:${ticket}`);
	if (!userId) return res.status(401).json({ error: "invalid or expired ticket" });
	req.userId = Number(userId);
	next();
}
```

Redis `GETDEL` makes redemption atomic, so two simultaneous connects can't both redeem one ticket. **Which to choose?** For a same-origin browser dashboard — this guide's case — **use the cookie.** Keep the ticket for genuinely cross-origin or non-browser clients. Never put a long-lived JWT directly in the `EventSource` URL.

---

## 7. Persistence and Gap-Free Resume

### 7.1 Why in-memory channels are not enough **[I]**

A channel broadcasts to *currently connected* sessions. But real clients disconnect constantly — a tab sleeps, a train enters a tunnel, your server redeploys. During those seconds the channel broadcasts into the void for that client, and when the browser reconnects a moment later those events are **gone**. For a chat toy that's fine; for a banking audit feed, silently dropping "a large transfer was flagged" because the reviewer's Wi-Fi blipped is unacceptable.

The fix is why `id:` and `Last-Event-ID` exist: **persist every event with a monotonic ID**, and when a client reconnects presenting its last-seen ID, **replay everything after it from the database** before resuming the live stream. Combined with the browser's automatic reconnect, this gives *at-least-once, gap-free* delivery — the client sees every event in order, whether or not it was connected at the time.

### 7.2 The events table (node-pg-migrate) **[I]**

The event store is an **append-only outbox**: every event worth delivering is inserted here first, then broadcast. A `BIGSERIAL` id gives the monotonic, gap-detecting sequence resume needs.

```sql
-- migrations/1700000000000_events.sql  (node-pg-migrate raw-SQL migration)
CREATE TABLE events (
	id          BIGSERIAL PRIMARY KEY,
	-- Who may see this event. NULL = global (everyone); otherwise one user.
	user_id     BIGINT REFERENCES users(id),
	name        TEXT        NOT NULL,        -- the SSE `event:` name
	data        JSONB       NOT NULL,        -- the payload
	created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Replay filters by "id > last-seen", scoped to a recipient; index for it.
CREATE INDEX events_user_id_id_idx ON events (user_id, id);
```

### 7.3 Persist-then-broadcast **[I/A]**

The rule is **persist first, broadcast second.** If you broadcast before the insert commits and the insert then fails, connected clients saw an event that doesn't exist in history — so a *different* client that reconnects and replays from the DB won't see it, and now two clients disagree. Insert first; the DB is the source of truth; the live broadcast is an optimization.

```ts
import { Pool } from "pg";
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// The single choke point for emitting events. Call from anywhere.
async function publishEvent(userId: number | null, name: string, data: unknown) {
	// 1) PERSIST FIRST — parameterized query ($1..$3) is injection-proof. The
	//    RETURNING id gives the authoritative monotonic id.
	const { rows } = await pool.query(
		`INSERT INTO events (user_id, name, data) VALUES ($1, $2, $3) RETURNING id`,
		[userId, name, JSON.stringify(data)],
	);
	const id = String(rows[0].id);

	// 2) BROADCAST SECOND, tagging the event with the DB id so the browser stores
	//    it as Last-Event-ID and can resume correctly.
	if (userId === null) {
		events.broadcast(data, name, { eventId: id });          // global
	} else {
		userChannels.get(userId)?.broadcast(data, name, { eventId: id }); // scoped
	}
}
```

Every event now has a durable home and a stable ID *before* any client hears about it — so a reconnecting client can always be caught up, and the DB doubles as the audit log a banking system needed anyway.

### 7.4 Replaying missed events on reconnect **[I/A]**

When a browser reconnects it *automatically* sends `Last-Event-ID: <n>`, which better-sse exposes as `session.lastId`. Catch the client up from the DB **before** registering it with the live channel — doing replay first preserves order and avoids a gap between "end of replay" and "start of live."

```ts
app.get("/api/stream", requireAuth, async (req, res) => {
	const session = await createSession(req, res);
	const userId = req.userId!;
	session.state.userId = userId;

	// 1) REPLAY: if the browser presented a Last-Event-ID, stream what it missed.
	const lastId = session.lastId ? BigInt(session.lastId) : 0n;
	if (lastId > 0n) {
		// Scoped query: id > lastId AND (this user's events OR global). The SQL
		// itself is the authorization boundary — a forged Last-Event-ID can NEVER
		// surface another user's events. Limit bounds a long-offline catch-up.
		const { rows } = await pool.query(
			`SELECT id, name, data FROM events
			 WHERE id > $1 AND (user_id = $2 OR user_id IS NULL)
			 ORDER BY id ASC LIMIT 1000`,
			[lastId.toString(), userId],
		);
		for (const row of rows) {
			session.push(row.data, row.name, String(row.id)); // replay in order
		}
	}

	// 2) GO LIVE: only now join the channel for live events.
	channelFor(userId).register(session);
});
```

Two banking-grade details are baked in: the `(user_id = $2 OR user_id IS NULL)` predicate is **server-side authorization** (a forged `Last-Event-ID` can't leak another user's events), and `LIMIT 1000` bounds catch-up so a client offline for a week can't force a million-row load. Resume is powerful; it must also be bounded.

---

## 8. Postgres LISTEN NOTIFY as an Event Source

### 8.1 The idea — let the database tell you **[A]**

So far, events enter the system because *application code* called `publishEvent`. But often the thing that changes is a **row in the database**, changed by another service, a batch job, or a trigger — code this process never runs. Polling the table reintroduces exactly the latency and waste SSE exists to avoid.

PostgreSQL has a built-in answer: **`LISTEN` / `NOTIFY`**, an in-database pub/sub. A session runs `NOTIFY channel, 'payload'`; every session that ran `LISTEN channel` receives it instantly. If your Node server `LISTEN`s, then *anything* that can `NOTIFY` — a trigger on the `events` table, another microservice, a human in `psql` — pushes an event into your live SSE stream with no coupling to your code. The database becomes the event bus.

### 8.2 A dedicated pg client for the listener **[A]**

`LISTEN/NOTIFY` needs a connection **dedicated** to waiting for notifications — it parks and blocks until something arrives, which is a poor fit for a pooled connection that must be shared and returned quickly. So use a **standalone `pg.Client`** (not a pooled `Pool` connection), `LISTEN`, and handle its `notification` event for the process's whole life.

```sql
-- migrations/..._events_notify.sql — NOTIFY on insert. Send only the id: NOTIFY
-- payloads are capped at 8000 bytes, so notify the id and read the row by it.
CREATE FUNCTION notify_event() RETURNS trigger AS $$
BEGIN
	PERFORM pg_notify('events', NEW.id::text);
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER events_notify
	AFTER INSERT ON events
	FOR EACH ROW EXECUTE FUNCTION notify_event();
```

```ts
import { Client } from "pg";

// A standalone client dedicated to LISTEN — NOT from the pool.
async function startListener() {
	const client = new Client({ connectionString: process.env.DATABASE_URL });
	await client.connect();
	await client.query("LISTEN events");

	client.on("notification", async (msg) => {
		// msg.payload is the event id (a string). Read the full, untruncated row.
		const id = msg.payload!;
		const { rows } = await pool.query(
			`SELECT id, user_id, name, data FROM events WHERE id = $1`, [id],
		);
		if (rows.length === 0) return;
		const ev = rows[0];
		if (ev.user_id === null) {
			events.broadcast(ev.data, ev.name, { eventId: String(ev.id) });
		} else {
			userChannels.get(Number(ev.user_id))
				?.broadcast(ev.data, ev.name, { eventId: String(ev.id) });
		}
	});

	// A dedicated connection WILL eventually drop; reconnect with backoff + a
	// catch-up read (events WHERE id > last_processed) so you miss nothing.
	client.on("error", (err) => {
		console.error("listener error, reconnecting", err);
		setTimeout(startListener, 1000);
	});
}
```

The **NOTIFY-an-id-then-read** pattern is the best practice: `NOTIFY` payloads cap at 8000 bytes, so notify only the tiny id and read the full row from the table. The notification is a doorbell, not the delivery. Now *anything* that inserts an `events` row — your code, another service, a trigger — lights up every relevant dashboard, fully decoupled.

> **⚡ Reconnect the listener.** The `error` handler above must re-`LISTEN` on reconnect and do a catch-up read for notifications missed during the gap. A listener that silently dies is a dashboard that silently stops updating.

---

## 9. Scaling with a Redis Backplane

### 9.1 The multi-instance problem **[A]**

One Node process holds its channels in memory. Run **two** instances behind a load balancer — which you will, for availability and capacity — and those in-memory channels become islands. Admin Alice connects to instance A; a transfer service on instance B calls `publishEvent`; B broadcasts to *its* sessions, but Alice is on A and never sees it. Each instance only knows its own connections.

With the `LISTEN/NOTIFY` design (§8) this is *partly* solved already: if every instance `LISTEN`s, an insert notifies *all* instances, and A broadcasts to Alice. For many systems that's enough — Postgres `LISTEN/NOTIFY` is itself a cross-instance fan-out. **Add Redis** when you don't want every event to require a DB write+trigger (high-frequency ephemeral events like per-second metrics or "typing…"), when you want pub/sub load off the primary database, or when Redis is already there for tickets/rate-limits.

### 9.2 The backplane pattern **[A]**

Each instance **publishes** every event it originates to a Redis channel and **subscribes** to that channel, broadcasting anything received to its *local* better-sse channels. An event thus reaches every instance's sessions regardless of origin. Use a **separate subscriber connection** — an ioredis connection in subscribe mode can't run other commands, so `duplicate()` a dedicated one.

```ts
import Redis from "ioredis";

const redis = new Redis(process.env.REDIS_URL!);      // publisher / normal commands
const sub = redis.duplicate();                         // dedicated subscriber

// Publish every originated event to the backplane.
async function publishToBackplane(userId: number | null, name: string, data: unknown, id: string) {
	await redis.publish("sse:events", JSON.stringify({ userId, name, data, id }));
}

// Subscribe once at boot: everything on the channel → local better-sse channels.
async function startBackplane() {
	await sub.subscribe("sse:events");
	sub.on("message", (_channel, payload) => {
		const m = JSON.parse(payload);
		if (m.userId === null) {
			events.broadcast(m.data, m.name, { eventId: m.id });
		} else {
			userChannels.get(m.userId)?.broadcast(m.data, m.name, { eventId: m.id });
		}
	});
}
```

`publishEvent` now persists to Postgres (durability/replay) and publishes to Redis (live cross-instance fan-out); each instance's subscriber feeds its local channels; every dashboard on any instance receives the event. Durability comes from Postgres; speed and fan-out from Redis — complementary, not redundant.

### 9.3 Nginx in front of SSE **[A]**

The reverse proxy needs specific settings or it will quietly ruin the stream (buffering) and cut it off (timeouts). Serve over **HTTP/2** to dissolve the 6-connections-per-domain limit (§2.4), and configure the SSE location to pass frames straight through:

```nginx
server {
	listen 443 ssl;
	http2 on;                          # many EventSource streams share one connection
	server_name dashboard.example.com;

	location /api/stream {
		proxy_pass http://backend;
		proxy_http_version 1.1;

		# THE critical line: never buffer the stream. Pairs with the
		# X-Accel-Buffering: no response header (§3.2) — set BOTH.
		proxy_buffering off;
		proxy_cache off;

		# SSE connections are long-lived; a 60s default timeout would kill them.
		proxy_read_timeout 3600s;
		proxy_set_header Connection '';   # clear 'close' so keep-alive holds
		proxy_set_header Host $host;
	}

	location / { proxy_pass http://backend; }   # normal API + static app
}
```

> **⚡ Compression is the Express-specific trap.** If you use the `compression` middleware, it will buffer/batch the SSE response and break streaming. **Exclude `text/event-stream`** from compression (via its `filter` option) or don't apply it to the stream route. This bites Express apps specifically and is easy to miss because non-stream routes work fine.

The two lines that matter most in Nginx are `proxy_buffering off` and the long `proxy_read_timeout`. Serving over one origin also makes your same-site auth cookie (§6.2) "just work" — dashboard and API are the same site, so `SameSite=Strict` is both maximally secure and fully functional.

---

## 10. The Admin Dashboard UI

### 10.1 A complete, self-contained client **[I]**

The front end is deliberately the smallest part — that's SSE's promise. Here is a full admin dashboard as one self-contained HTML file: it opens the stream with `withCredentials` (sending the auth cookie), routes named events to panels, escapes all server text, and handles reconnection. No framework, so you can read every line; §10.3 notes the React equivalent.

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="utf-8" />
	<title>Admin Live Dashboard</title>
	<style>
		body { font: 14px system-ui; margin: 0; display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; padding: 1rem; }
		.panel { border: 1px solid #ccc; border-radius: 8px; padding: 1rem; height: 40vh; overflow-y: auto; }
		.status { grid-column: 1 / -1; padding: .5rem 1rem; border-radius: 6px; }
		.status.open { background: #e6ffed; } .status.closed { background: #ffeef0; }
		.item { padding: .35rem 0; border-bottom: 1px solid #eee; }
		.warning { color: #b7791f; } .critical { color: #c53030; font-weight: 600; }
	</style>
</head>
<body>
	<div id="status" class="status closed">Connecting…</div>
	<div class="panel"><h3>Notifications</h3><div id="notes"></div></div>
	<div class="panel"><h3>Audit Feed</h3><div id="audit"></div></div>
	<div class="panel"><h3>Live Metrics</h3><div id="metrics"></div></div>

	<script>
		const statusEl = document.getElementById("status");

		// NEVER inject server text as innerHTML raw (stored XSS). Escape everything.
		function escapeHTML(s) {
			return String(s).replace(/[&<>"']/g, (c) => (
				{ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]
			));
		}
		function prepend(id, html) {
			document.getElementById(id).insertAdjacentHTML("afterbegin", `<div class="item">${html}</div>`);
		}

		// withCredentials → browser sends the HttpOnly auth cookie on connect AND on
		// every automatic reconnect. No token in the URL.
		const es = new EventSource("/api/stream", { withCredentials: true });

		es.onopen = () => { statusEl.textContent = "Live"; statusEl.className = "status open"; };
		es.onerror = () => { statusEl.textContent = "Reconnecting…"; statusEl.className = "status closed"; };

		es.addEventListener("notification", (e) => {
			const n = JSON.parse(e.data);
			prepend("notes", `<span class="${escapeHTML(n.level)}">${escapeHTML(n.text)}</span>`);
		});
		es.addEventListener("audit", (e) => {
			const a = JSON.parse(e.data);
			prepend("audit", `${escapeHTML(a.actor)} — ${escapeHTML(a.action)}`);
		});
		es.addEventListener("metric", (e) => {
			const m = JSON.parse(e.data);
			document.getElementById("metrics").textContent =
				`active sessions: ${m.sessions} · events/s: ${m.rate}`;
		});
	</script>
</body>
</html>
```

That is the entire client: ~60 lines, no dependencies, and it transparently handles cookie auth, reconnection, resume (the browser sends `Last-Event-ID` automatically; the server replays), and per-event routing. Compare that to the equivalent WebSocket client with its manual reconnect/backoff/heartbeat logic, and SSE's ergonomic win for one-way feeds is obvious.

### 10.2 Why the escaping matters **[I]**

Notice `escapeHTML` wrapping *every* piece of server text before it touches the DOM. An SSE payload is attacker-influenceable — a notification's text might contain whatever a user typed into a transfer memo. `innerHTML` on `<img src=x onerror=alert(document.cookie)>` executes script in your admin's session. Because the cookie is `HttpOnly` the script can't read *it*, but it can still act as the admin. Escape on the way in, always — and better, sanitize on the way *into* the event store so bad data never persists.

### 10.3 Consuming SSE in React or Next **[I]**

In React the pattern is identical, wrapped in an effect: open the `EventSource` in `useEffect`, attach listeners that push into state, and **return a cleanup that calls `es.close()`** so an unmount (or a route change, or React StrictMode's double-invoke) doesn't leak a stream. The one React-specific trap is forgetting that cleanup — every re-render without it opens another `EventSource` and marches you into the 6-connection limit (§2.4). Open one stream at the app shell, fan its events into your state manager (or TanStack Query cache), and never open a second per component.

---

## 11. Banking-Grade Security

Real-time endpoints have every vulnerability a normal endpoint has, plus a few unique to long-lived, header-less, fan-out connections. This is the checklist that separates a demo from something you'd put in front of money. Nothing here is optional in a regulated system.

### 11.1 Authenticate before the first byte **[A]**

Every stream connection must be authenticated *before* any event is sent — which the §6 `requireAuth` middleware guarantees by running before the handler. There are **no anonymous streams**: a connection that fails auth gets `401` and is closed. Because the browser reconnects automatically, auth runs again on *every reconnect*, so a token that expired mid-stream fails the next reconnect and the connection dies — exactly what you want. Keep access tokens short (minutes) so a stolen one has a small window.

### 11.2 Authorize every event, server-side **[A]**

This is the vulnerability unique to fan-out systems and the one most often missed: **a stream must deliver only events the connected user is authorized to see.** If your routing is wrong, User A's private notification lands in User B's dashboard — a data breach. Defense is layered:

- **Scope at the source.** Per-user channels (§5.2) mean a user-scoped event only reaches that user's sessions.
- **Scope at replay.** The `(user_id = $2 OR user_id IS NULL)` predicate (§7.4) means a forged `Last-Event-ID` can *never* surface another user's events — the SQL itself filters by recipient.
- **Scope by role for shared events.** For events visible to a *class* of user (e.g. "all admins"), check the role from the verified JWT (`req.role`), never a client-supplied value.

The golden rule: **authorization is decided from server-verified identity (the JWT claims), never from anything the client sends on the stream.** The client cannot ask for events; it only receives what the server routes to it.

### 11.3 Never leak secrets into URLs or logs **[A]**

Because `EventSource` can't set headers, the temptation is a token in the query string — which then lands in Nginx access logs, app logs, browser history, and `Referer` headers. §6 avoids this with the cookie (nothing in the URL) or the **single-use 30-second ticket** (worthless by the time it hits a log). If you must log stream requests, **strip or redact the query string**. Audit your log config for this specifically.

### 11.4 Rate-limit and cap connections **[A]**

Long-lived connections are a DoS vector: each holds a socket and memory. An attacker (or a buggy client in a reconnect storm) can open thousands and exhaust file descriptors. Defenses:

- **Max concurrent streams per user/IP** — track open sessions and reject new ones past a small limit (e.g. 5 per user).
- **Rate-limit the connect and ticket endpoints** — `express-rate-limit` backed by Redis so it works across instances.
- **A global connection ceiling** — refuse new streams past an instance's capacity rather than falling over.
- **Idle/absolute timeouts** — even authenticated streams get a maximum lifetime, forcing periodic reconnect (and re-auth).

```ts
import rateLimit from "express-rate-limit";
import helmet from "helmet";

app.use(helmet());                    // sane security headers everywhere

// Throttle the connect + ticket endpoints (not the stream body — that's long-lived).
const connectLimiter = rateLimit({ windowMs: 60_000, limit: 30 });
app.post("/api/sse-ticket", connectLimiter, requireBearerAuth, issueTicket);

// Per-user open-stream cap, tracked in a Map (or Redis for multi-instance).
const openPerUser = new Map<number, number>();
function reserveSlot(userId: number): boolean {
	const n = openPerUser.get(userId) ?? 0;
	if (n >= 5) return false;            // one user does not need 50 streams
	openPerUser.set(userId, n + 1);
	return true;
}
```

### 11.5 Transport, origin, and input hygiene **[A]**

- **TLS always.** A stream over plain HTTP sends the auth cookie and every event in cleartext. `Secure` cookies + HTTPS-only, no exceptions.
- **CORS deliberately.** If same-origin (recommended), you need no CORS and cross-site JS can't open your stream. If you must allow another origin, set an explicit allow-list (never `*` with credentials) and require `withCredentials`. Unrestricted CORS on a credentialed stream is a cross-site data leak.
- **Escape/sanitize all event data** — it flows to the DOM; treat it as hostile (§10.2).
- **Minimize what you stream** — send the notification text, not the customer's full record, so a client-side compromise leaks less.

---

## 12. Graceful Shutdown and Memory Management

### 12.1 Why streams make shutdown and leaks harder **[I/A]**

Every open SSE session owns a socket, an entry in a channel, and any timers/listeners you attached. Two failure modes follow: on **deploy**, killing the process mid-frame drops every client rudely (they reconnect, but you lose in-flight cleanliness); and over **time**, any session whose cleanup you forgot becomes a leak — a `setInterval` that never clears, a listener never removed, a `Map` entry never deleted. On a long-running Node process these accumulate until the heap grows and the event loop slows.

### 12.2 Clean shutdown on SIGTERM **[I/A]**

better-sse deregisters sessions from channels automatically on disconnect, which removes the most common leak. For deploys, track your sessions and close them on `SIGTERM`, then stop accepting new connections:

```ts
const server = app.listen(8080);
const liveSessions = new Set<{ close: () => void }>(); // register sessions here on connect

process.on("SIGTERM", async () => {
	// 1) Stop accepting new HTTP connections.
	server.close();
	// 2) Politely end open streams; browsers will reconnect to a healthy instance.
	for (const s of liveSessions) s.close?.();
	// 3) Close DB pool, Redis, and the listener client so the process can exit.
	await Promise.allSettled([pool.end(), redis.quit(), sub.quit()]);
	process.exit(0);
});
```

### 12.3 The leak checklist **[A]**

- **Always clear timers** attached to a session in its `disconnected` handler.
- **Always remove listeners** you added (`emitter.off(...)`) when the session ends.
- **Prune empty per-user channels** from your `Map` (§5.2) or it grows without bound.
- **Cap catch-up replay** (`LIMIT 1000`, §7.4) so a long-offline client can't load huge result sets into memory.
- **Watch RSS and open-handle count in production** — a steadily climbing memory or handle count is a leak (a session not cleaned, a channel not pruned), and it will eventually take the process down.

---

## 13. Testing SSE

### 13.1 What's actually hard to test **[I/A]**

SSE handlers are streaming and long-lived, so the usual "call handler, assert on the finished response" doesn't fit — the response never finishes. The approach: start the real app, connect with an HTTP client that lets you read the body *incrementally*, assert on the frames that arrive, then abort to end the stream. You test three things: (1) the handler emits correctly-framed events, (2) channel fan-out reaches the right sessions and not the wrong ones, (3) auth rejects the unauthenticated. See the [Testing in Node.js](NODE_TESTING_GUIDE.md) guide for the runner setup; here we focus on the streaming-specific technique.

### 13.2 Reading a stream in a test **[I/A]**

`fetch` (built into Node 18+) exposes the response body as a readable stream, which is perfect for reading SSE frames as they arrive. Using `node:test` and `AbortController`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";

test("stream delivers an event to an authorized user", async () => {
	const base = await startTestServer();          // your app on an ephemeral port
	const ac = new AbortController();

	// Open the stream (auth cookie set via a prior login in startTestServer).
	const res = await fetch(`${base}/api/stream`, {
		headers: { cookie: testSessionCookie },
		signal: ac.signal,
	});
	assert.equal(res.headers.get("content-type"), "text/event-stream");

	// Trigger an event after we're connected.
	setTimeout(() => publishEvent(TEST_USER_ID, "notification", { text: "hello" }), 50);

	// Read the body incrementally until we see our frame, then abort to finish.
	const reader = res.body!.getReader();
	const decoder = new TextDecoder();
	let buffer = "";
	const deadline = Date.now() + 2000;
	while (Date.now() < deadline) {
		const { value, done } = await reader.read();
		if (done) break;
		buffer += decoder.decode(value, { stream: true });
		if (buffer.includes('"hello"')) {
			ac.abort();                              // success — end the stream
			return;
		}
	}
	assert.fail("did not receive event in time");
});
```

The pattern generalizes: connect, trigger an event, read the body for the expected frame, abort to finish. Add negative tests — an unauthenticated connect returns `401`, and an event for user A never appears in user B's stream (the authorization test that matters most for a banking dashboard). For the DB-backed pieces (replay, `LISTEN/NOTIFY`), use **Testcontainers for Node** to run a real Postgres exactly as the testing guide describes.

---

## 14. Gotchas and Best Practices

A concentrated list of the mistakes that actually bite, many learnable only the hard way.

| Pitfall | Symptom | Fix |
|---|---|---|
| **`compression` middleware on the stream** | Stream stalls or batches; "not real-time". | Exclude `text/event-stream` from compression (its `filter` option), or don't apply it to the stream route. **The #1 Express-specific SSE bug.** |
| **Missing blank line** | Connected but no `message` events fire. | Every frame ends with `\n\n`. Or use better-sse, which frames for you. |
| **Proxy buffering** | Works locally, "freezes" behind Nginx. | `X-Accel-Buffering: no` header **and** `proxy_buffering off` in Nginx. |
| **No `req.on("close")` cleanup (raw)** | Timers/CPU leak for every dead client. | Clean up on disconnect; better-sse's `disconnected` event handles this. |
| **HTTP/1.1 6-connection limit** | 7th request to the origin hangs. | Serve over **HTTP/2**; one `EventSource` per tab. |
| **Token in the URL** | Credentials in access logs/history. | Cookie auth, or single-use short-lived ticket; strip/redact query strings in logs. |
| **No `id:` / ignoring `Last-Event-ID`** | Reconnecting clients miss events. | Emit `id:` on every event; replay `WHERE id > last` on reconnect (§7.4). |
| **Broadcast before persist** | Reconnecting clients disagree with live ones. | Persist first, broadcast second (§7.3). |
| **NOTIFY payload too big** | Truncated/garbled events. | NOTIFY the id only; read the full row by id (§8.2). |
| **Subscriber connection runs commands** | ioredis throws in subscribe mode. | Use a dedicated `redis.duplicate()` for the subscriber. |
| **Per-user channel Map grows forever** | Slow memory leak. | Prune empty channels on `session-deregistered` (§5.2). |
| **No connection cap** | FD/memory exhaustion; self-DoS. | Per-user stream limit + global ceiling + rate-limit connects (§11.4). |
| **Unescaped payload in DOM** | Stored XSS in the admin dashboard. | Escape all server text before the DOM (§10.2); sanitize on ingest. |
| **No heartbeat (raw)** | Idle connections dropped by proxies/NAT. | Send a `:` comment periodically; better-sse does this automatically. |

**Best-practice summary:** use **better-sse** rather than hand-rolling; one `EventSource` per tab over HTTP/2; authenticate with a same-site `HttpOnly` cookie; give every event an `id` and persist it before broadcasting; scope events to their recipient server-side; exclude `text/event-stream` from compression; drive events from `LISTEN/NOTIFY` and fan out with Redis when you outgrow one instance; cap and rate-limit connections; and set `proxy_buffering off` in Nginx before you deploy, not after the incident.

---

## 15. Study Path and Build-to-Learn Projects

### 15.1 A staged path **[B→A]**

1. **Frames first (§2–§3).** Build the raw clock endpoint. Watch the bytes with `curl -N http://localhost:8080/api/clock`. Then consume it with a 10-line `EventSource` page.
2. **Adopt the library (§4–§5).** Rewrite the clock with `createSession`, then add a `Channel` and a second endpoint (`POST /api/emit`) that broadcasts to three open tabs at once.
3. **Auth (§6).** Add Argon2id login that sets an `HttpOnly` cookie, protect the stream, and confirm an unauthenticated `EventSource` gets `401`.
4. **Durability (§7).** Add the `events` table via node-pg-migrate. Kill the server mid-stream, restart, and watch the browser reconnect and *replay what it missed* via `Last-Event-ID`. This is the moment SSE becomes production-real.
5. **Database-driven (§8).** Add the trigger and the `LISTEN` client. Insert a row with `psql` by hand and watch it appear on the dashboard — no Node code ran to send it.
6. **Scale (§9).** Run two instances behind Nginx with a Redis backplane; connect to one, emit on the other, confirm delivery. Then harden with §11.

### 15.2 Build-to-learn projects **[A]**

- **Live audit feed.** Every privileged action inserts an `events` row via the trigger; the admin dashboard shows the org's activity in real time with per-role filtering. Exercises §7, §8, §11.2.
- **Deployment / job log tail.** Stream a running job's log lines with resume, so a reviewer who reconnects sees the whole log. Exercises multi-line data, resume, backpressure.
- **Fraud-alert dashboard.** A scoring service `NOTIFY`s on flagged transactions; each reviewer sees only alerts for their region (role/scope authorization); alerts persist for replay and audit.
- **Live metrics wall.** A per-second `metric` event broadcast globally via Redis, rendered as moving numbers/sparklines. Exercises high-frequency ephemeral events (Redis, not DB) and coalescing.

### 15.3 Where to go next **[A]**

When a feature needs the *client* to stream to the server continuously — collaborative editing, chat, live cursors — you've outgrown SSE's one-way model; move to WebSockets with the [WebSockets in Node](NODE_WEBSOCKETS_GUIDE.md) guide, which reuses this guide's channel, auth, backplane, and persistence patterns over a bidirectional transport. For the data layer, deepen with [PostgreSQL](POSTGRESQL_GUIDE.md) and, if you prefer an ORM over raw `pg`, [Prisma](PRISMA_ORM_GUIDE.md); for the fan-out layer, [Redis](REDIS_GUIDE.md); to prove it all works, [Testing in Node.js](NODE_TESTING_GUIDE.md). And if your stack is NestJS rather than Express, the sibling [Server-Sent Events in NestJS](NESTJS_SSE_GUIDE.md) guide builds this same system with the framework's `@Sse()` decorator and RxJS streams. You now have everything to ship a real-time, banking-grade, reconnect-proof server-push system on nothing more exotic than an HTTP response you choose not to end.
