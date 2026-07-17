# Server-Sent Events in Go — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Any Go developer who has never *streamed* anything to a browser and wants to reach the point where they can build a production, banking-grade real-time feed — live notifications, an audit-log tail, a metrics dashboard — that survives reconnects, authenticates every connection, authorizes every event, and scales across many servers. You need only to be able to read Go and know what an HTTP handler is; this guide teaches Server-Sent Events (SSE) from the wire protocol up. It is deliberately **explain-first**: every concept leads with prose — *what* it is, *why* it exists, *when* to reach for it, *how* it works, the *best practice*, and the *gotcha* — and only then shows heavily-commented code. Real-time systems fail in subtle ways (a proxy that buffers your stream, a channel that blocks your whole hub, a token in a URL that leaks into an access log), so the "why" matters as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Go 1.25 / 1.26** with **Gin v1.10+**, **golang-jwt/jwt v5** for tokens, **Argon2id** (`golang.org/x/crypto/argon2`) for password hashing, **Ent v0.14+** as the type-safe query layer, **goose v3.27+** for migrations, **pgx v5.10+** as the PostgreSQL driver and pool, **Air** for hot-reload in development, **Redis** (via `redis/go-redis/v9`) for the cross-instance backplane, and **Nginx** as the reverse proxy. SSE itself is a stable web standard (part of the HTML Living Standard) and has not changed in years — the moving parts are the Go libraries around it. Fast-moving details are flagged **⚡**. The author is on **Windows 11**, so shell commands are shown for PowerShell and POSIX where they differ.
>
> **This guide's place in the library:** SSE is the *one-way* half of real-time. It rides on the HTTP server you already know — [Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) is the framework here — and reuses the exact auth you built in [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md). Its data layer is the "one pgxpool" stack: [goose](GO_GOOSE_MIGRATIONS_GUIDE.md) owns the schema, [Ent](GO_ENT_ORM_GUIDE.md) is the type-safe query layer, and [pgx](GO_PGX_GUIDE.md) owns the pool and the `LISTEN/NOTIFY` connection — all talking to [PostgreSQL](POSTGRESQL_GUIDE.md). When you need *bidirectional* messaging (chat, collaborative editing), reach for WebSockets instead — see [Coder WebSocket](GO_CODER_WEBSOCKETS_GUIDE.md) for that, and §1 here for the exact decision. To scale past one process you'll add a [Redis](REDIS_GUIDE.md) backplane behind [Nginx](NGINX_GUIDE.md), and ship it in [Docker](DOCKER_GUIDE.md). This guide assumes those tools exist; it teaches SSE.

---

## Table of Contents

1. [What SSE Is and When to Use It](#1-what-sse-is-and-when-to-use-it) **[B]**
2. [The event-stream Wire Protocol](#2-the-event-stream-wire-protocol) **[B]**
3. [The Browser EventSource API](#3-the-browser-eventsource-api) **[B]**
4. [Your First SSE Endpoint in Gin](#4-your-first-sse-endpoint-in-gin) **[B]**
5. [The Broker Hub Pattern for Many Clients](#5-the-broker-hub-pattern-for-many-clients) **[I]**
6. [Authenticating the Stream](#6-authenticating-the-stream) **[I/A]**
7. [Persisting Events and the Data Layer](#7-persisting-events-and-the-data-layer) **[I/A]**
8. [Postgres LISTEN NOTIFY as an Event Source](#8-postgres-listen-notify-as-an-event-source) **[A]**
9. [Scaling Across Instances with a Redis Backplane](#9-scaling-across-instances-with-a-redis-backplane) **[A]**
10. [The Admin Dashboard UI](#10-the-admin-dashboard-ui) **[I]**
11. [Banking-Grade Security](#11-banking-grade-security) **[A]**
12. [Testing SSE](#12-testing-sse) **[I/A]**
13. [Development Workflow with Air](#13-development-workflow-with-air) **[B/I]**
14. [Gotchas and Best Practices](#14-gotchas-and-best-practices) **[A]**
15. [Study Path and Build-to-Learn Projects](#15-study-path-and-build-to-learn-projects)

---

## 1. What SSE Is and When to Use It

### 1.1 The problem SSE solves **[B]**

Plain HTTP is a question-and-answer protocol: the client asks (a request), the server answers (a response), and the connection is done. That model is perfect for "load this page" or "save this form," but it has no way to express "tell me when something happens *later*." If your server learns something new a second after the response finished — a payment cleared, an alert fired, a background job completed — it has no channel to reach the browser. The browser would have to *ask again*.

For years the workaround was **polling**: the browser asks "anything new?" every few seconds. Polling works but is wasteful — most requests return "nothing new," yet each one pays the full cost of a request (headers, TLS, a round trip, a server handler). Worse, it is *laggy*: an event that happens just after a poll waits the whole interval before the user sees it. You can shorten the interval, but then you multiply the wasted requests. Polling is a tax you pay whether or not anything is happening.

**Server-Sent Events** removes the tax. The client opens **one** long-lived HTTP response and simply *leaves it open*. The server holds that response and, whenever it has something to say, writes a small text-framed message into the still-open body and flushes it. The browser receives each message the instant it is written. One connection, opened once, carries an unbounded stream of server→client messages. There is no polling, no per-event request overhead, and no artificial latency — the user sees the event as soon as the server emits it.

The crucial word is **one-way**. SSE is a server→client pipe. The client cannot send messages back *over the SSE connection* — if it needs to talk to the server, it makes a normal HTTP request like any other. That one-way limitation is not a weakness; it is the simplification that makes SSE dramatically easier and cheaper than WebSockets for the very common case where only the server has news to push.

### 1.2 What SSE actually is, concretely **[B]**

Strip away the jargon and SSE is just three agreements:

1. **A response that never ends (until you close it).** The server sends HTTP 200 with the header `Content-Type: text/event-stream` and then keeps the body open, writing text into it over time instead of writing it all at once and finishing.
2. **A tiny text format for framing messages.** Each message is a few lines of UTF-8 text — `data: hello` — terminated by a blank line. That's the entire "protocol on top of HTTP." §2 covers every field.
3. **A built-in browser client that reconnects for you.** Browsers ship a native object, `EventSource`, that opens the stream, parses the frames, hands you each message as a JavaScript event, and — this is the killer feature — **automatically reconnects** if the connection drops, even telling the server the ID of the last event it saw so you can resume without gaps. You write almost no client code. §3 covers it.

That is the whole idea. SSE is "a normal HTTP response you never finish, carrying newline-framed text, consumed by a browser object that reconnects automatically." Everything else in this guide is a consequence of those three facts.

### 1.3 SSE vs WebSockets vs long-polling vs HTTP/2 push **[B/I]**

Choosing the right real-time transport is the most important decision you'll make here, because picking the heavy tool for a light job is a common and expensive mistake. Here is the honest comparison:

| Transport | Direction | Reconnect | Protocol | Complexity | Best for |
|---|---|---|---|---|---|
| **Polling** | client pulls | n/a | plain HTTP | trivial | Rare, non-urgent updates; the fallback of last resort. |
| **Long-polling** | server→client (faked) | manual | plain HTTP | medium | Legacy environments where SSE/WS are blocked. |
| **SSE** | **server→client only** | **automatic, built-in** | plain HTTP (`text/event-stream`) | **low** | **Live feeds, notifications, dashboards, progress, log tails — anything where only the server pushes.** |
| **WebSockets** | **full duplex** | manual (you build it) | `ws://` upgrade | high | Chat, multiplayer, collaborative editing — anything where the *client* also streams. |
| **HTTP/2 server push** | server→client | n/a | HTTP/2 frames | — | *Deprecated / removed* — do not use; it pushed *assets*, not application events. |

Read the table as a decision tree:

- **Does the client need to stream messages to the server continuously?** If yes → **WebSockets** (see the [Coder WebSocket guide](GO_CODER_WEBSOCKETS_GUIDE.md)). SSE cannot do this; the client's only back-channel is ordinary HTTP requests.
- **Does only the server push, and the client just reacts (plus the occasional normal POST)?** → **SSE.** This describes the overwhelming majority of "real-time" features: a notification bell, a live order status, a deploy-log tail, a metrics graph, an admin activity feed. For all of these, SSE is less code, less infrastructure, and fewer failure modes than WebSockets.
- **Is even SSE unavailable** (an ancient proxy that strips streaming)? → **long-polling** as a graceful degradation.

The reason SSE is "less" than WebSockets is that it *is just HTTP*. It flows through every proxy, load balancer, and firewall that already understands HTTP. It uses your existing cookies, your existing auth, your existing TLS, your existing logging. A WebSocket, by contrast, is a *protocol upgrade* — it starts as HTTP then switches to a different framing, which means every hop in your infrastructure needs to explicitly support it, and you must build reconnection, heartbeats, and resume logic yourself. When the client doesn't need to stream, all of that is complexity you're buying for nothing.

> **The one-sentence rule:** *If only the server talks, use SSE; if the client also streams, use WebSockets.* Reach for the heavier tool only when the lighter one genuinely can't express your problem.

### 1.4 What we will build **[B]**

To make every concept concrete, this guide builds one running example incrementally: a **secure admin dashboard** for a small banking-style back office. Staff log in (Argon2id-verified password → JWT), and the dashboard opens an SSE stream that shows, in real time:

- **notifications** — "a large transfer was flagged," "a new account opened,"
- an **audit feed** — who did what, streamed as it happens,
- **live metrics** — active sessions, events/second.

By the end, that stream will authenticate on connect, deliver each admin only the events they're authorized to see, persist every event so a reconnecting browser can replay what it missed, be driven directly by the database via Postgres `LISTEN/NOTIFY`, fan out across multiple server instances through a Redis backplane, and sit behind Nginx with all the banking-grade hardening spelled out. We start with the smallest possible stream and add one capability per section.

---

## 2. The event-stream Wire Protocol

### 2.1 Why a protocol at all **[B]**

Once you decide to keep an HTTP response open and dribble text into it over time, you need a way to mark where one *message* ends and the next begins — otherwise the client just sees an ever-growing blob of characters. SSE solves this with an intentionally tiny line-based format. It is so small you could implement it with `fmt.Fprintf` and a flush (and in §4 we essentially do). Understanding the raw format first — before any library hides it — is what lets you debug a stream that "isn't working" by simply reading the bytes.

### 2.2 The frame format **[B]**

An SSE stream is UTF-8 text. It is a sequence of **fields**, one per line, in the form `field: value`. A message (the spec calls it an *event*) is a group of fields terminated by a **blank line**. There are exactly four fields plus comments:

| Line | Meaning |
|---|---|
| `data: <text>` | The payload. This is what the client receives. A message can have several `data:` lines; they are joined with `\n`. |
| `event: <name>` | An optional **event type/name**. Lets the client register different handlers for different kinds of message (e.g. `notification` vs `metric`). Default is `message`. |
| `id: <string>` | An optional **event ID**. The browser remembers the last ID it saw and sends it back as the `Last-Event-ID` header when it reconnects — this is how you resume without gaps (§7). |
| `retry: <ms>` | Tells the browser how long to wait before reconnecting after a drop. A number of milliseconds. |
| `: <text>` | A **comment** — a line starting with a colon. Ignored by the client. Used as a **heartbeat** to keep the connection and intermediary proxies alive (§5). |

The one framing rule you must never get wrong: **a message is terminated by a blank line** — that is, `\n\n`. Forgetting the second newline is the single most common reason a hand-written SSE endpoint "sends nothing": the bytes are on the wire, but without the blank-line terminator the client is still waiting for the message to end.

### 2.3 Worked examples of raw frames **[B]**

Here is exactly what travels down the wire. Read these as the literal bytes in the response body.

```text
data: hello world

```

That is a complete, minimal message: one `data` field and the terminating blank line. The client fires a `message` event whose `.data` is `"hello world"`.

```text
event: notification
id: 42
data: {"level":"warning","text":"Large transfer flagged"}

```

A **named** event with an **ID** and a JSON payload. The client code that did `addEventListener("notification", ...)` receives this; `.data` is the JSON string (you `JSON.parse` it yourself), and the browser records `42` as the last-seen ID.

```text
data: line one
data: line two

```

**Multi-line data.** The two `data:` lines are joined with a newline, so the client's `.data` is `"line one\nline two"`. This is how you stream a multi-line log entry without it being split into two messages.

```text
: keep-alive

retry: 5000
data: reconnect hint above

```

The first line is a **comment heartbeat** (a `:` line) — it keeps the socket warm and is otherwise ignored. Then a `retry` field sets the browser's reconnect delay to 5 seconds, followed by a normal message. Note the comment is its own "message" terminated by a blank line.

### 2.4 The rules that bite you **[B/I]**

A few protocol details cause almost every real-world SSE bug, so internalize them now:

- **UTF-8 only.** The stream is text. To send binary, base64-encode it into `data`.
- **No blank line inside a value.** A blank line *always* means "message over." If your payload could contain `\n\n`, encode it (JSON does this for you — a newline inside a JSON string is `\n`, not a real newline).
- **Strip newlines from a single `data:` value**, or split them deliberately into multiple `data:` lines. A raw `\n` in the middle of what you thought was one value silently starts a second `data` line.
- **IDs must not contain newlines**; keep them simple (a monotonic integer or a ULID).
- **The client reconnects on its own** when the stream drops. That means if your server crashes and restarts, every browser will silently reconnect a few seconds later — which is wonderful, but it also means a bug that closes streams immediately becomes a *reconnect storm* (§14). Design as if reconnection is constant, because it is.

Everything from here on is about *producing* these frames correctly in Go and *managing* the long-lived connections that carry them.

---

## 3. The Browser EventSource API

### 3.1 Why the client is almost free **[B]**

The reason SSE feels effortless on the front end is that browsers implement the entire client for you as a built-in object called **`EventSource`**. You give it a URL; it opens the stream, parses the wire format from §2, dispatches each message as a DOM event, and — critically — **reconnects automatically** when the connection drops, resuming from the last event ID. You do not write a parser, a reconnect loop, or a backoff timer. This is the half of SSE that WebSockets makes you build by hand.

### 3.2 The core API **[B]**

```js
// Open the stream. The browser immediately issues a GET to this URL with
// `Accept: text/event-stream` and holds the response open.
const es = new EventSource("/api/stream");

// The default handler: fires for any message with NO `event:` field
// (i.e. event type "message").
es.onmessage = (e) => {
	console.log("data:", e.data);       // the text after `data:`
	console.log("lastEventId:", e.lastEventId); // the `id:` if one was sent
};

// A handler for a NAMED event: fires only for frames with `event: notification`.
es.addEventListener("notification", (e) => {
	const payload = JSON.parse(e.data);  // you parse JSON yourself
	showToast(payload.text);
});

// Fires on connection error/drop. The browser will AUTOMATICALLY try to
// reconnect afterwards unless you call es.close(). You usually just log here.
es.onerror = (err) => {
	console.warn("SSE error; browser will retry", es.readyState);
};

// readyState: 0 = CONNECTING, 1 = OPEN, 2 = CLOSED.
// To stop for good (e.g. on logout) you must close it explicitly:
// es.close();
```

Three things deserve emphasis because they shape how you design the *server*:

- **Named events route to named listeners.** If you send `event: notification`, only `addEventListener("notification", …)` receives it — *not* `onmessage`. This lets your dashboard cleanly separate notification frames from metric frames from audit frames, all on one connection. Design your server's `event:` names as a small, deliberate vocabulary.
- **`lastEventId` and automatic resume.** Every time the browser receives a frame with an `id:`, it stores it. If the connection drops and the browser reconnects, it *automatically* adds the header `Last-Event-ID: <that id>` to the new request. Your server reads that header and replays everything after it (§7). You get gap-free delivery across reconnects almost for free — but only if you send `id:` fields and honor the header server-side.
- **Reconnection is the default, not the exception.** `onerror` is not "give up" — it's "the browser noticed a drop and is about to retry." The connection lifecycle is *open → drop → reconnect → open …* forever, until the tab closes or you call `close()`. Your server must be built to be reconnected to constantly.

### 3.3 The two hard limits you must design around **[B/I]**

`EventSource` has two constraints that dictate real architectural choices, so meet them now rather than being surprised later:

1. **You cannot set request headers.** The `EventSource` constructor takes a URL and (optionally) `{ withCredentials: true }` — and nothing else. There is **no way to add an `Authorization: Bearer …` header.** This single limitation drives all of §6: because you can't send a bearer token the normal way, you authenticate the stream with either a **cookie** (which the browser attaches automatically) or a **short-lived ticket in the query string**. Do not skip §6 — getting auth right on a header-less connection is where security bugs live.

2. **HTTP/1.1 caps you at ~6 connections per domain.** Browsers allow only about six simultaneous HTTP/1.1 connections to one origin. Each `EventSource` holds one open for its whole life. Open six across a few tabs and the *seventh* request to that origin — including your normal API calls — **hangs** waiting for a slot. The fix is **HTTP/2** (or HTTP/3), where all streams share one connection and the limit effectively vanishes. In practice: serve your app over HTTP/2 (Nginx does this trivially, §9), and never open more than one `EventSource` per tab. This is a real production incident waiting to happen if you ignore it.

> **⚡ `withCredentials` and cookies:** `new EventSource(url, { withCredentials: true })` makes the browser send cookies (and honor CORS credentials) on the stream request and its reconnects. This is exactly what you want for cookie-based auth (§6) when the dashboard and API share a site. Without it, cross-origin cookies are not sent.

With the protocol (§2) and the client (§3) understood, we can now produce real frames from Go.

---

## 4. Your First SSE Endpoint in Gin

### 4.1 The three things a handler must do **[B]**

An SSE handler differs from a normal Gin handler in exactly three ways, and every one of them matters:

1. **Set the SSE headers** so the browser (and every proxy in between) treats the response as a stream, not a document to buffer.
2. **Write frames and flush after each one.** Go's HTTP server buffers writes by default; without an explicit flush, your frames sit in a buffer and the browser sees nothing until the buffer fills or the handler returns — which for a stream is *never*. Flushing is what makes "real-time" real.
3. **Stop when the client goes away.** A browser tab closes, a laptop sleeps, a network drops — you must notice and return from the handler, or you leak a goroutine (and whatever it holds) for every dead client.

### 4.2 The headers, and why each one **[B]**

```go
func setSSEHeaders(c *gin.Context) {
	h := c.Writer.Header()
	// The MIME type that makes the browser treat this as an event stream and
	// hand it to EventSource's parser rather than rendering it as a page.
	h.Set("Content-Type", "text/event-stream")
	// Never cache a live stream. Without this, a proxy or the browser may serve
	// a stale, finite copy and your "live" feed silently freezes.
	h.Set("Cache-Control", "no-cache")
	// Ask intermediaries to keep the TCP connection open for the stream's life.
	h.Set("Connection", "keep-alive")
	// CRITICAL behind Nginx: disable proxy response buffering for THIS response.
	// Nginx buffers upstream responses by default; that buffering holds your
	// frames until the buffer fills, destroying real-time delivery. This header
	// tells Nginx "stream this one straight through." (Global Nginx config in §9.)
	h.Set("X-Accel-Buffering", "no")
}
```

That `X-Accel-Buffering: no` line is the most commonly-missed piece of the entire topic. Everything works on your laptop (no proxy), then you deploy behind Nginx and the stream "stops working" — because Nginx is dutifully buffering it. Set the header now and save yourself the incident.

### 4.3 A minimal streaming handler **[B]**

Gin gives you `c.Stream`, which loops as long as you return `true` and stops when you return `false` *or* the client disconnects — Gin wires the disconnect check into the loop for you. It also gives `c.SSEvent`, which writes a correctly-framed SSE message (it appends the blank line for you). Here is a complete endpoint that pushes the server clock once per second:

```go
package main

import (
	"time"

	"github.com/gin-gonic/gin"
)

func streamClock(c *gin.Context) {
	setSSEHeaders(c)

	// A ticker is our "something happened" source for this first example.
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	// c.Stream runs the callback repeatedly. Returning true = keep streaming;
	// returning false = end the response. Gin ALSO ends it automatically when the
	// client's connection drops, so we don't have to poll for that ourselves here.
	c.Stream(func(w io.Writer) bool {
		select {
		case <-c.Request.Context().Done():
			// The client disconnected (tab closed, network died) OR the server is
			// shutting down. Either way, stop — returning false ends the handler.
			return false
		case t := <-ticker.C:
			// SSEvent writes:  event: tick\n  data: <json>\n\n  and Gin flushes it.
			c.SSEvent("tick", gin.H{"time": t.Format(time.RFC3339)})
			return true
		}
	})
}

func main() {
	r := gin.Default()
	r.GET("/api/clock", streamClock)
	r.Run(":8080")
}
```

Point a browser's `new EventSource("http://localhost:8080/api/clock")` at it with an `addEventListener("tick", …)` and you have a live stream in under 30 lines. Two things are doing quiet but essential work: `c.Request.Context().Done()` is the disconnect signal (Gin cancels the request context when the client goes away or the server shuts down), and `c.SSEvent` handles the framing and the flush so you never forget the blank line.

### 4.4 Writing frames by hand **[B/I]**

`c.SSEvent` is convenient but hides the format, and sometimes you need control it doesn't give — a raw `id:`, a comment heartbeat, a custom `retry:`. It's worth writing one frame by hand so the format from §2 is not magic. You need the `http.Flusher` interface to force bytes onto the wire:

```go
func streamManual(c *gin.Context) {
	setSSEHeaders(c)

	// The ResponseWriter must support flushing for streaming to work. Every
	// standard Go/Gin writer does, but check rather than panic on a type assert.
	flusher, ok := c.Writer.(http.Flusher)
	if !ok {
		c.String(http.StatusInternalServerError, "streaming unsupported")
		return
	}

	// Send a comment heartbeat and a retry hint up front.
	fmt.Fprintf(c.Writer, ": connected\n\n")     // comment: ignored by client
	fmt.Fprintf(c.Writer, "retry: 3000\n\n")     // reconnect after 3s if dropped
	flusher.Flush()

	id := 0
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	for {
		select {
		case <-c.Request.Context().Done():
			return
		case t := <-ticker.C:
			id++
			// A full frame, by hand: id + named event + JSON data + BLANK LINE.
			fmt.Fprintf(c.Writer, "id: %d\n", id)
			fmt.Fprintf(c.Writer, "event: tick\n")
			fmt.Fprintf(c.Writer, "data: {\"time\":%q}\n\n", t.Format(time.RFC3339))
			flusher.Flush() // WITHOUT THIS, nothing reaches the client.
		}
	}
}
```

Notice the anatomy matches §2 exactly: `id:` then `event:` then `data:` then the mandatory blank line, then a **flush**. If you remember one thing from this section: **every frame ends in `\n\n` and is followed by `Flush()`.** Miss the blank line and the client waits forever; miss the flush and the bytes never leave Go's buffer.

### 4.5 Why one handler per client doesn't scale — and what's next **[B/I]**

The handlers above each own their event source (a ticker). That's fine for a clock, but a real dashboard has *many* clients who must all receive the *same* events — a notification isn't generated by a ticker inside one request, it's generated somewhere else in your app (a service handling a transfer) and must reach *every* connected admin. You cannot have each handler independently poll the database; that's just polling with extra steps. What you need is a central place that accepts "here is an event" from anywhere in the app and delivers it to every open stream. That is the **broker/hub**, and it is the heart of a production SSE system.

---

## 5. The Broker Hub Pattern for Many Clients

### 5.1 The design, in words **[I]**

A **hub** (a.k.a. broker) is a single long-lived object, owned by the app, that knows about every currently-connected client and can broadcast an event to all of them (or a filtered subset). Each SSE handler, when it starts, **registers** itself with the hub and receives a private Go channel; the handler then loops, forwarding whatever arrives on its channel out to its client as an SSE frame. Anywhere else in the app — a transfer service, a `LISTEN/NOTIFY` listener, a Redis subscriber — code calls `hub.Broadcast(event)`, and the hub drops that event onto every registered client's channel.

The reason this is a *pattern* and not just "a slice of channels" is **concurrency safety**. Clients connect and disconnect constantly, from many goroutines, while broadcasts fire from other goroutines. If you naively share a map of clients across all of them you get data races and panics. The classic, race-free solution is to give the hub its *own* goroutine that owns the client set exclusively, and have everyone else talk to it over channels — register, unregister, and broadcast all become messages to that one goroutine. No mutex, no races, because only one goroutine ever touches the map.

### 5.2 The hub, fully commented **[I]**

```go
package sse

import (
	"encoding/json"
)

// Event is one thing worth telling clients about. Name maps to the SSE `event:`
// field; Data becomes the `data:` payload; ID (optional) becomes `id:` for resume.
type Event struct {
	ID   string
	Name string
	Data any
	// UserID scopes delivery: if non-zero, only this user's clients get it.
	// Zero means "broadcast to everyone" (e.g. a global metric). This is the
	// seed of per-event authorization we harden in §11.
	UserID int64
}

// client is one connected SSE stream. send is its private outbound channel.
type client struct {
	userID int64
	send   chan Event
}

// Hub owns all clients and runs its own goroutine so the client set is touched
// by exactly one goroutine — no locks, no races.
type Hub struct {
	register   chan *client
	unregister chan *client
	broadcast  chan Event
	clients    map[*client]struct{}
}

func NewHub() *Hub {
	return &Hub{
		register:   make(chan *client),
		unregister: make(chan *client),
		broadcast:  make(chan Event, 256), // buffered: producers don't block briefly
		clients:    make(map[*client]struct{}),
	}
}

// Run is the hub's single owning goroutine. Start it once at boot: go hub.Run().
func (h *Hub) Run() {
	for {
		select {
		case c := <-h.register:
			h.clients[c] = struct{}{}
		case c := <-h.unregister:
			if _, ok := h.clients[c]; ok {
				delete(h.clients, c)
				close(c.send) // signal the handler loop to end
			}
		case ev := <-h.broadcast:
			for c := range h.clients {
				// Deliver only to the right audience (see Event.UserID).
				if ev.UserID != 0 && c.userID != ev.UserID {
					continue
				}
				// NON-BLOCKING send. If a client's buffer is full it is a slow
				// consumer; we DROP it rather than let one stuck browser freeze
				// the whole hub. Backpressure policy, made explicit — see 5.4.
				select {
				case c.send <- ev:
				default:
					// too slow: disconnect it. delete+close here is safe because
					// we're inside the hub's own goroutine.
					delete(h.clients, c)
					close(c.send)
				}
			}
		}
	}
}

// Broadcast is the app-wide entry point. ANY goroutine may call it safely.
func (h *Hub) Broadcast(ev Event) {
	h.broadcast <- ev
}
```

The shape to internalize: **one goroutine owns the state; everyone else sends it messages.** `register`, `unregister`, and `broadcast` are all just channels into that goroutine. Because the map is only ever read or written inside `Run`, there is no shared mutable state and therefore no race — the Go race detector (`go test -race`) will confirm it.

### 5.3 The handler that bridges a client to the hub **[I]**

Now the SSE handler's job is small: register with the hub, then forward its channel to the wire until the client leaves.

```go
func (h *Hub) StreamHandler(c *gin.Context) {
	setSSEHeaders(c)

	// userID comes from auth middleware (§6). It scopes which events this
	// stream is allowed to receive.
	userID := c.GetInt64("userID")

	cl := &client{
		userID: userID,
		send:   make(chan Event, 16), // per-client buffer; full => slow consumer
	}
	h.register <- cl
	// Guarantee we unregister no matter how we exit (disconnect, error, shutdown).
	defer func() { h.unregister <- cl }()

	// Optional: a heartbeat so idle connections and proxies stay alive.
	heartbeat := time.NewTicker(15 * time.Second)
	defer heartbeat.Stop()

	c.Stream(func(w io.Writer) bool {
		select {
		case <-c.Request.Context().Done():
			return false // client gone or server shutting down
		case <-heartbeat.C:
			// A comment frame: keeps the socket and proxies warm, ignored by client.
			c.Render(-1, sseComment("keep-alive"))
			return true
		case ev, ok := <-cl.send:
			if !ok {
				return false // hub closed our channel (we were evicted)
			}
			// Marshal once; send as a named event with an id for resume.
			payload, _ := json.Marshal(ev.Data)
			c.Render(-1, sse.Event{ // gin's sse.Event renderer
				Id:    ev.ID,
				Event: ev.Name,
				Data:  string(payload),
			})
			return true
		}
	})
}
```

Every connected browser now has: one `client` in the hub, one buffered `send` channel, and one goroutine (the request handler) draining that channel to the wire. A transfer service elsewhere calls `hub.Broadcast(Event{Name: "notification", UserID: adminID, Data: …})` and the right admin's dashboard lights up — with zero polling.

### 5.4 Backpressure and slow consumers — the make-or-break detail **[I/A]**

The most important line in the hub is the **non-blocking send** in `Run`. Here is why it is not optional. Suppose one admin's laptop goes to sleep with the tab open. Its TCP connection is alive but *stalled* — nothing is being read. Your handler goroutine blocks trying to write to it, so it stops draining that client's `send` channel, so the channel fills up. Now, if the hub's broadcast used a **blocking** send (`c.send <- ev` with no `default`), the hub's *own goroutine* would block trying to hand an event to that one frozen client — and because the hub is a single goroutine, **every other client stops receiving events too.** One sleeping laptop freezes the entire dashboard for everyone.

The fix is the policy encoded above: the hub's broadcast send is `select { case c.send <- ev: default: evict }`. If a client's buffer is full, that client is *too slow* and gets dropped (its channel closed, its handler exits, the browser reconnects later and resumes from its last event ID). We sacrifice one slow client to protect the many. This "bounded buffer + evict on overflow" is the canonical backpressure policy for fan-out systems, and getting it right is the difference between a dashboard that's rock-solid and one that mysteriously freezes under load.

> **Best practice:** size the per-client buffer for your event rate (16–256 is typical), always send non-blocking from the hub, and treat eviction as normal — the client's automatic reconnect + `Last-Event-ID` replay (§7) makes a dropped-and-resumed client nearly seamless to the user.

### 5.5 Graceful shutdown **[I/A]**

When the server shuts down (a deploy), you want open streams to end cleanly rather than be killed mid-frame. Because every handler selects on `c.Request.Context().Done()`, Go's `http.Server.Shutdown(ctx)` already does most of the work: it cancels in-flight request contexts, so every stream handler returns `false` and unwinds. Wire it up so a `SIGTERM` triggers `Shutdown`, give it a few seconds' grace, and your clients disconnect politely and reconnect to the new instance. We revisit shutdown security (draining, connection limits) in §11.

---

## 6. Authenticating the Stream

### 6.1 The core constraint, restated **[I]**

Recall from §3.3 the hard limit that shapes this entire section: **`EventSource` cannot send custom headers.** Your normal API sends `Authorization: Bearer <jwt>`; the SSE connection *cannot*. So the question "how do I know who is on the other end of this stream, and that they're allowed to be?" needs a different answer than your REST endpoints use. There are two industry-standard answers, and a professional system usually uses the first for browsers and keeps the second in its back pocket.

### 6.2 Approach A — the HttpOnly cookie (best for same-site dashboards) **[I/A]**

The browser attaches cookies to requests automatically, *including* the `EventSource` request and all its reconnects, as long as you pass `{ withCredentials: true }`. So if your login sets a secure, `HttpOnly` cookie containing (or referencing) the session/JWT, the SSE endpoint gets authenticated on every connect and reconnect with **zero client code** and **no token in the URL**. This is the cleanest and most secure option when the dashboard and the API are the same site (which, behind Nginx as one origin, they are — §9).

```go
// On successful Argon2id login, set the JWT in an HttpOnly, Secure, SameSite cookie.
func setSessionCookie(c *gin.Context, jwt string) {
	c.SetSameSite(http.SameSiteStrictMode) // stream is same-site; Strict is safe & strong
	c.SetCookie(
		"session",   // name
		jwt,         // value (a short-lived access JWT, or an opaque session id)
		15*60,       // maxAge seconds — keep access tokens short
		"/",         // path
		"",          // domain (current)
		true,        // Secure — HTTPS only; never send over plain HTTP
		true,        // HttpOnly — JavaScript CANNOT read it → immune to XSS token theft
	)
}
```

The properties that make this banking-grade: `HttpOnly` means a successful XSS **cannot read the token** (the number-one way tokens get stolen); `Secure` means it never traverses plain HTTP; `SameSite=Strict` means another site cannot make the browser send it (CSRF-resistant), and since your stream is same-origin, Strict costs you nothing. The auth middleware then reads the cookie exactly like a bearer token:

```go
func AuthFromCookie(secret []byte) gin.HandlerFunc {
	return func(c *gin.Context) {
		raw, err := c.Cookie("session")
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "no session"})
			return
		}
		claims, err := verifyJWT(raw, secret) // rigorous verification — see 6.4
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid session"})
			return
		}
		c.Set("userID", claims.UserID)
		c.Set("role", claims.Role)
		c.Next()
	}
}
```

### 6.3 Approach B — the short-lived ticket (query-param token) **[I/A]**

Sometimes a cookie isn't available or desirable — the stream is cross-origin, or you're on a native/mobile client, or you deliberately avoid cookies. The standard alternative is a **ticket**: the client, already authenticated via its normal bearer token, makes a quick authenticated `POST /api/sse-ticket`; the server returns a **single-use, short-lived (say 30-second) signed token**; the client then opens `new EventSource("/api/stream?ticket=" + t)`. The stream endpoint validates and immediately burns the ticket.

The reason the ticket must be *short-lived and single-use* is the elephant in the room with query-string auth: **URLs leak.** They land in server access logs, proxy logs, browser history, and `Referer` headers. A long-lived JWT in a URL is a credential sitting in a dozen log files. A 30-second single-use ticket that's already been consumed by the time anyone reads the log is nearly worthless to an attacker — that's what makes the pattern acceptable.

```go
// Issue a ticket: called via a NORMAL authenticated request (bearer token in header).
func IssueTicket(rdb *redis.Client, secret []byte) gin.HandlerFunc {
	return func(c *gin.Context) {
		userID := c.GetInt64("userID") // set by the normal bearer-auth middleware
		ticket := randomToken(32)      // crypto/rand, URL-safe
		// Store ticket→userID in Redis with a 30s TTL. Single-use: we DEL on redeem.
		key := "sse_ticket:" + ticket
		rdb.Set(c, key, userID, 30*time.Second)
		c.JSON(http.StatusOK, gin.H{"ticket": ticket})
	}
}

// Redeem it on the stream connect. Validate, then DELETE so it can't be reused.
func AuthFromTicket(rdb *redis.Client) gin.HandlerFunc {
	return func(c *gin.Context) {
		ticket := c.Query("ticket")
		if ticket == "" {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "no ticket"})
			return
		}
		key := "sse_ticket:" + ticket
		// GETDEL: atomically read and delete — enforces single use, closes the
		// race where two connects redeem the same ticket.
		uid, err := rdb.GetDel(c, key).Int64()
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired ticket"})
			return
		}
		c.Set("userID", uid)
		c.Next()
	}
}
```

Using Redis `GETDEL` makes redemption **atomic** — the read and the delete happen as one operation, so two simultaneous connects can't both redeem the same ticket. That atomicity is what makes "single-use" actually single-use under concurrency.

> **Which to choose?** For a browser dashboard on the same origin as your API — the case in this guide — **use the cookie (Approach A).** It keeps every credential out of URLs and logs, survives reconnects transparently, and is XSS-hardened by `HttpOnly`. Keep the ticket pattern for genuinely cross-origin or non-browser clients. Never put a long-lived JWT directly in the `EventSource` URL; that is the one clearly-wrong option.

### 6.4 Verifying the JWT rigorously **[A]**

However the token arrives — cookie or ticket-then-lookup — you must verify it *correctly*. The infamous JWT vulnerability is **algorithm confusion**: an attacker changes the token's `alg` header to `none`, or swaps `RS256` for `HS256` to trick a server into verifying an RSA-signed token with the RSA public key as an HMAC secret. golang-jwt v5 defends against this only if you **pin the expected algorithm** and reject everything else:

```go
func verifyJWT(tokenStr string, secret []byte) (*Claims, error) {
	claims := &Claims{}
	token, err := jwt.ParseWithClaims(tokenStr, claims,
		func(t *jwt.Token) (any, error) {
			// PIN the algorithm. If the token says anything other than HS256,
			// reject it — this is the line that stops alg-confusion attacks.
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}
			return secret, nil
		},
		// Belt and suspenders: restrict accepted algorithms at the parser level too.
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithExpirationRequired(), // reject tokens with no exp
	)
	if err != nil || !token.Valid {
		return nil, fmt.Errorf("invalid token: %w", err)
	}
	return claims, nil
}
```

This is the same verification discipline the [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md) covers in depth (rotation, refresh, revocation) — SSE reuses it unchanged. The only SSE-specific twist is *how the token gets to the endpoint* (cookie/ticket), not how it's validated once there. Password verification on login uses Argon2id exactly as that guide describes; the stream never sees a password, only the resulting token.

### 6.5 Wiring it into the routes **[I]**

```go
func RegisterRoutes(r *gin.Engine, h *Hub, secret []byte) {
	api := r.Group("/api")

	// Normal bearer-authenticated endpoints (login, ticket issue, actions).
	api.POST("/login", loginHandler)                 // Argon2id verify → set cookie/JWT

	// The stream: authenticated by the SAME-SITE COOKIE the login set.
	// withCredentials on the client makes the browser send it on connect+reconnect.
	authed := api.Group("", AuthFromCookie(secret))
	authed.GET("/stream", h.StreamHandler)           // <-- the SSE endpoint
}
```

The stream is now a first-class authenticated endpoint: no anonymous connections, identity established before a single frame is sent, and `userID`/`role` in the context ready for the per-event authorization we build in §11.

---

## 7. Persisting Events and the Data Layer

### 7.1 Why an in-memory hub is not enough **[I]**

The hub from §5 broadcasts to *currently connected* clients. But real clients disconnect constantly — a tab sleeps, a train enters a tunnel, your server redeploys. During those seconds the hub happily broadcasts events into the void for that client, and when the browser reconnects a moment later, those events are **gone**. For a chat toy that's fine; for a banking audit feed, silently dropping "a large transfer was flagged" because the reviewer's Wi-Fi blipped is unacceptable.

The fix is the reason `id:` and `Last-Event-ID` exist (§2, §3): **persist every event with a monotonic ID**, and when a client reconnects presenting its last-seen ID, **replay everything after it from the database** before resuming the live stream. Combined with the browser's automatic reconnect, this gives you *at-least-once, gap-free* delivery across disconnects — the client sees every event exactly in order, whether it was connected at the time or not.

### 7.2 The architecture — one pool, three roles **[I]**

This guide uses the same data-layer division as the [pgx](GO_PGX_GUIDE.md) and [goose](GO_GOOSE_MIGRATIONS_GUIDE.md) guides, because it is the production-proven shape:

- **goose owns the schema.** All DDL — the `events` table, the `users` table — is versioned migrations, reviewed in PRs, never auto-migrated at runtime.
- **Ent is the type-safe query layer.** Every read and write of an event goes through Ent's generated, compile-checked API. No hand-written SQL strings in application code.
- **pgx owns the pool** (`*pgxpool.Pool`) that Ent runs on, *and* owns the dedicated connection for `LISTEN/NOTIFY` in §8. One pool, opened once at boot.

This is "one pgxpool, one query layer" — goose shapes the database, Ent queries it, pgx connects to it. If any of that is unfamiliar, the three sibling guides cover it; here we just use it.

### 7.3 The events table (goose migration) **[I]**

The event store is an **append-only outbox**: every event worth delivering is inserted here first, then broadcast. The `id BIGSERIAL` gives us the monotonic, gap-detecting sequence `Last-Event-ID` needs.

```sql
-- +goose Up
-- Append-only event outbox. id is the monotonic sequence used for SSE resume.
CREATE TABLE events (
	id          BIGSERIAL PRIMARY KEY,
	-- Who may see this event. NULL = global (everyone); otherwise scoped to one user.
	user_id     BIGINT REFERENCES users(id),
	-- The SSE `event:` name (notification, audit, metric, ...).
	name        TEXT        NOT NULL,
	-- The payload, stored as jsonb so we can query/inspect it later.
	data        JSONB       NOT NULL,
	created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Replay queries filter by "id > last-seen" and often by recipient; index for it.
CREATE INDEX events_id_idx           ON events (id);
CREATE INDEX events_user_id_id_idx   ON events (user_id, id);

-- +goose Down
DROP TABLE events;
```

Because `id` is a `BIGSERIAL`, Postgres hands out strictly increasing values. That monotonicity is the backbone of resume: "everything the client missed" is exactly "`WHERE id > last_seen_id`."

### 7.4 The persist-then-broadcast flow **[I/A]**

The rule is **persist first, broadcast second.** If you broadcast before the insert commits and the insert then fails, connected clients saw an event that doesn't exist in history — so a *different* client that reconnects and replays from the DB won't see it, and now two clients disagree about what happened. Insert first; the DB is the source of truth; the live broadcast is an optimization on top of it.

```go
// Publish is the single choke point for emitting an event. Call it from anywhere
// (a transfer service, an admin action handler). It persists, then broadcasts.
func (s *EventService) Publish(ctx context.Context, userID int64, name string, data any) error {
	// 1) PERSIST FIRST via Ent. The returned row has the authoritative id.
	raw, err := json.Marshal(data)
	if err != nil {
		return fmt.Errorf("marshal event: %w", err)
	}
	row, err := s.ent.Event.Create().
		SetNillableUserID(nilIfZero(userID)). // NULL for global events
		SetName(name).
		SetData(raw).
		Save(ctx)
	if err != nil {
		return fmt.Errorf("persist event: %w", err) // if this fails, we DON'T broadcast
	}

	// 2) BROADCAST SECOND, using the id the DB assigned so clients store it as
	//    Last-Event-ID and can resume correctly.
	s.hub.Broadcast(Event{
		ID:     strconv.FormatInt(row.ID, 10),
		Name:   name,
		Data:   data,
		UserID: userID,
	})
	return nil
}
```

Now every event has a durable home and a stable ID *before* any client hears about it. Two consequences: a reconnecting client can always be caught up from the DB, and the DB doubles as an audit log (which, for a banking system, you needed anyway).

### 7.5 Replaying missed events on reconnect **[I/A]**

When a browser reconnects, the browser *automatically* sends `Last-Event-ID: <n>`. The handler reads it, queries the DB for everything newer that this user is allowed to see, streams those first (catching the client up), *then* joins the live hub. Doing catch-up **before** registering with the hub is important — it preserves order and avoids a gap between "end of replay" and "start of live."

```go
func (h *Hub) StreamHandler(c *gin.Context) {
	setSSEHeaders(c)
	userID := c.GetInt64("userID")

	// 1) REPLAY: if the browser sent Last-Event-ID, catch it up from the DB first.
	lastID := parseLastEventID(c.Request.Header.Get("Last-Event-ID")) // 0 if none/new
	if lastID > 0 {
		missed, err := h.events.Since(c.Request.Context(), userID, lastID)
		if err == nil {
			for _, ev := range missed {
				// Stream each missed event with its real id, in order.
				c.Render(-1, sse.Event{Id: ev.ID, Event: ev.Name, Data: ev.Data})
			}
			c.Writer.Flush()
		}
	}

	// 2) GO LIVE: only now register with the hub and forward live events.
	//    (the §5.3 client loop follows here, unchanged)
	// ...
}
```

The `Since` query is a textbook scoped read — "events newer than N that this user may see" — expressed in Ent so it's injection-proof and compile-checked:

```go
// Since returns events with id > lastID visible to userID (their own + globals),
// in ascending id order so the client replays them in the original sequence.
func (s *EventService) Since(ctx context.Context, userID, lastID int64) ([]Event, error) {
	rows, err := s.ent.Event.Query().
		Where(
			event.IDGT(lastID), // id > lastID  → only what was missed
			event.Or(           // visibility: this user's events OR global (null user)
				event.UserID(userID),
				event.UserIDIsNil(),
			),
		).
		Order(event.ByID(sql.OrderAsc())). // ascending → correct replay order
		Limit(1000).                        // cap a catch-up so a long-offline client can't OOM us
		All(ctx)
	if err != nil {
		return nil, err
	}
	return toEvents(rows), nil
}
```

Two banking-grade details are baked in: the `event.Or(UserID, UserIDIsNil)` predicate is **server-side authorization** — a client can *never* replay another user's events even by forging a `Last-Event-ID`, because the query itself filters by recipient (we harden this further in §11). And the `Limit(1000)` bounds catch-up so a client that was offline for a week can't force the server to load a million rows into memory. Resume is powerful; it must also be bounded.

---

## 8. Postgres LISTEN NOTIFY as an Event Source

### 8.1 The idea — let the database tell you **[A]**

So far, events enter the system because *application code* called `Publish`. But often the thing that changes is a **row in the database**, possibly changed by another service, a batch job, or a database trigger — code your Go process never runs. How does your SSE server find out? Polling the table ("any new events since I last looked?") reintroduces exactly the latency and waste SSE exists to avoid.

PostgreSQL has a built-in answer: **`LISTEN` / `NOTIFY`**, an in-database publish/subscribe system. A session runs `NOTIFY channel, 'payload'`; every session that has run `LISTEN channel` receives that payload asynchronously, instantly. If your Go server `LISTEN`s on a channel, then *anything* that can run `NOTIFY` — a trigger on the `events` table, a stored procedure, another microservice, a human in `psql` — can push an event into your live SSE stream without any coupling to your Go code. The database becomes the event bus.

### 8.2 Why pgx specifically **[A]**

`LISTEN/NOTIFY` needs a connection **dedicated** to waiting for notifications — it parks on the socket and blocks until something arrives. That's a poor fit for a pooled connection (which must be shared and returned quickly), which is exactly why §3 of the [pgx guide](GO_PGX_GUIDE.md) warns that a listener wants its *own* connection, not a pooled one. pgx exposes `conn.WaitForNotification(ctx)` for precisely this. We acquire one dedicated connection from the pool (or open a standalone `pgx.Conn`), `LISTEN`, and loop on `WaitForNotification` in a background goroutine for the process's whole life.

### 8.3 The trigger that fires NOTIFY **[A]**

We make the `events` table notify automatically on insert. Now `EventService.Publish` (§7) doesn't even need to call the hub directly — inserting the row *is* the signal. (You can keep the direct broadcast for the local instance and rely on NOTIFY for the others; §9 shows how these compose. For clarity here, NOTIFY drives everything.)

```sql
-- +goose Up
-- +goose StatementBegin
-- On each new event row, NOTIFY the 'events' channel with the new id as payload.
-- We send only the id, not the whole row: NOTIFY payloads are capped at 8000 bytes,
-- and the listener will read the full row by id anyway (avoids truncation bugs).
CREATE FUNCTION notify_event() RETURNS trigger AS $$
BEGIN
	PERFORM pg_notify('events', NEW.id::text);
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- +goose StatementEnd

CREATE TRIGGER events_notify
	AFTER INSERT ON events
	FOR EACH ROW EXECUTE FUNCTION notify_event();

-- +goose Down
DROP TRIGGER events_notify ON events;
DROP FUNCTION notify_event;
```

The **NOTIFY-an-id-then-read** pattern is the important best practice here. `NOTIFY` payloads have a hard 8000-byte limit, and stuffing a JSON blob into one is a truncation bug waiting to happen. So we notify only the *id* — always tiny — and the listener reads the full, untruncated row from the table by that id. The notification is a doorbell, not the delivery.

### 8.4 The listener goroutine **[A]**

```go
// RunListener holds ONE dedicated connection, LISTENs, and turns every NOTIFY
// into a hub broadcast. Start once at boot: go listener.Run(ctx).
func (l *Listener) Run(ctx context.Context) error {
	// A dedicated connection — NOT from the shared pool's normal rotation. We
	// acquire and keep it for this goroutine's entire life.
	conn, err := l.pool.Acquire(ctx)
	if err != nil {
		return fmt.Errorf("acquire listener conn: %w", err)
	}
	defer conn.Release()

	// Subscribe to the channel the trigger notifies.
	if _, err := conn.Exec(ctx, "LISTEN events"); err != nil {
		return fmt.Errorf("LISTEN: %w", err)
	}

	for {
		// Blocks until a NOTIFY arrives OR ctx is cancelled (shutdown).
		n, err := conn.Conn().WaitForNotification(ctx)
		if err != nil {
			if ctx.Err() != nil {
				return nil // clean shutdown
			}
			return fmt.Errorf("wait for notification: %w", err) // real error → caller reconnects
		}

		// n.Payload is the event id (a string). Read the full row and broadcast.
		id, _ := strconv.ParseInt(n.Payload, 10, 64)
		ev, err := l.events.Get(ctx, id) // Ent read of the full row
		if err != nil {
			log.Printf("listener: load event %d: %v", id, err)
			continue
		}
		l.hub.Broadcast(Event{
			ID:     strconv.FormatInt(ev.ID, 10),
			Name:   ev.Name,
			Data:   ev.Data,
			UserID: ev.UserID,
		})
	}
}
```

Now the flow is beautifully decoupled: *anything* that inserts an `events` row — your Go code, another service, a trigger elsewhere in the schema, an operator running SQL — causes the trigger to fire `NOTIFY`, the listener wakes, reads the row, and broadcasts it to every relevant connected dashboard. The database is the single source of truth *and* the event bus.

> **⚡ Reconnect the listener.** A dedicated long-lived connection *will* eventually drop (a network blip, a Postgres restart, `MaxConnLifetime`). Wrap `Run` in a supervising loop that re-acquires and re-`LISTEN`s on error with backoff, and — because you may have missed notifications during the gap — do a **catch-up read** (`events WHERE id > last_processed_id`) on reconnect, exactly like client resume but server-side. A listener that silently dies is a dashboard that silently stops updating.

---

## 9. Scaling Across Instances with a Redis Backplane

### 9.1 The multi-instance problem **[A]**

One Go process holds its clients in one in-memory hub. The moment you run **two** instances behind a load balancer — which you will, for availability and capacity — that in-memory hub becomes a liability. Admin Alice's browser connects to instance A. A transfer service running on instance B calls `Publish`. Instance B's hub broadcasts to *its* clients — but Alice isn't one of them; she's on A. She never sees the event. Each instance is an island that only knows its own connections.

With the `LISTEN/NOTIFY` design from §8 this is *partially* solved already: if every instance `LISTEN`s on the `events` channel, then an insert on B's connection notifies *all* instances, including A, and A broadcasts to Alice. Postgres `LISTEN/NOTIFY` is itself a cross-instance fan-out. For many systems that is enough and you can skip Redis.

**When do you add Redis anyway?** When you don't want every application event to require a database write and trigger (high-frequency ephemeral events like "typing…" or per-second metrics), when you want to keep pub/sub load off the primary database, or when you're already running Redis for tickets/rate-limits and want one fan-out mechanism. Redis pub/sub is a purpose-built, extremely fast fan-out bus. The pattern mirrors the WebSocket backplane from the realtime guides exactly.

### 9.2 The backplane pattern **[A]**

Each instance does two things: it **publishes** every event it originates to a Redis channel, and it **subscribes** to that channel and broadcasts anything it receives to its *local* hub. An event thus reaches every instance's clients regardless of which instance created it. The key subtlety is avoiding a double-delivery loop: tag each message with the originating instance id and skip re-broadcasting your own (or simply always deliver from Redis and never locally — pick one path and stick to it).

```go
// Publish to the backplane instead of (or in addition to) the local hub.
func (b *Backplane) Publish(ctx context.Context, ev Event) error {
	msg, _ := json.Marshal(backplaneMsg{Origin: b.instanceID, Event: ev})
	return b.rdb.Publish(ctx, "sse:events", msg).Err()
}

// Subscribe runs for the process's life: everything on the channel → local hub.
func (b *Backplane) Subscribe(ctx context.Context) {
	sub := b.rdb.Subscribe(ctx, "sse:events")
	defer sub.Close()
	ch := sub.Channel()
	for {
		select {
		case <-ctx.Done():
			return
		case m := <-ch:
			var bm backplaneMsg
			if err := json.Unmarshal([]byte(m.Payload), &bm); err != nil {
				continue
			}
			// Deliver to THIS instance's local clients. (Origin tag is available
			// if you choose the skip-your-own-origin dedup strategy.)
			b.hub.Broadcast(bm.Event)
		}
	}
}
```

With this in place, `EventService.Publish` persists to Postgres (for durability/replay) and publishes to Redis (for live cross-instance fan-out); each instance's `Subscribe` goroutine feeds its local hub; and every connected dashboard, on any instance, receives the event. Durability comes from Postgres; speed and fan-out come from Redis; the two are complementary, not redundant.

### 9.3 Nginx in front of SSE **[A]**

The reverse proxy needs specific settings or it will quietly ruin your stream (buffering) and cut it off (timeouts). Serve the app over **HTTP/2** to dissolve the 6-connections-per-domain limit from §3.3, and configure the SSE location to pass frames straight through:

```nginx
# Serve over HTTP/2 so many EventSource streams share one connection.
server {
	listen 443 ssl;
	http2 on;
	server_name dashboard.example.com;

	# ... ssl_certificate etc ...

	# The SSE endpoint needs streaming-specific proxy settings.
	location /api/stream {
		proxy_pass http://backend;
		proxy_http_version 1.1;          # keep-alive upstream

		# THE critical line: never buffer the streamed response. Without this
		# Nginx holds frames until its buffer fills and "real-time" dies.
		proxy_buffering off;
		proxy_cache off;

		# SSE connections are long-lived; don't let Nginx time them out at 60s.
		proxy_read_timeout 3600s;
		proxy_send_timeout 3600s;

		# Pass through the headers that matter.
		proxy_set_header Connection '';   # clear 'close' so keep-alive holds
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
	}

	# Normal API + static app served here (buffering fine for these).
	location / {
		proxy_pass http://backend;
	}
}
```

The two lines that matter most are `proxy_buffering off` (pairs with the `X-Accel-Buffering: no` response header from §4.2 — set both) and the long `proxy_read_timeout` (an SSE stream that's quiet for 60 seconds is normal, not dead; the default timeout would kill it). Serving over one origin also means your same-site auth cookie (§6.2) "just works" — the dashboard and API are the same site, so `SameSite=Strict` is both maximally secure and fully functional.

---

## 10. The Admin Dashboard UI

### 10.1 A complete, self-contained client **[I]**

The front end is deliberately the smallest part — that's SSE's promise. Here is a full admin dashboard as one self-contained HTML file: it logs in (which sets the auth cookie), opens the stream with `withCredentials`, routes named events to the right panels, and handles reconnection gracefully. It uses no framework so you can read every line; §10.3 notes the React equivalent.

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

		// escapeHTML: NEVER inject server text as innerHTML directly (XSS). We build
		// text nodes / escape everything. See §11 for why this is non-negotiable.
		function escapeHTML(s) {
			return String(s).replace(/[&<>"']/g, (c) => (
				{ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]
			));
		}
		function prepend(panelId, html) {
			const el = document.getElementById(panelId);
			el.insertAdjacentHTML("afterbegin", `<div class="item">${html}</div>`);
		}

		// Open the stream. withCredentials → the browser sends our HttpOnly auth
		// cookie on connect AND on every automatic reconnect. No token in the URL.
		const es = new EventSource("/api/stream", { withCredentials: true });

		es.onopen = () => { statusEl.textContent = "Live"; statusEl.className = "status open"; };
		es.onerror = () => {
			// Not "give up" — the browser is about to auto-reconnect. Just reflect it.
			statusEl.textContent = "Reconnecting…"; statusEl.className = "status closed";
		};

		// Named events route to their panels. Each .data is a JSON string we parse.
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

That is the entire client: ~60 lines, no dependencies, and it transparently handles login-cookie auth, reconnection, resume (the browser sends `Last-Event-ID` automatically — the server replays), and per-event routing. Compare that to the equivalent WebSocket client with its manual reconnect/backoff/heartbeat logic, and SSE's ergonomic win for one-way feeds is obvious.

### 10.2 Why the escaping matters, briefly **[I]**

Notice `escapeHTML` wrapping *every* piece of server-supplied text before it touches the DOM. An SSE payload is attacker-influenceable data — a notification's text might contain whatever a user typed into a transfer memo. If you `innerHTML` that raw, a memo of `<img src=x onerror=alert(document.cookie)>` executes script in your admin's session. Because the auth cookie is `HttpOnly` the script can't read *it*, but it can still act as the admin. Escape on the way in, always. This is the client half of the per-event security we formalize next.

### 10.3 Consuming SSE in React or Next **[I]**

In a React app the pattern is identical, wrapped in an effect: open the `EventSource` in `useEffect`, attach listeners that push into state, and **return a cleanup that calls `es.close()`** so a component unmount (or a route change, or React 18 StrictMode's double-invoke) doesn't leak a stream. The one React-specific trap is forgetting that cleanup — every re-render without it opens another `EventSource`, and you march straight into the 6-connection limit from §3.3. Open one stream at the app shell level, fan its events into your state manager (or TanStack Query cache), and never open a second per component.

---

## 11. Banking-Grade Security

Real-time endpoints have every vulnerability a normal endpoint has, plus a few unique to long-lived, header-less, fan-out connections. This section is the checklist that separates a demo from something you'd put in front of money. Nothing here is optional in a regulated system.

### 11.1 Authenticate before the first byte **[A]**

Every stream connection must be authenticated *before* any event is sent — which the §6 middleware guarantees by running before the handler. There are **no anonymous streams**. A connection that fails auth gets `401` and is closed, never a stream. And because the browser reconnects automatically, auth runs again on *every reconnect* — so a token that expired mid-stream fails the next reconnect and the connection dies, which is what you want. Keep access tokens short (minutes) so a stolen one has a small window.

### 11.2 Authorize every event, server-side **[A]**

This is the vulnerability unique to fan-out systems and the one most often missed: **a stream must deliver only events the connected user is authorized to see.** The hub broadcasts to many clients; if your filtering is wrong, User A's private notification lands in User B's dashboard — a data breach. Defense is layered:

- **Scope at the source.** The `Event.UserID` field and the hub's `if ev.UserID != 0 && c.userID != ev.UserID { continue }` check (§5.2) mean a user-scoped event only reaches that user's clients.
- **Scope at replay.** The `Since` query's `event.Or(UserID(userID), UserIDIsNil())` predicate (§7.5) means a forged `Last-Event-ID` can *never* surface another user's events — the SQL itself filters by recipient.
- **Scope by role for global events.** For events visible to a *class* of user (e.g. "all admins"), check the role from the verified JWT, not a client-supplied value. Add role to the filter:

```go
// Deliver a role-scoped event only to clients whose VERIFIED role qualifies.
func (h *Hub) authorized(c *client, ev Event) bool {
	if ev.UserID != 0 {           // user-scoped: must be that exact user
		return c.userID == ev.UserID
	}
	if ev.MinRole != "" {          // role-scoped: client's JWT role must qualify
		return roleAtLeast(c.role, ev.MinRole)
	}
	return true                    // truly global (rare; use sparingly)
}
```

The golden rule: **authorization is decided from server-verified identity (the JWT claims), never from anything the client sends on the stream.** The client cannot ask for events; it can only receive what the server decides it may see.

### 11.3 Never leak secrets into URLs or logs **[A]**

Because `EventSource` can't set headers, the temptation is to put the token in the query string — and that token then lands in Nginx access logs, application logs, browser history, and `Referer` headers. §6 avoids this with the cookie (nothing in the URL) or the **single-use 30-second ticket** (worthless by the time it hits a log). If you must log stream requests, **strip the query string** or redact the ticket parameter. Audit your log config for this specifically.

### 11.4 Rate-limit and cap connections **[A]**

Long-lived connections are a DoS vector: each one holds a goroutine, a buffer, and a socket. An attacker (or a buggy client in a reconnect storm) can open thousands and exhaust your file descriptors and memory. Defenses:

- **Max concurrent streams per user/IP.** Track open streams (in the hub or Redis) and reject new ones past a small limit (e.g. 5 per user). One user does not need fifty streams.
- **Rate-limit the connect endpoint** (and the ticket endpoint) so an attacker can't hammer reconnects — a Redis token-bucket keyed by user/IP.
- **A global connection ceiling** so one instance refuses new streams past its capacity rather than falling over — shed load deliberately.
- **Idle/absolute timeouts.** Even authenticated streams get a maximum lifetime, forcing periodic reconnect (and thus re-auth). A stream open for 24 hours is a stale credential held open.

```go
// Reject a new stream if this user already has too many open.
func (h *Hub) tryReserveSlot(userID int64) bool {
	h.mu.Lock()
	defer h.mu.Unlock()
	if h.perUser[userID] >= maxStreamsPerUser { // e.g. 5
		return false
	}
	h.perUser[userID]++
	return true
}
```

### 11.5 Transport, origin, and input hygiene **[A]**

- **TLS always (`wss`-grade for SSE means plain `https`).** A stream over plain HTTP sends the auth cookie and every event in cleartext. `Secure` cookies + HTTPS-only, no exceptions.
- **CORS deliberately.** If the dashboard is same-origin (recommended), you need no CORS at all and cross-site JS can't open your stream. If you must allow another origin, set `Access-Control-Allow-Origin` to an explicit allow-list (never `*` with credentials) and require `withCredentials`. An unrestricted CORS policy on a credentialed stream is a cross-site data leak.
- **Validate/encode all event data.** Payloads flow to the DOM; treat them as hostile. Escape on the client (§10.2) and, better, sanitize on the way *in* to the event store so bad data never persists.
- **Don't stream more than the user needs.** Send the notification's text, not the customer's full record. Minimize what crosses the wire so a client-side compromise leaks less.

### 11.6 Clean shutdown and leak prevention **[A]**

Every open stream owns resources; a leak is both a reliability bug and a slow DoS on yourself. The `defer h.unregister <- cl` in the handler (§5.3) and the context-driven loop guarantee cleanup on *every* exit path — client disconnect, error, or shutdown. On `SIGTERM`, `http.Server.Shutdown` cancels request contexts so streams drain and clients reconnect to a healthy instance. Run the race detector in tests (§12) to prove the hub has no data races, and watch goroutine count in production — a steadily climbing count is a leak (a handler not exiting, a channel not closed), and it will eventually take the process down.

---

## 12. Testing SSE

### 12.1 What's actually hard to test **[I/A]**

SSE handlers are streaming and long-lived, so the usual "call handler, assert on the finished response" doesn't fit — the response never finishes. The trick is to drive the handler with `httptest`, read the response body *incrementally* as a stream, assert on the frames that arrive, then cancel the request context to end it. You test three things: (1) the handler emits correctly-framed events, (2) the hub fans out and evicts slow clients, (3) auth rejects the unauthenticated.

### 12.2 Testing the hub in isolation **[I]**

The hub is pure Go with no HTTP — the easiest and most valuable thing to test, especially under `-race`:

```go
func TestHubBroadcastScopesByUser(t *testing.T) {
	hub := NewHub()
	go hub.Run()

	alice := &client{userID: 1, send: make(chan Event, 4)}
	bob := &client{userID: 2, send: make(chan Event, 4)}
	hub.register <- alice
	hub.register <- bob

	// A user-scoped event for Alice must NOT reach Bob.
	hub.Broadcast(Event{Name: "notification", UserID: 1, Data: "hi alice"})

	select {
	case ev := <-alice.send:
		if ev.Data != "hi alice" {
			t.Fatalf("alice got wrong event: %v", ev.Data)
		}
	case <-time.After(time.Second):
		t.Fatal("alice did not receive her event")
	}

	// Bob's channel must stay empty.
	select {
	case ev := <-bob.send:
		t.Fatalf("bob received an event he wasn't authorized for: %v", ev)
	case <-time.After(100 * time.Millisecond):
		// correct: nothing for bob
	}
}
```

Run it with `go test -race ./...` — because the hub is concurrent, the race detector is what proves the single-owner-goroutine design is actually race-free (see the [Testing in Go guide](GO_TESTING_GUIDE.md) for `-race` in depth).

### 12.3 Testing the endpoint end-to-end **[I/A]**

Spin the real handler with `httptest.NewServer`, connect with a plain HTTP client, and read frames off the body with a `bufio.Scanner`:

```go
func TestStreamDeliversEvent(t *testing.T) {
	hub := NewHub()
	go hub.Run()
	r := gin.New()
	r.GET("/stream", injectUser(1), hub.StreamHandler) // test middleware sets userID=1
	srv := httptest.NewServer(r)
	defer srv.Close()

	// Open the stream. Use a cancelable context so we can end it.
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	req, _ := http.NewRequestWithContext(ctx, "GET", srv.URL+"/stream", nil)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		t.Fatal(err)
	}
	defer resp.Body.Close()

	// Emit an event after the client is connected.
	go func() {
		time.Sleep(50 * time.Millisecond)
		hub.Broadcast(Event{Name: "notification", UserID: 1, Data: "hello"})
	}()

	// Read frames until we see our data line, or time out.
	scanner := bufio.NewScanner(resp.Body)
	deadline := time.After(2 * time.Second)
	for {
		select {
		case <-deadline:
			t.Fatal("did not receive event in time")
		default:
		}
		if !scanner.Scan() {
			t.Fatal("stream closed unexpectedly")
		}
		if strings.Contains(scanner.Text(), `"hello"`) {
			return // success: our event framed and delivered
		}
	}
}
```

The pattern generalizes: connect, trigger an event, scan the body for the expected frame, cancel to finish. For the DB-backed pieces (replay, `LISTEN/NOTIFY`), use **Testcontainers** to run a real Postgres exactly as the [Testing in Go guide](GO_TESTING_GUIDE.md) describes — insert an event, assert the listener broadcasts it; connect with a `Last-Event-ID`, assert the missed events replay in order.

---

## 13. Development Workflow with Air

### 13.1 Why hot-reload matters for streams **[B/I]**

Iterating on an SSE server by hand is painful: change code, stop the server, rebuild, restart, re-open the dashboard, log in again, wait for the stream to reconnect. **Air** removes all of that — it watches your source, rebuilds and restarts on save, and because the browser's `EventSource` *automatically reconnects*, your dashboard re-attaches to the new build within seconds without you touching it. SSE's auto-reconnect and a hot-reloader are a genuinely delightful pairing.

### 13.2 Setup **[B]**

```bash
# Install Air (Go 1.24+ tool style, pinned in go.mod — reproducible across the team).
go get -tool github.com/air-verse/air@latest
# Generate a starter config you can edit.
go tool air init      # writes .air.toml
```

```toml
# .air.toml — minimal config for the dashboard server.
root = "."
tmp_dir = "tmp"

[build]
  # Build the server command; adjust to your main package path.
  cmd = "go build -o ./tmp/server ./cmd/server"
  bin = "./tmp/server"
  # Watch Go files; ignore generated Ent code and test files to avoid churn.
  include_ext = ["go", "html", "toml"]
  exclude_dir = ["tmp", "ent/gen", "node_modules"]
  exclude_regex = ["_test\\.go"]
  # Give in-flight streams a moment to drain on restart.
  kill_delay = "1s"
```

Now `go tool air` runs the server, and every save rebuilds it. The `kill_delay` gives `http.Server.Shutdown` a beat to end open streams cleanly (§5.5) before the new binary takes over — so you're exercising your graceful-shutdown path on every reload, which is a nice free test of it.

> **Gotcha:** if you regenerate Ent code on save, make sure `ent/gen` (or wherever your generated code lives) is in `exclude_dir`, or Air will rebuild in a loop as codegen rewrites files. And never run Air in production — it's a dev tool that shells out to the compiler.

---

## 14. Gotchas and Best Practices

A concentrated list of the mistakes that actually bite, most learnable only the hard way. Skim it now; return to it when something "doesn't work."

| Pitfall | Symptom | Fix |
|---|---|---|
| **Forgot to flush** | Client receives nothing; bytes stuck in Go's buffer. | Call `flusher.Flush()` after every frame (or use `c.SSEvent`/`c.Stream`, which flush for you). |
| **Missing blank line** | Client connected but no `message` events fire. | Every frame ends with `\n\n`. The blank line *is* the message terminator. |
| **Proxy buffering** | Works locally, "freezes" behind Nginx. | `X-Accel-Buffering: no` response header **and** `proxy_buffering off` in Nginx. |
| **Compression middleware** | Stream stalls or batches. | Exclude `text/event-stream` from gzip/compression middleware — buffering compressors defeat streaming. |
| **Blocking hub send** | One slow/sleeping client freezes the whole dashboard. | Non-blocking send from the hub; evict slow consumers (§5.4). |
| **HTTP/1.1 6-connection limit** | 7th request to the origin hangs; app feels frozen with a few tabs open. | Serve over **HTTP/2**; open **one** `EventSource` per tab. |
| **Token in the URL** | Credentials in access logs / history. | Cookie auth, or single-use short-lived ticket; strip/redact query strings in logs (§11.3). |
| **No heartbeat** | Idle connections silently dropped by proxies/NAT. | Send a `: keep-alive` comment every 15–30s (§5.3). |
| **No `id:` / ignoring `Last-Event-ID`** | Reconnecting clients miss events. | Emit `id:` on every event; replay `WHERE id > last` on reconnect (§7.5). |
| **Broadcast before persist** | Reconnecting clients disagree with live clients. | Persist first, broadcast second (§7.4). |
| **NOTIFY payload too big** | Truncated/garbled events. | NOTIFY the id only; read the full row by id (§8.3). |
| **Listener never reconnects** | Dashboard silently stops updating after a DB blip. | Supervise the `LISTEN` loop with backoff + catch-up read (§8.4). |
| **No connection cap** | FD/memory exhaustion; self-inflicted DoS. | Per-user stream limit + global ceiling + rate-limit connects (§11.4). |
| **Leaked goroutines** | Goroutine count climbs; eventual OOM. | `defer` unregister; select on `ctx.Done()`; verify with `-race` and goroutine metrics. |
| **Unescaped payload in DOM** | Stored XSS in the admin dashboard. | Escape all server text before it touches the DOM (§10.2); sanitize on ingest. |
| **Reconnect storm** | Every client reconnects at once after a restart, hammering the server. | Set a sensible `retry:`; add jitter; make auth/replay cheap; scale connects. |

**Best-practice summary:** one `EventSource` per tab over HTTP/2; authenticate with a same-site `HttpOnly` cookie; give every event an `id` and persist it before broadcasting; make the hub single-owner and non-blocking; heartbeat idle streams; drive events from `LISTEN/NOTIFY` and fan out with Redis when you outgrow one instance; cap and rate-limit connections; and put `proxy_buffering off` in Nginx before you deploy, not after the incident.

---

## 15. Study Path and Build-to-Learn Projects

### 15.1 A staged path **[B→A]**

1. **Frames first (§2–§4).** Build the clock endpoint. Open it in a browser with `curl -N http://localhost:8080/api/clock` and *watch the raw frames* scroll by — seeing the `data:`/blank-line bytes makes the protocol click. Then consume it with a 10-line `EventSource` page.
2. **Fan-out (§5).** Add the hub. Open the same stream in three browser tabs; trigger a broadcast from a second endpoint (`POST /api/emit`) and watch all three light up. Then simulate a slow client (a `time.Sleep` in one consumer) and confirm the *others* keep flowing — you've proven your backpressure policy.
3. **Auth (§6).** Add Argon2id login that sets an `HttpOnly` cookie, protect the stream, and confirm an unauthenticated `EventSource` gets `401`. Then implement the ticket flow as an exercise and compare.
4. **Durability (§7).** Add the goose `events` table and Ent layer. Kill the server mid-stream, restart, and watch the browser reconnect and *replay the events it missed* via `Last-Event-ID`. This is the moment SSE becomes production-real.
5. **Database-driven (§8).** Add the trigger and the `LISTEN` goroutine. Insert a row with `psql` by hand and watch it appear on the dashboard — no Go code ran to send it. That decoupling is the payoff.
6. **Scale (§9).** Run two instances behind Nginx with a Redis backplane; connect to one, emit on the other, confirm delivery. Then harden with §11.

### 15.2 Build-to-learn projects **[A]**

- **Live audit feed.** Every privileged action (login, transfer approval, role change) inserts an `events` row via the trigger; the admin dashboard shows the org's activity in real time with per-role filtering. Exercises §7, §8, §11.2.
- **Deployment / job log tail.** Stream a running background job's log lines to an admin panel with resume, so a reviewer who reconnects sees the whole log, not just what happened while connected. Exercises multi-line `data`, resume, backpressure.
- **Fraud-alert dashboard.** A scoring service `NOTIFY`s on flagged transactions; each reviewer sees only alerts for their assigned region (role/scope authorization); alerts persist for replay and audit. Exercises the full stack end-to-end.
- **Live metrics wall.** A per-second `metric` event (active sessions, events/sec, error rate) broadcast globally via Redis, rendered as moving numbers/sparklines. Exercises high-frequency ephemeral events (Redis, not DB) and the global-broadcast path.

### 15.3 Where to go next **[A]**

When you hit a feature that needs the *client* to stream to the server continuously — collaborative editing, a chat, live cursors — you've outgrown SSE's one-way model; move to WebSockets with the [Coder WebSocket guide](GO_CODER_WEBSOCKETS_GUIDE.md), which reuses this guide's hub, auth, backplane, and persistence patterns almost verbatim over a bidirectional transport. For the data layer, go deeper with [pgx](GO_PGX_GUIDE.md) (pooling, `LISTEN/NOTIFY`, high-traffic tuning), [Ent](GO_ENT_ORM_GUIDE.md), and [goose](GO_GOOSE_MIGRATIONS_GUIDE.md); for the auth half, [JWT + Argon2](GO_JWT_ARGON2_GUIDE.md); and to prove it all works, [Testing in Go](GO_TESTING_GUIDE.md). You now have everything to ship a real-time, banking-grade, reconnect-proof server-push system on nothing more exotic than an HTTP response you choose not to end.
