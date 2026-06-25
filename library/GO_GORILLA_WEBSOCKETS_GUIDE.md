# Go Gorilla WebSockets — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have used HTTP but never opened a WebSocket" to "I can design, secure, and horizontally scale a production real-time Go server." This is a **learn-offline study guide**: every concept is explained in *prose first* — what it is, the logic and *why* it exists, what it is for and when to reach for it, how to use it, the key options and parameters, best practices, and explicit **security recommendations** — and only *then* shown as heavily-commented, runnable Go code. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Go 1.25 / 1.26** (current in 2026) and **`github.com/gorilla/websocket` v1.5.x**. The Gorilla toolkit went dormant around 2022, was archived, and is now **actively maintained again** under community stewardship (the `gorilla/*` repos were un-archived in 2023 and continue to receive releases through 2026). The v1 API is stable; there is **no v2 migration** to worry about. Modern Go features used here: the **Go 1.22 loop-variable-per-iteration** fix (so `for client := range hub.clients` is safe to close over), `log/slog` structured logging, `errors.As`/`errors.Is`, and `http.Server.Shutdown` with `context`. Where behaviour is version-sensitive it is flagged with **⚡ Version note**.
>
> **This guide's place in the library:** it assumes you already know the Go *language* (`GO_GUIDE.md`) and HTTP basics. WebSockets begin life as an HTTP request, so the **`GO_NET_HTTP_REST_API_GUIDE.md`** (servers, handlers, middleware, graceful shutdown) and **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** (running WebSockets inside the Gin router) are direct companions and are cross-referenced throughout. For the multi-instance scaling backplane see **`REDIS_GUIDE.md`**; for connection authentication see **`GO_JWT_ARGON2_GUIDE.md`**.

---

## Table of Contents

1. [What WebSockets Are & Why They Exist](#1-what-websockets-are--why-they-exist) **[B]**
2. [The WebSocket Handshake — HTTP Upgrade](#2-the-websocket-handshake--http-upgrade) **[B]**
3. [Project Setup & a Hello-World Echo Server](#3-project-setup--a-hello-world-echo-server) **[B]**
4. [The Upgrader — Options, CheckOrigin & CSWSH](#4-the-upgrader--options-checkorigin--cswsh) **[B/I]**
5. [Reading & Writing Messages](#5-reading--writing-messages) **[B/I]**
6. [The Critical Concurrency Model — One Reader, One Writer](#6-the-critical-concurrency-model--one-reader-one-writer) **[I/A]**
7. [Ping/Pong, Deadlines & Connection Health](#7-pingpong-deadlines--connection-health) **[I]**
8. [Handling Disconnects & Close Codes](#8-handling-disconnects--close-codes) **[I]**
9. [The Hub / Broadcast Pattern — Full Chat Server](#9-the-hub--broadcast-pattern--full-chat-server) **[I/A]**
10. [Backpressure & Slow Clients](#10-backpressure--slow-clients) **[A]**
11. [Clients — Browser JS & the Go Dialer](#11-clients--browser-js--the-go-dialer) **[I]**
12. [Running WebSockets Inside Gin](#12-running-websockets-inside-gin) **[I]**
13. [Graceful Shutdown](#13-graceful-shutdown) **[A]**
14. [Security — CheckOrigin, Auth, Limits, Rate Limiting, TLS](#14-security--checkorigin-auth-limits-rate-limiting-tls) **[A]**
15. [Scaling Beyond a Single Process — Redis Backplane](#15-scaling-beyond-a-single-process--redis-backplane) **[A]**
16. [Tips, Tricks & Gotchas](#16-tips-tricks--gotchas) **[I/A]**
17. [API Quick Reference](#17-api-quick-reference)
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What WebSockets Are & Why They Exist

### 1.1 The problem WebSockets solve **[B]**

Plain HTTP is a **request-response** protocol with a strict rule: *the client always speaks first.* The browser asks ("GET /messages"), the server answers, and then the exchange is over. The server has no way to say "hey, something just happened" on its own. For a CRUD app — load a page, submit a form — this is perfect. But for anything *live* — a chat, a multiplayer game, a trading dashboard, a collaborative editor, a notification bell — the server constantly needs to push data to a client that did not ask for it at that instant. HTTP simply cannot do that natively.

Before WebSockets, developers faked server-push with a series of increasingly painful workarounds. Understanding them is the *why* behind WebSockets:

- **Short polling** — the client asks "anything new?" every few seconds on a timer. Simple, but it is wasteful: most requests return "nothing new," and each one pays the full cost of a new HTTP request (headers, possibly a TCP/TLS handshake, server routing). Latency is bounded by the poll interval — set it to 1 s and you hammer the server; set it to 30 s and the app feels dead.
- **Long polling** — the client makes a request and the server *holds it open* until it has something to say, then responds; the client immediately reconnects. Lower latency and less waste than short polling, but you still pay a full HTTP round-trip per message and tie up server resources holding requests open. It is a clever hack, not a real bidirectional channel.
- **Server-Sent Events (SSE)** — a standardized *one-way* stream from server to client over a single long-lived HTTP response (`text/event-stream`). Genuinely good when the data only flows **server → client** (live scores, a notification feed, log tailing). It is simpler than WebSockets, auto-reconnects, and rides normal HTTP/2. But it is unidirectional — the client still has to use a separate HTTP request to send anything back.

**WebSocket** is the real answer: a **persistent, full-duplex** connection. After a one-time handshake, a single TCP connection stays open and *both sides can send messages at any time*, independently, with tiny per-message overhead (about 2–14 bytes of framing versus the hundreds of bytes an HTTP request spends on headers every single time).

### 1.2 HTTP vs WebSocket at a glance **[B]**

| Property | HTTP (poll/long-poll) | Server-Sent Events | WebSocket |
|---|---|---|---|
| Connection lifetime | New per request (or held) | One long-lived response | Persistent until closed |
| Direction | Client → Server (server replies) | Server → Client only | **Full-duplex (both ways)** |
| Who can initiate a message | Client only | Server only | **Either side, anytime** |
| Per-message overhead | High (full HTTP headers) | Low after first | **Very low (small frame header)** |
| Latency | Round-trip / poll interval | Low | **Minimal (persistent TCP)** |
| Browser API | `fetch` / `XMLHttpRequest` | `EventSource` | `WebSocket` |
| Binary support | Awkward | No (text only) | **Yes (text *and* binary)** |
| Auto-reconnect built in | No | Yes | No (you implement it) |

### 1.3 When to use which — the decision **[B]**

This is the practical question, so be deliberate about it. Choosing WebSockets when you don't need them adds operational complexity (sticky load balancing, keepalive, a scaling backplane) for no benefit.

**Reach for WebSockets when:**
- Messages flow in **both** directions and timing matters — chat, multiplayer games, collaborative editing, live cursors.
- The server must push *and* the client must push frequently, on the same logical channel.
- You need sub-second, sustained, low-overhead two-way traffic.

**Use Server-Sent Events when:**
- Data only flows **server → client** — live dashboards, notifications, stock tickers, progress bars, log streaming. SSE is simpler, auto-reconnects, and works over plain HTTP/2 without special proxy config.

**Stick with plain HTTP / polling when:**
- Updates are infrequent (polling every 30–60 s is fine), or the client is a script/cron that never needs an unsolicited push. Don't pay the WebSocket tax for an occasional refresh.

> **Best practice:** default to the *simplest* transport that meets the latency and direction requirements. Many "we need WebSockets" features are actually one-directional and are better served by SSE.

### 1.4 What WebSocket is *not* **[B]**
- It is **not** a message queue. There is no durability, no delivery guarantee beyond TCP, no replay. If a client is offline when you broadcast, the message is gone unless *you* persist it (see the Notifications project in §18).
- It is **not** request-response. There is no built-in correlation between a message you send and a reply. If you want RPC semantics over a socket, you add your own message IDs — or use **gRPC** instead (see `GO_GRPC_RPC_GUIDE.md`).
- It is **not** automatically reconnecting. When the TCP connection drops, it is gone; the client must detect the close and dial again.

---

## 2. The WebSocket Handshake — HTTP Upgrade

### 2.1 The logic: reuse HTTP, then leave it behind **[B]**

A WebSocket connection does not get its own port or its own initial protocol. It **begins as an ordinary HTTP/1.1 GET request** and then asks the server to "upgrade" that same TCP connection to the WebSocket protocol. This design is deliberate and clever: it means WebSockets travel over ports 80/443 like normal web traffic, pass through most firewalls, can share a domain and TLS certificate with your website, and can be authenticated using the same cookies and headers your HTTP app already uses. The handshake is the *only* part that is HTTP; once it succeeds, the bytes on the wire are WebSocket frames, not HTTP.

### 2.2 The wire exchange **[B]**

The client sends a normal GET with a few special headers:

```
Client → Server:
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

The server, if it agrees, replies with status **101 Switching Protocols** — the only successful response code in all of HTTP that means "we are abandoning HTTP on this connection":

```
Server → Client (101 Switching Protocols):
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

Header by header, and *why each matters*:
- **`Upgrade: websocket`** + **`Connection: Upgrade`** — the request to switch protocols. Both must be present. (This pair is exactly what reverse proxies must forward, see §16.)
- **`Sec-WebSocket-Key`** — a random base64 nonce from the client. The server hashes it with a fixed "magic" GUID (`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`) and returns the SHA-1, base64-encoded, in **`Sec-WebSocket-Accept`**. This is *not* security — it is a sanity check that proves the server actually understood the WebSocket handshake and isn't a cache or a confused HTTP server blindly echoing data.
- **`Sec-WebSocket-Version: 13`** — the only version in real use (RFC 6455).
- **`Origin`** — set automatically by browsers. This is your one chance to reject cross-site connection attempts (CSWSH) *before* the upgrade. It is the heart of `CheckOrigin` (§4, §14).
- **`Sec-WebSocket-Protocol`** (optional) — a list of application subprotocols the client speaks; the server picks one.

After the `101`, the TCP socket is "promoted": both sides speak the WebSocket framing protocol (opcodes, masking, fragmentation) from that point on. **Gorilla performs this entire dance for you** when you call `upgrader.Upgrade()` — you never compute the accept hash or parse frames by hand. But knowing what is happening explains every option on the `Upgrader` and every security concern in this guide.

### 2.3 The consequence you must internalize **[B]**

Because the handshake *is* an HTTP request, you get exactly one HTTP request to inspect — cookies, headers, query string, TLS state — **before** you call `Upgrade`. The instant the upgrade succeeds, the HTTP response is finished; you can no longer write `401 Unauthorized` or set a status code. **Therefore: do all authentication, authorization, and origin checking *before* `Upgrade()`.** This single fact drives the security architecture in §14.

---

## 3. Project Setup & a Hello-World Echo Server

### 3.1 Prerequisites & install **[B]**

You need Go 1.21+ (1.25/1.26 recommended) and a Go module. Inside your module directory:

```bash
go get github.com/gorilla/websocket@latest
```

This records `github.com/gorilla/websocket v1.5.x` in `go.mod` and a checksum in `go.sum`. Gorilla has **zero third-party dependencies** — it only uses the standard library — which is one reason it remains the default WebSocket library for Go.

> **⚡ Version note:** `gorilla/websocket` follows semantic versioning and v1.x is the only stable major version as of 2026. Despite the project's dormant period (2022–2023), v1.5.x is current and maintained. There is no v2.

### 3.2 Minimal project layout **[B]**

```
myapp/
├── go.mod
├── go.sum
├── main.go          ← HTTP server entry point + upgrade handler
├── hub.go           ← Hub broadcast logic (added in §9)
├── client.go        ← Client read/write pumps (added in §9)
└── static/
    └── index.html   ← Browser client (added in §11)
```

A representative `go.mod`:

```go
module github.com/yourname/myapp

go 1.24

require github.com/gorilla/websocket v1.5.3
```

### 3.3 The simplest possible server: an echo **[B]**

Before any concurrency model, understand the *lifecycle* of one connection: upgrade → loop reading → echo back → the loop ends on error → close. This server is deliberately naive (it reads and writes from the same goroutine, which is only safe because it never writes except as a direct reply to a read), but it is the right first thing to run.

```go
// main.go — bare-minimum WebSocket echo server.
// Run with `go run main.go`, then test with `wscat -c ws://localhost:8080/ws`.
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

// The Upgrader converts an HTTP connection into a WebSocket connection.
// Declare it ONCE at package level and reuse it for every request — it is
// safe for concurrent use and holds your configuration (buffer sizes, the
// all-important CheckOrigin, etc.).
var upgrader = websocket.Upgrader{
	// CheckOrigin decides whether to accept the cross-origin handshake.
	// Returning true for EVERYTHING is acceptable ONLY on localhost during
	// development. In production this is a security hole (see §4 and §14).
	CheckOrigin: func(r *http.Request) bool { return true },
}

// echoHandler upgrades the request, then echoes every message back.
func echoHandler(w http.ResponseWriter, r *http.Request) {
	// Upgrade swaps the HTTP connection for a *websocket.Conn.
	// On failure it has ALREADY written an HTTP error to w — we just log + return.
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println("upgrade error:", err)
		return
	}
	// Always close to release the socket and its OS file descriptor.
	defer conn.Close()

	for {
		// ReadMessage blocks until a full message arrives or an error occurs
		// (peer closed, network died, deadline exceeded).
		msgType, msg, err := conn.ReadMessage()
		if err != nil {
			log.Println("read error:", err)
			break // leave the loop -> deferred Close() fires
		}
		log.Printf("received: %s", msg)

		// Echo the exact same message (and type) straight back.
		if err := conn.WriteMessage(msgType, msg); err != nil {
			log.Println("write error:", err)
			break
		}
	}
}

func main() {
	http.HandleFunc("/ws", echoHandler)
	// Serve the browser client from ./static (added in §11).
	http.Handle("/", http.FileServer(http.Dir("./static")))
	log.Println("listening on http://localhost:8080  (ws at /ws)")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**How to test it without writing a client:** install `wscat` (`npm install -g wscat`) and run `wscat -c ws://localhost:8080/ws`. Type a line; it comes back. Or open the browser console on any page and run `new WebSocket('ws://localhost:8080/ws')`.

> **Why this naive version is unsafe for real apps:** it only ever writes *in response to* a read, on the same goroutine, so there is never a concurrent write. The moment you need to push *unsolicited* messages (a broadcast, a server-initiated notification, a periodic ping) you have two goroutines touching the connection — and you must adopt the concurrency model in §6. Do not ship the echo pattern.

---

## 4. The Upgrader — Options, CheckOrigin & CSWSH

### 4.1 What the Upgrader is and why it is configured once **[B]**

`websocket.Upgrader` is the bridge between Go's `net/http` and the WebSocket protocol. It holds your handshake policy — buffer sizes, timeout, which origins to trust, which subprotocols you speak — and exposes a single method, `Upgrade(w, r, responseHeader)`, that runs the handshake from §2 and hands back a `*websocket.Conn`. You declare it as a package-level variable because it is **stateless and concurrency-safe**: one Upgrader serves every incoming connection. Creating a fresh one per request just wastes allocations.

### 4.2 Every option, explained **[I]**

```go
import "time"

var upgrader = websocket.Upgrader{
	// ReadBufferSize / WriteBufferSize: the size (bytes) of the per-connection
	// I/O buffers Gorilla allocates. They affect EFFICIENCY, not the maximum
	// message size — a message bigger than the buffer is still handled, it just
	// takes more syscalls. Default 4096 each. With tens of thousands of mostly
	// idle connections, shrink these (e.g. 1024 or 512) to cut memory; for
	// large binary payloads, grow them (32k–64k) to reduce syscalls.
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,

	// WriteBufferPool: an optional sync.Pool-backed buffer pool SHARED across
	// connections. With many connections, this dramatically reduces memory
	// because write buffers are borrowed only while a write is in flight,
	// instead of being permanently allocated per connection. Strongly
	// recommended at scale. (When set, WriteBufferSize sizes pooled buffers.)
	WriteBufferPool: &websocket.BufferPool{}, // see note below

	// HandshakeTimeout: max duration for the upgrade handshake itself.
	// Zero means NO timeout — not recommended; a slow-loris client could hold
	// the handshake open. 10s is a sane default.
	HandshakeTimeout: 10 * time.Second,

	// CheckOrigin: called on EVERY upgrade. Return true to allow, false to
	// reject with HTTP 403. THE most important security knob — see §4.4/§14.
	CheckOrigin: func(r *http.Request) bool {
		return r.Header.Get("Origin") == "https://myapp.example.com"
	},

	// Subprotocols: application-level subprotocols the server supports. The
	// client lists its choices in Sec-WebSocket-Protocol; Gorilla picks the
	// first match and echoes it in the 101 response. Use this to version your
	// protocol ("chat.v2") or negotiate a format ("json", "msgpack").
	Subprotocols: []string{"chat.v2", "chat.v1"},

	// Error: customize the HTTP error written when the upgrade is rejected.
	// Defaults to http.Error. Use it to log or return a structured body.
	Error: func(w http.ResponseWriter, r *http.Request, status int, reason error) {
		http.Error(w, reason.Error(), status)
	},

	// EnableCompression: negotiate per-message DEFLATE (RFC 7692). Saves
	// bandwidth on repetitive text/JSON, but costs CPU and memory per message.
	// Measure before enabling; it is OFF by default for good reason.
	EnableCompression: false,
}
```

> **Note on `WriteBufferPool`:** the real type is the `websocket.BufferPool` *interface*; the idiomatic value is a `&sync.Pool{}` wrapped to satisfy it, or simply Gorilla's behavior of using the pool when one is provided. For a single small app you can omit it; at scale, share a pool. Cross-reference: this is the same memory-amortization idea as connection pooling in `GO_NET_HTTP_REST_API_GUIDE.md`.

### 4.3 Upgrading inside a handler — validate first **[B/I]**

The handler is where you exploit the "one HTTP request before upgrade" rule from §2.3: inspect everything, reject early, and only then upgrade.

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
	// 1) AUTH and any other policy MUST happen here, while we can still send a
	//    normal HTTP status. After Upgrade succeeds you cannot send 401/403.
	if !isAuthenticated(r) {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return // never reaches Upgrade
	}

	// 2) responseHeader lets you add headers to the 101 response (e.g. a
	//    Set-Cookie, or echo a chosen subprotocol). Pass nil if you have none.
	responseHeader := http.Header{}
	responseHeader.Set("X-Server", "myapp")

	// 3) Run the handshake.
	conn, err := upgrader.Upgrade(w, r, responseHeader)
	if err != nil {
		// Upgrade already wrote the HTTP error (via Upgrader.Error / http.Error).
		// Do NOT write to w again here. Just log and return.
		log.Printf("upgrade failed for %s: %v", r.RemoteAddr, err)
		return
	}
	defer conn.Close()

	// conn is a *websocket.Conn. Hand it to your connection logic.
	handleConnection(conn)
}
```

### 4.4 CheckOrigin and Cross-Site WebSocket Hijacking (CSWSH) **[A]**

This is the single most important security concept for WebSocket servers, so it gets its own subsection here and a full treatment in §14.

**The logic:** WebSocket connections are **not protected by the Same-Origin Policy** the way `fetch` is, and they **do not** trigger CORS preflight. When a browser opens a WebSocket, it **automatically attaches the user's cookies** for the target domain — exactly as it would for any request to that domain. So if a victim is logged in to `bank.example.com` and then visits `evil.example`, JavaScript on the evil page can do `new WebSocket('wss://bank.example.com/ws')` and the browser will happily send the victim's auth cookie along with the handshake. If your server accepts any origin, the attacker now has an authenticated socket acting as the victim. That attack is **Cross-Site WebSocket Hijacking (CSWSH)**.

**The defense:** the browser also sends the `Origin` header (which page opened the socket), and the attacker's page *cannot forge it*. `CheckOrigin` is where you reject any origin you do not own. Gorilla's **default** `CheckOrigin` (when you leave it `nil`) already rejects requests whose `Origin` host differs from the `Host` header — a safe default. The dangerous thing is *overriding* it with `return true`, which people copy from tutorials and forget to remove.

```go
// DANGEROUS — disables CSWSH protection. Acceptable ONLY on localhost dev.
CheckOrigin: func(r *http.Request) bool { return true }
```

```go
// SAFE — explicit allowlist. The attacker cannot forge Origin from a browser.
var allowedOrigins = map[string]bool{
	"https://myapp.example.com": true,
	"https://www.example.com":   true,
	// "http://localhost:3000":  true, // dev only — REMOVE for production
}

CheckOrigin: func(r *http.Request) bool {
	origin := r.Header.Get("Origin")
	if origin == "" {
		// No Origin header = not a browser (e.g. a Go client, curl).
		// Decide your policy: browsers always send it. If your API is
		// browser-only, reject empty Origin; if you also serve native
		// clients that authenticate by token, you may allow it.
		return false
	}
	return allowedOrigins[origin]
}
```

> **Security checklist (CheckOrigin):** never ship `return true`; prefer an explicit allowlist over substring matching (a naive `strings.Contains(origin, "example.com")` matches `evil-example.com.attacker.io`); remember `Origin` is reliable *only from browsers* — native clients can set anything, so do not rely on origin alone for authentication (§14).

---

## 5. Reading & Writing Messages

### 5.1 Message types (opcodes) **[B]**

Every WebSocket frame carries a numeric opcode. Gorilla exposes them as constants. You will use `TextMessage` and `BinaryMessage` constantly; the control frames (`Close`, `Ping`, `Pong`) are usually handled for you but you can send them explicitly.

| Constant | Value | Meaning | When you use it |
|---|---|---|---|
| `websocket.TextMessage` | 1 | UTF-8 text (JSON, plain text) | Almost all app messages |
| `websocket.BinaryMessage` | 2 | Raw bytes (protobuf, images, msgpack) | Binary payloads |
| `websocket.CloseMessage` | 8 | Begin/echo the close handshake | Graceful shutdown |
| `websocket.PingMessage` | 9 | Keepalive ping (control frame) | Liveness checks (§7) |
| `websocket.PongMessage` | 10 | Reply to a ping (control frame) | Auto-sent by Gorilla |

> **Why Text vs Binary matters:** Text frames are validated as UTF-8 by spec and are what middleboxes/proxies expect; some firewalls are stricter about binary. Send JSON as `TextMessage` (semantically correct), and reserve `BinaryMessage` for actual binary encodings.

### 5.2 The high-level read/write helpers **[B]**

`ReadMessage` and `WriteMessage` are the simplest API: one call moves one *complete* message.

```go
// READ — blocks until a whole message arrives, the peer closes, or a deadline
// is hit. messageType is TextMessage or BinaryMessage; p is the full payload.
messageType, p, err := conn.ReadMessage()
if err != nil {
	// Distinguish expected vs unexpected close — see §8.
	return
}

// WRITE — sends exactly one complete message of the given type.
err = conn.WriteMessage(websocket.TextMessage, []byte(`{"hello":"world"}`))
```

### 5.3 ReadJSON / WriteJSON — typed envelopes **[B/I]**

Real apps rarely shuttle raw strings; they send **typed JSON envelopes**. Gorilla's `ReadJSON`/`WriteJSON` wrap `encoding/json` over a single message.

```go
// A typed message envelope is the standard pattern: a "type" discriminator
// plus payload fields. The client and server agree on this shape.
type Event struct {
	Type    string `json:"type"`              // "chat", "join", "ping", ...
	User    string `json:"user,omitempty"`
	Content string `json:"content,omitempty"`
	Time    int64  `json:"time,omitempty"`    // Unix millis, set by server
}

// WriteJSON marshals v to JSON and sends it as ONE TextMessage.
if err := conn.WriteJSON(Event{Type: "chat", User: "Alice", Content: "hi"}); err != nil {
	return
}

// ReadJSON reads the next message and json.Unmarshal's it into v.
var in Event
if err := conn.ReadJSON(&in); err != nil {
	// NOTE: a non-JSON payload yields a *json* decode error, NOT a websocket
	// error. And ReadJSON reads the ENTIRE message into memory first — so you
	// MUST set SetReadLimit (§7/§14) or a huge payload can OOM you.
	log.Println("bad message:", err)
	return
}
```

> **⚡ Gotcha:** `ReadJSON` buffers the *whole* message before decoding. Without `conn.SetReadLimit(maxBytes)` a malicious 100 MB frame allocates 100 MB on your heap. Always set a read limit before the first read.

### 5.4 Streaming large messages with NextReader / NextWriter **[I/A]**

For large payloads you do not want to materialize the whole message in RAM. `NextReader` gives you an `io.Reader` over the incoming message; `NextWriter` gives you an `io.WriteCloser` you stream into. `NextWriter` is also the key to **batching** several queued small messages into one frame (used in the writePump in §6).

```go
// READ as a stream — proxy or incrementally decode without buffering it all.
msgType, r, err := conn.NextReader()
if err != nil {
	return
}
_, _ = io.Copy(dst, r) // r is valid only until the NEXT read call

// WRITE as a stream — you MUST call Close() to flush the final frame.
w, err := conn.NextWriter(websocket.BinaryMessage)
if err != nil {
	return
}
_, _ = io.Copy(w, src)
if err := w.Close(); err != nil { // Close flushes and finalizes the frame
	return
}
```

> **Rule:** while a `NextWriter` is open you must not call `WriteMessage`/`WriteJSON` on the same conn — the in-progress frame would be corrupted. Close the writer first.

### 5.5 A complete (single-goroutine) read loop **[B/I]**

```go
// handleConnection runs the basic message loop for ONE connection.
// (Single-goroutine; safe only because it writes solely as a reply. For
// broadcasts/pushes use the two-pump model in §6.)
func handleConnection(conn *websocket.Conn) {
	defer conn.Close()

	conn.SetReadLimit(512 * 1024) // 512 KB cap — protect memory (§14)

	for {
		msgType, data, err := conn.ReadMessage()
		if err != nil {
			// IsUnexpectedCloseError: log only the *unexpected* closes; treat
			// normal/going-away as routine.
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseNormalClosure,
				websocket.CloseGoingAway,
			) {
				log.Printf("unexpected close: %v", err)
			}
			return
		}

		switch msgType {
		case websocket.TextMessage:
			log.Printf("text: %s", data)
			_ = conn.WriteMessage(websocket.TextMessage, data) // reply (safe here)
		case websocket.BinaryMessage:
			log.Printf("binary: %d bytes", len(data))
		}
	}
}
```

---

## 6. The Critical Concurrency Model — One Reader, One Writer

This is **the most important section in the guide.** Get it wrong and your server corrupts its streams or panics under load — and it will *look* fine in single-client testing, then explode in production.

### 6.1 The rule **[I]**

> A `*websocket.Conn` is **not** safe for concurrent reads, and **not** safe for concurrent writes.
> - At most **ONE goroutine** may call read methods (`ReadMessage`, `ReadJSON`, `NextReader`, `SetReadDeadline`, `SetPongHandler`...) at a time.
> - At most **ONE goroutine** may call write methods (`WriteMessage`, `WriteJSON`, `NextWriter`, `WriteControl`...) at a time.
> - A read and a write *may* happen concurrently with each other — that is fine and expected (one reader goroutine + one writer goroutine).

The one documented exception: `WriteControl` (used for ping/pong/close control frames) **may** be called concurrently with the writer goroutine. Everything else obeys the one-reader/one-writer rule.

### 6.2 Why concurrent writes break things **[I]**

`WriteMessage` writes a frame **header** followed by the **payload** as separate operations on the socket. If two goroutines call it at the same instant, their bytes interleave on the wire: half of message A's payload lands inside message B's frame. The receiver then sees a malformed frame and the connection is dead — or Gorilla detects the reentrancy and panics. Gorilla deliberately does **not** add internal locking; it documents the constraint so you build the right architecture instead of paying for a mutex on every write.

```go
// BAD — two goroutines writing the same conn at once. Corruption or panic.
go func() { conn.WriteMessage(websocket.TextMessage, []byte("a")) }()
go func() { conn.WriteMessage(websocket.TextMessage, []byte("b")) }() // RACE
```

### 6.3 The canonical solution: a `send` channel + a single writePump **[I/A]**

The idiom that resolves this — straight from Gorilla's official chat example and used in every serious Go WebSocket codebase — is the **two-goroutine-per-connection** model:

- **`readPump`** — the *only* goroutine that reads from the conn. It reads messages and hands them off to application logic (the Hub) via channels. It never writes to the conn.
- **`writePump`** — the *only* goroutine that writes to the conn. Everything that wants to send a message instead pushes it onto a buffered `send chan []byte`; the writePump drains that channel and is the sole writer. It also owns the periodic ping.

Because each goroutine touches the conn for only one direction, the one-reader/one-writer rule is satisfied *by construction* with no locks. The `send` channel is a **serialization point**: many producers (the Hub, timers, the readPump signalling a reply) push to it, one consumer (the writePump) writes to the socket.

```go
package main

import (
	"time"

	"github.com/gorilla/websocket"
)

// Timing constants — tuned together. pingPeriod MUST be < pongWait so a ping
// goes out before the read deadline expires (see §7 for the full logic).
const (
	writeWait      = 10 * time.Second    // max time allowed for a single write
	pongWait       = 60 * time.Second    // how long we wait for a pong before giving up
	pingPeriod     = (pongWait * 9) / 10 // 54s — ping a bit before pongWait elapses
	maxMessageSize = 512 * 1024          // 512 KB read cap
)

// Client wraps one connection plus its outbound queue.
type Client struct {
	conn *websocket.Conn
	// send is a BUFFERED channel of outbound messages. Producers push here;
	// writePump (the only writer) drains it. Buffering absorbs short bursts.
	send chan []byte
}

// writePump is the ONLY goroutine that writes to c.conn.
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod) // drives periodic keepalive pings
	defer func() {
		ticker.Stop()
		c.conn.Close() // closing the conn unblocks readPump's ReadMessage too
	}()

	for {
		select {
		case message, ok := <-c.send:
			// Bound every write with a deadline so a stuck/slow client cannot
			// block this goroutine forever.
			_ = c.conn.SetWriteDeadline(time.Now().Add(writeWait))

			if !ok {
				// The Hub closed c.send (the client is being removed). Send a
				// clean close frame and exit.
				_ = c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}

			// Use NextWriter so we can coalesce any other already-queued
			// messages into ONE WebSocket frame — a real efficiency win under
			// load (fewer frames, fewer syscalls).
			w, err := c.conn.NextWriter(websocket.TextMessage)
			if err != nil {
				return
			}
			_, _ = w.Write(message)

			n := len(c.send) // how many more are waiting RIGHT NOW
			for i := 0; i < n; i++ {
				_, _ = w.Write([]byte{'\n'}) // newline separator between messages
				_, _ = w.Write(<-c.send)
			}

			if err := w.Close(); err != nil { // flush the coalesced frame
				return
			}

		case <-ticker.C:
			// Time to ping. If the write fails, the peer is gone — exit.
			_ = c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}

// readPump is the ONLY goroutine that reads from c.conn.
func (c *Client) readPump() {
	defer c.conn.Close() // ensures writePump's select unblocks via a write error

	// Cap message size BEFORE the first read to prevent memory exhaustion.
	c.conn.SetReadLimit(maxMessageSize)

	// The first read must complete within pongWait; thereafter the pong handler
	// keeps extending the deadline (see §7).
	_ = c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		// A pong proves the client is alive -> push the deadline forward.
		return c.conn.SetReadDeadline(time.Now().Add(pongWait))
	})

	for {
		_, message, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseAbnormalClosure,
			) {
				// log it; an expected close needs no noise
			}
			break
		}
		// Hand off to app logic — NEVER write to the conn from here.
		c.handleIncoming(message)
	}
}

func (c *Client) handleIncoming(msg []byte) {
	// e.g. validate, then push onto the Hub's broadcast channel (§9).
}
```

### 6.4 The mental model **[I]**

```
                          ┌─────────────┐
                          │  net/http   │  serveWs(): Upgrade(), build Client,
                          │  goroutine  │  spawn the two pumps, then RETURN.
                          └──────┬──────┘
                                 │ go writePump(); go readPump()
              ┌──────────────────┴──────────────────┐
              ▼                                       ▼
      ┌───────────────┐                    ┌───────────────────┐
      │  readPump()   │   c.send (chan)    │   writePump()     │
      │  goroutine    │ ◀───────────────── │   goroutine       │
      │ ONLY reads    │   (Hub & timers    │ ONLY writes       │
      │ conn.Read*()  │    push here)       │ conn.Write*()     │
      └──────┬────────┘                    └─────────┬─────────┘
             ▼                                       ▼
        frames in  ◀──────  one TCP socket  ──────▶  frames out
```

The HTTP handler goroutine **must return** after spawning the pumps — do not block it. Blocking it ties up the server's request-handling goroutine for the entire life of the socket and defeats the whole design (see Gotcha §16.3).

---

## 7. Ping/Pong, Deadlines & Connection Health

### 7.1 Why you cannot trust TCP to tell you about death **[I]**

A TCP connection can **silently die** with no FIN packet: a NAT/firewall idle-timeout drops the mapping, a mobile device switches from Wi-Fi to cellular, a laptop sleeps, a client process is `kill -9`'d. In all these cases your server's `ReadMessage` just *blocks forever* waiting for bytes that will never come. You leak the connection's memory, its two goroutines, and its slot in the Hub — indefinitely. With enough ghosts you exhaust memory or file descriptors and crash. So you need an **application-level liveness check**: ping/pong plus read deadlines.

### 7.2 How ping/pong works **[I]**

The protocol defines two control frames: **ping** (opcode 9) and **pong** (opcode 10). The rule is simple: any endpoint that receives a ping must reply with a pong carrying the same payload. **Gorilla auto-replies to incoming pings for you** — you rarely send pongs by hand. The strategy:

1. The server's `writePump` sends a **ping** every `pingPeriod` (54 s).
2. A live client's WebSocket stack auto-replies with a **pong**.
3. The server's `readPump` has a **read deadline** of `pongWait` (60 s). Its `PongHandler` pushes that deadline forward every time a pong arrives.
4. If the client is dead, no pong comes, the read deadline expires, `ReadMessage` returns an error, `readPump` exits, it closes the conn, `writePump`'s next write fails and *it* exits — both goroutines and all memory are reclaimed within ~`pongWait`.

The key invariant: **`pingPeriod < pongWait`.** You must send a ping (and get a pong) *before* the read deadline elapses, with margin. `(pongWait * 9) / 10` gives a 10% safety margin.

### 7.3 Deadlines are absolute times, reset on activity **[I]**

`SetReadDeadline`/`SetWriteDeadline` take an *absolute* `time.Time`, not a duration — so you must keep re-arming them. Reads re-arm in the `PongHandler` (and optionally after each real message); writes re-arm before *every* write.

```go
const (
	writeWait  = 10 * time.Second
	pongWait   = 60 * time.Second
	pingPeriod = (pongWait * 9) / 10 // MUST be < pongWait
)

// --- read side (inside readPump, before the loop) ---
_ = conn.SetReadLimit(512 * 1024)
_ = conn.SetReadDeadline(time.Now().Add(pongWait)) // arm the initial deadline
conn.SetPongHandler(func(appData string) error {
	// Client answered our ping -> it is alive -> push the deadline forward.
	return conn.SetReadDeadline(time.Now().Add(pongWait))
})

// --- write side (inside writePump, before every write) ---
_ = conn.SetWriteDeadline(time.Now().Add(writeWait))
if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
	return // pong never came in time, or the socket is broken
}
```

### 7.4 SetReadLimit — the cheap, essential cap **[I]**

```go
// Reject any single message larger than this. Gorilla closes the connection
// with CloseMessageTooBig (1009). This is your first line of defense against a
// client (malicious or buggy) sending a giant payload to exhaust your heap.
// Set it BEFORE the first read. Cross-reference §14 (security).
conn.SetReadLimit(512 * 1024) // 512 KB
```

### 7.5 Overriding control-frame handlers (advanced) **[A]**

You rarely need these — Gorilla's defaults are correct — but here is how, and why you usually shouldn't.

```go
// SetPingHandler — the DEFAULT already sends the matching pong. Override only
// if you need to observe pings; you MUST still send the pong yourself.
conn.SetPingHandler(func(appData string) error {
	err := conn.WriteControl(websocket.PongMessage, []byte(appData),
		time.Now().Add(writeWait))
	if err == websocket.ErrCloseSent {
		return nil // peer already closing; not an error
	}
	return err
})

// SetCloseHandler — the DEFAULT echoes a close frame back (correct behavior).
// Override only to observe the code/reason; preserve the echo.
conn.SetCloseHandler(func(code int, text string) error {
	msg := websocket.FormatCloseMessage(code, "")
	_ = conn.WriteControl(websocket.CloseMessage, msg, time.Now().Add(writeWait))
	return nil
})
```

> **Note:** control-frame handlers run *inside* the read goroutine (during `ReadMessage`). They use `WriteControl`, which is the one write method safe to call concurrently with the writePump — that is *why* they can write a pong without violating the one-writer rule.

---

## 8. Handling Disconnects & Close Codes

### 8.1 The close handshake **[I]**

A clean disconnect is a small handshake: one side sends a **close frame** (optionally with a numeric code and a short UTF-8 reason), the peer **echoes** a close frame back, and both close the TCP socket. Gorilla performs the echo automatically inside `ReadMessage`: when it receives a close frame it sends the reply and returns a `*websocket.CloseError` from `ReadMessage`. So in your read loop, *any* error means "the connection is over" — your job is to classify it (expected vs unexpected) for logging and to always exit the loop.

### 8.2 Standard close codes **[I]**

| Code | Constant | Meaning |
|---|---|---|
| 1000 | `CloseNormalClosure` | Clean, intentional shutdown |
| 1001 | `CloseGoingAway` | Browser tab closed, or server going down |
| 1002 | `CloseProtocolError` | Protocol violation |
| 1003 | `CloseUnsupportedData` | Got a message type it can't accept |
| 1006 | `CloseAbnormalClosure` | No close frame (TCP died) — never *sent*, only observed |
| 1008 | `ClosePolicyViolation` | App-level rejection (auth/policy) |
| 1009 | `CloseMessageTooBig` | Payload exceeded the read limit |
| 1011 | `CloseInternalServerErr` | Server-side error |

### 8.3 Classifying close errors **[I]**

```go
import "errors"

func (c *Client) readPump() {
	defer c.conn.Close()

	for {
		_, msg, err := c.conn.ReadMessage()
		if err != nil {
			var closeErr *websocket.CloseError
			switch {
			case errors.As(err, &closeErr):
				// We received a proper close frame — expected, log the details.
				log.Printf("client closed: code=%d reason=%q", closeErr.Code, closeErr.Text)
			case websocket.IsUnexpectedCloseError(err,
				websocket.CloseNormalClosure,
				websocket.CloseGoingAway):
				// Abnormal: TCP reset, deadline exceeded, crash — worth logging.
				log.Printf("unexpected close: %v", err)
			default:
				// Normal/going-away or a benign read error — usually quiet.
			}
			return // ALWAYS exit the loop on any read error
		}
		c.handleIncoming(msg)
	}
}
```

`websocket.IsUnexpectedCloseError(err, expectedCodes...)` returns true only when the error is a close *and* its code is **not** in your "expected" list — perfect for "log the surprising ones, ignore the routine ones."

### 8.4 Initiating a clean close from the server **[I]**

```go
// closeConn sends a close frame, gives the peer a moment to echo, then closes.
func closeConn(conn *websocket.Conn, code int, reason string) {
	msg := websocket.FormatCloseMessage(code, reason)
	// WriteControl is safe even alongside the writePump (see §6.1/§7.5).
	_ = conn.WriteControl(websocket.CloseMessage, msg, time.Now().Add(5*time.Second))
	// Brief grace period for the peer's echo, then drop the TCP socket.
	time.Sleep(200 * time.Millisecond)
	_ = conn.Close()
}

// Usage during shutdown:
closeConn(conn, websocket.CloseGoingAway, "server restarting")
```

---

## 9. The Hub / Broadcast Pattern — Full Chat Server

### 9.1 The pattern and its logic **[I/A]**

The Hub is the standard architecture for *any* multi-client real-time server — chat rooms, live dashboards, notification fan-out, presence. The core insight is a beautiful application of Go's "share memory by communicating" philosophy:

> A **Hub is a single goroutine that exclusively owns the map of connected clients.** Because only that one goroutine ever touches the map, **no mutex is needed.** Everyone else interacts with the Hub by sending on channels — `register`, `unregister`, `broadcast`.

This sidesteps an entire class of concurrency bugs. There is no shared mutable map guarded by a lock that you might forget to take; there is one owner and a set of typed mailboxes (channels) feeding it. The Hub's `run()` loop is a `select` over those channels — it is a tiny state machine processing one event at a time, serially, safely.

### 9.2 Architecture **[I]**

```
  HTTP handler (serveWs)
        │ Upgrade -> *websocket.Conn -> build Client
        │ hub.register <- client ; go writePump ; go readPump
        ▼
   ┌────────┐   register / unregister / broadcast (channels)
   │ Client │ ───────────────────────────────────────▶ ┌─────────────────┐
   │ struct │                                           │       Hub       │
   └───┬────┘ ◀────────────── c.send (chan) ─────────── │ map[*Client]bool│
 readPump│  pushes incoming msgs to hub.broadcast        │ (single owner)  │
 writePump│ drains c.send, writes to socket              └─────────────────┘
          │                                                   ▲
          └──── other clients' readPumps push here ───────────┘
```

### 9.3 hub.go **[I]**

```go
package main

import "log/slog"

// Hub owns the registry of clients and fans messages out to them.
type Hub struct {
	// clients: the set of connected clients. ONLY the Hub goroutine (run())
	// reads or writes this map, so no mutex is required.
	clients map[*Client]bool

	// broadcast: messages destined for every client. Producers are clients'
	// readPumps (and, when scaling, the Redis subscriber in §15).
	broadcast chan []byte

	// register / unregister: lifecycle events from serveWs and the pumps.
	register   chan *Client
	unregister chan *Client
}

func newHub() *Hub {
	return &Hub{
		clients:    make(map[*Client]bool),
		broadcast:  make(chan []byte, 256), // buffer absorbs bursts
		register:   make(chan *Client),
		unregister: make(chan *Client),
	}
}

// run is the Hub's event loop. Start it in EXACTLY ONE goroutine (go hub.run()).
func (h *Hub) run() {
	for {
		select {
		case client := <-h.register:
			h.clients[client] = true
			slog.Info("client registered", "total", len(h.clients))

		case client := <-h.unregister:
			if _, ok := h.clients[client]; ok {
				delete(h.clients, client)
				close(client.send) // signal the client's writePump to exit
				slog.Info("client unregistered", "total", len(h.clients))
			}

		case message := <-h.broadcast:
			// Fan out to every client via its buffered send channel.
			for client := range h.clients {
				select {
				case client.send <- message:
					// queued OK
				default:
					// The client's send buffer is FULL -> it is too slow to keep
					// up. Drop it rather than let one slow client stall the Hub
					// (backpressure — see §10). Closing send makes writePump exit.
					delete(h.clients, client)
					close(client.send)
					slog.Warn("dropped slow client")
				}
			}
		}
	}
}
```

> **Why the `default` case in the broadcast loop is non-negotiable:** without it, `client.send <- message` *blocks* when a client's buffer is full. That blocks the Hub goroutine, which blocks *every other client's* broadcast, which blocks their readPumps trying to push to `hub.broadcast` — the whole server stalls because of one slow connection. The `default` makes the send non-blocking and converts "slow client" into "dropped client." See §10 for nuanced alternatives.

### 9.4 client.go **[I]**

This is the §6 two-pump model, wired to the Hub.

```go
package main

import (
	"bytes"
	"time"

	"github.com/gorilla/websocket"
)

const (
	writeWait      = 10 * time.Second
	pongWait       = 60 * time.Second
	pingPeriod     = (pongWait * 9) / 10
	maxMessageSize = 512 * 1024
)

type Client struct {
	hub  *Hub
	conn *websocket.Conn
	send chan []byte // outbound queue; writePump is the sole consumer
}

// readPump: the ONLY reader. Pumps inbound messages to the Hub.
func (c *Client) readPump() {
	defer func() {
		c.hub.unregister <- c // tell the Hub we're gone
		c.conn.Close()        // unblocks writePump via a write error too
	}()

	c.conn.SetReadLimit(maxMessageSize)
	_ = c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		return c.conn.SetReadDeadline(time.Now().Add(pongWait))
	})

	for {
		_, message, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				// log unexpected closes
			}
			break
		}
		// Normalize whitespace, then hand to the Hub for broadcast.
		message = bytes.TrimSpace(bytes.Replace(message, []byte{'\n'}, []byte{' '}, -1))
		c.hub.broadcast <- message
	}
}

// writePump: the ONLY writer. Drains c.send, coalesces, and pings.
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case message, ok := <-c.send:
			_ = c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				// Hub closed the channel -> send close frame and exit.
				_ = c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			w, err := c.conn.NextWriter(websocket.TextMessage)
			if err != nil {
				return
			}
			_, _ = w.Write(message)
			// Coalesce any other queued messages into this one frame.
			n := len(c.send)
			for i := 0; i < n; i++ {
				_, _ = w.Write([]byte{'\n'})
				_, _ = w.Write(<-c.send)
			}
			if err := w.Close(); err != nil {
				return
			}

		case <-ticker.C:
			_ = c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}
```

### 9.5 main.go — wiring it together **[I]**

```go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		// PRODUCTION: replace with an allowlist (see §4.4 / §14).
		return true
	},
}

// serveWs upgrades the request and registers a new client with the hub.
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	// (Authenticate r BEFORE this point — see §14.)
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println("upgrade:", err)
		return
	}
	client := &Client{
		hub:  hub,
		conn: conn,
		send: make(chan []byte, 256), // buffered: absorbs bursts before drop
	}
	client.hub.register <- client

	// Spawn the pumps; the HTTP handler then RETURNS (do not block it).
	go client.writePump()
	go client.readPump()
}

func main() {
	hub := newHub()
	go hub.run() // single Hub goroutine

	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWs(hub, w, r)
	})
	http.Handle("/", http.FileServer(http.Dir("./static")))

	log.Println("chat server on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 9.6 Rooms — scoping broadcasts **[A]**

A flat Hub broadcasts to *everyone*. Real chat has **rooms**. The cleanest extension keeps the single-owner-goroutine property by making the Hub own a nested map and routing on a room key. Messages become typed envelopes carrying their room.

```go
// A typed envelope. The "room" routes the message; the server stamps Time.
type Message struct {
	Type    string `json:"type"`    // "chat" | "join" | "leave" | "system"
	Room    string `json:"room"`
	User    string `json:"user,omitempty"`
	Content string `json:"content,omitempty"`
	Time    int64  `json:"time"`    // server-set Unix millis
}

// RoomHub: the Hub now keys clients by room. Still one owner goroutine.
type RoomHub struct {
	rooms      map[string]map[*Client]bool // room -> set of clients
	register   chan *Client
	unregister chan *Client
	broadcast  chan Message // carries its own Room
}

func (h *RoomHub) run() {
	for {
		select {
		case c := <-h.register:
			if h.rooms[c.room] == nil {
				h.rooms[c.room] = make(map[*Client]bool)
			}
			h.rooms[c.room][c] = true

		case c := <-h.unregister:
			if set, ok := h.rooms[c.room]; ok {
				if _, ok := set[c]; ok {
					delete(set, c)
					close(c.send)
					if len(set) == 0 {
						delete(h.rooms, c.room) // tidy up empty rooms
					}
				}
			}

		case msg := <-h.broadcast:
			msg.Time = time.Now().UnixMilli()
			data, _ := json.Marshal(msg)
			for c := range h.rooms[msg.Room] { // only this room's clients
				select {
				case c.send <- data:
				default:
					delete(h.rooms[msg.Room], c)
					close(c.send)
				}
			}
		}
	}
}
```

> **Scaling caveat:** rooms work perfectly within one process, but across multiple instances a client in room "X" on Server A and another in room "X" on Server B won't see each other unless you add a backplane (§15). The backplane simply re-injects messages into each instance's `broadcast`.

---

## 10. Backpressure & Slow Clients

### 10.1 The problem, precisely **[A]**

"Backpressure" is what happens when a producer outpaces a consumer. In a Hub, the producer is "everyone broadcasting" and the per-client consumer is `writePump`, whose speed is limited by *that client's network*. A client on a flaky mobile link, or one that has stopped reading, drains its `send` channel slowly or not at all. Messages pile up. You have a finite amount of memory; you must decide what happens when a client cannot keep up. There is no free lunch — every strategy trades something.

### 10.2 The strategies and when to use each **[A]**

| Strategy | Behavior | Good for | Cost |
|---|---|---|---|
| **Bounded buffer + drop the client** (default) | When `send` is full, unregister and close the client | Chat, presence — a client that can't keep up is broken anyway | Slow client loses the connection (it should reconnect) |
| **Drop the message, keep the client** | Skip the send; client stays connected | Telemetry/dashboards where stale data is worthless and the next update supersedes it | Client silently misses updates |
| **Conflate / latest-wins** | Replace the queued message with the newest | Live price/metric where only the current value matters | Intermediate values lost (often desirable) |
| **Block (no `default`)** | Hub waits for the slow client | Almost never — one slow client stalls everyone | Total server stall — **avoid** |
| **Per-client overflow with grace** | Allow N strikes before dropping | Tolerant of brief stalls | More state, more complexity |

The Hub in §9 uses the first strategy. The decision hinges on your **delivery semantics**: must every client receive every message (then a dropped client must reconnect and resync), or is newest-only acceptable (then conflate)?

### 10.3 Conflation example (latest-wins) **[A]**

```go
// For a metrics dashboard: keep only the freshest value per client. The send
// channel has capacity 1; if it's full, replace the pending value.
func pushLatest(c *Client, msg []byte) {
	for {
		select {
		case c.send <- msg:
			return // delivered (or queued)
		default:
			// Buffer full: discard the stale pending message, then retry.
			select {
			case <-c.send: // drop the old one
			default:
			}
			// loop and try to enqueue the fresh msg again
		}
	}
}
```

### 10.4 Best practices **[A]**
- **Size `send` deliberately.** Too small drops healthy clients during normal bursts; too large lets a dead client hoard memory before you notice. 256 is a common starting point for chat; tune with metrics.
- **Always bound writes with `SetWriteDeadline`** (§7) so a stuck TCP write cannot wedge a writePump forever.
- **Emit metrics**: dropped-client count, average `len(send)`, broadcast queue depth. A rising drop rate is your early warning that the system is overloaded.
- **Give clients reconnect logic** (§11.4) so a drop is a hiccup, not a permanent disconnect.

---

## 11. Clients — Browser JS & the Go Dialer

### 11.1 The browser `WebSocket` API **[I]**

The browser ships a native `WebSocket` object — no library needed. It exposes four event callbacks (`onopen`, `onmessage`, `onclose`, `onerror`), a `send()` method, and a `readyState`. The important detail: it does **not** auto-reconnect (§11.4 adds that).

```html
<!-- static/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8" />
	<title>Go Chat</title>
</head>
<body>
	<div id="log" style="height:400px;overflow:auto;border:1px solid #ccc;padding:8px;font-family:monospace"></div>
	<input id="msg" type="text" placeholder="Type a message…" style="width:80%" />
	<button id="send">Send</button>

	<script>
		const logEl = document.getElementById('log');
		const input = document.getElementById('msg');

		// Use wss:// in production. Build the URL from location for portability.
		const proto = location.protocol === 'https:' ? 'wss' : 'ws';
		const ws = new WebSocket(`${proto}://${location.host}/ws`);

		ws.onopen = () => append('Connected');

		ws.onmessage = (event) => {
			// The Go writePump may coalesce several messages, newline-separated.
			for (const line of event.data.split('\n')) {
				try {
					const m = JSON.parse(line);
					append(`[${m.user}] ${m.content}`);
				} catch {
					append(line); // plain-text fallback
				}
			}
		};

		ws.onclose = (e) => append(`Disconnected: code=${e.code} reason=${e.reason}`);
		ws.onerror = () => append('Socket error');

		document.getElementById('send').addEventListener('click', sendMessage);
		input.addEventListener('keydown', (e) => { if (e.key === 'Enter') sendMessage(); });

		function sendMessage() {
			const text = input.value.trim();
			if (!text || ws.readyState !== WebSocket.OPEN) return;
			ws.send(JSON.stringify({ type: 'chat', user: 'Alice', content: text }));
			input.value = '';
		}

		function append(text) {
			const p = document.createElement('p');
			p.textContent = text;
			logEl.appendChild(p);
			logEl.scrollTop = logEl.scrollHeight;
		}
	</script>
</body>
</html>
```

### 11.2 `readyState` reference **[B]**

| Value | Constant | Meaning |
|---|---|---|
| 0 | `CONNECTING` | Handshake in progress |
| 1 | `OPEN` | Connected; safe to `send()` |
| 2 | `CLOSING` | Close handshake underway |
| 3 | `CLOSED` | Connection closed/failed |

Always guard `ws.send()` with `ws.readyState === WebSocket.OPEN`; sending while CONNECTING throws.

### 11.3 The Go client (`websocket.Dialer`) **[I]**

Gorilla also dials *out*. This is invaluable for integration tests, service-to-service streaming, CLIs, and load generators. `Dialer` mirrors the `Upgrader`'s options for the client side: timeouts, TLS config, proxy, custom request headers (your chance to send `Authorization`).

```go
// goclient/main.go — connect to a WebSocket server from Go.
package main

import (
	"crypto/tls"
	"encoding/json"
	"log"
	"net/url"
	"os"
	"os/signal"
	"time"

	"github.com/gorilla/websocket"
)

type Message struct {
	Type    string `json:"type"`
	User    string `json:"user"`
	Content string `json:"content"`
}

func main() {
	u := url.URL{Scheme: "ws", Host: "localhost:8080", Path: "/ws"}
	log.Printf("connecting to %s", u.String())

	dialer := websocket.Dialer{
		HandshakeTimeout: 10 * time.Second,
		// For wss:// against a real cert, leave TLSClientConfig nil. For a
		// self-signed dev cert, set RootCAs — NEVER InsecureSkipVerify in prod.
		TLSClientConfig: &tls.Config{InsecureSkipVerify: false},
	}

	// RequestHeader carries auth/cookies into the handshake.
	header := map[string][]string{"Authorization": {"Bearer my-token"}}

	conn, resp, err := dialer.Dial(u.String(), header)
	if err != nil {
		// On failure, resp holds the HTTP response (e.g. 401/403) — inspect it.
		log.Fatalf("dial failed: %v (status %v)", err, resp)
	}
	defer conn.Close()

	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt)

	// Reader goroutine (the ONE reader, per §6).
	done := make(chan struct{})
	go func() {
		defer close(done)
		for {
			_, data, err := conn.ReadMessage()
			if err != nil {
				log.Println("read:", err)
				return
			}
			var m Message
			if json.Unmarshal(data, &m) == nil {
				log.Printf("[%s] %s", m.User, m.Content)
			} else {
				log.Printf("raw: %s", data)
			}
		}
	}()

	ticker := time.NewTicker(3 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			m := Message{Type: "chat", User: "GoClient", Content: "ping from Go"}
			data, _ := json.Marshal(m)
			// This goroutine is the ONE writer here.
			if err := conn.WriteMessage(websocket.TextMessage, data); err != nil {
				log.Println("write:", err)
				return
			}

		case <-interrupt:
			log.Println("interrupt — closing cleanly")
			// Polite close: send a close frame, then wait briefly for the echo.
			_ = conn.WriteMessage(websocket.CloseMessage,
				websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
			select {
			case <-done:
			case <-time.After(time.Second):
			}
			return

		case <-done:
			return // server closed first
		}
	}
}
```

### 11.4 Reconnection with backoff (browser) **[I]**

Because neither the browser nor Gorilla reconnects for you, production clients need a small reconnect loop with **exponential backoff and jitter** — so that when your server restarts, ten thousand clients don't all reconnect in the same millisecond (a "thundering herd").

```html
<script>
	let backoff = 1000;             // start at 1s
	const maxBackoff = 30000;       // cap at 30s
	let ws;

	function connect() {
		const proto = location.protocol === 'https:' ? 'wss' : 'ws';
		ws = new WebSocket(`${proto}://${location.host}/ws`);

		ws.onopen = () => { backoff = 1000; /* reset on success */ };

		ws.onclose = () => {
			// Reconnect after backoff + random jitter, then grow backoff.
			const jitter = Math.random() * 1000;
			setTimeout(connect, backoff + jitter);
			backoff = Math.min(backoff * 2, maxBackoff);
		};
	}
	connect();
</script>
```

---

## 12. Running WebSockets Inside Gin

### 12.1 The logic: Gin is still net/http underneath **[I]**

If your REST API uses **Gin** (see `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`), you don't need a separate server for WebSockets. Gin's `*gin.Context` exposes the raw `http.ResponseWriter` (`c.Writer`) and `*http.Request` (`c.Request`), and `upgrader.Upgrade` only needs those two. So you upgrade *inside* a Gin handler and get all of Gin's routing, middleware, and parameter binding for free — including auth middleware that runs *before* the upgrade (exactly where §2.3 says it must).

### 12.2 The handler **[I]**

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool {
		return r.Header.Get("Origin") == "https://myapp.example.com"
	},
}

func wsHandler(hub *Hub) gin.HandlerFunc {
	return func(c *gin.Context) {
		// Gin auth middleware has already run; pull the authenticated user.
		userID := c.GetString("userID") // set by your JWT middleware
		if userID == "" {
			c.AbortWithStatus(http.StatusUnauthorized) // still HTTP — safe pre-upgrade
			return
		}

		// Upgrade using Gin's underlying writer + request.
		conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
		if err != nil {
			// Upgrade wrote the error already; do not touch c.Writer again.
			return
		}

		client := &Client{hub: hub, conn: conn, send: make(chan []byte, 256)}
		hub.register <- client
		go client.writePump()
		go client.readPump()
	}
}

func main() {
	r := gin.Default()
	hub := newHub()
	go hub.run()

	// Apply JWT middleware to the WS route so auth runs before the upgrade.
	r.GET("/ws", authMiddleware(), wsHandler(hub))
	_ = r.Run(":8080")
}
```

> **Gotcha:** after a successful `Upgrade`, do **not** call any `c.JSON`/`c.String`/`c.Status` — the HTTP response is already finished. Gin's logger middleware will report the request as a 101 (or sometimes 200); that's expected.

> **Cross-reference:** for JWT verification middleware and Argon2 password hashing used by `authMiddleware`, see `GO_JWT_ARGON2_GUIDE.md`. For the Gin router/middleware mechanics, see `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`.

---

## 13. Graceful Shutdown

### 13.1 Why it matters **[A]**

When you deploy a new version or scale down, the process gets a `SIGTERM`. If you just exit, every open WebSocket dies abruptly (clients see code 1006, abnormal closure) and any in-flight broadcast is lost. **Graceful shutdown** means: stop accepting new connections, tell existing clients you are going away (close code 1001 `CloseGoingAway`), let the pumps drain and exit, then stop the HTTP server — all within a deadline. This is the same `http.Server.Shutdown` pattern from `GO_NET_HTTP_REST_API_GUIDE.md`, extended to also close the WebSocket clients (which `Shutdown` does **not** do on its own — `Shutdown` waits for idle connections, but a live WebSocket is never "idle").

### 13.2 Implementation **[A]**

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gorilla/websocket"
)

// closeAll asks every client to close cleanly. Run it from the Hub goroutine
// (e.g. via a dedicated channel) so the map access stays single-owner.
func (h *Hub) closeAll() {
	msg := websocket.FormatCloseMessage(websocket.CloseGoingAway, "server shutting down")
	for c := range h.clients {
		// WriteControl is safe alongside writePump (§6.1).
		_ = c.conn.WriteControl(websocket.CloseMessage, msg, time.Now().Add(time.Second))
		close(c.send) // make each writePump exit
		delete(h.clients, c)
	}
}

func main() {
	hub := newHub()
	go hub.run()

	mux := http.NewServeMux()
	mux.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) { serveWs(hub, w, r) })
	srv := &http.Server{Addr: ":8080", Handler: mux}

	// Listen for SIGTERM / Ctrl-C.
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, syscall.SIGTERM, os.Interrupt)

	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			slog.Error("listen", "err", err)
		}
	}()
	slog.Info("server up on :8080")

	<-stop // block until a shutdown signal arrives
	slog.Info("shutting down…")

	// 1) Tell all WebSocket clients we're going away (must run on Hub goroutine
	//    in real code — shown inline for brevity).
	hub.closeAll()

	// 2) Drain in-flight HTTP work with a deadline; stop accepting new conns.
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		slog.Error("graceful shutdown failed", "err", err)
	}
	slog.Info("bye")
}
```

> **Note:** to keep the single-owner invariant, the *clean* way to trigger `closeAll` is to add a `shutdown chan struct{}` to the Hub and handle it inside `run()`'s `select`, so the map is only ever touched by the Hub goroutine. The inline call above is simplified for the example.

---

## 14. Security — CheckOrigin, Auth, Limits, Rate Limiting, TLS

Security for WebSockets is mostly about the handshake (where you still have HTTP) and about bounding what a connected client can do to your memory and CPU. Treat every connected socket as hostile until proven otherwise.

### 14.1 Origin checking / CSWSH (recap + rules) **[A]**

Covered in depth in §4.4. The rules, restated as a checklist:
- **Never** ship `CheckOrigin: func(r) bool { return true }`. Use an **allowlist**.
- Prefer exact origin equality over substring matching (`evil-example.com` defeats `Contains("example.com")`).
- `Origin` is trustworthy only from **browsers**; native clients can forge it, so origin checking is *defense in depth*, **not** authentication.

### 14.2 Authenticate BEFORE the upgrade **[A]**

The handshake is HTTP; that is your only chance to return `401`. Validate identity, then upgrade. Patterns, best to worst:

1. **Session cookie** (browser, same-site): the cookie rides the handshake automatically. Combine with strict `CheckOrigin` to block CSWSH. Best for first-party web apps.
2. **Short-lived "ticket"**: the client first calls an authenticated REST endpoint (`POST /ws-ticket`) to get a single-use, short-TTL token, then connects with `?ticket=...`. The ticket is useless if leaked (one-time, seconds-long). **Recommended** when you cannot rely on cookies.
3. **First-message auth**: upgrade, then require the client's *first* WebSocket message to be a valid token within a short deadline; otherwise close. Keeps the token out of URLs/logs.
4. **JWT in query string** (`?token=...`): simplest, but the token lands in server logs, proxy logs, and browser history. Acceptable only for short-lived tokens; prefer #2 or #3.

```go
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	// Validate BEFORE upgrading — after Upgrade you can't send 401.
	userID, err := authenticate(r) // cookie / ticket / token — see GO_JWT_ARGON2_GUIDE.md
	if err != nil {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	client := &Client{hub: hub, conn: conn, userID: userID, send: make(chan []byte, 256)}
	hub.register <- client
	go client.writePump()
	go client.readPump()
}
```

> **First-message auth sketch:** after upgrade, `conn.SetReadDeadline(time.Now().Add(5*time.Second))`, read one message, verify the token, then clear the deadline and continue — or close with `ClosePolicyViolation` (1008) if invalid.

> **Cross-reference:** JWT signing/verification and Argon2id password hashing live in `GO_JWT_ARGON2_GUIDE.md`.

### 14.3 Message size limits **[A]**

Always cap inbound message size *before the first read*. Without it, `ReadMessage`/`ReadJSON` will buffer whatever the client sends.

```go
conn.SetReadLimit(64 * 1024) // 64 KB; oversize -> server closes with 1009
```

### 14.4 Rate limiting **[A]**

A connected client can flood you with messages. Limit per-connection message rate with a token bucket (`golang.org/x/time/rate`). Apply it in `readPump`, *before* processing each message.

```go
import "golang.org/x/time/rate"

// 10 messages/sec sustained, bursts up to 20. Tune per app.
func (c *Client) readPump() {
	defer func() { c.hub.unregister <- c; c.conn.Close() }()
	c.conn.SetReadLimit(maxMessageSize)
	limiter := rate.NewLimiter(10, 20)

	for {
		_, msg, err := c.conn.ReadMessage()
		if err != nil {
			break
		}
		if !limiter.Allow() {
			// Over the limit: drop the message, or close with a policy violation.
			closeConn(c.conn, websocket.ClosePolicyViolation, "rate limit exceeded")
			break
		}
		c.hub.broadcast <- msg
	}
}
```

Also rate-limit **new connections** per IP (a connection-storm DoS) in your HTTP layer — middleware in front of `/ws` — before the upgrade ever runs.

### 14.5 TLS / `wss://` **[A]**

Plain `ws://` sends frames in cleartext: tokens, chat content, everything is sniffable and tamperable on the wire. **On the public internet, always use `wss://`** (WebSocket over TLS), exactly as you'd use `https://`. Two deployment shapes:

```go
// A) Terminate TLS in Go directly.
log.Fatal(srv.ListenAndServeTLS("cert.pem", "key.pem"))
```

```nginx
# B) Terminate TLS at a reverse proxy (common). Nginx MUST forward the
#    Upgrade/Connection headers or the handshake silently fails.
location /ws {
	proxy_pass         http://localhost:8080;
	proxy_http_version 1.1;
	proxy_set_header   Upgrade $http_upgrade;
	proxy_set_header   Connection "Upgrade";
	proxy_set_header   Host $host;
	proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_read_timeout 86400s; # MUST exceed your ping period or idle conns get killed
}
```

### 14.6 Input validation & the rest **[A]**
- **Validate and sanitize every message.** Treat payloads as untrusted: validate JSON shape, enforce field lengths, and **escape on output** to prevent stored XSS when chat content is rendered in browsers. The server relaying user content verbatim is a classic XSS vector.
- **Never echo server internals** in close reasons or error messages.
- **Bound goroutines**: ensure readPump/writePump *always* exit on disconnect (defers + the deadline machinery) so a flood of connect/disconnect cycles can't leak goroutines.

### 14.7 Security checklist **[A]**

| Concern | Action |
|---|---|
| CSWSH | Allowlist origins in `CheckOrigin`; never `return true` |
| Authentication | Validate before `Upgrade()`; prefer tickets/first-message over `?token=` |
| Message size | `SetReadLimit` before first read |
| Message rate | Per-connection token-bucket limiter in readPump |
| Connection storms | Per-IP connection rate limit in HTTP middleware |
| Transport | `wss://` only on the internet; forward Upgrade headers at proxies |
| XSS | Escape user content on output |
| Goroutine leaks | Deferred cleanup + read/write deadlines guarantee pump exit |

---

## 15. Scaling Beyond a Single Process — Redis Backplane

### 15.1 How far a single process goes **[A]**

Be honest about whether you *need* to scale out. One well-tuned Go process can hold **tens of thousands to ~100k** concurrent WebSocket connections, bounded by: per-connection memory (two goroutines + buffers — shrink buffers and use a `WriteBufferPool`, §4.2), message rate and payload size, total RAM, and the OS file-descriptor limit. For most apps (< ~50k concurrent), **a single instance plus a hot standby is the right answer** — scaling out adds real operational cost. Profile first.

OS tuning that lets one box go big:

```bash
ulimit -n 1000000              # each WS = 1 file descriptor
# /etc/sysctl.conf (persistent):
# net.core.somaxconn = 65535
# net.ipv4.tcp_max_syn_backlog = 65535
# fs.file-max = 1000000
```

### 15.2 The multi-instance problem **[A]**

The moment you run **two or more** instances behind a load balancer, each instance's Hub only knows its *own* clients. A message from a client on Server A must also reach clients connected to Servers B and C — but A's Hub has never heard of them. You need a **backplane**: a shared pub/sub bus that every instance subscribes to. When any instance receives a message to broadcast, it **publishes** to the bus; every instance (including itself) **receives** it from the bus and fans it out to *its* local clients. **Redis Pub/Sub** is the most common backplane — see `REDIS_GUIDE.md` for Redis itself.

```
Client A (Server 1) ── sends ──▶ Server 1 Hub ── PUBLISH "chat:room1" ──▶ Redis
                                                                            │
                       ┌────────────────────────┬───────────────────────┐  │
                       ▼                         ▼                       ▼  │ (fan-out)
                  Server 1 SUB             Server 2 SUB            Server 3 SUB
                       │                         │                       │
                  local clients             local clients           local clients
```

### 15.3 Redis Pub/Sub backplane in Go **[A]**

Using `github.com/redis/go-redis/v9`. Two halves: **publish** every broadcast to Redis instead of (or in addition to) the local Hub, and a **subscriber goroutine** that injects everything from Redis into the local Hub's `broadcast`.

```go
package main

import (
	"context"
	"log/slog"

	"github.com/redis/go-redis/v9"
)

type RedisBackplane struct {
	rdb     *redis.Client
	channel string
	hub     *Hub
}

func newBackplane(addr, channel string, hub *Hub) *RedisBackplane {
	return &RedisBackplane{
		rdb:     redis.NewClient(&redis.Options{Addr: addr}),
		channel: channel,
		hub:     hub,
	}
}

// Publish: send a message to ALL instances via Redis (instead of local-only).
func (b *RedisBackplane) Publish(ctx context.Context, msg []byte) error {
	return b.rdb.Publish(ctx, b.channel, msg).Err()
}

// Subscribe: receive every published message (from any instance, including us)
// and inject it into THIS instance's local Hub for local fan-out. Run once.
func (b *RedisBackplane) Subscribe(ctx context.Context) {
	pubsub := b.rdb.Subscribe(ctx, b.channel)
	defer pubsub.Close()
	for {
		msg, err := pubsub.ReceiveMessage(ctx)
		if err != nil {
			slog.Error("redis receive", "err", err)
			return // a supervisor should restart this
		}
		b.hub.broadcast <- []byte(msg.Payload) // local fan-out only
	}
}
```

The wiring change: a client's `readPump` now calls `backplane.Publish(...)` **instead of** `hub.broadcast <- msg`. The Hub's `broadcast` channel is fed *only* by the Redis subscriber, so every instance treats local and remote messages identically. (Watch out: the publishing instance also receives its own message back from Redis — which is exactly what you want, so it fans out to its local clients too. Don't double-deliver by *also* sending to the local Hub directly.)

### 15.4 Backplane options compared **[A]**

| Backplane | Use case | Trade-offs |
|---|---|---|
| **Redis Pub/Sub** | The default; simple fan-out across instances | Fire-and-forget (no persistence/replay); a client offline at broadcast misses the message |
| **Redis Streams** | Need replay / consumer groups / at-least-once | More complex than Pub/Sub |
| **NATS** | Lower latency, purpose-built messaging, subjects/wildcards | Another system to run |
| **Kafka** | High-throughput, durable, replayable event log | Heavyweight; overkill for chat |
| **Sticky sessions only** | Tiny scale; avoid fan-out by pinning a user to one instance | Uneven load; cross-instance rooms still broken |

> **Important:** none of these provide *delivery guarantees to the end client.* WebSocket itself isn't durable; the backplane only moves messages *between servers*. For "user must eventually receive this even if offline," persist to a DB and flush on (re)connect — see the Notifications project in §18.

> **Load balancer note:** WebSockets are long-lived, so configure your LB for WebSocket upgrade pass-through and long idle timeouts (> ping period). With a backplane you generally do **not** need sticky sessions for correctness, but they can still help connection distribution.

---

## 16. Tips, Tricks & Gotchas

### 16.1 Concurrent write panic — the #1 mistake **[I]**
```go
// WRONG — two goroutines writing the same conn.
go func() { conn.WriteJSON(a) }()
go func() { conn.WriteJSON(b) }() // corruption or panic
```
**Fix:** all writes go through one `writePump`; producers push to a `send` channel (§6). The only write call safe to use concurrently is `WriteControl`.

### 16.2 No deadlines = a goroutine-leak factory **[I]**
A silently-dropped client leaves `ReadMessage` blocked forever; its goroutines never exit; memory and goroutine count climb until you crash. **Fix:** always `SetReadDeadline` + `SetPongHandler` and a periodic ping; always `SetWriteDeadline` before each write (§7).

### 16.3 Don't block the HTTP handler **[I]**
```go
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, _ := upgrader.Upgrade(w, r, nil)
	client := &Client{conn: conn, send: make(chan []byte, 256)}
	hub.register <- client
	go client.writePump()
	go client.readPump()
	// CORRECT: handler returns; the two goroutines run independently.
}
```
Running the read loop *inline* in the handler ties up a server request goroutine for the socket's lifetime.

### 16.4 Backpressure: keep the `default` in the broadcast loop **[A]**
Without the non-blocking `default` (§9.3/§10), one slow client stalls the Hub and therefore every other client. Drop or conflate; never block.

### 16.5 Check `ok` on the send channel **[I]**
```go
case message, ok := <-c.send:
	if !ok { // Hub closed it -> send close frame and EXIT
		_ = c.conn.WriteMessage(websocket.CloseMessage, []byte{})
		return
	}
```
Ignoring `ok` makes you write `nil` on every tick after the channel closes.

### 16.6 Don't mix `NextWriter` and `WriteMessage` **[I]**
While a `NextWriter` is open you hold the write side; calling `WriteMessage` starts a second frame over the unfinished one. Always `w.Close()` first.

### 16.7 `ReadJSON` does not bound size **[I]**
It buffers the whole message before decoding. Set `SetReadLimit` first (§7.4/§14.3) or a giant frame OOMs you.

### 16.8 Send JSON as Text, not Binary **[B]**
Browsers accept either, but `TextMessage` is semantically correct and friendlier to middleboxes that scrutinize binary frames.

### 16.9 Reverse proxies must forward Upgrade headers **[A]**
Nginx/Envoy/HAProxy each need explicit WebSocket config (the `Upgrade`/`Connection` headers and a long `proxy_read_timeout`, §14.5). Symptom of missing config: the handshake returns 200/400 instead of 101, or idle connections die after ~60 s.

### 16.10 Test with `go test -race` **[A]**
The race detector catches accidental concurrent conn access during development. Run your two-pump model under `-race` with many simulated clients; a clean run is strong evidence your concurrency model is correct.

### 16.11 `WriteControl` for pings/closes when racing the writer **[A]**
If you must send a ping or close from outside the writePump (e.g. during shutdown), use `WriteControl` — it's the documented exception to the one-writer rule.

---

## 17. API Quick Reference

**Upgrader / handshake**
| Symbol | Purpose |
|---|---|
| `websocket.Upgrader{...}` | Holds handshake config; reuse one package-level value |
| `upgrader.Upgrade(w, r, hdr)` | Run the handshake; returns `*websocket.Conn` |
| `Upgrader.CheckOrigin` | CSWSH defense — allowlist origins |
| `Upgrader.ReadBufferSize` / `WriteBufferSize` | I/O buffer sizes (efficiency, not max msg) |
| `Upgrader.WriteBufferPool` | Shared write buffers — saves memory at scale |
| `Upgrader.Subprotocols` | Negotiated app subprotocols |

**Conn — read/write**
| Symbol | Purpose |
|---|---|
| `conn.ReadMessage()` | Read one full message (`type, data, err`) |
| `conn.WriteMessage(type, data)` | Write one full message |
| `conn.ReadJSON(&v)` / `WriteJSON(v)` | JSON over one message |
| `conn.NextReader()` / `NextWriter(type)` | Stream a message; `Close()` the writer to flush |
| `conn.WriteControl(type, data, deadline)` | Send ping/pong/close (safe alongside writePump) |

**Conn — health & lifecycle**
| Symbol | Purpose |
|---|---|
| `conn.SetReadLimit(n)` | Max inbound message bytes |
| `conn.SetReadDeadline(t)` / `SetWriteDeadline(t)` | Absolute deadlines (re-arm them) |
| `conn.SetPongHandler(fn)` | Extend read deadline on pong |
| `conn.SetPingHandler(fn)` / `SetCloseHandler(fn)` | Override defaults (rarely needed) |
| `conn.Close()` | Close the TCP socket |

**Client (dialing out)**
| Symbol | Purpose |
|---|---|
| `websocket.Dialer{...}` | Client-side handshake config (TLS, timeout, proxy) |
| `dialer.Dial(url, header)` | Connect; returns `conn, *http.Response, err` |

**Helpers / constants**
| Symbol | Purpose |
|---|---|
| `websocket.TextMessage` / `BinaryMessage` | App message opcodes |
| `websocket.Ping/Pong/CloseMessage` | Control opcodes |
| `websocket.FormatCloseMessage(code, text)` | Build a close-frame payload |
| `websocket.IsUnexpectedCloseError(err, codes...)` | True if close code not in expected set |
| `*websocket.CloseError` | Typed close error (`.Code`, `.Text`) — use `errors.As` |
| `websocket.CloseNormalClosure` (1000), `CloseGoingAway` (1001), `CloseMessageTooBig` (1009), `ClosePolicyViolation` (1008) | Common close codes |

---

## 18. Study Path & Build-to-Learn Projects

Work through these in order — each builds on the last. **Implement, don't just read.** Run everything under `go test -race` where applicable.

### Stage 1 — Foundation (Week 1) **[B]**
1. **Echo server (§3).** Upgrade, read, echo, close. Understand the upgrade lifecycle and the `101`. Test with `wscat`.
2. **Text & binary (§5).** Handle both message types; log the opcode.
3. **Ping/pong (§7).** Add `SetReadDeadline`, `SetPongHandler`, a periodic ping. `kill -9` the client and confirm the server reaps it within `pongWait`.
4. **Close codes (§8).** Send `CloseNormalClosure` after 10 s; verify the browser handles it.

### Stage 2 — The Concurrency Model (Week 2) **[I/A]**
5. **readPump + writePump (§6).** Refactor the echo server into the two-goroutine model with a `send` channel. Run 100 concurrent clients under `-race`; zero races.
6. **Hub (§9).** Build the broadcast Hub. Open 5 browser tabs; a message in one appears in all.
7. **Slow-client test (§10).** Buffer = 1, add a sleep in writePump, blast 100 messages; confirm the slow client is dropped without stalling the others.

### Stage 3 — Build Project 1: Chat Server (Week 3) **[I]**
8. **Rooms (§9.6).** `map[string]map[*Client]bool`; room-scoped broadcasts.
9. **JSON protocol.** A typed `Message` envelope (`type`, `room`, `user`, `content`, `time`).
10. **Join/leave events.** Broadcast system messages on connect/disconnect.
11. **Browser UI (§11).** Multi-room vanilla-JS client with reconnect + backoff (§11.4).

### Stage 4 — Build Project 2: Live Dashboard (Week 4) **[I/A]**
12. **Server-push only.** Publish CPU/mem/RPS every second; clients only receive (consider whether **SSE** would be simpler here — §1.3).
13. **Per-client subscriptions.** Clients send a `subscribe` list; the Hub filters per subscription.
14. **Conflation (§10.3).** Latest-wins so a slow client never lags behind on metrics.

### Stage 5 — Build Project 3: Notifications Service (Week 5) **[A]**
15. **User-targeted delivery.** Hub maps `userID → *Client`.
16. **REST trigger.** Internal `POST /notify {userID, message}` routes to the live client.
17. **Persistence & flush.** Store undelivered notifications (SQLite/Postgres — see `RELATIONAL_DB_DESIGN_GUIDE.md`/`SQLITE3_GUIDE.md`); flush on reconnect. This is how you get "eventual delivery" that WebSocket alone can't provide (§15.4).

### Stage 6 — Production Readiness (Week 6) **[A]**
18. **TLS (§14.5).** `ListenAndServeTLS` or `wss://` behind nginx with correct upgrade headers.
19. **Auth (§14.2).** Verify before upgrade; use short-lived tickets or first-message auth (`GO_JWT_ARGON2_GUIDE.md`).
20. **Rate limiting (§14.4).** Per-connection token bucket + per-IP connection limits.
21. **Graceful shutdown (§13).** On SIGTERM, send `CloseGoingAway`, drain pumps, `srv.Shutdown`.
22. **Metrics.** Prometheus: active connections, messages/sec, dropped clients, goroutine count.
23. **Redis fan-out (§15).** Run two instances + Redis Pub/Sub behind nginx; verify cross-instance rooms work (`REDIS_GUIDE.md`).

### Reference bookmarks
| Resource | URL |
|---|---|
| Gorilla WebSocket docs | https://pkg.go.dev/github.com/gorilla/websocket |
| Official examples (chat) | https://github.com/gorilla/websocket/tree/main/examples |
| WebSocket RFC 6455 | https://datatracker.ietf.org/doc/html/rfc6455 |
| Per-message DEFLATE RFC 7692 | https://datatracker.ietf.org/doc/html/rfc7692 |
| Go `net/http` docs | https://pkg.go.dev/net/http |
| `wscat` CLI | `npm i -g wscat` → `wscat -c ws://localhost:8080/ws` |

### Companion guides in this library
- **`GO_GUIDE.md`** — the Go language itself (concurrency, channels, `select`, `context`).
- **`GO_NET_HTTP_REST_API_GUIDE.md`** — servers, handlers, middleware, graceful shutdown (the HTTP foundation under every upgrade).
- **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** — running WebSockets inside the Gin router (§12).
- **`GO_JWT_ARGON2_GUIDE.md`** — token auth used to authenticate the handshake (§14.2).
- **`REDIS_GUIDE.md`** — the pub/sub backplane for horizontal scaling (§15).
- **`GO_GRPC_RPC_GUIDE.md`** — when you actually want streaming RPC instead of a raw socket.

---

*Guide accurate as of Go 1.25/1.26 and gorilla/websocket v1.5.x (2026). The Gorilla toolkit is actively maintained again. Always confirm exact API signatures at pkg.go.dev.*
