# Coder WebSocket (github.com/coder/websocket) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have used HTTP but never opened a WebSocket in my life" to "I can design, secure, authenticate, and horizontally scale a production, thread-safe, real-time Go backend." This is a **learn-offline study guide**: every concept is explained in *prose first* — what it is, the underlying logic and *why* it works this way, what it is for and *when* you reach for it, *how* to use it, the key parameters and options, best practices, and explicit **security recommendations** — and only *then* shown as heavily-commented, runnable Go code. Code alone is never enough; the point is that you understand the machinery, not that you copy snippets. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **`github.com/coder/websocket` v1.8.x** on **Go 1.25 / 1.26** (current in 2026). The production stack woven throughout is **Gin v1.10+**, **golang-jwt/jwt v5**, **`golang.org/x/crypto/argon2`** (Argon2id), **Ent v0.14+**, **pgx v5**, and **Air** for live reload. Where an API is fast-moving or version-sensitive it is flagged with **⚡**. The library was formerly published as **`nhooyr.io/websocket`** (by Anmol Sethi / "nhooyr") and in 2024 moved under Coder's stewardship as **`github.com/coder/websocket`**. The migration is a *pure import-path change* — the package name is still `websocket` and the API is identical — so almost every `nhooyr.io/websocket` example on the internet works verbatim once you swap the import. This is covered in [§2](#2-why-coderwebsocket-install--the-nhooyrcoder-history).
>
> **This guide's place in the library:** it assumes you know the Go *language* ([Go](GO_GUIDE.md)) and basic HTTP. A WebSocket *is* an HTTP request that gets upgraded, so [Go net/http REST](GO_NET_HTTP_REST_API_GUIDE.md) and [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) are direct companions. For the *other* Go WebSocket library and a head-to-head comparison see [Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md); for the Node.js equivalent see [Node WebSockets (50k+)](NODE_WEBSOCKETS_GUIDE.md). Connection auth builds on [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md); persistence uses [Go ent ORM](GO_ENT_ORM_GUIDE.md); multi-node scaling uses [Redis](REDIS_GUIDE.md); the reverse proxy is [Nginx](NGINX_GUIDE.md).

---

## Table of Contents

1. [WebSockets From Zero — The Problem & The Protocol](#1-websockets-from-zero--the-problem--the-protocol) **[B]**
2. [Why coder/websocket, Install & the nhooyr→coder History](#2-why-coderwebsocket-install--the-nhooyrcoder-history) **[B]**
3. [Server Basics — Accept, Echo, wsjson, Read/Write, Close](#3-server-basics--accept-echo-wsjson-readwrite-close) **[B]**
4. [Client Basics — Dial, Go Client, Browser JS, Reconnection](#4-client-basics--dial-go-client-browser-js-reconnection) **[B/I]**
5. [The Concurrency Model In Depth — The Headline Feature](#5-the-concurrency-model-in-depth--the-headline-feature) **[I/A]**
6. [The Production Hub & Client Pattern](#6-the-production-hub--client-pattern) **[I/A]**
7. [Integrating With Gin](#7-integrating-with-gin) **[I]**
8. [Authenticating the Upgrade — Banking-Grade](#8-authenticating-the-upgrade--banking-grade) **[A]**
9. [The Data Layer — pgx, Ent, Air & Persist-Then-Broadcast](#9-the-data-layer--pgx-ent-air--persist-then-broadcast) **[I/A]**
10. [Banking-Grade Security](#10-banking-grade-security) **[A]**
11. [Scaling Out — Redis Pub/Sub Backplane & Nginx](#11-scaling-out--redis-pubsub-backplane--nginx) **[A]**
12. [Complete Worked Example — Real-Time Notifications Service](#12-complete-worked-example--real-time-notifications-service) **[A]**
13. [Testing WebSocket Code](#13-testing-websocket-code) **[I]**
14. [Gotchas & Best Practices](#14-gotchas--best-practices) **[I/A]**
15. [API Quick Reference](#15-api-quick-reference) **[ref]**
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects) **[B→A]**

---

## 1. WebSockets From Zero — The Problem & The Protocol

### 1.1 The problem WebSockets solve **[B]**

Plain HTTP is a **request-response** protocol with one iron rule: *the client always speaks first.* The browser asks ("`GET /messages`"), the server answers, and the exchange is over. The server has no channel to say "hey, something just happened" on its own initiative. For a CRUD app — load a page, submit a form — that is perfect and you should not reach for anything fancier. But for anything *live* — a chat, a multiplayer game, a trading dashboard, a collaborative document, a notification bell, a live-ops admin panel watching bank transactions — the server continually needs to **push** data to a client that did not just ask for it. Vanilla HTTP cannot do that.

Before WebSockets, developers faked server-push with a ladder of increasingly painful workarounds. Knowing them is the *why* behind WebSockets:

- **Short polling** — the client asks "anything new?" every few seconds on a timer. Dead simple, but wasteful: most responses are "nothing new," and each request pays the full cost of HTTP (headers, routing, maybe a fresh TCP/TLS handshake). Latency is bounded by the interval — 1 s hammers the server, 30 s feels dead.
- **Long polling** — the client makes a request and the server *holds it open* until it has something to say, then responds; the client immediately reconnects. Lower latency and less waste, but you still pay a full HTTP round-trip per message and you tie up a request slot per waiting client. A clever hack, not a real duplex channel.
- **Server-Sent Events (SSE)** — a standardized *one-way* stream from server to client over one long-lived HTTP response (`Content-Type: text/event-stream`). Genuinely good when data only flows **server → client** (live scores, a notification feed, log tailing): simpler than WebSockets, auto-reconnects, rides normal HTTP/2. But it is unidirectional; the client still opens separate HTTP requests to send anything back, and browsers cap concurrent SSE connections per host over HTTP/1.1.

A **WebSocket** is the real answer: a **persistent, full-duplex** connection. After a one-time handshake, a single TCP connection stays open and *both sides can send messages at any moment*, independently, with tiny per-message overhead (roughly 2–14 bytes of framing versus the hundreds of bytes an HTTP request burns on headers *every single time*).

### 1.2 HTTP vs SSE vs WebSocket vs gRPC-streaming at a glance **[B]**

| Property | HTTP poll/long-poll | Server-Sent Events | **WebSocket** | gRPC streaming |
|---|---|---|---|---|
| Connection lifetime | New per request (or held) | One long-lived response | **Persistent until closed** | Persistent (HTTP/2 stream) |
| Direction | Client → Server | Server → Client only | **Full-duplex** | Uni- or bi-directional |
| Who initiates a message | Client only | Server only | **Either side, anytime** | Either side |
| Per-message overhead | High (full headers) | Low after first | **Very low (frame header)** | Low (HTTP/2 framing) |
| Payload | Text/JSON/binary | Text only (UTF-8) | **Text *and* binary** | Protobuf (binary) |
| Browser support | Native `fetch` | Native `EventSource` | **Native `WebSocket`** | Needs grpc-web + proxy |
| Auto-reconnect | No | **Yes (built in)** | No — you implement it | No |
| Best for | CRUD, occasional fetch | One-way live feeds | **Interactive real-time** | Service-to-service, typed streams |

**Decision rule.** If data flows only server→client and the client is a browser, prefer **SSE** — it is simpler, reconnects itself, and is easier to scale. If you need *bidirectional*, low-latency, ordered messaging (chat, presence, live collaboration, games, order books), use **WebSocket**. If you are doing *service-to-service* streaming with strong typing across your backend, use **gRPC streaming** ([Go gRPC](GO_GRPC_RPC_GUIDE.md)). Do not reach for WebSockets just because they are cool; they are a stateful, long-lived resource that complicates load balancing, deploys, and scaling — as the rest of this guide will make very concrete.

### 1.3 The handshake — how an HTTP request becomes a WebSocket **[B]**

A WebSocket connection is *born as HTTP* and then **upgraded**. This is deliberate: it lets WebSockets travel over ports 80/443, pass through existing proxies and firewalls, and reuse HTTP authentication and routing. The client sends an ordinary `GET` with special headers:

```http
GET /ws HTTP/1.1
Host: api.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://app.example.com
Sec-WebSocket-Protocol: json.v1, auth.jwt.<token>
```

- `Upgrade: websocket` + `Connection: Upgrade` — "please switch protocols on this TCP socket."
- `Sec-WebSocket-Key` — a random 16-byte base64 nonce. **Not security** — it just proves the peer speaks WebSocket and defeats caching proxies from replaying a cached 101.
- `Sec-WebSocket-Version: 13` — the only version in real use (RFC 6455).
- `Origin` — set *by the browser* to the page's origin. The server uses it to reject cross-site connections (see CSWSH in [§10](#10-banking-grade-security)). Non-browser clients can forge it, so Origin is a defense against *browser*-driven attacks, not a general authenticator.
- `Sec-WebSocket-Protocol` — a client-proposed list of **subprotocols**; the server picks at most one and echoes it. We will exploit this header to smuggle a JWT past the browser's inability to set custom headers ([§8](#8-authenticating-the-upgrade--banking-grade)).

The server, if it agrees, replies with the magic **101 Switching Protocols**:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: json.v1
```

`Sec-WebSocket-Accept` is `base64(SHA1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`. That constant GUID is fixed in the RFC. Once the 101 is sent, the TCP socket is **no longer HTTP** — it is a raw bidirectional WebSocket frame stream. coder/websocket's `Accept` does all of this for you and hands back a `*websocket.Conn`.

### 1.4 Frames, opcodes, masking, text vs binary **[I]**

After the handshake, data travels in **frames**. A frame has a small binary header then a payload. The header encodes:

- **FIN bit** — is this the final frame of a message? A single logical message may be split across multiple frames (fragmentation); FIN=1 marks the last.
- **Opcode** (4 bits) — the frame's kind:
  - `0x1` **text** — payload is UTF-8. `MessageText` in the library.
  - `0x2` **binary** — arbitrary bytes. `MessageBinary`.
  - `0x0` continuation — a follow-on fragment of a text/binary message.
  - `0x8` **close**, `0x9` **ping**, `0xA` **pong** — *control frames*. These may be interleaved between fragments and carry ≤125 bytes.
- **MASK bit + masking-key** — **every client→server frame MUST be masked** with a 4-byte key XORed over the payload; server→client frames MUST NOT be masked. Masking exists to stop malicious JS from crafting bytes that confuse intermediary proxies into cache-poisoning attacks. The library handles masking/unmasking for you; you never touch it, but you should know it is why a client cannot "just send raw bytes."
- **Payload length** — 7 bits, or 7+16, or 7+64 bits, so a single frame can carry up to 2^63 bytes. This is exactly why `SetReadLimit` matters: without a cap, a hostile peer announces an enormous length and forces you to allocate ([§3.6](#36-setreadlimit--dont-let-a-peer-ram-your-heap-b), [§10](#10-banking-grade-security)).

**Text vs binary — which to send?** Use **text** (usually JSON) for structured application messages; it is human-debuggable, plays nicely with `wsjson`, and is what browsers hand you as a string. Use **binary** for already-encoded payloads (protobuf, MessagePack, CBOR, media chunks, file transfer) where you want density and zero double-encoding. The frame opcode is the *only* difference at the wire level; the library keeps them straight so a `MessageText` you send arrives as `MessageText`.

> **⚡ Version note:** coder/websocket implements **RFC 6455** framing and the **RFC 7692** `permessage-deflate` compression extension, and passes the **Autobahn** conformance test suite (the industry reference for WebSocket correctness). You do not implement any of §1.3–1.4 yourself — but every design decision later in this guide (read limits, close codes, ping/pong, compression) maps directly onto these wire concepts.

---

## 2. Why coder/websocket, Install & the nhooyr→coder History

### 2.1 What this library is and the design philosophy **[B]**

`github.com/coder/websocket` is a **minimal, idiomatic, context-first, zero-dependency** WebSocket implementation for Go. "Zero-dependency" is literal: it imports only the standard library, so it never drags transitive packages into your `go.mod` — a real supply-chain and audit advantage for a banking backend. "Context-first" is the headline ergonomic difference from older libraries: **every blocking method takes a `context.Context`**, so timeouts and cancellation compose with the rest of your Go code instead of the awkward `SetReadDeadline(time.Now().Add(...))` dance. And, crucially for the concurrent, thread-safe backend this guide is about, its `Conn` is **safe to `Write` to from multiple goroutines** — the library serializes writes internally with a context-aware mutex ([§5](#5-the-concurrency-model-in-depth--the-headline-feature)).

The API surface is deliberately tiny — a handful of functions and about a dozen `Conn` methods — which makes it fast to learn and hard to misuse. It ships a companion package `wsjson` for the ubiquitous "send/receive a JSON value" case, a `NetConn` adapter that turns a `Conn` into a `net.Conn` (so you can layer TLS, `bufio`, or any `io`-based protocol on top), and **WebAssembly** support so the *same* client code compiles to run in the browser.

### 2.2 coder/websocket vs gorilla/websocket **[B]**

The two mainstream choices are this library and [Gorilla](GO_GORILLA_WEBSOCKETS_GUIDE.md). They make different tradeoffs:

| Dimension | **coder/websocket** | gorilla/websocket |
|---|---|---|
| API style | **Context-first** (`Read(ctx)`, `Write(ctx,…)`) | Deadline-based (`SetReadDeadline`) |
| Dependencies | **Zero** (stdlib only) | Zero (stdlib only) |
| Concurrent `Write` | **Safe** — internally synchronized | **Unsafe** — you must add a mutex / single writer |
| JSON helper | Built-in `wsjson` | `conn.ReadJSON`/`WriteJSON` |
| Compression | `permessage-deflate` (RFC 7692) | `permessage-deflate` |
| Close/status API | Typed `StatusCode`, `CloseStatus(err)` | `CloseError`, `IsCloseError`, `CloseNormalClosure` |
| `net.Conn` adapter | **`NetConn`** built in | Via `UnderlyingConn` (lower level) |
| WebAssembly client | **Yes** (same API in browser) | No |
| Autobahn compliance | **Yes** | Yes |
| Age / ecosystem | Newer, modern-Go idioms | Older, huge install base, tons of examples |

The single most consequential row is **concurrent `Write`**. With Gorilla, two goroutines calling `WriteMessage` on the same connection *corrupt the frame stream* — a whole class of production bugs — so the canonical Gorilla pattern *forces* a single dedicated writer goroutine plus a mutex. coder/websocket removes that footgun: concurrent `Write` is safe. **But — and this is a point junior engineers routinely get wrong — you still build a single-writer-goroutine + bounded send-channel design in production.** Not for safety anymore, but for **backpressure, ordering, and slow-client eviction** ([§5.4](#54-so-why-still-use-one-writer-goroutine-a), [§6](#6-the-production-hub--client-pattern)). Safety and architecture are different problems; the library fixes the first, you still own the second.

### 2.3 The nhooyr.io → coder rename and migration **[B]**

This library began as **`nhooyr.io/websocket`**, written by Anmol Sethi. In 2024 maintenance moved to **Coder** (the company behind the `coder` remote-dev platform) and the canonical import path became **`github.com/coder/websocket`**. Nothing else changed: the Go *package name* is still `websocket`, and every type, function, and method signature is identical. So migrating an existing codebase is a mechanical find-and-replace:

```bash
# Migrate an existing project from the old path to the new one.
grep -rl 'nhooyr.io/websocket' . | xargs sed -i 's#nhooyr.io/websocket#github.com/coder/websocket#g'
go mod tidy
```

Practically, this means **every `nhooyr.io/websocket` tutorial, Stack Overflow answer, and blog post you find is valid** — just swap the import line. When you read older material, mentally translate `nhooyr.io/websocket` → `github.com/coder/websocket` and carry on.

### 2.4 Installing **[B]**

```bash
# Inside a Go module (go 1.25+). If you don't have one yet:
go mod init github.com/acme/realtime

# Add the library and its JSON helper (same module, single go get).
go get github.com/coder/websocket@latest   # pulls v1.8.x as of 2026

# The rest of the production stack used later in this guide:
go get github.com/gin-gonic/gin@latest
go get github.com/golang-jwt/jwt/v5@latest
go get golang.org/x/crypto/argon2
go get entgo.io/ent@latest
go get github.com/jackc/pgx/v5@latest
go get github.com/redis/go-redis/v9@latest
```

Imports you will use constantly:

```go
import (
	"github.com/coder/websocket"        // core: Accept, Dial, Conn, StatusCode, ...
	"github.com/coder/websocket/wsjson" // JSON convenience: wsjson.Read / wsjson.Write
)
```

> **⚡ Version note:** Pin the version in `go.mod` (`github.com/coder/websocket v1.8.x`) and let Renovate/Dependabot bump it. v1.x is API-stable; the `v1.8` line is what you target on Go 1.25/1.26. Because the library has **no dependencies**, a `go get -u` on it can never cascade into a hundred transitive upgrades — a genuine operational nicety.

---

## 3. Server Basics — Accept, Echo, wsjson, Read/Write, Close

### 3.1 `Accept` — turning the request into a Conn **[B]**

On the server, one function does the handshake:

```go
func Accept(w http.ResponseWriter, r *http.Request, opts *AcceptOptions) (*Conn, error)
```

You call it from an ordinary `http.HandlerFunc`. It reads the upgrade headers off `r`, writes the `101 Switching Protocols` to `w`, hijacks the TCP connection, and returns a `*websocket.Conn`. After a successful `Accept` you must **not** touch `w` or `r` for HTTP anymore — the socket belongs to the WebSocket now.

Two things about `Accept` you must internalize on day one:

1. **It blocks cross-origin requests by default.** If the `Origin` header's host differs from the request host and you did not allow it via `OriginPatterns`, `Accept` returns an error and refuses the upgrade. This is the built-in **CSWSH** defense ([§10.2](#102-origin-checking--cswsh-a)). During local development where your frontend runs on a *different* port than your API, you *will* hit this, and the fix is `OriginPatterns`, **not** `InsecureSkipVerify`.
2. **On error, respond and return.** `Accept` writes an HTTP error response itself when it fails, so your handler should just log and return.

### 3.2 A minimal echo server **[B]**

The "hello world" of WebSockets: read a message, write it back. Every line here matters, so read the comments.

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"time"

	"github.com/coder/websocket"
)

func echoHandler(w http.ResponseWriter, r *http.Request) {
	// Accept performs the HTTP->WS upgrade. OriginPatterns whitelists the
	// browser origins allowed to connect (CSWSH defense). If Origin's host
	// equals the request host it is always allowed.
	c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
		OriginPatterns: []string{"localhost:5173", "app.example.com"},
	})
	if err != nil {
		log.Printf("accept: %v", err) // Accept already wrote an HTTP error.
		return
	}
	// CloseNow is the "always run, never blocks" cleanup. Even if we return
	// via an error path, this guarantees the socket and goroutines are freed.
	// A *graceful* Close (with a code+reason) is done on the happy path below;
	// this deferred CloseNow is the safety net for panics/early returns.
	defer c.CloseNow()

	// Cap the largest message we will read into memory. Without this a hostile
	// client can announce a huge frame and OOM us. Default is 32 KiB.
	c.SetReadLimit(32 * 1024)

	for {
		// Give each read a deadline via context. If the client goes silent for
		// 60s we cancel the read, break the loop, and free the connection.
		ctx, cancel := context.WithTimeout(r.Context(), 60*time.Second)

		typ, data, err := c.Read(ctx)
		cancel() // Always cancel to release the timer, even on success.
		if err != nil {
			// Distinguish a clean client close from a real error.
			if websocket.CloseStatus(err) == websocket.StatusNormalClosure ||
				websocket.CloseStatus(err) == websocket.StatusGoingAway {
				log.Println("client closed normally")
			} else if errors.Is(err, context.DeadlineExceeded) {
				log.Println("client idle timeout")
			} else {
				log.Printf("read: %v", err)
			}
			return
		}

		// Echo the exact bytes back with the same message type.
		wctx, wcancel := context.WithTimeout(r.Context(), 10*time.Second)
		err = c.Write(wctx, typ, data)
		wcancel()
		if err != nil {
			log.Printf("write: %v", err)
			return
		}
	}
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/ws", echoHandler)
	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

There is already a lot of production DNA here: an origin whitelist, a read limit, per-read timeouts via context, and correct close-status handling. Hold onto the shape — the whole guide elaborates it.

### 3.3 `Read`, `Reader`, `Write`, `Writer` **[B]**

Four methods move data. Two "whole message" convenience methods and two streaming `io` methods:

- `Read(ctx) (MessageType, []byte, error)` — reads *one complete message* into a byte slice. Simple; allocates the whole message. Bounded by `SetReadLimit`.
- `Reader(ctx) (MessageType, io.Reader, error)` — returns an `io.Reader` for the next message so you can stream it (e.g. `io.Copy` a large upload to disk) without buffering it all. You **must** drain the reader before calling `Reader`/`Read` again — the one-reader rule ([§5](#5-the-concurrency-model-in-depth--the-headline-feature)).
- `Write(ctx, MessageType, []byte) error` — writes *one complete message* from a byte slice. **Safe to call concurrently** from multiple goroutines.
- `Writer(ctx, MessageType) (io.WriteCloser, error)` — returns an `io.WriteCloser` to stream a message out in chunks; `Close()` the writer to finalize the frame. Only one writer may be open at a time.

Rule of thumb: use `Read`/`Write` for normal JSON-sized app messages (kilobytes); use `Reader`/`Writer` when a message is large enough that you would not want it fully in RAM (media, file transfer).

### 3.4 `wsjson` — the 90% case **[B]**

Almost every real WebSocket app sends JSON envelopes. `wsjson` marshals/unmarshals for you against a `MessageText` frame:

```go
func Read(ctx context.Context, c *websocket.Conn, v any) error  // decode next msg into v
func Write(ctx context.Context, c *websocket.Conn, v any) error // encode v and send
```

```go
type Envelope struct {
	Type string          `json:"type"` // "chat", "presence", "ping", ...
	Data json.RawMessage `json:"data"` // deferred-decoded payload
}

// Read a typed message:
var env Envelope
if err := wsjson.Read(ctx, c, &env); err != nil {
	// EOF/close vs decode error handled by caller
	return err
}

// Write a typed message:
_ = wsjson.Write(ctx, c, Envelope{Type: "ack", Data: json.RawMessage(`{"ok":true}`)})
```

`wsjson.Read` respects `SetReadLimit` (it reads a whole message, then decodes), so malformed or oversized JSON cannot blow your heap. Design your protocol around a small **envelope** with a `type` discriminator plus a `data` payload; it makes routing, versioning, and per-message authorization ([§8.5](#85-per-message-authorization-a)) clean.

### 3.5 The close handshake, `Close`, `CloseNow`, `CloseStatus` **[B]**

Closing a WebSocket is *not* just dropping the TCP socket — RFC 6455 defines a **close handshake**: one side sends a close frame carrying a **status code** and optional UTF-8 reason, the peer echoes a close frame, then the TCP connection tears down. Doing it properly lets the other side learn *why* you left.

- `Close(code StatusCode, reason string) error` — the **graceful** close. Sends a close frame with your code/reason and waits (bounded internally) for the peer's echo. Use it on the happy path. `reason` must be ≤123 bytes (it lives in a control frame).
- `CloseNow() error` — the **immediate** close. Skips the handshake and drops the socket now. Use it as a `defer` safety net and when you have already given up on the peer (e.g. after eviction). It never blocks, which is why it is the ideal deferred cleanup.
- `CloseStatus(err error) StatusCode` — inspect a `Read`/`Write` error to find *the peer's* close code. Returns **-1** if the error was not a `CloseError` at all (e.g. a network reset). This is how you tell "client said goodbye normally" from "the wire exploded."

```go
// Happy-path graceful close after finishing work:
c.Close(websocket.StatusNormalClosure, "bye")

// Detect how the peer left when Read returns an error:
switch websocket.CloseStatus(err) {
case websocket.StatusNormalClosure, websocket.StatusGoingAway:
	// expected — browser tab closed, navigation, etc.
case -1:
	// not a close frame at all: TCP reset, timeout, read-limit tripped, ...
default:
	log.Printf("peer closed with code %d", websocket.CloseStatus(err))
}
```

The standard pattern is **both**: `defer c.CloseNow()` at the top as the guaranteed-cleanup net, and an explicit `c.Close(StatusNormalClosure, …)` at the natural end of the happy path. Calling `CloseNow` after a successful `Close` is harmless.

### 3.6 `SetReadLimit` — don't let a peer ram your heap **[B]**

`SetReadLimit(n int64)` caps the byte size of a single message the connection will read; exceeding it fails the read with a close of `StatusMessageTooBig` (1009). **The default is 32768 bytes (32 KiB).** That default is fine for chat and control traffic and *actively protects you* — but if your protocol legitimately sends larger messages you must raise it, and if you accept untrusted input you should set it as low as your app allows. Treat it as a required security control, not an optional tuning knob ([§10.4](#104-input-validation--read-limits-a)).

```go
c.SetReadLimit(64 * 1024) // allow up to 64 KiB messages; reject anything bigger
```

### 3.7 Status codes reference **[I]**

Every close carries a `StatusCode`. Know the common ones — you *send* them to tell clients why, and you *read* them to react:

| Code | Constant | Meaning / when you send it |
|---|---|---|
| 1000 | `StatusNormalClosure` | Clean, intended shutdown |
| 1001 | `StatusGoingAway` | Server shutting down / browser navigating away |
| 1002 | `StatusProtocolError` | Peer violated the protocol |
| 1003 | `StatusUnsupportedData` | Got binary when expecting text (or vice-versa) |
| 1005 | `StatusNoStatusRcvd` | *Reserved* — no code was present (never send) |
| 1006 | `StatusAbnormalClosure` | *Reserved* — connection dropped without a close frame (never send) |
| 1007 | `StatusInvalidFramePayloadData` | Message wasn't valid UTF-8 / bad payload |
| 1008 | `StatusPolicyViolation` | Generic "you broke a rule" (auth fail, rate limit) |
| 1009 | `StatusMessageTooBig` | Message exceeded `SetReadLimit` |
| 1010 | `StatusMandatoryExtension` | Client needed an extension server won't give |
| 1011 | `StatusInternalError` | Server hit an unexpected condition |
| 1012 | `StatusServiceRestart` | Server restarting — reconnect later |
| 1013 | `StatusTryAgainLater` | Overloaded — back off and retry (slow-client eviction) |
| 1014 | `StatusBadGateway` | Upstream/gateway failure |
| 1015 | `StatusTLSHandshake` | *Reserved* — TLS failure (never send) |

Codes **3000–3999** are reserved for frameworks/libraries (register-able), and **4000–4999** are yours to use privately for application-specific reasons (e.g. `4001 token expired`, `4002 kicked by admin`). Use the private range for anything your own client should react to specifically.

---

## 4. Client Basics — Dial, Go Client, Browser JS, Reconnection

### 4.1 `Dial` — the Go client **[B]**

On the client side, one function opens a connection:

```go
func Dial(ctx context.Context, u string, opts *DialOptions) (*Conn, *http.Response, error)
```

The URL scheme is written as `ws://` or `wss://` — but note the library also accepts `http`/`https` and interprets them as `ws`/`wss`. The `ctx` bounds the *handshake* (not the lifetime of the connection); pass a `context.WithTimeout` so a dead server does not hang your dialer. The returned `*http.Response` is the handshake response — useful to inspect the negotiated subprotocol or a rejection body.

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

c, resp, err := websocket.Dial(ctx, "wss://api.example.com/ws", &websocket.DialOptions{
	Subprotocols: []string{"json.v1"},
	HTTPHeader:   http.Header{"Authorization": {"Bearer " + token}}, // Go clients CAN set headers
})
if err != nil {
	// resp may carry the rejection status/body when the server refused the upgrade.
	log.Fatalf("dial: %v", err)
}
defer c.CloseNow()
_ = resp

// Send and receive a JSON message:
_ = wsjson.Write(ctx, c, map[string]any{"type": "hello"})
var reply Envelope
_ = wsjson.Read(context.Background(), c, &reply)
```

### 4.2 `DialOptions` and the browser header limitation **[B/I]**

```go
type DialOptions struct {
	HTTPClient     *http.Client   // custom client: proxies, TLS config, timeouts
	HTTPHeader     http.Header    // extra handshake headers (Go clients only)
	Host           string         // override the Host header
	Subprotocols   []string       // proposed Sec-WebSocket-Protocol values
	CompressionMode CompressionMode
	CompressionThreshold int
	OnPingReceived func(ctx context.Context, payload []byte) bool
	OnPongReceived func(ctx context.Context, payload []byte)
}
```

The single most important fact for authentication: **a browser's `WebSocket` constructor cannot set custom request headers.** There is no `Authorization` header option in the browser API — full stop. A *Go* client can set `HTTPHeader` (as above), but browser JS cannot. This forces one of two auth patterns for browser clients — **subprotocol-smuggled token** or **short-lived query-param ticket** — both covered in depth in [§8](#8-authenticating-the-upgrade--banking-grade). Internalize this now: it shapes your entire auth design.

### 4.3 The browser JavaScript client **[B]**

In the browser you use the native `WebSocket` API. It takes a URL and an optional subprotocol (string or array). This is the *only* place you can influence the handshake from a browser besides the URL itself:

```javascript
// The 2nd argument becomes Sec-WebSocket-Protocol. We use it to carry the JWT
// (see §8) since the browser cannot set an Authorization header.
const ws = new WebSocket("wss://api.example.com/ws", ["json.v1", `auth.jwt.${token}`]);

ws.binaryType = "arraybuffer"; // how binary frames surface: "blob" (default) or "arraybuffer"

ws.addEventListener("open", () => {
	ws.send(JSON.stringify({ type: "hello" }));
});

ws.addEventListener("message", (ev) => {
	// ev.data is a string for text frames, ArrayBuffer/Blob for binary.
	const msg = JSON.parse(ev.data);
	console.log("recv", msg);
});

ws.addEventListener("close", (ev) => {
	console.log("closed", ev.code, ev.reason); // ev.code is the StatusCode from the server
});

ws.addEventListener("error", () => {
	// Fires before close on failure; details are deliberately vague for security.
});

// Close gracefully with a code + reason (reason <= 123 bytes):
// ws.close(1000, "bye");
```

Two browser quirks worth knowing: (1) the `error` event never tells you *why* (browsers hide it to prevent probing), so rely on the `close` event's `code`; (2) `send()` on a not-yet-open socket throws, so gate sends on the `open` event or check `ws.readyState === WebSocket.OPEN`.

### 4.4 Reconnection with exponential backoff and jitter **[I]**

Neither the browser `WebSocket` nor coder/websocket reconnects automatically — you own it. A production client wraps the socket in a reconnection loop with **exponential backoff plus jitter** (randomization) so that when a server restarts, thousands of clients do not reconnect in a synchronized "thundering herd" that knocks it over again. On each successful open you reset the backoff; on close you back off, capped at a ceiling.

```javascript
// Browser: resilient reconnecting client with capped backoff + jitter.
function connect(url, tokenProvider) {
	let attempt = 0;
	let ws;

	const open = async () => {
		const token = await tokenProvider(); // fetch a *fresh*, short-lived token each time
		ws = new WebSocket(url, ["json.v1", `auth.jwt.${token}`]);

		ws.addEventListener("open", () => { attempt = 0; /* reset backoff */ });

		ws.addEventListener("close", (ev) => {
			// 4001 = our private "token expired" — reconnect will fetch a new one.
			// 1008 = policy violation (auth/rate) — maybe stop & surface an error.
			const base = Math.min(30000, 1000 * 2 ** attempt); // cap at 30s
			const jitter = Math.random() * base;               // full jitter
			attempt++;
			setTimeout(open, base / 2 + jitter);
		});
	};

	open();
	return () => ws && ws.close(1000, "client shutdown");
}
```

The equivalent Go client loop uses the same math around `websocket.Dial`, re-issuing a token each attempt. **Always fetch a fresh token per reconnect** — connections are long-lived and tokens are short-lived, so caching the first token guarantees failures after it expires ([§8.6](#86-token-expiry-on-a-long-lived-connection-a)).

---

## 5. The Concurrency Model In Depth — The Headline Feature

This is the section that most differentiates coder/websocket and directly answers the "thread-safe concurrent" requirement. Read it slowly; getting the mental model right prevents an entire category of production bugs.

### 5.1 The rules, stated precisely **[I]**

From the library's contract, for a single `*Conn`:

1. **All methods are safe to call concurrently — EXCEPT `Read`/`Reader`.** You may call `Write`, `Ping`, `Close`, `CloseNow`, etc. from many goroutines at once.
2. **Only one reader at a time.** At most one `Read`/`Reader` may be in flight. If you have a `Reader`'s `io.Reader` open, you must fully drain (or the message ends) before reading the next message. In practice this means **exactly one goroutine ever reads** from a connection.
3. **Only one writer open at a time — but `Write` is internally synchronized.** You may not have two `Writer()` `io.WriteCloser`s open simultaneously, but you *can* call the whole-message `Write(ctx, …)` from multiple goroutines concurrently: the library serializes them with a **context-aware mutex**, so frames never interleave or corrupt. If two goroutines `Write` at once, one proceeds and the other blocks on the mutex until the first frame completes (or its context is cancelled).
4. **`Ping` does not read its own pong.** `Ping(ctx)` sends a ping and waits for the matching pong — but *the pong arrives via the read path*, which `Ping` does not drive. Therefore **`Ping` only completes if some goroutine is concurrently reading** (your read pump, or `CloseRead`). Pinging on a connection nobody is reading will hang until the context expires.

### 5.2 Why one reader **[I]**

A WebSocket message can be fragmented across frames, and control frames (ping/pong/close) can be interleaved. Reassembling that stream is inherently *stateful and sequential* — there is exactly one byte stream and one "current message." Two concurrent readers would race over that shared parse state, so the library forbids it by contract. The clean consequence: dedicate **one goroutine** to reading (the "read pump"). Everything the connection receives flows through that single goroutine, which then dispatches to your app.

### 5.3 Why concurrent Write is safe (and what that buys you) **[I]**

Older libraries treat the write side like the read side: you must serialize it yourself, and forgetting a lock silently corrupts frames. coder/websocket instead puts a mutex *inside* `Write`. The payoff: a broadcast function can loop over connections and call `c.Write(...)` directly; a request handler on another goroutine can also `c.Write(...)` to the same connection; neither corrupts the other. This removes the "I forgot the write lock" bug entirely.

```go
// This is SAFE with coder/websocket — the library serializes the two writes.
// (It would corrupt frames with gorilla/websocket.)
go func() { c.Write(ctx, websocket.MessageText, []byte(`{"type":"a"}`)) }()
go func() { c.Write(ctx, websocket.MessageText, []byte(`{"type":"b"}`)) }()
```

### 5.4 So why still use one writer goroutine? **[A]**

Because **safety is not the same as good architecture.** The library guarantees frames won't corrupt, but it does *not* solve three problems you absolutely have in production:

- **Backpressure.** If a client's TCP send buffer is full (slow network, dead-but-not-yet-timed-out peer), a direct `c.Write` **blocks** the calling goroutine until the write drains or its context expires. If that caller is your *broadcast loop*, one slow client stalls delivery to everyone. You need a way to *decouple* "produce a message" from "actually write it," and to *give up* on a client that cannot keep up.
- **Ordering & fairness under load.** Even though writes are serialized, the *order* in which competing goroutines win the mutex is not something you control. Routing all outbound traffic for a connection through a single **bounded channel** drained by one goroutine gives you a clean FIFO per client.
- **Slow-consumer eviction.** With a bounded `send chan []byte`, a non-blocking send (`select { case send <- msg: default: evict }`) lets you *detect* that a client is not draining fast enough and drop them deterministically, instead of letting them consume memory or block producers.

So the production pattern is: **one read-pump goroutine, one write-pump goroutine per connection, and a bounded outbound channel between your app and the write pump.** The write pump is also the natural home for the periodic `Ping` keepalive. This is exactly [§6](#6-the-production-hub--client-pattern).

### 5.5 `CloseRead` — for write-only connections **[I]**

Sometimes a connection is **write-only from your side**: a server that only *pushes* notifications and never expects the client to send app messages. You still must read, though — because control frames (ping, pong, and the peer's *close*) arrive on the read path, and the read limit is only enforced while reading. If nobody reads, you never notice the client left, `Ping` can't get its pong, and buffers back up.

`CloseRead(ctx) context.Context` solves this: it spins up a background goroutine that **reads and discards** all incoming messages (failing the connection if a data message exceeds the read limit) purely so control frames are processed. It returns a context that is cancelled when the connection dies — handy to select on. After `CloseRead`, you must not call `Read` yourself (the helper owns the reader).

```go
// A push-only connection: we never expect app messages from the client.
ctx := c.CloseRead(r.Context()) // background reader drains control frames + close

// Now safely push and ping; Ping's pong is consumed by the CloseRead goroutine.
for {
	select {
	case <-ctx.Done():
		return // client went away
	case msg := <-outbound:
		if err := c.Write(ctx, websocket.MessageText, msg); err != nil {
			return
		}
	case <-ticker.C:
		if err := c.Ping(ctx); err != nil { // works because CloseRead is reading
			return
		}
	}
}
```

### 5.6 Ping/pong keepalive and dead-connection detection **[I]**

TCP does not reliably tell you a peer vanished (pulled cable, crashed, NAT timeout). The WebSocket answer is application-level heartbeats: periodically `Ping`; if the pong does not come back within your timeout, treat the connection as dead and close it. Because `Ping` blocks until the pong (or ctx cancel), a *timed* `Ping` doubles as your liveness probe:

```go
ctx, cancel := context.WithTimeout(baseCtx, 10*time.Second)
err := c.Ping(ctx) // returns when pong arrives, or ctx times out (=> peer is dead)
cancel()
if err != nil {
	c.Close(websocket.StatusPolicyViolation, "ping timeout")
	return
}
```

Remember §5.1 rule 4: this only works if a reader is active (your read pump or `CloseRead`). You can also hook `OnPongReceived` / `OnPingReceived` in the options if you want to observe or customize heartbeat handling; returning `false` from `OnPingReceived` suppresses the library's automatic pong so you can send your own.

---

## 6. The Production Hub & Client Pattern

Now we assemble §5's rules into the canonical real-time architecture: a **Hub** that owns the set of connected **Clients**, with per-client read and write pumps and bounded channels. This is the beating heart of chat servers, presence systems, live dashboards, and notification fan-out.

### 6.1 The mental model **[I]**

Picture a telephone switchboard. Each **Client** is one caller's line. Each line has:

- a **read pump** — one goroutine that does nothing but `Read` from the socket and hand messages to the Hub (§5.2: one reader);
- a **write pump** — one goroutine that drains a **bounded `send` channel** and writes to the socket, and also fires periodic pings (§5.4);
- a bounded outbound buffer (`send chan []byte`) so producers never block on a slow line.

The **Hub** is the switchboard operator: it keeps the registry of clients (and which rooms/topics each belongs to) and routes messages — broadcast to all, or targeted to some. Two Hub designs exist; we teach both.

### 6.2 Design A — channel-serialized Hub (actor style) **[I/A]**

The Hub owns its state and is driven by a single `run()` goroutine reading from `register`, `unregister`, and `broadcast` channels. Because only that one goroutine touches the `clients` map, **no mutex is needed** — the channel *is* the synchronization. This is the idiomatic-Go "share memory by communicating" style; it is easy to reason about and impossible to race, at the cost of all hub operations funnelling through one goroutine (rarely a bottleneck until very high fan-out).

```go
// internal/hub/hub.go  (Client lives in internal/hub/client.go; shown together here)
package hub

import (
	"context"
	"log/slog"
	"time"

	"github.com/coder/websocket"
)

// Client is one WebSocket connection plus its identity and outbound buffer.
type Client struct {
	hub    *Hub
	conn   *websocket.Conn
	send   chan []byte      // bounded outbound queue; the write pump drains it
	userID string           // authenticated identity (set at upgrade, see §8)
	rooms  map[string]bool  // topic/room membership
}

// Hub is the actor that owns all client state. Only run() touches `clients`.
type Hub struct {
	clients    map[*Client]bool
	register   chan *Client
	unregister chan *Client
	broadcast  chan []byte
	log        *slog.Logger
}

func NewHub(log *slog.Logger) *Hub {
	return &Hub{
		clients:    make(map[*Client]bool),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan []byte, 256), // buffered so publishers rarely block
		log:        log,
	}
}

// Run is the single goroutine that mutates hub state. Start it once at boot.
func (h *Hub) Run(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			// Graceful shutdown: close every client politely (§6.6).
			for c := range h.clients {
				c.conn.Close(websocket.StatusGoingAway, "server shutting down")
				delete(h.clients, c)
			}
			return
		case c := <-h.register:
			h.clients[c] = true
			h.log.Info("client registered", "user", c.userID, "total", len(h.clients))
		case c := <-h.unregister:
			if _, ok := h.clients[c]; ok {
				delete(h.clients, c)
				close(c.send) // signal the write pump to exit
			}
		case msg := <-h.broadcast:
			for c := range h.clients {
				// NON-blocking send: if a client's buffer is full it is a slow
				// consumer. We evict it rather than let it stall the whole hub.
				select {
				case c.send <- msg:
				default:
					h.log.Warn("evicting slow client", "user", c.userID)
					delete(h.clients, c)
					close(c.send)
				}
			}
		}
	}
}
```

The **non-blocking `select`/`default`** on `c.send` is the linchpin of backpressure: a client that cannot keep up gets dropped instead of freezing everyone. That is slow-consumer eviction, done deterministically.

### 6.3 Design B — mutex-guarded map **[I/A]**

The alternative keeps the `clients` map behind a `sync.RWMutex`. Any goroutine may broadcast by taking a read lock and ranging the map. This avoids funnelling everything through one goroutine and can have lower latency for targeted sends, at the cost of you having to reason about lock scope and never doing slow work while holding the lock.

```go
type Hub struct {
	mu      sync.RWMutex
	clients map[string]map[*Client]bool // userID -> set of that user's connections
	log     *slog.Logger
}

func (h *Hub) Add(c *Client) {
	h.mu.Lock()
	defer h.mu.Unlock()
	set := h.clients[c.userID]
	if set == nil {
		set = make(map[*Client]bool)
		h.clients[c.userID] = set
	}
	set[c] = true
}

func (h *Hub) Remove(c *Client) {
	h.mu.Lock()
	defer h.mu.Unlock()
	if set := h.clients[c.userID]; set != nil {
		delete(set, c)
		if len(set) == 0 {
			delete(h.clients, c.userID)
		}
	}
}

// Targeted delivery: push to every live connection of one user (multi-device).
func (h *Hub) SendToUser(userID string, msg []byte) {
	h.mu.RLock()
	defer h.mu.RUnlock()
	for c := range h.clients[userID] {
		select {
		case c.send <- msg:
		default: // slow consumer — flag for eviction outside the lock
			go h.evict(c)
		}
	}
}
```

**Which to choose?** Design A (channels) is the most idiomatic and the easiest to keep race-free; reach for it first. Design B (mutex) shines when you need *targeted* delivery keyed by user/room and want to skip the single-goroutine hop — note the `map[string]map[*Client]bool` shape, which supports the very common "one user, many devices" case. In both designs the golden rule holds: **never do blocking I/O (an actual `c.Write`) while holding the lock or inside `Run`** — you only ever *enqueue* to `c.send`; the write pump does the real I/O.

### 6.4 The read pump and write pump **[I/A]**

Each client runs exactly two goroutines. The read pump reads messages and forwards them; the write pump drains `send`, writes, and pings.

```go
const (
	writeWait    = 10 * time.Second  // max time for a single Write
	pongWait     = 60 * time.Second  // if no traffic/pong in this window, peer is dead
	pingInterval = (pongWait * 9) / 10 // ping a bit before pongWait elapses
	maxMessage   = 32 * 1024
	sendBuffer   = 32               // per-client outbound queue depth
)

// readPump is the ONLY goroutine that reads this connection (§5.2).
func (c *Client) readPump(ctx context.Context) {
	defer func() {
		if r := recover(); r != nil { // never let a panic in one client kill the process
			c.hub.log.Error("readPump panic", "user", c.userID, "err", r)
		}
		c.hub.unregister <- c
		c.conn.CloseNow()
	}()

	c.conn.SetReadLimit(maxMessage)

	for {
		rctx, cancel := context.WithTimeout(ctx, pongWait)
		typ, data, err := c.conn.Read(rctx)
		cancel()
		if err != nil {
			if websocket.CloseStatus(err) == websocket.StatusNormalClosure {
				return // clean goodbye
			}
			c.hub.log.Debug("read ended", "user", c.userID, "err", err)
			return
		}
		if typ != websocket.MessageText {
			c.conn.Close(websocket.StatusUnsupportedData, "text only")
			return
		}
		c.hub.HandleInbound(c, data) // validate + authorize + route (§8.5)
	}
}

// writePump is the ONLY goroutine that writes this connection (backpressure + ping).
func (c *Client) writePump(ctx context.Context) {
	ticker := time.NewTicker(pingInterval)
	defer func() {
		ticker.Stop()
		c.conn.CloseNow()
		if r := recover(); r != nil {
			c.hub.log.Error("writePump panic", "user", c.userID, "err", r)
		}
	}()

	for {
		select {
		case <-ctx.Done():
			c.conn.Close(websocket.StatusGoingAway, "server shutdown")
			return
		case msg, ok := <-c.send:
			if !ok { // hub closed our send channel => we were unregistered/evicted
				c.conn.Close(websocket.StatusTryAgainLater, "evicted")
				return
			}
			wctx, cancel := context.WithTimeout(ctx, writeWait)
			err := c.conn.Write(wctx, websocket.MessageText, msg)
			cancel()
			if err != nil {
				return // write failed => connection is gone; readPump will unregister
			}
		case <-ticker.C:
			pctx, cancel := context.WithTimeout(ctx, writeWait)
			err := c.conn.Ping(pctx) // pong is consumed by readPump (§5.6)
			cancel()
			if err != nil {
				return // no pong in time => dead peer
			}
		}
	}
}
```

Note the elegant division of labor: the **write pump** owns *all* outbound traffic (app messages **and** pings) so there is never a second writer; the **read pump** owns all inbound traffic (app messages **and** the pongs that make `Ping` complete). Neither steps on the other. When either exits, it triggers cleanup and the other soon follows because the socket is closed.

### 6.5 Wiring a connection into the Hub **[I]**

```go
// ServeWS is called from your HTTP/Gin handler after auth (§7, §8).
func (h *Hub) ServeWS(w http.ResponseWriter, r *http.Request, userID string) {
	c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
		OriginPatterns: []string{"app.example.com"},
		Subprotocols:   []string{"json.v1"},
	})
	if err != nil {
		return
	}

	client := &Client{
		hub:    h,
		conn:   c,
		send:   make(chan []byte, sendBuffer), // BOUNDED — never make this unbuffered-infinite
		userID: userID,
		rooms:  map[string]bool{},
	}
	h.register <- client

	// Each connection gets exactly two goroutines. Use the request context so
	// server shutdown / client disconnect cancels both.
	ctx := context.Background() // or a derived server-lifetime context
	go client.writePump(ctx)
	client.readPump(ctx) // run read pump on this goroutine; returns on disconnect
}
```

### 6.6 Rooms, topics, presence & targeted delivery **[A]**

Real apps rarely broadcast to *everyone*; they broadcast to a **room** (a chat channel), a **topic** (a document being edited), or a **user** (all that user's devices). Model this as an index from key → set of clients. Presence ("who is online in this room?") is derived from that index; typing indicators and read receipts are just messages routed to a room.

```go
// Extend the channel-style Hub with rooms. rooms: roomID -> set of clients.
type Hub struct {
	// ...existing fields...
	rooms map[string]map[*Client]bool
}

func (h *Hub) joinRoom(c *Client, room string) {
	if h.rooms[room] == nil {
		h.rooms[room] = make(map[*Client]bool)
	}
	h.rooms[room][c] = true
	c.rooms[room] = true
	h.broadcastPresence(room) // tell the room someone joined
}

func (h *Hub) leaveRoom(c *Client, room string) {
	if set := h.rooms[room]; set != nil {
		delete(set, c)
		if len(set) == 0 {
			delete(h.rooms, room)
		}
	}
	delete(c.rooms, room)
	h.broadcastPresence(room)
}

// Deliver only to members of one room (targeted broadcast).
func (h *Hub) sendToRoom(room string, msg []byte) {
	for c := range h.rooms[room] {
		select {
		case c.send <- msg:
		default:
			h.evict(c)
		}
	}
}

func (h *Hub) broadcastPresence(room string) {
	members := make([]string, 0, len(h.rooms[room]))
	for c := range h.rooms[room] {
		members = append(members, c.userID)
	}
	payload, _ := json.Marshal(Envelope{Type: "presence", Data: mustJSON(map[string]any{
		"room": room, "online": members,
	})})
	h.sendToRoom(room, payload)
}
```

All of this stays *single-node* for now. The exact same room/broadcast calls become multi-node once we put a Redis backplane behind them ([§11](#11-scaling-out--redis-pubsub-backplane--nginx)).

### 6.7 Graceful drain on shutdown **[A]**

When a deploy rolls, you do not want to `kill -9` thousands of live connections — that shows up as `1006 abnormal closure` on the client and triggers reconnect storms. Instead, on `SIGTERM`: stop accepting new upgrades, then close every client with `StatusGoingAway (1001)` (or `StatusServiceRestart 1012`), giving clients a *reason* to reconnect with backoff after a delay. Bound the drain with a timeout so a stuck client cannot hold the deploy hostage.

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	h := hub.NewHub(logger)
	go h.Run(ctx) // when ctx is cancelled, Run closes all clients (see §6.2)

	srv := &http.Server{Addr: ":8080", Handler: router}
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("listen", "err", err)
		}
	}()

	<-ctx.Done() // SIGTERM received
	logger.Info("draining connections...")

	// Give in-flight closes up to 15s, then force the HTTP server down.
	shutCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutCtx)
}
```

---

## 7. Integrating With Gin

Most real Go backends route with [Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md), not bare `net/http`. Mounting a WebSocket on a Gin route is trivial because `Accept` only needs the underlying `http.ResponseWriter` and `*http.Request`, both of which Gin exposes on its `*gin.Context`.

### 7.1 Mounting the handler **[I]**

```go
// cmd/server/main.go
func main() {
	r := gin.New()
	r.Use(gin.Recovery(), RequestLogger()) // your normal HTTP middleware

	h := hub.NewHub(logger)
	go h.Run(context.Background())

	// REST endpoints (login, history) live alongside the WS route.
	r.POST("/api/login", loginHandler)
	r.GET("/api/messages/:room", authREST(), historyHandler)

	// The WebSocket upgrade endpoint.
	r.GET("/ws", func(c *gin.Context) {
		userID, ok := authenticateUpgrade(c) // §8
		if !ok {
			c.AbortWithStatus(http.StatusUnauthorized)
			return
		}
		// Hand Gin's raw writer/request to Accept. After this, do NOT use c
		// for HTTP responses — the socket is hijacked.
		h.ServeWS(c.Writer, c.Request, userID)
	})

	_ = r.Run(":8080")
}
```

### 7.2 Why WebSockets bypass your JSON middleware **[I]**

This trips people up: once `Accept` hijacks the connection, **Gin's response-oriented middleware no longer applies** to WebSocket traffic. Middleware that sets JSON headers, wraps the writer to capture status codes, gzips the *HTTP* response, or renders errors as JSON simply does not run against WebSocket frames — those frames never pass back through Gin's writer. Concretely:

- **Auth middleware that reads an `Authorization` header still works** *for the handshake* — because the handshake *is* still an HTTP request when your middleware runs, before `Accept`. But remember browsers can't set that header, so you usually authenticate inside the handler from the subprotocol/query instead ([§8](#8-authenticating-the-upgrade--banking-grade)).
- **Response-writing middleware (loggers that record status/bytes, gzip, CORS response headers) is effectively bypassed** for the WS lifetime. Do WebSocket-specific logging inside the pumps, not via HTTP middleware.
- **Panic recovery**: `gin.Recovery()` covers panics during the handshake, but once your pumps are running on their own goroutines you need your *own* `recover()` in each pump (as in §6.4) — a panic on a pump goroutine is not caught by Gin.

A clean split: use Gin middleware for the *handshake* (rate-limit the upgrade, parse/verify the token, set request-scoped identity), then hand off to the hub which owns everything post-upgrade.

### 7.3 Sharing request-scoped values across the boundary **[I]**

If your handshake middleware resolves the user (from a cookie session, a header on Go clients, or a query ticket), stash the identity on the `gin.Context` and read it in the handler:

```go
func authUpgradeMW() gin.HandlerFunc {
	return func(c *gin.Context) {
		claims, err := verifyUpgradeToken(c) // §8.2/§8.3
		if err != nil {
			c.AbortWithStatus(http.StatusUnauthorized)
			return
		}
		c.Set("userID", claims.Subject)
		c.Set("scopes", claims.Scopes)
		c.Next()
	}
}

// In the handler:
r.GET("/ws", authUpgradeMW(), func(c *gin.Context) {
	userID := c.GetString("userID")
	h.ServeWS(c.Writer, c.Request, userID)
})
```

---

## 8. Authenticating the Upgrade — Banking-Grade

Authentication is where WebSocket security gets subtle, and where the banking-grade bar matters most. The core principle: **authenticate at the upgrade, bind an identity to the connection for its whole life, and re-authorize every sensitive message.** We build on [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) for the token machinery and only cover the WebSocket-specific parts here.

### 8.1 The full auth flow **[A]**

1. User logs in over normal HTTPS `POST /api/login` with credentials. The server verifies the password with **Argon2id** (memory-hard, GPU-resistant) and, on success, issues a **short-lived access JWT** (e.g. 15 min) signed with a rotating key.
2. The browser opens the WebSocket, presenting that JWT. Because browsers can't set headers, it travels via the **subprotocol** or a **one-time query ticket** (§8.3).
3. The `/ws` handler **verifies the JWT before or at `Accept`**, extracts the identity (`sub`) and scopes, and — only then — upgrades. Unauthenticated upgrades are rejected with `401` (pre-upgrade) or closed with `StatusPolicyViolation 1008` / a private `4001`.
4. The verified identity is stored on the `Client` and used to authorize each inbound message (§8.5).
5. Because the connection outlives the token, the client refreshes and the server re-checks expiry (§8.6).

### 8.2 Issuing the JWT at login (Argon2id) **[A]**

```go
// POST /api/login — verify password with Argon2id, mint a short-lived access JWT.
func loginHandler(c *gin.Context) {
	var req struct{ Email, Password string }
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(400, gin.H{"error": "bad request"})
		return
	}

	user, err := lookupUser(c.Request.Context(), req.Email)
	// Argon2id verify. Compare against the stored encoded hash. Use a constant-
	// time comparison inside verifyArgon2id (see the JWT+Argon2 guide).
	if err != nil || !verifyArgon2id(req.Password, user.PasswordHash) {
		// Identical response & timing for "no user" and "bad password" to avoid
		// user enumeration.
		c.JSON(401, gin.H{"error": "invalid credentials"})
		return
	}

	claims := AccessClaims{
		Scopes: user.Scopes,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   user.ID,
			Issuer:    "realtime.example.com",
			Audience:  jwt.ClaimStrings{"ws"},
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			ID:        uuid.NewString(), // jti, for revocation lists if needed
		},
	}
	tok := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signed, _ := tok.SignedString(signingKey) // prefer RS256/EdDSA in prod (see JWT guide)
	c.JSON(200, gin.H{"token": signed, "expiresIn": 900})
}
```

### 8.3 Passing the token past the browser's header limit **[A]**

Recall [§4.2](#42-dialoptions-and-the-browser-header-limitation-bi): browser `WebSocket` cannot set `Authorization`. Two viable patterns, each with tradeoffs:

**Pattern 1 — Subprotocol-smuggled token (preferred).** The client puts the token in `Sec-WebSocket-Protocol`. The server parses it out, verifies it, and — importantly — **echoes back a real subprotocol** it selected so the handshake is RFC-valid.

```javascript
// Client: first entry is the app subprotocol, second carries the JWT.
new WebSocket("wss://api.example.com/ws", ["json.v1", `auth.jwt.${token}`]);
```

```go
// internal/auth/upgrade.go
// Server: pull the token out of the requested subprotocols BEFORE Accept.
func authenticateUpgrade(c *gin.Context) (string, bool) {
	var token string
	// The browser's requested subprotocols arrive in the comma-separated
	// Sec-WebSocket-Protocol request header. Unlike gorilla, coder/websocket has
	// no websocket.Subprotocols() helper, so parse the header yourself.
	for _, p := range strings.Split(c.Request.Header.Get("Sec-WebSocket-Protocol"), ",") {
		p = strings.TrimSpace(p)
		if strings.HasPrefix(p, "auth.jwt.") {
			token = strings.TrimPrefix(p, "auth.jwt.")
			break
		}
	}
	if token == "" {
		return "", false
	}
	claims, err := verifyAccessJWT(token)
	if err != nil {
		return "", false
	}
	return claims.Subject, true
}

// In ServeWS, we must select a subprotocol we actually support so the browser's
// ws.protocol is set and the handshake validates. Never echo the token itself.
c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
	Subprotocols:   []string{"json.v1"},          // we pick this one
	OriginPatterns: []string{"app.example.com"},
})
```

Why preferred: the token never lands in a URL, so it does not leak into **access logs, `Referer` headers, browser history, or proxy logs**. The cost is the mild ugliness of overloading the subprotocol field.

**Pattern 2 — Short-lived query-param ticket.** The client first calls an authenticated REST endpoint to exchange its access JWT for a **single-use, ~30-second "ticket"**, then opens `wss://…/ws?ticket=…`. The server verifies and immediately burns the ticket.

```go
// GET /api/ws-ticket (authenticated over normal HTTPS): mint a one-time ticket.
func wsTicketHandler(c *gin.Context) {
	userID := c.GetString("userID")
	ticket := uuid.NewString()
	// Store in Redis with a 30s TTL, bound to the user, single-use.
	redisClient.Set(c, "wsticket:"+ticket, userID, 30*time.Second)
	c.JSON(200, gin.H{"ticket": ticket})
}

// At /ws: consume the ticket exactly once (atomic GETDEL).
func authenticateUpgrade(c *gin.Context) (string, bool) {
	ticket := c.Query("ticket")
	userID, err := redisClient.GetDel(c, "wsticket:"+ticket).Result() // atomic, single-use
	if err != nil || userID == "" {
		return "", false
	}
	return userID, true
}
```

Why sometimes necessary: some infrastructure or client libraries make subprotocol manipulation awkward. The **critical caveat**: raw access tokens must **never** go directly in the query string — URLs leak everywhere. A *ticket* is acceptable *only because* it is single-use and expires in seconds, so a leaked URL is worthless moments later. Never put the long-lived JWT itself in the URL.

> **⚡ Security note:** Whatever you choose, do it **consistently** and reject anything else. Do not accept "token in query OR subprotocol OR header" as a permissive fallback — each accepted channel is attack surface. Pick one browser path (subprotocol) plus the header path for trusted Go clients, and refuse the rest.

### 8.4 Verifying and binding identity **[A]**

```go
func verifyAccessJWT(raw string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	_, err := jwt.ParseWithClaims(raw, claims, func(t *jwt.Token) (any, error) {
		// Pin the algorithm — never trust the token's alg header (alg-confusion attack).
		if t.Method.Alg() != jwt.SigningMethodHS256.Alg() {
			return nil, errors.New("unexpected signing method")
		}
		return signingKey, nil
	},
		jwt.WithAudience("ws"),                     // this token is meant for the WS service
		jwt.WithIssuer("realtime.example.com"),
		jwt.WithExpirationRequired(),               // reject tokens with no exp
		jwt.WithValidMethods([]string{"HS256"}),
	)
	if err != nil {
		return nil, err
	}
	return claims, nil
}
```

The verified `claims.Subject` becomes the `Client.userID` and `claims.Scopes` the connection's authority — set once, at the upgrade, and never re-derived from client-supplied data thereafter.

### 8.5 Per-message authorization **[A]**

Authenticating the *connection* is necessary but not sufficient. A client authenticated as Alice must not be able to post into a room she is not a member of, or address a message to another user, simply because the socket is open. **Authorize every inbound message against the connection's bound identity and scopes.**

```go
func (h *Hub) HandleInbound(c *Client, raw []byte) {
	var env Envelope
	if err := json.Unmarshal(raw, &env); err != nil {
		// Malformed input from an authenticated user: log and ignore (don't crash).
		return
	}
	switch env.Type {
	case "chat":
		var m struct{ Room, Body string }
		if json.Unmarshal(env.Data, &m) != nil || len(m.Body) == 0 || len(m.Body) > 4000 {
			return // input validation
		}
		// AUTHORIZATION: is *this connection's user* actually in that room?
		if !c.rooms[m.Room] {
			c.send <- mustJSON(Envelope{Type: "error", Data: mustJSON(map[string]string{
				"error": "not a member of room", "room": m.Room,
			})})
			return
		}
		// Persist-then-broadcast (§9.3), stamping the SERVER's notion of identity,
		// never trusting any "from" field the client sends.
		h.persistAndBroadcast(c.userID, m.Room, m.Body)
	case "join":
		var m struct{ Room string }
		_ = json.Unmarshal(env.Data, &m)
		if h.userMayJoin(c.userID, m.Room) { // scope/ACL check
			h.joinRoom(c, m.Room)
		}
	default:
		// Unknown message type from a client is suspicious — count it toward
		// a rate/abuse budget (§10.5).
	}
}
```

Two non-negotiables visible here: **(1) never trust a client-supplied `from`/`sender`** — the server stamps identity from `c.userID`; **(2) validate every field** (length caps, room membership) before acting.

### 8.6 Token expiry on a long-lived connection **[A]**

A connection can stay open for hours; an access token expires in minutes. You must decide what expiry *means* for an open socket. Common, defensible policy:

- The token authorizes the *upgrade*. Once upgraded, the connection stays valid until it closes — but you enforce a **maximum session lifetime** (e.g. close after 8 hours regardless), and any **sensitive** action re-checks a fresh credential.
- For stricter (bank-grade) posture: the client periodically sends a **re-auth message** carrying a fresh token; the server re-verifies and slides the connection's authority window. If the client fails to re-auth before a deadline, the server closes with a private **`4001` "re-auth required"** so the client silently reconnects with a new token.

```go
// Server tracks a re-auth deadline per client; the read pump enforces it.
func (h *Hub) handleReauth(c *Client, raw json.RawMessage) {
	var m struct{ Token string }
	if json.Unmarshal(raw, &m) != nil {
		return
	}
	claims, err := verifyAccessJWT(m.Token)
	if err != nil || claims.Subject != c.userID { // token must belong to same user
		c.conn.Close(4001, "re-auth failed")
		return
	}
	c.reauthDeadline = time.Now().Add(15 * time.Minute) // slide the window
}
```

---

## 9. The Data Layer — pgx, Ent, Air & Persist-Then-Broadcast

A real-time backend is not just a message router — messages usually need to be **persisted** (chat history, notification records, audit trail) and users authenticated against a database. We use **pgx v5** for the Postgres driver/pool, **[Ent](GO_ENT_ORM_GUIDE.md)** for the typed ORM, and **Air** for live reload during development.

### 9.1 pgx v5 connection pool **[I]**

`pgx` is the fast, actively-maintained Postgres driver for Go; its `pgxpool` gives you a concurrency-safe connection pool that all your goroutines (HTTP handlers *and* WebSocket pumps) share.

```go
import "github.com/jackc/pgx/v5/pgxpool"

func newPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn) // dsn: postgres://user:pass@host:5432/db?sslmode=require
	if err != nil {
		return nil, err
	}
	cfg.MaxConns = 20                       // size for your concurrency, not "as big as possible"
	cfg.MaxConnIdleTime = 5 * time.Minute
	cfg.HealthCheckPeriod = 1 * time.Minute
	return pgxpool.NewWithConfig(ctx, cfg)
}
```

### 9.2 Ent schema for users, rooms, messages **[I]**

Ent generates a fully-typed, query-builder API from schema definitions. Here is a minimal schema for our real-time service:

```go
// ent/schema/user.go
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").DefaultFunc(uuid.NewString).Immutable(),
		field.String("email").Unique(),
		field.String("password_hash").Sensitive(), // omitted from logs/JSON by Ent
		field.Strings("scopes").Optional(),
		field.Time("created_at").Default(time.Now).Immutable(),
	}
}

// ent/schema/message.go
func (Message) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").DefaultFunc(uuid.NewString).Immutable(),
		field.String("room").NotEmpty(),
		field.String("body").MaxLen(4000),
		field.Time("sent_at").Default(time.Now).Immutable(),
	}
}
func (Message) Edges() []ent.Edge {
	return []ent.Edge{edge.From("author", User.Type).Ref("messages").Unique().Required()}
}
```

For a demo you can skip migration files with `client.Schema.Create(ctx)` (Ent's auto-migration creates/updates tables to match the schema); for production use versioned migrations ([Ent guide](GO_ENT_ORM_GUIDE.md)).

### 9.3 Persist-then-broadcast **[I/A]**

The canonical write flow for a durable real-time app: when a message arrives and is authorized, **persist it first**, then broadcast the *stored* record (with its DB-assigned id and timestamp). Persisting first means a client that reloads and fetches history sees exactly what live subscribers saw — no ghost messages that were broadcast but never saved, and no ordering skew.

```go
// internal/store/messages.go
func (h *Hub) persistAndBroadcast(userID, room, body string) {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	// 1) PERSIST — the DB assigns id + sent_at, the source of truth.
	msg, err := h.db.Message.Create().
		SetRoom(room).
		SetBody(body).
		SetAuthorID(userID).
		Save(ctx)
	if err != nil {
		h.log.Error("persist message", "err", err)
		return // do NOT broadcast something we failed to store
	}

	// 2) BROADCAST the stored record (server-stamped identity + timestamp).
	payload := mustJSON(Envelope{Type: "chat", Data: mustJSON(map[string]any{
		"id": msg.ID, "room": room, "from": userID, "body": body, "sentAt": msg.SentAt,
	})})

	// Single node: fan out locally. Multi-node: publish to Redis (§11).
	h.sendToRoom(room, payload)
}
```

### 9.4 Air live reload for development **[I]**

Air watches your source and rebuilds/reruns the server on save so you iterate fast. WebSocket dev especially benefits — you keep a browser tab connected and watch behavior change. A minimal `.air.toml`:

```toml
# .air.toml — live reload for the realtime server.
root = "."
tmp_dir = "tmp"

[build]
  cmd = "go build -o ./tmp/server ./cmd/server"
  bin = "./tmp/server"
  include_ext = ["go", "tpl", "tmpl", "html"]
  exclude_dir = ["tmp", "ent/schema", "node_modules"]
  delay = 200           # ms debounce
  stop_on_error = true

[log]
  time = true
```

```bash
go install github.com/air-verse/air@latest
air   # rebuilds + restarts on save; connected WS clients will get 1006/close and reconnect
```

> **⚡ Note:** On each Air rebuild the process restarts, so open WebSocket connections drop (clients see an abnormal close and reconnect via your backoff logic). That is expected in dev. Air's import path is now `github.com/air-verse/air` (formerly `cosmtrek/air`).

---

## 10. Banking-Grade Security

WebSockets widen your attack surface: a long-lived, stateful, bidirectional channel that bypasses much of your HTTP middleware. Treat this section as a checklist you actively verify, not read.

### 10.1 Always wss:// (TLS) **[A]**

Never run production WebSockets over `ws://`. Plaintext exposes tokens, messages, and session data to anyone on the path and enables trivial MITM/downgrade. Terminate TLS at the edge (Nginx/load balancer) or in-process, and serve **`wss://`** exclusively. Set `Strict-Transport-Security` on the HTTPS app that hosts the client so the browser refuses plaintext. Inside a trusted cluster you may terminate TLS at the proxy and speak `ws://` on the loopback hop, but never across untrusted networks.

### 10.2 Origin checking / CSWSH **[A]**

**Cross-Site WebSocket Hijacking (CSWSH)** is the WebSocket cousin of CSRF. Because the browser automatically attaches cookies to a WebSocket handshake, a malicious page at `evil.com` can open a socket to `your-api.com` and — if you authenticate by cookie and don't check origin — ride the victim's session. coder/websocket defends by default: **`Accept` rejects cross-origin handshakes** unless the origin's host matches the request host or is listed in `OriginPatterns`. Two rules:

- **Set `OriginPatterns` to your exact allowed hosts.** Do not use wildcards you don't need.
- **Never set `InsecureSkipVerify: true` in production.** It disables the origin check entirely; it exists only for local tooling and tests.

```go
websocket.Accept(w, r, &websocket.AcceptOptions{
	OriginPatterns: []string{"app.example.com", "admin.example.com"},
	// InsecureSkipVerify: true, // DEV ONLY. Never ship this.
})
```

Belt-and-braces: prefer **token-based** auth (§8) over cookie auth for WebSockets; a token the attacker's page cannot read is not automatically attached, so CSWSH is defused even if an origin check were misconfigured.

### 10.3 Auth on upgrade + per-message authz **[A]**

Covered in [§8](#8-authenticating-the-upgrade--banking-grade): verify a token at the upgrade, bind identity, and re-authorize each sensitive message against that identity. The security failure mode to avoid is "authenticated ≠ authorized" — an open, authenticated socket is not a license to perform any action.

### 10.4 Input validation & read limits **[A]**

Every byte from a client is hostile until validated. Two layers:

- **`SetReadLimit`** caps message size at the transport (default 32 KiB) — set it as low as your protocol allows so a peer cannot force large allocations ([§3.6](#36-setreadlimit--dont-let-a-peer-ram-your-heap-b)).
- **Application validation** on every decoded message: enforce field lengths, enum/`type` whitelists, UTF-8 validity, numeric ranges, and reject unknown message types. Decode into typed structs, not `map[string]any`, and treat `json.Unmarshal` errors as "drop the message," never "crash."

### 10.5 Per-connection rate limiting **[A]**

An authenticated client can still flood you. Rate-limit **per connection** (and optionally per user across connections). A `golang.org/x/time/rate` token bucket per client is simple and effective; exceed it and close with `StatusPolicyViolation` or `StatusTryAgainLater`.

```go
import "golang.org/x/time/rate"

// e.g. 20 messages/sec sustained, burst of 40, per connection.
c.limiter = rate.NewLimiter(20, 40)

// In the read pump, before handling each inbound message:
if !c.limiter.Allow() {
	c.conn.Close(websocket.StatusPolicyViolation, "rate limit exceeded")
	return
}
```

Also cap **message rate of unusual events** (unknown types, auth failures) and **connection rate** at the proxy (§11.4).

### 10.6 Max connections / DoS **[A]**

Each connection costs two goroutines, socket buffers, and a send-channel buffer. An attacker opening millions of idle sockets is a resource-exhaustion attack. Defenses: cap total connections (a counting semaphore checked in `ServeWS`), cap connections per user/IP, enforce **idle read timeouts** so silent sockets get reaped (`context.WithTimeout(ctx, pongWait)` on each read, §6.4), and set OS `ulimit`/file-descriptor limits appropriately. Reject over-limit upgrades with `503`/`StatusTryAgainLater`.

```go
var connSem = make(chan struct{}, 50_000) // global cap on concurrent connections

func (h *Hub) ServeWS(w http.ResponseWriter, r *http.Request, userID string) {
	select {
	case connSem <- struct{}{}:
		defer func() { <-connSem }()
	default:
		http.Error(w, "server at capacity", http.StatusServiceUnavailable)
		return
	}
	// ...accept + pumps...
}
```

### 10.7 Idle/read timeouts & graceful shutdown **[A]**

Every read has a deadline (context timeout ≈ `pongWait`); a client that neither sends nor pongs within the window is dropped, freeing resources and detecting dead peers. On deploy, drain gracefully with `StatusGoingAway`/`StatusServiceRestart` ([§6.7](#67-graceful-drain-on-shutdown-a)) instead of hard-killing sockets.

### 10.8 No secrets in URLs **[A]**

URLs leak into access logs, `Referer` headers, browser history, and proxy logs. Never put a long-lived JWT in the query string. Use the subprotocol channel, or a single-use short-TTL ticket if you must use a query param ([§8.3](#83-passing-the-token-past-the-browsers-header-limit-a)).

### 10.9 Compression & CRIME/BREACH caution **[A]**

`permessage-deflate` (RFC 7692) can dramatically cut bandwidth for repetitive JSON, but compression + secrets is dangerous. **CRIME/BREACH-style attacks** exploit the fact that compressed size *leaks information*: if a message mixes attacker-controlled data with a secret (e.g. a token echoed alongside user input), the attacker can infer the secret from observed compressed sizes. Rules:

- Do **not** compress messages that combine attacker-influenced content with secrets.
- Prefer `CompressionDisabled` (the default) unless you have measured a real bandwidth win.
- If you enable it, understand the context-takeover modes (below) and keep secrets out of compressed frames.

| Mode | Behavior | Threshold |
|---|---|---|
| `CompressionDisabled` | No compression (default) | — |
| `CompressionContextTakeover` | Reuses a 32 KB sliding window across messages — best ratio, more memory/connection | 128 B |
| `CompressionNoContextTakeover` | Resets the window each message — less memory, worse ratio, safer | 512 B |

### 10.10 Structured logging & audit **[A]**

Log connection lifecycle (open/close with code, user, remote addr), auth outcomes, rate-limit trips, and evictions with `log/slog` in structured form. For banking, keep an **audit trail** of sensitive actions performed over the socket (who did what, when) — persisted, tamper-evident, and never containing secrets/tokens/PII beyond what compliance requires. Do **not** log message bodies containing sensitive data or the tokens themselves.

### 10.11 Goroutine leaks & panic recovery **[A]**

Two connection goroutines that never exit are a slow memory leak that eventually kills the process. Guarantees: every pump has a `defer` that closes the connection and unregisters the client; every read/write is context-bounded so it cannot block forever; and every pump `recover()`s so a panic in one connection's handler cannot crash the whole server (§6.4). Test for leaks with `runtime.NumGoroutine()` before/after a churn of connections in your test suite.

### 10.12 Security checklist **[A]**

| Control | Mechanism |
|---|---|
| Transport encryption | `wss://` only; HSTS on the app |
| CSWSH / origin | `OriginPatterns` set; never `InsecureSkipVerify` in prod |
| Authentication | JWT verified at upgrade; token via subprotocol/ticket, not URL |
| Authorization | Per-message check against bound identity + scopes |
| Input size | `SetReadLimit` as low as protocol allows |
| Input content | Typed decode + field validation + type whitelist |
| Rate limiting | Per-connection token bucket + proxy limits |
| Connection caps | Global + per-user/IP semaphores |
| Liveness/idle | Ping/pong + per-read deadlines |
| Shutdown | Graceful drain with `StatusGoingAway` |
| Secrets | Never in URLs/logs; short-lived tokens |
| Compression | Disabled by default; CRIME/BREACH aware |
| Resilience | Panic recovery + bounded goroutines in every pump |

---

## 11. Scaling Out — Redis Pub/Sub Backplane & Nginx

### 11.1 The single-process limit **[A]**

The Hub from §6 lives in **one process's memory**. It works beautifully — until you run two instances. Now Alice is connected to instance A and Bob to instance B; when Alice posts to a room, instance A's hub fans out only to *its* local clients, so Bob (on B) never sees it. The moment you scale horizontally (for capacity or high availability), each node only knows about its *own* connections. You need a **backplane**: a shared channel every node subscribes to, so a message published on any node reaches the clients on *every* node.

### 11.2 The Redis pub/sub backplane **[A]**

[Redis](REDIS_GUIDE.md) pub/sub is the classic backplane. The pattern:

- Each node **subscribes** to relevant channels (e.g. a channel per room, or one firehose channel).
- When a node needs to broadcast, it **publishes** the message to Redis instead of (or in addition to) fanning out locally.
- Every node's subscription handler receives the published message and fans it out to *its own* local clients via the existing hub.

So `sendToRoom` changes from "fan out locally" to "publish to Redis," and a background subscriber does the local fan-out on every node. Persistence still happens once (persist-then-publish).

```go
// internal/hub/backplane.go
import "github.com/redis/go-redis/v9"

type Backplane struct {
	rdb *redis.Client
	hub *Hub
}

// Publish is called instead of local fan-out. The message reaches ALL nodes.
func (b *Backplane) PublishToRoom(ctx context.Context, room string, payload []byte) error {
	return b.rdb.Publish(ctx, "room:"+room, payload).Err()
}

// Subscribe runs once per node at boot; it delivers messages to LOCAL clients.
func (b *Backplane) Subscribe(ctx context.Context, rooms ...string) {
	// Pattern-subscribe to every room channel this node cares about.
	sub := b.rdb.PSubscribe(ctx, "room:*")
	defer sub.Close()

	ch := sub.Channel()
	for {
		select {
		case <-ctx.Done():
			return
		case msg := <-ch:
			room := strings.TrimPrefix(msg.Channel, "room:")
			// Local fan-out only: each node handles its own connections.
			b.hub.sendToRoomLocal(room, []byte(msg.Payload))
		}
	}
}
```

The persist-then-broadcast flow becomes **persist → publish**; local delivery is now a side effect of the subscription, uniform across nodes:

```go
// internal/store/messages.go
func (h *Hub) persistAndBroadcast(userID, room, body string) {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	msg, err := h.db.Message.Create().
		SetRoom(room).SetBody(body).SetAuthorID(userID).Save(ctx)
	if err != nil {
		return
	}
	payload := mustJSON(Envelope{Type: "chat", Data: mustJSON(map[string]any{
		"id": msg.ID, "room": room, "from": userID, "body": body, "sentAt": msg.SentAt,
	})})

	// Publish to the backplane. EVERY node (including this one, via its own
	// subscription) delivers to its local room members. Do NOT also call the
	// local fan-out here, or this node's clients get the message twice.
	_ = h.backplane.PublishToRoom(ctx, room, payload)
}
```

> **⚡ Delivery-semantics note:** Redis pub/sub is **fire-and-forget, at-most-once** — if a node is momentarily disconnected from Redis it *misses* messages published during that gap. For chat that is usually fine because the durable copy is in Postgres and the client re-fetches history on reconnect. If you need stronger guarantees (replay, consumer groups, backlog), use **Redis Streams** or a real broker (NATS/Kafka) instead of plain pub/sub.

### 11.3 Presence in Redis **[A]**

With multiple nodes, "who is online in room X?" can no longer be answered from one node's memory. Store presence in Redis: a `SET` per room (`presence:room:X`) of online user IDs, with each membership carrying a TTL you refresh via heartbeat so crashed nodes' entries expire. On join/leave, update the set and publish a presence event. `SMEMBERS presence:room:X` answers the query cluster-wide.

```go
func (b *Backplane) MarkOnline(ctx context.Context, room, userID string) {
	key := "presence:room:" + room
	b.rdb.SAdd(ctx, key, userID)
	b.rdb.Expire(ctx, key, 90*time.Second) // refreshed by heartbeat; stale nodes expire
	b.rdb.Publish(ctx, "presence:"+room, []byte(userID+" joined"))
}
```

### 11.4 Sticky vs stateless, and the reverse proxy (Nginx) **[A]**

WebSocket connections are **long-lived and stateful**, so the load balancer must (a) support the HTTP Upgrade and (b) keep a connection pinned to the node that owns it — you cannot mid-stream move an open socket to another node. With the Redis backplane the *application* is effectively stateless (any node can serve any user because messages fan out cluster-wide), but a single *TCP connection* still lives on one node for its lifetime. Configure [Nginx](NGINX_GUIDE.md) to proxy the upgrade and set generous timeouts, because the default 60 s proxy read timeout will kill idle WebSockets:

```nginx
# nginx.conf — reverse-proxying WebSockets
map $http_upgrade $connection_upgrade {
	default upgrade;
	''      close;
}

upstream realtime {
	# ip_hash gives simple stickiness by client IP; or use a cookie-based method.
	ip_hash;
	server 10.0.0.11:8080;
	server 10.0.0.12:8080;
}

server {
	listen 443 ssl http2;
	server_name api.example.com;
	# ssl_certificate ...;

	location /ws {
		proxy_pass http://realtime;
		proxy_http_version 1.1;                     # required for Upgrade
		proxy_set_header Upgrade $http_upgrade;     # forward the Upgrade header
		proxy_set_header Connection $connection_upgrade;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		proxy_read_timeout 3600s;   # keep idle WS alive up to 1h (pair with app pings)
		proxy_send_timeout 3600s;
		proxy_buffering off;        # stream frames straight through
	}
}
```

Note the app-level ping interval (§6.4) should be *shorter* than the proxy's read timeout so traffic keeps the connection marked alive.

### 11.5 Load testing **[A]**

Real-time systems fail differently under load — not with slow responses but with memory growth, goroutine leaks, and event-loop-style stalls. Test with tools that open *many concurrent connections* and measure fan-out latency: `k6` (has a WebSocket module), Tsung, `websocket-bench`, or a custom Go harness using `Dial` in thousands of goroutines. Track: connections established/sec, message round-trip latency at the p50/p99, memory and goroutine count over time, and behavior when you kill and restart a node (do clients reconnect and rejoin rooms cleanly?).

---

## 12. Complete Worked Example — Real-Time Notifications Service

This ties everything together into a coherent, production-shaped service: **live notifications + chat** with Gin + coder/websocket + JWT + Argon2 + Ent + pgx + Redis backplane, with graceful shutdown. It is intentionally realistic; treat it as a skeleton you extend.

### 12.1 Project layout **[A]**

Here is the full directory structure, mapping every file to the section that builds it — so you always know *which file* a given code block belongs in. The components you met in §6–§11 assemble here; **each block in this section is headed with a comment naming its file**, and the earlier component blocks (§6 Hub, §8 auth, §9 persistence) note their destination file too.

```text
realtime/
├── cmd/
│   └── server/
│       └── main.go              # §12.2  wiring: config → pool → ent → hub → gin → shutdown
├── internal/
│   ├── hub/
│   │   ├── hub.go               # §6   the Hub (register/unregister/broadcast, rooms)
│   │   ├── client.go            # §6   the Client (read/write pumps, bounded send, backpressure)
│   │   └── backplane.go         # §11  Redis pub/sub cross-instance fan-out
│   ├── auth/
│   │   ├── password.go          # §8   Argon2id verify
│   │   ├── jwt.go               # §8   JWT issue/verify (rigorous validation)
│   │   └── upgrade.go           # §8   authenticate the WS upgrade (subprotocol/ticket)
│   ├── handlers/
│   │   ├── login.go             # §8   POST /login → JWT
│   │   ├── ticket.go            # §8   POST /ws-ticket → single-use ticket
│   │   └── history.go           # §9   REST history (Ent reads)
│   └── store/
│       └── messages.go          # §9   persist-then-broadcast (Ent + pgx)
├── ent/                         # generated ORM (schema in ent/schema/*.go)
│   └── schema/
│       └── message.go           # the Message entity
├── web/
│   └── index.html               # the browser WS client (§4)
├── nginx/
│   └── ws.conf                  # §11  reverse proxy (Upgrade headers, timeouts)
├── .air.toml                    # hot-reload for development
├── .env                         # DATABASE_URL, JWT_SECRET, REDIS_URL — never committed
├── go.mod
└── go.sum
```

### 12.2 `main.go` — wiring and shutdown **[A]**

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

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"

	"github.com/acme/realtime/ent"
	"github.com/acme/realtime/internal/auth"
	"github.com/acme/realtime/internal/handlers"
	"github.com/acme/realtime/internal/hub"
)

func main() {
	log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Root context cancelled on SIGINT/SIGTERM — drives graceful shutdown.
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	// --- Data layer: pgx pool via ent's sql driver, + Redis for backplane/tickets.
	client, err := ent.Open("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Error("ent open", "err", err)
		os.Exit(1)
	}
	defer client.Close()
	if err := client.Schema.Create(ctx); err != nil { // demo auto-migrate
		log.Error("schema create", "err", err)
		os.Exit(1)
	}
	rdb := redis.NewClient(&redis.Options{Addr: os.Getenv("REDIS_ADDR")})

	// --- Hub + backplane.
	h := hub.New(client, rdb, log)
	go h.Run(ctx)
	go h.Backplane().Subscribe(ctx) // node-local fan-out from Redis

	// --- HTTP/Gin.
	r := gin.New()
	r.Use(gin.Recovery())
	r.POST("/api/login", handlers.Login(client))
	r.GET("/api/ws-ticket", auth.RequireJWT(), handlers.WSTicket(rdb))
	r.GET("/api/messages/:room", auth.RequireJWT(), handlers.History(client))
	r.GET("/ws", func(c *gin.Context) {
		userID, ok := auth.AuthenticateUpgrade(c, rdb)
		if !ok {
			c.AbortWithStatus(http.StatusUnauthorized)
			return
		}
		h.ServeWS(c.Writer, c.Request, userID)
	})

	srv := &http.Server{Addr: ":8080", Handler: r}
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Error("listen", "err", err)
		}
	}()
	log.Info("realtime up on :8080")

	<-ctx.Done() // signal received
	log.Info("shutting down; draining connections")
	// h.Run's ctx is already cancelled → it closes all clients with GoingAway.
	shutCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutCtx)
}
```

### 12.3 The hub package (condensed) **[A]**

```go
package hub

// New builds a Hub bound to the DB and Redis backplane.
func New(db *ent.Client, rdb *redis.Client, log *slog.Logger) *Hub {
	h := &Hub{
		clients:    map[*Client]bool{},
		rooms:      map[string]map[*Client]bool{},
		register:   make(chan *Client),
		unregister: make(chan *Client),
		db:         db,
		log:        log,
	}
	h.backplane = &Backplane{rdb: rdb, hub: h}
	return h
}

func (h *Hub) Backplane() *Backplane { return h.backplane }

// ServeWS: upgrade (origin + subprotocol), register, spin up pumps.
func (h *Hub) ServeWS(w http.ResponseWriter, r *http.Request, userID string) {
	select { // global connection cap (§10.6)
	case connSem <- struct{}{}:
		defer func() { <-connSem }()
	default:
		http.Error(w, "capacity", http.StatusServiceUnavailable)
		return
	}

	c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
		OriginPatterns: []string{"app.example.com"},
		Subprotocols:   []string{"json.v1"},
	})
	if err != nil {
		h.log.Warn("accept failed", "err", err)
		return
	}

	client := &Client{
		hub: h, conn: c, userID: userID,
		send:    make(chan []byte, 32),
		rooms:   map[string]bool{},
		limiter: rate.NewLimiter(20, 40),
	}
	h.register <- client

	go client.writePump(context.Background())
	client.readPump(context.Background()) // blocks until disconnect
}
```

The `Client`, `readPump`, `writePump`, `HandleInbound`, `joinRoom`, `sendToRoomLocal`, `persistAndBroadcast`, and `Backplane` methods are exactly as developed in §6, §8.5, §9.3, and §11.2 — assembled here they form a complete, coherent service. On the wire the lifecycle is: browser logs in over HTTPS → gets a JWT → opens `wss://…/ws` with subprotocol `["json.v1","auth.jwt.<jwt>"]` → server verifies, registers, starts pumps → client `join`s rooms → messages are validated, authorized, persisted, and published to Redis → every node delivers to its local room members → on deploy, all clients get `1001 GoingAway` and reconnect with backoff.

### 12.4 The browser client (condensed) **[A]**

```javascript
async function start() {
	const { token } = await (await fetch("/api/login", {
		method: "POST", headers: { "Content-Type": "application/json" },
		body: JSON.stringify({ email, password }),
	})).json();

	const ws = new WebSocket("wss://api.example.com/ws", ["json.v1", `auth.jwt.${token}`]);
	ws.addEventListener("open", () => ws.send(JSON.stringify({ type: "join", data: { room: "general" } })));
	ws.addEventListener("message", (ev) => {
		const env = JSON.parse(ev.data);
		if (env.type === "chat")     renderMessage(env.data);
		if (env.type === "presence") renderPresence(env.data);
	});
	ws.addEventListener("close", (ev) => scheduleReconnect(ev.code)); // backoff, §4.4
	// Send a chat message:
	// ws.send(JSON.stringify({ type: "chat", data: { room: "general", body } }));
}
```

---

## 13. Testing WebSocket Code

WebSocket handlers are very testable because `httptest.NewServer` gives you a real HTTP server you can `Dial` with the same library. The pattern: start the server on your handler, `Dial` it, exercise the protocol, assert.

### 13.1 A round-trip test **[I]**

```go
func TestEchoRoundTrip(t *testing.T) {
	// Real server bound to the handler under test.
	srv := httptest.NewServer(http.HandlerFunc(echoHandler))
	defer srv.Close()

	// httptest gives an http:// URL; the library accepts it as ws://.
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	c, _, err := websocket.Dial(ctx, srv.URL+"/ws", &websocket.DialOptions{
		// Same-origin in tests; if your handler sets OriginPatterns you may need
		// to set the Origin header or InsecureSkipVerify *in the test only*.
	})
	if err != nil {
		t.Fatalf("dial: %v", err)
	}
	defer c.CloseNow()

	want := map[string]any{"type": "hello", "n": 1.0}
	if err := wsjson.Write(ctx, c, want); err != nil {
		t.Fatalf("write: %v", err)
	}
	var got map[string]any
	if err := wsjson.Read(ctx, c, &got); err != nil {
		t.Fatalf("read: %v", err)
	}
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("got %v want %v", got, want)
	}

	// Assert a clean close handshake.
	if err := c.Close(websocket.StatusNormalClosure, ""); err != nil {
		t.Fatalf("close: %v", err)
	}
}
```

### 13.2 Table-driven protocol tests **[I]**

Drive multiple message types through one connection and assert responses. This is the natural way to test `HandleInbound` routing and authorization rules:

```go
func TestInboundRouting(t *testing.T) {
	cases := []struct {
		name    string
		send    Envelope
		wantTyp string
	}{
		{"join then chat", Envelope{Type: "join", Data: mustJSON(map[string]string{"room": "r1"})}, "presence"},
		{"chat in joined room", Envelope{Type: "chat", Data: mustJSON(map[string]string{"room": "r1", "body": "hi"})}, "chat"},
		{"chat in foreign room", Envelope{Type: "chat", Data: mustJSON(map[string]string{"room": "r2", "body": "x"})}, "error"},
	}
	// ...dial once, then for each case Write then Read and assert env.Type == wantTyp...
	_ = cases
}
```

### 13.3 Testing concurrency & leaks **[A]**

Because the library allows concurrent writes, write a test that fires N goroutines writing to one connection and asserts frames arrive intact and complete (count them on the reader). For leak detection, record `runtime.NumGoroutine()` before opening/closing many connections and assert it returns to baseline (allow a small settle with `time.Sleep`/`runtime.Gosched`). Use the race detector: `go test -race ./...` — it will catch any accidental shared-state race in your hub.

---

## 14. Gotchas & Best Practices

The distilled list of things that bite people. Skim it now; return to it when something misbehaves.

| # | Gotcha | Why it hurts | Fix |
|---|---|---|---|
| 1 | No `defer c.CloseNow()` | Connection + goroutines leak on the error path | Always `defer c.CloseNow()` at the top; graceful `Close` on happy path |
| 2 | Two goroutines `Read`ing | Violates the one-reader rule; garbage/panics | Exactly one read-pump goroutine per connection (§5.2) |
| 3 | Blocking work in the read loop | Stalls all inbound processing for that client | Read → enqueue → return fast; do work elsewhere |
| 4 | Unbounded `send` channel | A slow client grows memory without limit | **Bounded** channel + non-blocking send + eviction (§6.2) |
| 5 | Forgetting `SetReadLimit` (or leaving default too high) | Peer forces huge allocations (DoS) | Set it as low as your protocol allows (§3.6) |
| 6 | Direct `c.Write` in the broadcast loop | One slow client blocks delivery to all | Enqueue to `send`; single write-pump does the I/O (§5.4) |
| 7 | `Ping` on a conn nobody reads | `Ping` hangs until ctx timeout | Ensure a reader is active, or use `CloseRead` for push-only (§5.5) |
| 8 | Using `context.Background()` for a read forever | Never times out; dead peers linger | Per-read `context.WithTimeout(ctx, pongWait)` (§6.4) |
| 9 | Assuming browser can send `Authorization` | It cannot | Token via subprotocol or short-TTL ticket (§8.3) |
| 10 | Long-lived JWT in the URL | Leaks into logs/history/referer | Subprotocol, or single-use ticket only |
| 11 | `InsecureSkipVerify: true` in prod | Disables CSWSH origin defense | `OriginPatterns` with exact hosts (§10.2) |
| 12 | Trusting a client `from`/`sender` field | Spoofed identity | Stamp identity from bound `c.userID` (§8.5) |
| 13 | Broadcasting before persisting | Ghost messages / history mismatch | Persist-then-broadcast (§9.3) |
| 14 | Double delivery with Redis backplane | Local fan-out **and** subscription both fire | Publish only; deliver via subscription on every node (§11.2) |
| 15 | Nginx default 60 s read timeout | Idle sockets die mid-session | `proxy_read_timeout 3600s` + app pings shorter (§11.4) |
| 16 | No panic recovery in pumps | One bad message crashes the whole server | `recover()` in every pump goroutine (§6.4) |
| 17 | Compressing secrets + attacker input | CRIME/BREACH size-oracle leak | Keep compression off by default; segregate secrets (§10.9) |
| 18 | `CloseStatus(err)` misread | -1 means "not a close frame," not code 0 | Handle -1 explicitly (§3.5) |
| 19 | Reusing the first token on reconnect | Fails after token expiry | Fetch a fresh token per reconnect attempt (§4.4) |
| 20 | Expecting Gin JSON middleware on WS | It's bypassed post-upgrade | Do WS logging/limits in the pumps (§7.2) |

**Best-practice summary:** one reader + one writer goroutine per connection; bounded outbound channel with eviction; context deadlines on every I/O; `wss://` + `OriginPatterns` + token-at-upgrade + per-message authz; `SetReadLimit` + validation + rate limits; persist-then-broadcast; Redis backplane with publish-only fan-out; graceful drain on shutdown; panic recovery and leak tests everywhere.

---

## 15. API Quick Reference

**Package:** `github.com/coder/websocket` (+ `github.com/coder/websocket/wsjson`)

**Functions**

| Signature | Purpose |
|---|---|
| `Accept(w, r, *AcceptOptions) (*Conn, error)` | Server handshake; blocks cross-origin by default |
| `Dial(ctx, url, *DialOptions) (*Conn, *http.Response, error)` | Client handshake; `http/https`→`ws/wss` |
| `NetConn(ctx, *Conn, MessageType) net.Conn` | Wrap a Conn as a `net.Conn` |
| `CloseStatus(err) StatusCode` | Peer's close code, or **-1** if not a CloseError |
| `Subprotocols(r *http.Request) []string` | Parse requested subprotocols from a handshake |

**Conn methods**

| Method | Concurrency | Notes |
|---|---|---|
| `Read(ctx) (MessageType, []byte, error)` | **One reader only** | Whole message; bounded by read limit |
| `Reader(ctx) (MessageType, io.Reader, error)` | One reader only | Stream a large message |
| `Write(ctx, MessageType, []byte) error` | **Safe concurrently** | Internally serialized |
| `Writer(ctx, MessageType) (io.WriteCloser, error)` | One writer open at a time | Stream out; `Close()` finalizes |
| `Ping(ctx) error` | Needs an active reader | Blocks for pong = liveness probe |
| `Close(code, reason) error` | Safe | Graceful close handshake; reason ≤123 B |
| `CloseNow() error` | Safe | Immediate; ideal `defer` cleanup |
| `CloseRead(ctx) context.Context` | — | Drains reads for push-only conns |
| `SetReadLimit(n int64)` | — | Default **32768 (32 KiB)** |
| `Subprotocol() string` | — | The negotiated subprotocol |

**AcceptOptions:** `Subprotocols []string`, `InsecureSkipVerify bool` (dev only), `OriginPatterns []string` (**CSWSH defense**), `CompressionMode`, `CompressionThreshold int`, `OnPingReceived func(ctx,[]byte) bool`, `OnPongReceived func(ctx,[]byte)`.

**DialOptions:** `HTTPClient *http.Client`, `HTTPHeader http.Header` (Go clients only), `Host string`, `Subprotocols []string`, `CompressionMode`, `CompressionThreshold int`, `OnPingReceived`, `OnPongReceived`.

**Types:** `MessageType` (`MessageText=1`, `MessageBinary=2`); `StatusCode` (1000–1015 + 3000–3999 framework + 4000–4999 private); `CloseError{Code, Reason}`; `CompressionMode` (`CompressionDisabled` default, `CompressionContextTakeover`, `CompressionNoContextTakeover`).

**wsjson:** `Read(ctx, *Conn, v any) error`, `Write(ctx, *Conn, v any) error`.

---

## 16. Study Path & Build-to-Learn Projects

Learning WebSockets sticks when you *build*. Progress through these; each adds exactly one hard concept.

**Phase 1 — Fundamentals [B].** Read §1–§4. Build the **echo server** (§3.2) and connect with both a Go `Dial` client and a browser `WebSocket`. Success = you can articulate the handshake, why one reader, and what `SetReadLimit`/`CloseStatus` do. Add graceful close on both ends.

**Phase 2 — The concurrency model [I].** Read §5. Prove to yourself that concurrent `Write` is safe (fire 100 goroutines writing, count intact frames on the reader) and that `Ping` hangs without a reader. Add ping/pong keepalive and idle read deadlines.

**Phase 3 — Authenticated chat room [I/A].** Build a single-node chat with the **Hub/Client pattern** (§6), mounted on **Gin** (§7). Add **JWT auth at the upgrade via subprotocol** and **Argon2id login** (§8), bounded send channels, and slow-client eviction. Success = an unauthenticated upgrade is rejected; a message to a room you didn't join is refused.

**Phase 4 — Presence, typing, persistence [A].** Add rooms, presence, and typing indicators (§6.6). Wire **Ent + pgx** and switch to **persist-then-broadcast** (§9). Add a `/api/messages/:room` history endpoint so reconnecting clients backfill. Use **Air** for iteration.

**Phase 5 — Scale to multiple nodes [A].** Introduce the **Redis pub/sub backplane** (§11): run two instances behind **Nginx** with the WebSocket proxy config, verify a message from a client on node A reaches a client on node B, and move presence into Redis. Kill a node and confirm clients reconnect and rejoin cleanly.

**Phase 6 — Harden & test [A].** Apply the full **security checklist** (§10): rate limits, connection caps, TLS, origin patterns, structured audit logging, graceful drain. Write the **round-trip, table-driven, concurrency, and leak tests** (§13) and run `go test -race`. Load-test with many concurrent connections and watch memory/goroutines.

**Capstone.** A production-shaped **real-time notifications + chat service** (§12): Gin + coder/websocket + JWT + Argon2 + Ent + pgx + Redis + Air, deployed behind Nginx, horizontally scaled, gracefully draining on deploy. If you can build, secure, and scale that, you have mastered production WebSockets in Go.

**Cross-references:** [Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md) (the other library + concurrency contrast) · [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) (auth machinery) · [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) (router) · [Go ent ORM](GO_ENT_ORM_GUIDE.md) (persistence) · [Redis](REDIS_GUIDE.md) (backplane/presence) · [Nginx](NGINX_GUIDE.md) (reverse proxy) · [Node WebSockets (50k+)](NODE_WEBSOCKETS_GUIDE.md) (the Node.js counterpart) · [Go gRPC](GO_GRPC_RPC_GUIDE.md) (when to prefer typed streaming) · [Go net/http REST](GO_NET_HTTP_REST_API_GUIDE.md) (the HTTP foundation).

---

> **You now have the full arc:** from *what a WebSocket frame is* to a *secure, authenticated, horizontally-scaled real-time backend*. The library's job is small and it does it well — safe concurrent writes, context-first I/O, correct framing. The architecture — one reader, one writer, bounded channels, auth at the edge, authz per message, persist-then-broadcast, a backplane to scale — is *yours*, and it is the same regardless of language. Build the capstone; that is where it becomes real.
