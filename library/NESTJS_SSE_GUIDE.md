# Server-Sent Events in NestJS — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Any NestJS developer who has never *streamed* anything to a browser and wants to reach the point where they can build a production, banking-grade real-time feed — live notifications, an audit-log tail, a metrics dashboard — that survives reconnects, authenticates every connection, authorizes every event, and scales across many servers. You need only to be able to read TypeScript and know what a Nest controller, module, and provider are; this guide teaches Server-Sent Events (SSE) from the wire protocol up, the *Nest way* — with the built-in `@Sse()` decorator and RxJS Observables. It is deliberately **explain-first**: every concept leads with prose — *what* it is, *why* it exists, *when* to reach for it, *how* it works, the *best practice*, and the *gotcha* — and only then shows heavily-commented code. Real-time systems fail in subtle ways (a proxy that buffers your stream, a hot Observable that leaks subscriptions, a token in a URL that lands in a log), so the "why" matters as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **NestJS 11** on **Node.js 22 / 24 LTS** with **TypeScript 5.7+**. SSE is built into Nest via the **`@Sse()`** decorator (from `@nestjs/common`), which serializes an **RxJS 7** `Observable<MessageEvent>` into the `text/event-stream` protocol for you — no third-party SSE library needed. Auth uses **`@nestjs/passport` + `passport-jwt`** (or a custom Guard) with **`jsonwebtoken`/`@nestjs/jwt`** and **`argon2`** (node-argon2) for Argon2id password hashing. The domain event bus is **`@nestjs/event-emitter` 3** (EventEmitter2); the database layer is **PostgreSQL** via **TypeORM 0.3** (or Prisma) with its migrations, plus a dedicated **`pg`** client for `LISTEN/NOTIFY`; **`ioredis` 5** provides the backplane; **`@nestjs/throttler` 6** rate-limits; **Nginx** is the reverse proxy. Fast-moving details are flagged **⚡**. The author is on **Windows 11**, so shell commands are shown for PowerShell and POSIX where they differ.
>
> **This guide's place in the library:** SSE is the *one-way* half of real-time. It builds directly on the framework taught in the [NestJS](NESTJS_GUIDE.md) guide (modules, providers, guards, the request lifecycle) and the reactive patterns you may know from [NestJS GraphQL](NESTJS_GRAPHQL_GUIDE.md) subscriptions. When you need *bidirectional* messaging (chat, collaborative editing), reach for WebSockets instead — see [WebSockets in Node](NODE_WEBSOCKETS_GUIDE.md), and §1 here for the exact decision. The data layer talks to [PostgreSQL](POSTGRESQL_GUIDE.md) (via TypeORM here; the [Prisma](PRISMA_ORM_GUIDE.md) guide slots in cleanly as an alternative). To scale past one process you'll add a [Redis](REDIS_GUIDE.md) backplane behind [Nginx](NGINX_GUIDE.md), ship it in [Docker](DOCKER_GUIDE.md), and run it in CI with [GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md). The exact same system built in Express (with the `better-sse` library) is the sibling [Server-Sent Events in Express](EXPRESS_SSE_GUIDE.md) guide — read whichever matches your framework. This guide assumes basic NestJS; it teaches SSE.

---

## Table of Contents

1. [What SSE Is and When to Use It](#1-what-sse-is-and-when-to-use-it) **[B]**
2. [The event-stream Protocol and EventSource](#2-the-event-stream-protocol-and-eventsource) **[B]**
3. [The Nest Sse Decorator and RxJS](#3-the-nest-sse-decorator-and-rxjs) **[B/I]**
4. [Streaming Domain Events with a Subject](#4-streaming-domain-events-with-a-subject) **[I]**
5. [Per-User Targeting with RxJS Filtering](#5-per-user-targeting-with-rxjs-filtering) **[I]**
6. [Authenticating the Stream with Guards](#6-authenticating-the-stream-with-guards) **[I/A]**
7. [Persistence and Gap-Free Resume](#7-persistence-and-gap-free-resume) **[I/A]**
8. [Postgres LISTEN NOTIFY as an Event Source](#8-postgres-listen-notify-as-an-event-source) **[A]**
9. [Scaling with a Redis Backplane](#9-scaling-with-a-redis-backplane) **[A]**
10. [The Admin Dashboard UI](#10-the-admin-dashboard-ui) **[I]**
11. [Banking-Grade Security](#11-banking-grade-security) **[A]**
12. [Graceful Shutdown and Lifecycle](#12-graceful-shutdown-and-lifecycle) **[I/A]**
13. [Testing SSE](#13-testing-sse) **[I/A]**
14. [Gotchas and Best Practices](#14-gotchas-and-best-practices) **[A]**
15. [Study Path and Build-to-Learn Projects](#15-study-path-and-build-to-learn-projects)

---

## 1. What SSE Is and When to Use It

### 1.1 The problem SSE solves **[B]**

Plain HTTP is a question-and-answer protocol: the client asks (a request), the server answers (a response), and the connection is done. That is perfect for "load this page" or "save this form," but it has no way to express "tell me when something happens *later*." If your server learns something new a second after the response finished — a payment cleared, an alert fired, a job completed — it has no channel to reach the browser. The browser would have to *ask again*.

For years the workaround was **polling**: ask "anything new?" every few seconds. It works but is wasteful (most responses are "nothing new," each paying full request cost) and laggy (an event just after a poll waits the whole interval). Shorten the interval and you multiply the waste. Polling is a tax you pay whether or not anything is happening.

**Server-Sent Events** removes the tax. The client opens **one** long-lived HTTP response and leaves it open; the server holds it and, whenever it has news, writes a small text-framed message and flushes it; the browser receives each message instantly. One connection carries an unbounded stream of server→client messages — no polling, no per-event overhead, no artificial latency.

The crucial word is **one-way**. SSE is a server→client pipe. The client cannot send messages back *over the SSE connection* — if it needs to talk to the server, it makes a normal HTTP request. That one-way limitation is the simplification that makes SSE dramatically easier and cheaper than WebSockets for the very common case where only the server has news to push.

### 1.2 SSE vs WebSockets vs long-polling **[B/I]**

| Transport | Direction | Reconnect | Protocol | Complexity | Best for |
|---|---|---|---|---|---|
| **Polling** | client pulls | n/a | plain HTTP | trivial | Rare, non-urgent updates. |
| **Long-polling** | server→client (faked) | manual | plain HTTP | medium | Legacy environments where SSE/WS are blocked. |
| **SSE** | **server→client only** | **automatic, built-in** | plain HTTP (`text/event-stream`) | **low** | **Live feeds, notifications, dashboards, progress, log tails.** |
| **WebSockets** | **full duplex** | manual | `ws://` upgrade | high | Chat, multiplayer, collaborative editing. |

The decision:

- **Client needs to stream to the server continuously?** → **WebSockets** (see [WebSockets in Node](NODE_WEBSOCKETS_GUIDE.md); in Nest, WebSocket Gateways). SSE can't; the client's only back-channel is ordinary HTTP requests.
- **Only the server pushes, client reacts (plus normal POSTs)?** → **SSE.** This is most "real-time" features — a notification bell, live order status, a deploy-log tail, a metrics graph, an admin activity feed. Less code, less infrastructure, fewer failure modes than WebSockets.

The reason SSE is "less" is that it *is just HTTP* — it flows through every proxy and load balancer, uses your existing cookies, auth, TLS, and logging, and the browser's `EventSource` reconnects for you. A WebSocket is a protocol upgrade every hop must support, with reconnection/heartbeats/resume you build yourself. **If only the server talks, use SSE.**

> **⚡ Nest-specific note:** unlike WebSockets, which in Nest need a **Gateway** (`@WebSocketGateway`) and a separate adapter, SSE in Nest is just a **controller method** with an `@Sse()` decorator returning an Observable. It lives inside your normal HTTP app — same guards, same interceptors, same module system — which is a big part of why it's the lighter choice.

### 1.3 What we will build **[B]**

To keep concepts concrete, this guide builds one running example incrementally: a **secure admin dashboard** for a small banking-style back office. Staff log in (Argon2id-verified password → JWT in a secure cookie), and the dashboard opens an SSE stream showing, in real time: **notifications** ("a large transfer was flagged"), an **audit feed** (who did what), and **live metrics** (active sessions, events/second). By the end it authenticates on connect via a Guard, delivers each admin only the events they're authorized to see (RxJS filtering), persists every event for reconnect replay, is driven by the database via `LISTEN/NOTIFY`, fans out across instances through Redis, and sits behind Nginx with banking-grade hardening.

---

## 2. The event-stream Protocol and EventSource

### 2.1 The frame format **[B]**

An SSE stream is UTF-8 text: **fields**, one per line, as `field: value`; a message is a group of fields terminated by a **blank line** (`\n\n`). Four fields plus comments:

| Line | Meaning |
|---|---|
| `data: <text>` | The payload. Several `data:` lines join with `\n`. |
| `event: <name>` | Optional **event type**; routes to a named client listener. Default `message`. |
| `id: <string>` | Optional **event ID**; the browser returns it as `Last-Event-ID` on reconnect (resume, §7). |
| `retry: <ms>` | How long the browser waits before reconnecting. |
| `: <text>` | A **comment/heartbeat**; ignored by the client, keeps the connection alive. |

Nest's `@Sse()` produces these frames *for you* from a `MessageEvent` object, so you rarely type the raw format — but you must understand it to debug a stream and to design your event names and IDs. The `MessageEvent` shape Nest serializes is:

```ts
// The object your Observable emits; Nest turns each into an SSE frame.
interface MessageEvent {
	data: string | object;   // → data: (objects are JSON-stringified for you)
	id?: string;             // → id:    (for Last-Event-ID resume)
	type?: string;           // → event: (the named event; "message" if omitted)
	retry?: number;          // → retry:
}
```

### 2.2 The browser EventSource API **[B]**

Browsers implement the entire client as the built-in **`EventSource`**: it opens the stream, parses frames, dispatches each as a DOM event, and **reconnects automatically**, resuming from the last event ID.

```js
// withCredentials → send the auth cookie on connect AND every auto-reconnect.
const es = new EventSource("/api/stream", { withCredentials: true });

es.addEventListener("notification", (e) => {   // matches MessageEvent.type
	const n = JSON.parse(e.data);
	showToast(n.text);
});
es.onerror = () => console.warn("dropped; browser will retry", es.readyState);
// es.close() to stop for good (e.g. on logout).
```

Three facts shape the server design: **named events** (`type`) route to named listeners, so design a small event vocabulary; **`lastEventId` + automatic resume** means every frame with an `id` is remembered and returned as `Last-Event-ID` on reconnect (§7); and **reconnection is the default** — the lifecycle is *open → drop → reconnect* forever, so your stream must tolerate constant reconnection.

### 2.3 The two hard limits **[B/I]**

1. **`EventSource` cannot set request headers** — no `Authorization: Bearer …`. This drives all of §6: authenticate with a **cookie** (sent automatically) or a **short-lived query-string ticket**, never a bearer header.
2. **HTTP/1.1 caps ~6 connections per domain** — each `EventSource` holds one for life; the 7th request to the origin hangs. Serve over **HTTP/2** (Nginx, §9) and open one `EventSource` per tab.

---

## 3. The Nest Sse Decorator and RxJS

### 3.1 How Nest models SSE **[B/I]**

Nest treats an SSE endpoint as a controller method decorated with **`@Sse()`** that returns an **RxJS `Observable<MessageEvent>`**. This is a beautiful fit: an Observable *is* "a stream of values over time," which is exactly what SSE delivers. Nest subscribes to your Observable when a client connects, serializes every emitted `MessageEvent` into a wire frame (§2.1), sets the `text/event-stream` headers, and — when the client disconnects — **unsubscribes** from the Observable for you. You never touch `res.write`, headers, or flushing; you produce a stream of objects and Nest does the plumbing.

To understand the whole topic you only need three RxJS concepts:

- **Observable** — a lazy stream of values you can subscribe to. `@Sse()` returns one.
- **Subject** — an Observable you can *also push into imperatively* (`subject.next(value)`). This is the bridge between your app's imperative code ("a transfer was flagged") and the reactive stream the client consumes. It's the Nest equivalent of the Go/Express "hub/channel."
- **Operators** (`map`, `filter`, `merge`) — functions that transform a stream. You'll `map` domain events into `MessageEvent`s and `filter` so a user only receives their own.

### 3.2 The simplest possible SSE endpoint **[B]**

```ts
import { Controller, Sse } from "@nestjs/common";
import { interval, map, Observable } from "rxjs";

interface MessageEvent { data: string | object; id?: string; type?: string; retry?: number; }

@Controller("api")
export class ClockController {
	// @Sse marks this as a Server-Sent Events endpoint. It MUST return an
	// Observable<MessageEvent>. Nest handles headers, framing, flushing, and
	// unsubscribing on disconnect — all of it.
	@Sse("clock")
	clock(): Observable<MessageEvent> {
		// interval(1000) emits 0,1,2,... every second (a "hot" ticking stream).
		return interval(1000).pipe(
			// map each tick into a MessageEvent. `type` becomes the SSE event name;
			// an object `data` is JSON-stringified for you.
			map((n) => ({
				id: String(n),
				type: "tick",
				data: { time: new Date().toISOString() },
			})),
		);
	}
}
```

Point `new EventSource("http://localhost:3000/api/clock")` with `addEventListener("tick", …)` at it and you have a live stream — no headers, no framing, no flush, no disconnect handling in *your* code. Nest's `@Sse()` is genuinely the least-boilerplate SSE of any framework in this library, precisely because RxJS already models "a stream of values" and Nest just serializes it.

### 3.3 Why `interval` isn't enough — the shape of a real app **[B/I]**

`interval` is a self-contained source, fine for a clock. But real events don't come from a timer inside one request — a notification is produced *elsewhere* (a service handling a transfer) and must reach *every* connected dashboard. So the real pattern is: a **long-lived Subject** owned by a provider, into which the whole app pushes events, and which every `@Sse()` subscription reads from. That Subject is the heart of the system, and it's what §4 builds.

> **⚡ Hot vs cold, and why it matters.** A plain `Observable` (like `interval`) is **cold**: each subscriber gets its own independent execution — two clients would each get their *own* counter starting at 0. A **Subject** is **hot**: all subscribers share one stream and see the same values from the moment they subscribe. For broadcasting the same event to many clients you need a *hot* source — a Subject. Using a cold Observable by mistake is a classic Nest-SSE bug where "each client sees different data."

---

## 4. Streaming Domain Events with a Subject

### 4.1 The event bus provider **[I]**

Create an injectable provider that owns a **Subject** — the single stream every event flows through and every `@Sse()` endpoint subscribes to. This is the Nest-idiomatic "hub."

```ts
import { Injectable } from "@nestjs/common";
import { Subject, Observable } from "rxjs";

// The shape of an event flowing through the system.
export interface AppEvent {
	id: string;            // monotonic id (from the DB) for Last-Event-ID resume
	name: string;          // the SSE event name (notification, audit, metric)
	data: unknown;         // the payload
	userId: number | null; // recipient; null = global (everyone)
}

@Injectable()
export class EventBus {
	// A hot multicast stream. Every connected @Sse subscription reads from this;
	// the whole app pushes into it via emit(). One Subject, shared by all clients.
	private readonly stream = new Subject<AppEvent>();

	// Called from ANYWHERE (a transfer service, a listener, a Redis subscriber).
	emit(event: AppEvent): void {
		this.stream.next(event);
	}

	// Exposed to the controller as a read-only Observable.
	asObservable(): Observable<AppEvent> {
		return this.stream.asObservable();
	}
}
```

### 4.2 The controller consumes the bus **[I]**

The `@Sse()` endpoint subscribes to the bus, maps each `AppEvent` to a `MessageEvent`, and returns it. Nest subscribes on connect and unsubscribes on disconnect — so a client that leaves is automatically removed from the Subject's subscriber list, with **no manual cleanup** (RxJS + Nest handle the teardown that the Go guide does with `defer` and the Express guide with `disconnected`).

```ts
import { Controller, Sse, UseGuards } from "@nestjs/common";
import { map, Observable } from "rxjs";

@Controller("api")
export class StreamController {
	constructor(private readonly bus: EventBus) {}

	@Sse("stream")
	stream(): Observable<MessageEvent> {
		return this.bus.asObservable().pipe(
			// Turn each domain event into an SSE MessageEvent.
			map((ev) => ({ id: ev.id, type: ev.name, data: ev.data })),
		);
	}
}
```

### 4.3 Bridging imperative code with EventEmitter2 **[I]**

Directly injecting `EventBus` everywhere couples your services to SSE. A cleaner Nest pattern is to let services emit **domain events** through Nest's `@nestjs/event-emitter` (EventEmitter2) — which they'd often do anyway for decoupling — and have a single listener translate those into `EventBus.emit`. Your transfer service knows nothing about SSE; it just announces "transfer.flagged," and the SSE layer decides that means a stream event.

```ts
import { Injectable } from "@nestjs/common";
import { OnEvent } from "@nestjs/event-emitter";

@Injectable()
export class EventBridge {
	constructor(private readonly bus: EventBus, private readonly store: EventStore) {}

	// A domain event emitted anywhere in the app (this.emitter.emit('transfer.flagged', …)).
	@OnEvent("transfer.flagged")
	async onTransferFlagged(payload: { userId: number; text: string }) {
		// Persist FIRST (durable id for resume), then push to the live bus (§7).
		const saved = await this.store.append(payload.userId, "notification", {
			level: "warning", text: payload.text,
		});
		this.bus.emit({
			id: String(saved.id),
			name: "notification",
			data: saved.data,
			userId: payload.userId,
		});
	}
}
```

Now the flow is decoupled and testable: services emit domain events; the bridge persists and forwards them to the SSE bus; the controller streams them. Each layer has one job.

---

## 5. Per-User Targeting with RxJS Filtering

### 5.1 Filtering the shared stream **[I]**

The Subject carries *every* event for *every* user. A banking dashboard must never show Alice's notification to Bob, so each connection filters the shared stream down to what that user is allowed to see. This is where RxJS shines: the per-connection Observable is just the bus filtered by the authenticated user id. Because filtering happens *per subscription*, each client gets its own view of the one hot stream.

```ts
import { Controller, Sse, Req, UseGuards } from "@nestjs/common";
import { filter, map, Observable } from "rxjs";
import type { Request } from "express";

@Controller("api")
@UseGuards(SseAuthGuard)                 // authenticates the connection — §6
export class StreamController {
	constructor(private readonly bus: EventBus) {}

	@Sse("stream")
	stream(@Req() req: Request): Observable<MessageEvent> {
		const userId = req.user!.id;       // set by the Guard from the verified JWT

		return this.bus.asObservable().pipe(
			// AUTHORIZATION as a stream operator: keep only events this user may see —
			// their own (userId match) or global (userId === null). A user can NEVER
			// receive another user's event because the filter runs server-side on
			// server-verified identity. This is the per-event authz boundary (§11).
			filter((ev) => ev.userId === null || ev.userId === userId),
			map((ev) => ({ id: ev.id, type: ev.name, data: ev.data })),
		);
	}
}
```

The important property: **which events a client receives is decided by server code from the Guard-verified `req.user`, never by anything the client sends.** The `filter` operator *is* the authorization gate. A client cannot widen its own filter; it only ever sees what the server's predicate allows.

### 5.2 Role-scoped events **[I]**

For events visible to a *class* of user (e.g. "all admins"), extend the filter with the role from the verified JWT:

```ts
filter((ev) => {
	if (ev.userId !== null) return ev.userId === userId;   // user-scoped
	if (ev.minRole) return roleAtLeast(req.user!.role, ev.minRole); // role-scoped
	return true;                                            // truly global
}),
```

Because `req.user.role` comes from the Guard-verified token, a client cannot promote itself to see admin-only events. Authorization is always from server-verified identity — the same rule as the Go and Express guides, expressed here as an RxJS predicate.

---

## 6. Authenticating the Stream with Guards

### 6.1 The constraint and the Nest tool **[I]**

Recall the hard limit (§2.3): **`EventSource` cannot send headers**, so no `Authorization: Bearer …` on the stream. Nest's idiomatic auth mechanism is a **Guard** — a class that runs before the handler and returns true/false (or throws) to allow/deny. Guards work perfectly for SSE because they run on the *connect* request, before Nest subscribes to your Observable. The only SSE twist is *where the Guard reads the token from*: not the `Authorization` header (the browser can't set it), but a **cookie** (browser sends it automatically) or a **short-lived ticket** in the query string.

### 6.2 Argon2id login issuing a secure cookie **[I/A]**

```ts
import { Body, Controller, Post, Res } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import type { Response } from "express";
import * as argon2 from "argon2";

@Controller("api")
export class AuthController {
	constructor(private readonly jwt: JwtService, private readonly users: UsersService) {}

	@Post("login")
	async login(@Body() dto: { email: string; password: string }, @Res({ passthrough: true }) res: Response) {
		const user = await this.users.findByEmail(dto.email);
		// Argon2id verify (constant-time, params read from the stored hash). Always
		// run a verify even if the user is missing (dummy hash) to avoid a timing
		// oracle that reveals which emails exist.
		if (!user || !(await argon2.verify(user.passwordHash, dto.password))) {
			throw new UnauthorizedException("invalid credentials");
		}
		// Short-lived access token; algorithm pinned in JwtModule config (6.4).
		const token = await this.jwt.signAsync({ sub: user.id, role: user.role });

		// Hardened cookie: HttpOnly (JS can't read it → XSS-safe), Secure (HTTPS
		// only), SameSite=Strict (CSRF-safe; free since the stream is same-origin).
		res.cookie("session", token, {
			httpOnly: true, secure: true, sameSite: "strict",
			maxAge: 15 * 60 * 1000, path: "/",
		});
		return { ok: true };
	}
}

// Registration hashing:
export async function hashPassword(plain: string): Promise<string> {
	return argon2.hash(plain, { type: argon2.argon2id });
}
```

### 6.3 The SSE auth Guard **[I/A]**

The Guard reads the cookie, verifies the JWT, and attaches `req.user`. The same Guard protects normal endpoints and the stream.

```ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import type { Request } from "express";

@Injectable()
export class SseAuthGuard implements CanActivate {
	constructor(private readonly jwt: JwtService) {}

	async canActivate(ctx: ExecutionContext): Promise<boolean> {
		const req = ctx.switchToHttp().getRequest<Request>();
		// Read from the COOKIE (not Authorization — EventSource can't set it).
		// requires cookie-parser middleware in main.ts.
		const token = (req as any).cookies?.session
			?? extractTicket(req);          // fall back to a ticket (6.5) if you support it
		if (!token) throw new UnauthorizedException("no session");
		try {
			// verifyAsync enforces the algorithm/secret from JwtModule (6.4).
			const claims = await this.jwt.verifyAsync(token);
			(req as any).user = { id: Number(claims.sub), role: String(claims.role) };
			return true;
		} catch {
			throw new UnauthorizedException("invalid session");
		}
	}
}
```

### 6.4 Configuring JWT rigorously **[A]**

The infamous JWT vulnerability is **algorithm confusion** (`alg: none`, or `RS256`→`HS256` swaps). Pin the algorithm in `JwtModule` so the verifier never trusts the token's own `alg` header:

```ts
import { JwtModule } from "@nestjs/jwt";

JwtModule.register({
	secret: process.env.JWT_SECRET,        // 32+ random bytes from a secret manager
	signOptions: { algorithm: "HS256", expiresIn: "15m" },
	verifyOptions: { algorithms: ["HS256"] }, // the line that stops alg-confusion attacks
});
```

### 6.5 The ticket alternative **[I/A]**

When a cookie isn't available (cross-origin dashboard, native client), issue a **single-use, short-lived (~30s) ticket** from a normal bearer-authenticated `POST /api/sse-ticket`, store `ticket→userId` in Redis, and have the client open `new EventSource("/api/stream?ticket=" + t)`. Redeem with Redis `GETDEL` (atomic single-use) in the Guard. The ticket must be short-lived and single-use because **URLs leak** into logs, history, and `Referer` headers — a 30-second consumed ticket is worthless to an attacker, whereas a long-lived JWT in a URL is a credential sitting in log files.

```ts
// Redemption inside the Guard (extractTicket + burn):
async function redeemTicket(redis: Redis, req: Request): Promise<number | null> {
	const ticket = String((req.query as any).ticket ?? "");
	if (!ticket) return null;
	const userId = await redis.getdel(`sse_ticket:${ticket}`); // atomic read+delete
	return userId ? Number(userId) : null;
}
```

**Which to choose?** For a same-origin browser dashboard — this guide's case — **use the cookie.** Keep the ticket for cross-origin/non-browser clients. Never put a long-lived JWT directly in the `EventSource` URL.

---

## 7. Persistence and Gap-Free Resume

### 7.1 Why the live Subject is not enough **[I]**

The Subject broadcasts to *currently subscribed* clients. But clients disconnect constantly — a tab sleeps, a train enters a tunnel, the server redeploys. During those seconds the Subject emits into the void for that client, and on reconnect those events are **gone**. For a banking audit feed, silently dropping "a large transfer was flagged" because the reviewer's Wi-Fi blipped is unacceptable.

The fix is why `id` and `Last-Event-ID` exist: **persist every event with a monotonic ID**, and when a client reconnects presenting its last-seen ID, **replay everything after it from the database** before merging in the live stream. Combined with the browser's automatic reconnect, this gives *at-least-once, gap-free* delivery. In RxJS this is elegant: the returned Observable is the **concatenation** of a "replay from DB" Observable followed by the live filtered Subject.

### 7.2 The events entity and store (TypeORM) **[I]**

The event store is an **append-only outbox**; a `bigint` generated id gives the monotonic sequence resume needs.

```ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, Index } from "typeorm";

@Entity("events")
@Index(["userId", "id"])          // replay filters by (recipient, id > last)
export class EventEntity {
	// bigint identity → strictly increasing; the basis of Last-Event-ID resume.
	@PrimaryGeneratedColumn({ type: "bigint" })
	id!: string;                    // TypeORM maps bigint to string to avoid JS precision loss

	@Column({ type: "bigint", nullable: true })
	userId!: number | null;         // null = global (everyone)

	@Column() name!: string;        // the SSE event name
	@Column({ type: "jsonb" }) data!: unknown;

	@CreateDateColumn({ type: "timestamptz" }) createdAt!: Date;
}
```

```ts
@Injectable()
export class EventStore {
	constructor(@InjectRepository(EventEntity) private readonly repo: Repository<EventEntity>) {}

	// Append + return the row (with its authoritative id) — persist BEFORE broadcast.
	async append(userId: number | null, name: string, data: unknown): Promise<EventEntity> {
		return this.repo.save(this.repo.create({ userId, name, data }));
	}

	// Events newer than lastId this user may see, ascending — the replay query.
	async since(userId: number, lastId: string): Promise<EventEntity[]> {
		return this.repo.createQueryBuilder("e")
			.where("e.id > :lastId", { lastId })
			// visibility filter: this user's events OR global. This SQL predicate is
			// the authorization boundary — a forged Last-Event-ID can't leak others'.
			.andWhere("(e.userId = :userId OR e.userId IS NULL)", { userId })
			.orderBy("e.id", "ASC")
			.limit(1000)               // bound catch-up so a long-offline client can't OOM us
			.getMany();
	}
}
```

### 7.3 Persist-then-broadcast **[I/A]**

The rule is **persist first, broadcast second** (as the `EventBridge` in §4.3 already does). If you broadcast before the insert commits and it then fails, live clients saw an event that isn't in history — so a client that reconnects and replays from the DB won't see it, and the two disagree. Insert first; the DB is the source of truth; the live emit is an optimization.

### 7.4 Replay-then-live with RxJS concat **[I/A]**

Here the reactive model pays off beautifully. On connect, read the client's `Last-Event-ID`, build an Observable that emits the missed rows in order, and **`concat`** it with the live filtered stream — replay finishes, *then* live begins, with no gap and correct ordering.

```ts
import { concat, from, filter, map, Observable } from "rxjs";

@Sse("stream")
stream(@Req() req: Request): Observable<MessageEvent> {
	const userId = req.user!.id;
	// The browser sends this header automatically on every reconnect.
	const lastId = (req.headers["last-event-id"] as string) ?? "0";

	// 1) REPLAY: a finite Observable of the events this client missed, in order.
	const replay$ = from(this.store.since(userId, lastId)).pipe(
		// from(Promise<EventEntity[]>) emits the array once; flatten to per-row events.
		concatMap((rows) => from(rows)),
		map((row) => ({ id: String(row.id), type: row.name, data: row.data })),
	);

	// 2) LIVE: the shared bus, filtered to this user (the §5 authorization filter).
	const live$ = this.bus.asObservable().pipe(
		filter((ev) => ev.userId === null || ev.userId === userId),
		map((ev) => ({ id: ev.id, type: ev.name, data: ev.data })),
	);

	// concat = replay FIRST (to completion), THEN live — gap-free, ordered resume.
	return concat(replay$, live$);
}
```

Two banking-grade details are baked in: the `since` query's `(userId = … OR userId IS NULL)` predicate is **server-side authorization** (a forged `Last-Event-ID` can't surface another user's events), and its `LIMIT 1000` bounds catch-up. The `concat` guarantees a reconnecting client sees exactly the missed events, in order, before any live event — the cleanest expression of resume in any of the three SSE guides, thanks to RxJS.

> **⚡ The tiny race, handled.** Between the replay query and subscribing to `live$`, an event *could* be emitted. Because you persist-then-broadcast and every event carries a monotonic id, a duplicate at the seam is harmless — the client can de-dupe by id (it already tracks `lastEventId`). If you want to be strict, capture the max id before the replay query and drop live events with `id <=` it. For most systems, id-based client de-dupe is enough.

---

## 8. Postgres LISTEN NOTIFY as an Event Source

### 8.1 The idea **[A]**

Often the thing that changes is a **row in the database**, changed by another service, a batch job, or a trigger — code this Nest process never runs. Polling reintroduces the latency and waste SSE exists to avoid. PostgreSQL's built-in **`LISTEN`/`NOTIFY`** is an in-database pub/sub: a session runs `NOTIFY channel, 'payload'`; every session that ran `LISTEN channel` receives it instantly. If your Nest app `LISTEN`s, then *anything* that can `NOTIFY` — a trigger on the `events` table, another microservice, a human in `psql` — pushes into your live SSE stream with no coupling to your code. The database becomes the event bus.

### 8.2 A dedicated pg client, wired to the EventBus **[A]**

`LISTEN/NOTIFY` needs a connection **dedicated** to waiting (it parks and blocks), which is a poor fit for TypeORM's pool. Use a standalone **`pg.Client`**, `LISTEN`, and on each notification read the row and push it into the `EventBus`. Run it from a provider's lifecycle hook so it starts with the app.

```sql
-- migration: NOTIFY on insert; send only the id (NOTIFY payloads cap at 8000 bytes).
CREATE FUNCTION notify_event() RETURNS trigger AS $$
BEGIN
	PERFORM pg_notify('events', NEW.id::text);
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER events_notify AFTER INSERT ON events
	FOR EACH ROW EXECUTE FUNCTION notify_event();
```

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import { Client } from "pg";

@Injectable()
export class PgListener implements OnModuleInit, OnModuleDestroy {
	private client!: Client;
	constructor(private readonly bus: EventBus, private readonly store: EventStore) {}

	async onModuleInit() {
		await this.connect();
	}

	private async connect() {
		this.client = new Client({ connectionString: process.env.DATABASE_URL });
		await this.client.connect();
		await this.client.query("LISTEN events");

		this.client.on("notification", async (msg) => {
			const id = msg.payload!;                 // the event id (a string)
			const ev = await this.store.getById(id); // read the full, untruncated row
			if (ev) {
				this.bus.emit({ id: String(ev.id), name: ev.name, data: ev.data, userId: ev.userId });
			}
		});

		// A dedicated connection WILL drop eventually; reconnect + catch-up read so
		// notifications missed during the gap aren't lost (§7 resume, server-side).
		this.client.on("error", () => setTimeout(() => this.connect(), 1000));
	}

	async onModuleDestroy() {
		await this.client?.end();
	}
}
```

The **NOTIFY-an-id-then-read** pattern is the best practice: `NOTIFY` payloads cap at 8000 bytes, so notify the tiny id and read the full row. The notification is a doorbell, not the delivery. Now *anything* that inserts an `events` row lights up every relevant dashboard, fully decoupled — and it composes perfectly with Nest's lifecycle hooks (`OnModuleInit`/`OnModuleDestroy`) for clean startup and shutdown.

---

## 9. Scaling with a Redis Backplane

### 9.1 The multi-instance problem **[A]**

One Nest process holds its `EventBus` Subject in memory. Run **two** instances behind a load balancer and those Subjects are islands: Alice connects to instance A; a service on instance B emits; B's Subject notifies *its* subscribers, but Alice is on A and never sees it.

With the `LISTEN/NOTIFY` design (§8) this is *partly* solved — if every instance `LISTEN`s, an insert notifies all instances, and A's bus emits to Alice. Postgres `LISTEN/NOTIFY` is itself a cross-instance fan-out, enough for many systems. **Add Redis** when you don't want every event to require a DB write+trigger (high-frequency ephemeral events like per-second metrics), to keep pub/sub load off the primary database, or when Redis is already present for tickets/rate-limits.

### 9.2 The backplane provider **[A]**

Each instance **publishes** originated events to a Redis channel and **subscribes** to that channel, pushing anything received into its *local* `EventBus`. Use a dedicated subscriber connection (ioredis in subscribe mode can't run other commands).

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import Redis from "ioredis";

@Injectable()
export class RedisBackplane implements OnModuleInit, OnModuleDestroy {
	private pub = new Redis(process.env.REDIS_URL!);
	private sub = this.pub.duplicate();     // dedicated subscriber connection
	constructor(private readonly bus: EventBus) {}

	async onModuleInit() {
		await this.sub.subscribe("sse:events");
		this.sub.on("message", (_ch, payload) => {
			// Everything from the channel → this instance's local bus → its clients.
			this.bus.emit(JSON.parse(payload));
		});
	}

	// Call this instead of bus.emit for events that must reach all instances.
	async publish(event: AppEvent) {
		await this.pub.publish("sse:events", JSON.stringify(event));
	}

	async onModuleDestroy() {
		await Promise.allSettled([this.pub.quit(), this.sub.quit()]);
	}
}
```

Now `EventBridge` persists to Postgres (durability/replay) and calls `RedisBackplane.publish` (live cross-instance fan-out); each instance's subscriber feeds its local `EventBus`; every dashboard on any instance receives the event. Durability from Postgres; speed and fan-out from Redis — complementary. (Avoid double-delivery: publish to Redis and let the subscriber be the *only* path into the local bus, rather than also calling `bus.emit` directly on the origin instance.)

### 9.3 Nginx in front of SSE **[A]**

```nginx
server {
	listen 443 ssl;
	http2 on;                          # many EventSource streams share one connection
	server_name dashboard.example.com;

	location /api/stream {
		proxy_pass http://backend;
		proxy_http_version 1.1;
		proxy_buffering off;             # THE critical line — never buffer the stream
		proxy_cache off;
		proxy_read_timeout 3600s;        # long-lived; a 60s default would kill it
		proxy_set_header Connection '';
		proxy_set_header Host $host;
	}
	location / { proxy_pass http://backend; }
}
```

Serve over **HTTP/2** to dissolve the 6-connection limit (§2.3); set `proxy_buffering off` (pairs with disabling any Nest/Express compression on the stream). One origin also makes the same-site auth cookie (§6.2) work with `SameSite=Strict` at zero cost.

> **⚡ Compression trap.** If you enable response compression (the `compression` middleware in `main.ts`), it will buffer/batch the SSE stream and break it. Exclude `text/event-stream` or don't compress the stream route.

---

## 10. The Admin Dashboard UI

### 10.1 A complete, self-contained client **[I]**

The front end is deliberately the smallest part. Here is a full admin dashboard as one self-contained HTML file: it opens the stream with `withCredentials` (sending the auth cookie), routes named events (Nest's `MessageEvent.type`) to panels, escapes all server text, and handles reconnection. No framework, so you can read every line.

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

		// Named events (MessageEvent.type) route to their listeners.
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

That is the entire client: ~60 lines, no dependencies, transparently handling cookie auth, reconnection, resume (the browser sends `Last-Event-ID`; Nest's `concat` replays), and per-event routing.

### 10.2 Escaping matters **[I]**

`escapeHTML` wraps *every* server-supplied value before it touches the DOM. An SSE payload is attacker-influenceable (a notification's text may be a user's transfer memo); `innerHTML` on `<img src=x onerror=…>` runs script in the admin's session. `HttpOnly` stops the script reading the cookie, but it can still act as the admin. Escape on the way in — and sanitize on ingest so bad data never persists.

### 10.3 React or Next **[I]**

In React, open the `EventSource` in `useEffect` and **return a cleanup that calls `es.close()`** so an unmount, route change, or StrictMode double-invoke doesn't leak a stream and march you into the 6-connection limit (§2.3). Open one stream at the app shell, fan events into your state manager (or TanStack Query cache), never a second per component.

---

## 11. Banking-Grade Security

Real-time endpoints have every vulnerability a normal endpoint has, plus a few unique to long-lived, header-less, fan-out connections. This is the checklist that separates a demo from something you'd put in front of money.

### 11.1 Authenticate before the first byte **[A]**

The `SseAuthGuard` (§6.3) runs before Nest subscribes to your Observable, so there are **no anonymous streams** — a failed connect gets `401` and no stream. Because the browser reconnects automatically, the Guard runs again on *every reconnect*, so an expired token fails the next reconnect and the connection dies. Keep access tokens short (minutes).

### 11.2 Authorize every event, server-side **[A]**

The vulnerability unique to fan-out systems: **a stream must deliver only events the connected user may see.** In Nest this is the RxJS `filter` predicate (§5) plus the replay query's `WHERE` (§7.2), both keyed on the **Guard-verified `req.user`**, never on client input. A client cannot widen its filter or forge a `Last-Event-ID` into another user's data — the predicate and the SQL are the authorization boundary. For role-scoped events, check `req.user.role` from the verified JWT.

The golden rule: **authorization is from server-verified identity, never from anything the client sends.**

### 11.3 Never leak secrets into URLs or logs **[A]**

`EventSource` can't set headers, so avoid tokens in the query string — they land in access logs, history, `Referer`. Use the cookie (nothing in the URL) or the single-use 30s ticket (worthless once consumed). If you log stream requests, strip/redact the query string.

### 11.4 Rate-limit and cap connections **[A]**

Long-lived connections are a DoS vector. Use `@nestjs/throttler` on the connect and ticket endpoints, cap concurrent streams per user (e.g. 5), enforce a global ceiling, and give streams an absolute lifetime so they periodically reconnect and re-auth.

```ts
import { Throttle } from "@nestjs/throttler";

@Throttle({ default: { limit: 30, ttl: 60_000 } }) // 30 connects/min per client
@Post("sse-ticket")
issueTicket(/* ... */) { /* ... */ }
```

A per-user open-stream cap is easy with the Subject model: increment a counter in the Guard (or a provider) on connect and decrement via RxJS `finalize` (which fires when the subscription ends — the natural place, since Nest unsubscribes on disconnect):

```ts
import { finalize } from "rxjs";

@Sse("stream")
stream(@Req() req: Request): Observable<MessageEvent> {
	const userId = req.user!.id;
	if (!this.slots.reserve(userId)) throw new ForbiddenException("too many streams");
	return concat(replay$, live$).pipe(
		finalize(() => this.slots.release(userId)), // runs on disconnect — no leak
	);
}
```

### 11.5 Transport, origin, input hygiene **[A]**

- **TLS always** — a stream over plain HTTP sends the cookie and every event in cleartext. `Secure` cookies + HTTPS-only.
- **CORS deliberately** — same-origin needs none; if you must allow another origin, an explicit allow-list with credentials (never `*`).
- **Escape/sanitize all event data** — it flows to the DOM (§10.2).
- **Minimize what you stream** — the notification text, not the customer's full record.

---

## 12. Graceful Shutdown and Lifecycle

### 12.1 Nest lifecycle does the heavy lifting **[I/A]**

Nest's lifecycle hooks make SSE cleanup unusually clean. **`@Sse()` + RxJS**: Nest unsubscribes from your Observable when a client disconnects, so subscriptions don't leak — the `finalize` operator (§11.4) is your hook for any per-connection cleanup (release a slot, clear a counter). **Providers** (`PgListener`, `RedisBackplane`): implement `OnModuleDestroy` to close the dedicated `pg` client and Redis connections. **The app**: enable shutdown hooks so `SIGTERM` triggers an orderly teardown.

```ts
// main.ts
async function bootstrap() {
	const app = await NestFactory.create(AppModule);
	app.use(cookieParser());          // needed to read the auth cookie (§6.3)
	// Complete the Subject and run every provider's OnModuleDestroy on SIGTERM/SIGINT,
	// so open streams end and dedicated connections close before the process exits.
	app.enableShutdownHooks();
	await app.listen(3000);
}
```

### 12.2 The leak checklist **[A]**

- **Complete the Subject on shutdown** so all `@Sse` subscriptions end (in an `OnModuleDestroy` on `EventBus`: `this.stream.complete()`).
- **Release per-user slots** in `finalize`, not in a disconnect handler you might forget.
- **Close dedicated connections** (`pg` listener, Redis pub/sub) in `OnModuleDestroy`.
- **Bound replay** (`LIMIT 1000`, §7.2) so a long-offline client can't load huge result sets.
- **Watch RSS and subscription counts** — a climbing count means a subscription or connection isn't being torn down.

---

## 13. Testing SSE

### 13.1 Two layers to test **[I/A]**

Because Nest models SSE as an Observable, you can test most logic **without HTTP at all** — call the controller method, subscribe to the returned Observable, push events into the `EventBus`, and assert on what the Observable emits. Then add one or two **e2e** tests that go over real HTTP to prove the framing, headers, and Guard work end-to-end. See [Testing in Node.js](NODE_TESTING_GUIDE.md) for the `@nestjs/testing` setup.

### 13.2 Unit-testing the stream logic **[I/A]**

```ts
import { Test } from "@nestjs/testing";
import { firstValueFrom, filter, take, toArray } from "rxjs";

it("delivers a user-scoped event only to that user", async () => {
	const moduleRef = await Test.createTestingModule({
		providers: [EventBus /* , mocked EventStore */],
	}).compile();
	const bus = moduleRef.get(EventBus);

	// Simulate the per-connection filtered stream for user 1 (the §5 predicate).
	const forUser1 = bus.asObservable().pipe(
		filter((ev) => ev.userId === null || ev.userId === 1),
		take(1), toArray(),
	);
	const received = firstValueFrom(forUser1);

	bus.emit({ id: "1", name: "notification", data: { text: "for 2" }, userId: 2 }); // must NOT arrive
	bus.emit({ id: "2", name: "notification", data: { text: "for 1" }, userId: 1 }); // must arrive

	const events = await received;
	expect(events).toHaveLength(1);
	expect((events[0].data as any).text).toBe("for 1");
});
```

This tests the authorization filter — the most security-critical logic — as pure RxJS, in milliseconds, with no server. That testability is a real advantage of the Observable model.

### 13.3 An e2e stream test **[A]**

For the full path (Guard + framing + headers), start the app with `@nestjs/testing` and read the stream over HTTP with `fetch` + `AbortController`, exactly as the Express testing section shows: connect with the auth cookie, trigger an event, read the response body until the expected frame appears, then abort. Add the two negative tests that matter: an unauthenticated connect returns `401`, and an event for user A never appears in user B's stream. For DB-backed pieces (replay, `LISTEN/NOTIFY`), use **Testcontainers** to run a real Postgres.

---

## 14. Gotchas and Best Practices

| Pitfall | Symptom | Fix |
|---|---|---|
| **Cold Observable for broadcast** | Each client sees its own/different data. | Use a hot **Subject** for shared events (§3.3). |
| **`compression` middleware on the stream** | Stream stalls or batches. | Exclude `text/event-stream` from compression. |
| **Proxy buffering** | Works locally, "freezes" behind Nginx. | `proxy_buffering off` in Nginx; serve over HTTP/2. |
| **Returning wrong shape from `@Sse`** | Malformed frames / errors. | Emit `MessageEvent` objects (`{ data, id?, type?, retry? }`); objects in `data` are JSON-stringified for you. |
| **HTTP/1.1 6-connection limit** | 7th request to the origin hangs. | HTTP/2; one `EventSource` per tab. |
| **Token in the URL** | Credentials in logs/history. | Cookie auth, or single-use short-lived ticket; redact query strings in logs. |
| **Reading token from `Authorization`** | Guard always fails on the stream. | `EventSource` can't set headers — read the **cookie** (or ticket) in the Guard (§6.3). |
| **Ignoring `Last-Event-ID`** | Reconnecting clients miss events. | Read `req.headers['last-event-id']`; `concat` a replay Observable before live (§7.4). |
| **Broadcast before persist** | Reconnecting clients disagree with live ones. | Persist first, then emit (§4.3, §7.3). |
| **NOTIFY payload too big** | Truncated/garbled events. | NOTIFY the id only; read the row by id (§8.2). |
| **Subscriber connection runs commands** | ioredis throws in subscribe mode. | Dedicated `redis.duplicate()` subscriber (§9.2). |
| **No per-connection cleanup** | Slot counters / subscriptions leak. | Use RxJS `finalize` (fires on disconnect) (§11.4). |
| **No connection cap** | FD/memory exhaustion; self-DoS. | Per-user cap + global ceiling + `@nestjs/throttler` (§11.4). |
| **Unescaped payload in DOM** | Stored XSS in the dashboard. | Escape server text before the DOM (§10.2). |

**Best-practice summary:** use `@Sse()` + a hot **Subject**; authenticate with a Guard reading a same-site `HttpOnly` cookie; give every event an `id` and persist it before emitting; make authorization an RxJS `filter` on Guard-verified identity; `concat` replay before live for gap-free resume; drive events from `LISTEN/NOTIFY` and fan out with Redis when you outgrow one instance; clean up with `finalize` and `OnModuleDestroy`; disable compression and set `proxy_buffering off` before you deploy.

---

## 15. Study Path and Build-to-Learn Projects

### 15.1 A staged path **[B→A]**

1. **The decorator (§3).** Build the `@Sse("clock")` endpoint. Watch the frames with `curl -N http://localhost:3000/api/clock`. Consume it with a 10-line `EventSource` page.
2. **The bus (§4–§5).** Add the `EventBus` Subject and a `POST /api/emit` that pushes an event; open three tabs and watch them all update. Add the `filter` and confirm a user-scoped event reaches only its user.
3. **Auth (§6).** Add Argon2id login setting an `HttpOnly` cookie and the `SseAuthGuard`; confirm an unauthenticated `EventSource` gets `401`.
4. **Durability (§7).** Add the `events` entity and `concat` replay. Kill the server mid-stream, restart, and watch the browser reconnect and *replay what it missed* via `Last-Event-ID`.
5. **Database-driven (§8).** Add the trigger and `PgListener`. Insert a row with `psql` and watch it appear on the dashboard — no Nest code ran to send it.
6. **Scale (§9).** Run two instances behind Nginx with the `RedisBackplane`; connect to one, emit on the other, confirm delivery. Harden with §11.

### 15.2 Build-to-learn projects **[A]**

- **Live audit feed.** Every privileged action emits a domain event; the bridge persists + streams it; the dashboard shows the org's activity with per-role filtering. Exercises §4, §7, §8, §11.2.
- **Deployment / job log tail.** Stream a running job's log lines with resume so a reconnecting reviewer sees the whole log.
- **Fraud-alert dashboard.** A scoring service `NOTIFY`s flagged transactions; each reviewer sees only their region's alerts (role/scope filter); alerts persist for replay and audit.
- **Live metrics wall.** A per-second `metric` event broadcast globally via Redis, rendered as moving numbers/sparklines — high-frequency ephemeral events (Redis, not DB).

### 15.3 Where to go next **[A]**

When a feature needs the *client* to stream continuously — collaborative editing, chat, live cursors — you've outgrown SSE; move to a Nest **WebSocket Gateway** (see [WebSockets in Node](NODE_WEBSOCKETS_GUIDE.md)), reusing this guide's bus, Guard, backplane, and persistence patterns over a bidirectional transport. Deepen the data layer with [PostgreSQL](POSTGRESQL_GUIDE.md) and [Prisma](PRISMA_ORM_GUIDE.md) (an alternative to TypeORM here), the fan-out layer with [Redis](REDIS_GUIDE.md), and prove it all with [Testing in Node.js](NODE_TESTING_GUIDE.md). If your stack is Express rather than Nest, the sibling [Server-Sent Events in Express](EXPRESS_SSE_GUIDE.md) guide builds this same system with the `better-sse` library. You now have everything to ship a real-time, banking-grade, reconnect-proof server-push system the Nest way — an `Observable` Nest serializes into an HTTP response it never ends.
