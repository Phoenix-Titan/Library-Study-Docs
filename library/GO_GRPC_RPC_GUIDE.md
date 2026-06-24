# gRPC & RPC with Go — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never made a remote procedure call" to "I can design, secure, test, and operate production gRPC microservices in Go." This is a **learn-offline** study document: every concept is explained in *prose first* — what it is, the logic and *why* it exists, what it is for and when to use it, how to use it, the key parameters/options, best practices, and **security recommendations** — and only *then* demonstrated with heavily-commented, runnable `.proto` and Go code you can paste into a module. Read it top-to-bottom once; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. No prior gRPC experience assumed; no internet required to learn from this document.
>
> **Version note:** This guide targets **Go 1.23 / 1.24** (current in 2026), **Protocol Buffers proto3**, and the current generation of modules: `google.golang.org/grpc` (v1.6x+) and `google.golang.org/protobuf` (v1.3x+, the "APIv2" runtime). It uses the modern codegen toolchain — `protoc` **or** [**buf**](https://buf.build), plus `protoc-gen-go` and `protoc-gen-go-grpc` — and the modern `grpc.NewClient` dialing API. It also covers **ConnectRPC** (`connectrpc.com/connect`) as a browser-friendly alternative that speaks gRPC, gRPC-Web, and its own Connect protocol. Fast-moving details are flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (paths, `.exe`, shells) are called out where they matter. Always confirm exact APIs at pkg.go.dev, protobuf.dev, buf.build, and connectrpc.com.
>
> **This guide's place in the library.** This is the definitive **RPC/gRPC** treatment. For the Go *language itself* (goroutines, channels, `context`, interfaces, generics, errors) see **`GO_GUIDE.md`** — gRPC leans on all of it and this guide assumes you can read Go. For **REST** alternatives and the trade-offs of REST-vs-gRPC, see **`GO_NET_HTTP_REST_API_GUIDE.md`** (the stdlib HTTP server) and **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** (the Gin framework, including file uploads — compare with gRPC client-streaming in §7). For JWT/Argon2 auth primitives reused by gRPC auth interceptors (§10–§11) see **`GO_JWT_ARGON2_GUIDE.md`**. For WebSockets (the other bidirectional-streaming option) see **`GO_GORILLA_WEBSOCKETS_GUIDE.md`**.

---

## Table of Contents

1. [RPC Fundamentals & the Standard Library `net/rpc`](#1-rpc-fundamentals--the-standard-library-netrpc) **[B]**
2. [gRPC & Protocol Buffers Overview — Why HTTP/2 & Binary](#2-grpc--protocol-buffers-overview--why-http2--binary) **[B]**
3. [REST vs gRPC — When to Use Each](#3-rest-vs-grpc--when-to-use-each) **[B/I]**
4. [Protocol Buffers (proto3) In Depth](#4-protocol-buffers-proto3-in-depth) **[B/I]**
5. [Toolchain Setup — protoc, Plugins & buf](#5-toolchain-setup--protoc-plugins--buf) **[B/I]**
6. [Building a gRPC Server in Go](#6-building-a-grpc-server-in-go) **[I]**
7. [The Four RPC Types End-to-End (Client + Server)](#7-the-four-rpc-types-end-to-end-client--server) **[I]**
8. [Metadata — Headers & Trailers](#8-metadata--headers--trailers) **[I]**
9. [Error Handling — status, codes & rich errors](#9-error-handling--status-codes--rich-errors) **[I]**
10. [Interceptors — gRPC's Middleware](#10-interceptors--grpcs-middleware) **[I/A]**
11. [Authentication & Security — TLS, mTLS, Tokens](#11-authentication--security--tls-mtls-tokens) **[I/A]**
12. [Deadlines, Cancellation, Retries, Keepalive & Load Balancing](#12-deadlines-cancellation-retries-keepalive--load-balancing) **[I/A]**
13. [Interoperability — gRPC-Gateway, gRPC-Web, ConnectRPC](#13-interoperability--grpc-gateway-grpc-web-connectrpc) **[A]**
14. [Health Checking & Reflection](#14-health-checking--reflection) **[I]**
15. [Testing gRPC — bufconn, Mocks & Streams](#15-testing-grpc--bufconn-mocks--streams) **[I/A]**
16. [Observability — OpenTelemetry, Prometheus, Logging](#16-observability--opentelemetry-prometheus-logging) **[A]**
17. [Performance — Message Size, Compression, Reuse](#17-performance--message-size-compression-reuse) **[A]**
18. [Deployment — Load Balancers, Docker, Graceful Shutdown](#18-deployment--load-balancers-docker-graceful-shutdown) **[A]**
19. [Gotchas & Best Practices](#19-gotchas--best-practices) **[A]**
20. [Study Path & Build-to-Learn Projects](#20-study-path--build-to-learn-projects)

---

## 1. RPC Fundamentals & the Standard Library `net/rpc`

### 1.1 What RPC is, and the logic behind it **[B]**

**Remote Procedure Call (RPC)** is a programming model where calling a function on a *remote* machine looks (almost) like calling a *local* function. You invoke `client.GetUser(ctx, id)` and the framework handles everything messy underneath: serializing the arguments into bytes, sending those bytes over a network connection, dispatching to the right function on the server, serializing the return value, and handing the result back to you as if nothing remote ever happened.

**Why does this model exist?** The idea is old (1980s) but enduring because of a single design goal: **make the network disappear behind a function call.** Distributed systems are made of many programs that need to ask each other to *do things*. The two dominant ways to express "ask another program to do a thing" are (a) **manipulate a resource over HTTP** (the REST mental model — `GET /users/1`) and (b) **call a procedure** (the RPC mental model — `GetUser(1)`). RPC is the more direct expression when the thing you want is fundamentally *an action*, not *a document*.

Every RPC system, no matter the era or language, must do **four jobs**. Understanding these four makes every framework — including gRPC — feel inevitable rather than magical:

1. **Serialization (a.k.a. marshaling):** turn in-memory data structures into a flat sequence of bytes, and turn bytes back into structures. (gRPC uses Protocol Buffers; `net/rpc` uses `gob` or JSON.)
2. **Transport:** move those bytes across a connection — TCP, HTTP/1.1, HTTP/2, Unix sockets, etc. (gRPC uses HTTP/2.)
3. **Dispatch:** on the server, route an incoming request to the correct handler function.
4. **Contract:** both sides must agree on method names, argument shapes, and return shapes. (gRPC makes this contract an explicit, versioned `.proto` file.)

**The crucial caveat — "almost like a local call."** The word *almost* is where all the engineering judgment lives. A local call cannot fail to reach its callee; a remote one can be **slow**, **fail partway**, **time out**, be **reordered**, or be **retried** (possibly executing twice). A *good* RPC framework gives you first-class tools for these realities — deadlines, cancellation, typed error codes, retries, back-pressure — while a *naive* one pretends the network is reliable and falls over under load. gRPC is firmly in the "good" category, which is much of why this guide is mostly about gRPC.

### 1.2 The Go standard library: `net/rpc` **[B]**

Before gRPC, Go shipped its own RPC package: `net/rpc`. It is **still in the standard library**, still works, and is worth ~20 minutes because it strips RPC to its essentials and demystifies what gRPC does under the hood. It is, however, **frozen** and **Go-specific** by default — its native wire format is `encoding/gob`, which only Go programs read (there is also `net/rpc/jsonrpc` for a language-neutral JSON-RPC 1.0 codec). **Why learn it if you'll ship gRPC?** It teaches the four jobs above with almost no ceremony, and seeing what it *lacks* (no streaming, deadlines, error codes, or cross-language story) motivates every feature gRPC adds. Think of it as the "hello world" of the concept.

#### The rules for an `net/rpc` method

A method is eligible to be exposed over RPC only if it satisfies a rigid signature. The framework uses reflection to find methods matching this exact shape:

```
func (t *T) MethodName(argType T1, replyType *T2) error
```

1. The method's receiver type (`T`) is **exported** (capitalized).
2. The method itself is **exported**.
3. It has exactly **two arguments**, both **exported** (or builtin) types.
4. The **second argument is a pointer** — this is the *reply*, which the method fills in (output parameters, because Go has no native "out" concept and RPC wants one allocation site).
5. The method returns an `error`. A non-nil error is transmitted to the client as a plain string.

#### Complete `net/rpc` example — server

```go
// file: rpcserver/main.go
package main

import (
	"errors"
	"log"
	"net"
	"net/http"
	"net/rpc"
)

// Args is the argument struct. Fields MUST be exported (capitalized) so that
// encoding/gob can serialize them across the wire — gob skips unexported fields.
type Args struct {
	A, B int
}

// Quotient is a richer reply type, included to show structs flow both ways.
type Quotient struct {
	Quo, Rem int
}

// Arith is the receiver type whose methods we expose over RPC. It carries no
// state here, but in a real service it could hold dependencies (DB handles, etc.).
type Arith int

// Multiply matches the required signature:  Method(args T1, reply *T2) error
// args is passed by value; reply is a pointer we fill in.
func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil // returning a non-nil error sends that error TEXT (no code) to the client
}

// Divide demonstrates returning an application error.
func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		// This string is transmitted to the client and surfaces there as a plain
		// *errors.errorString — there is NO error *code*, just text. That is a key
		// limitation gRPC fixes with its status/codes model (§9).
		return errors.New("divide by zero")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}

func main() {
	arith := new(Arith)

	// Register exposes all eligible methods of *Arith under the name "Arith".
	if err := rpc.Register(arith); err != nil {
		log.Fatal("register:", err)
	}

	// Two common transports: (a) raw TCP with the gob codec, or (b) over HTTP
	// (handy for sharing a port / debugging). We use HTTP here. HandleHTTP wires
	// RPC onto the default HTTP mux at the conventional /_goRPC_ path.
	rpc.HandleHTTP()

	ln, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("listen:", err)
	}
	log.Println("net/rpc server listening on :1234")

	// http.Serve drives the accept loop; each connection is handled concurrently
	// in its own goroutine — concurrency you get for free from the net/http server.
	log.Fatal(http.Serve(ln, nil))
}
```

#### Complete `net/rpc` example — client

```go
// file: rpcclient/main.go
package main

import (
	"fmt"
	"log"
	"net/rpc"
)

type Args struct{ A, B int }
type Quotient struct{ Quo, Rem int }

func main() {
	// DialHTTP connects to the RPC service registered with HandleHTTP.
	client, err := rpc.DialHTTP("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}
	defer client.Close()

	// --- Synchronous call ---
	// "Arith.Multiply" is "<registered name>.<method>". Note there is no context,
	// no deadline, no cancellation — a hung server hangs the client forever.
	args := &Args{A: 7, B: 8}
	var product int
	if err := client.Call("Arith.Multiply", args, &product); err != nil {
		log.Fatal("Arith.Multiply error:", err)
	}
	fmt.Printf("7 * 8 = %d\n", product) // -> 56

	// --- Asynchronous call ---
	// Go() returns immediately; the result lands on the Done channel.
	quot := new(Quotient)
	divCall := client.Go("Arith.Divide", &Args{A: 17, B: 5}, quot, nil)
	replyCall := <-divCall.Done // block until the async call completes
	if replyCall.Error != nil {
		log.Fatal("Arith.Divide error:", replyCall.Error)
	}
	fmt.Printf("17 / 5 = %d remainder %d\n", quot.Quo, quot.Rem) // -> 3 r 2

	// --- Error path ---
	err = client.Call("Arith.Divide", &Args{A: 1, B: 0}, new(Quotient))
	fmt.Println("divide-by-zero call returned:", err) // -> divide by zero (just a string)
}
```

#### JSON-RPC variant (`net/rpc/jsonrpc`)

If you want a language-neutral wire format, swap the *codec*. The server registers methods identically but serves each connection with `jsonrpc.NewServerCodec`. This shows the **serialization** job (#1 above) is pluggable, independent of transport and dispatch:

```go
// Server side: accept raw TCP and serve each conn with the JSON codec.
ln, _ := net.Listen("tcp", ":1234")
for {
	conn, err := ln.Accept()
	if err != nil {
		continue
	}
	// One goroutine per connection, JSON-RPC 1.0 framing over the conn.
	go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
}

// Client side:
conn, _ := net.Dial("tcp", "localhost:1234")
client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
var product int
_ = client.Call("Arith.Multiply", &Args{A: 7, B: 8}, &product)
```

The JSON on the wire is human-readable (which is its appeal):

```json
{"method":"Arith.Multiply","params":[{"A":7,"B":8}],"id":0}
{"id":0,"result":56,"error":null}
```

**Security note for `net/rpc`:** it has **no built-in authentication, authorization, or transport security**. The gob decoder also executes against attacker-controlled input, so never expose a `net/rpc` server to an untrusted network. If you must use it beyond a toy, wrap the listener in TLS (`tls.Listen`) and do auth yourself. This DIY-security burden is another reason production systems moved to gRPC, which has credentials and interceptors built in (§10–§11).

### 1.3 Why gRPC superseded `net/rpc` **[B]**

`net/rpc` is elegant and tiny, but for modern distributed systems it falls short on nearly every production axis. The table below is the entire motivation for the rest of this guide:

| Concern | `net/rpc` | gRPC |
|---|---|---|
| Cross-language | No (gob); JSON-RPC is weakly typed | Yes — generated stubs for 10+ languages |
| Schema / contract | Implicit (Go structs) | Explicit, versioned `.proto` (IDL) |
| Streaming | No | 4 first-class modes (§7) |
| Deadlines / cancellation | No `context` support | `context.Context` everywhere (§12) |
| Error model | Plain string | Rich `status` + `codes` + details (§9) |
| Metadata / headers | No | First-class metadata (§8) |
| Interceptors / middleware | No | Unary + stream interceptors (§10) |
| Transport | gob over TCP/HTTP/1.1 | HTTP/2, multiplexed, flow-controlled |
| Auth / TLS | DIY | Built-in credentials, mTLS, per-RPC creds (§11) |
| Tooling | None | reflection, grpcurl, gateway, codegen (§5,§14) |
| Maintenance | Frozen | Actively developed CNCF project |

In one sentence: **`net/rpc` teaches you the *concept*; gRPC is what you *ship*.** Everything from here on is gRPC.

---

## 2. gRPC & Protocol Buffers Overview — Why HTTP/2 & Binary

### 2.1 What gRPC is **[B]**

**gRPC** (officially "gRPC Remote Procedure Calls" — a playfully recursive acronym) is a high-performance, open-source RPC framework originally from Google and now a CNCF project. Conceptually it is just "the four RPC jobs, done extremely well," resting on **three pillars**:

1. **Protocol Buffers** as the Interface Definition Language (IDL) *and* the serialization format. One `.proto` file is the contract.
2. **HTTP/2** as the transport — chosen for multiplexing, streaming, and header compression.
3. **Code generation** that produces strongly typed client *and* server stubs from your `.proto`, so calling a remote method is as type-safe as calling a local one.

**The logic / why this combination.** Google needed service-to-service comms that were fast (millions of RPCs/sec), polyglot, evolvable (schemas change weekly without breaking callers), and streaming-capable. No single standard delivered all four, so gRPC bundles best-in-class answers: Protobuf for compact typed payloads + evolvable schema, HTTP/2 for transport, and codegen for type safety + polyglot.

### 2.2 Why HTTP/2? **[B]**

HTTP/2 is the substrate that makes gRPC fast and streaming-capable. Each property maps to a concrete gRPC benefit:

- **Multiplexing:** many concurrent RPCs share *one* TCP connection without HTTP-layer head-of-line blocking — parallelism with no connection pool; a single `ClientConn` handles thousands of in-flight calls (§6, §17).
- **Binary framing:** binary, not text, so it is compact and fast to parse.
- **Header compression (HPACK):** repeated headers (metadata, §8) cost far fewer bytes.
- **Bidirectional streams:** HTTP/2 streams map *directly* onto gRPC's four streaming modes (§7) — why gRPC does real-time bidi while plain HTTP/1.1 cannot.
- **Flow control:** per-stream and per-connection back-pressure stops a fast sender drowning a slow receiver. Streaming gets this for free.

On the wire, a gRPC call is an HTTP/2 `POST` to a path like `/package.Service/Method`, with **length-prefixed protobuf messages** in the body and gRPC-specific headers/trailers (`content-type: application/grpc`, `grpc-status`, `grpc-encoding`, etc.). The final status arrives as an HTTP/2 *trailer* — which is precisely why browsers (whose `fetch` API can't read trailers) need a bridge (§13).

### 2.3 Why Protocol Buffers? **[B]**

Protobuf is the serialization format and the schema language. It buys you:

- **Compactness:** binary encoding is typically **3–10× smaller** than equivalent JSON. Field *numbers*, not names, go on the wire.
- **Speed:** parsing is fast and allocation-light compared to JSON.
- **A strict, evolvable schema:** the contract is explicit and machine-checkable, and the field-number design enables *safe schema evolution* (§4.10) — the single most important operational property.
- **Polyglot codegen:** one `.proto`, stubs in Go, Java, Python, C++, Rust, TypeScript, Kotlin, and more — all guaranteed to agree on the wire format.

**The trade-off (be honest about it):** payloads are not human-readable, and you need codegen tooling installed. For internal services that is a great deal; for a public, "just curl it" API it is why teams add a JSON edge (§13). This trade-off is exactly the REST-vs-gRPC decision covered next in §3.

### 2.4 The four RPC types (the conceptual core) **[B]**

This is the heart of gRPC. Every method you define is exactly one of four kinds, chosen by where you put the `stream` keyword in the `.proto`:

| Type | Client sends | Server sends | Go call shape | Example |
|---|---|---|---|---|
| **Unary** | 1 message | 1 message | `Get(ctx, req) (resp, err)` | `GetUser` |
| **Server streaming** | 1 message | stream | `List(ctx, req) (stream, err)` | `ListUpdates` |
| **Client streaming** | stream | 1 message | `Upload(ctx) (stream, err)` | `UploadChunks` |
| **Bidirectional streaming** | stream | stream | `Chat(ctx) (stream, err)` | `Chat` |

```proto
service Demo {
  rpc Unary(Req) returns (Resp);                    // unary
  rpc ServerStream(Req) returns (stream Resp);      // server streaming
  rpc ClientStream(stream Req) returns (Resp);      // client streaming
  rpc BidiStream(stream Req) returns (stream Resp); // bidirectional
}
```

We implement all four fully, server *and* client, in §6 and §7.

### 2.5 The contract-first workflow **[B]**

gRPC is **schema-first** (a.k.a. contract-first). The canonical development loop is:

1. **Write `.proto`** — define messages and services. This file is the *source of truth* and the contract between teams.
2. **Generate code** — `protoc`/`buf` produces Go types (`*.pb.go`) and service stubs (`*_grpc.pb.go`).
3. **Implement the server** — fill in the generated service interface.
4. **Use the client stub** — call methods as if they were local functions.
5. **Evolve safely** — change the `.proto` following the compatibility rules (§4.10); regenerate; CI enforces compatibility.

The `.proto` lives in version control and is often shared via a **schema registry** (e.g. the Buf Schema Registry) or a shared repo so multiple services and languages consume the same contract. This "the schema is the API" discipline is a major cultural difference from REST, where the contract is frequently just convention plus documentation.

---

## 3. REST vs gRPC — When to Use Each

This decision comes up constantly, so it gets its own section. Cross-reference the REST guides — **`GO_NET_HTTP_REST_API_GUIDE.md`** and **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** — for the REST side in depth; here we focus on *choosing*.

### 3.1 The mental-model difference **[B]**

REST and gRPC answer "how does program A ask program B to do something?" with two metaphors:

- **REST** = *manipulate a resource.* You think in nouns (URLs) and a fixed set of HTTP verbs. "Get user 1" is `GET /users/1`. The protocol's verbs *are* the API's verbs.
- **gRPC** = *call a procedure.* You think in methods of arbitrary name and shape. "Get user 1" is `GetUser(GetUserRequest{id: 1})`. The API defines its own verbs; HTTP is plumbing you never see.

Neither is "more correct." CRUD over well-defined resources maps cleanly to REST; rich domain actions (`TransferFunds`, `RebalancePortfolio`, `StreamTelemetry`) map cleanly to RPC.

### 3.2 The comparison table **[B/I]**

| Dimension | gRPC | REST | GraphQL (for context) |
|---|---|---|---|
| Mental model | Call a function | Manipulate a resource | Query a graph; pick exact fields |
| Contract | Strongly typed `.proto` (enforced) | Convention + OpenAPI (optional) | Strongly typed SDL |
| Payload | Binary Protobuf (compact, opaque) | Usually JSON (verbose, readable) | JSON |
| Transport | HTTP/2 (multiplexed, streaming) | HTTP/1.1 or HTTP/2 | Usually HTTP/1.1 POST |
| Streaming | First-class (4 modes) | SSE / chunked (awkward) | Subscriptions (over WS) |
| Browser support | Needs gRPC-Web/Connect bridge (§13) | Native | Native |
| Discoverability | Reflection + generated stubs (§14) | URLs + docs | Introspection |
| Caching | Manual / none | HTTP caching built-in (big advantage) | Hard |
| Human-debuggable | `grpcurl` (not raw `curl`) | `curl` anything | GraphQL playground |
| Code generation | Yes, mandatory, polyglot | Optional (OpenAPI generators) | Yes |
| Best for | Internal microservices, low latency, polyglot, streaming | Public APIs, simple CRUD, cacheable reads | Aggregating many sources, client-driven fields |

### 3.3 Rules of thumb — when to pick which **[I]**

- **Reach for gRPC when** you control *both ends*, need low latency / high throughput, want streaming, work across multiple languages, and value a strict enforced contract. This is the classic **service-to-service** ("east-west") traffic inside a system.
- **Reach for REST when** the API is *public-facing* or consumed by browsers and third parties, where ubiquity, HTTP caching, and "anyone can curl it" matter more than raw performance. This is classic **north-south** edge traffic. See the REST guides for building these.
- **Performance reality check:** gRPC's wins (binary payloads, HTTP/2 multiplexing, no per-request connection setup) are real but matter most at scale or under latency pressure. For a low-traffic internal CRUD service, REST's simplicity and tooling may win on *total* engineering cost. Measure; don't cargo-cult.
- **They are not mutually exclusive.** A very common, recommended topology: **gRPC between internal services, with a gRPC-Gateway or ConnectRPC edge (§13) exposing REST/JSON** for browsers and partners. One `.proto` contract, two surfaces — you get gRPC's internal performance and REST's external reach without writing the contract twice.

**Security framing of the choice:** A public REST edge is your trust boundary — validate aggressively, authenticate every request, rate-limit (see the REST guides + `GO_JWT_ARGON2_GUIDE.md`). Internal gRPC traffic should still be authenticated (mTLS, §11) under a zero-trust posture — "internal" is not a synonym for "trusted."

---

## 4. Protocol Buffers (proto3) In Depth

Protobuf is the contract language. This section is the reference for proto3 syntax — what each construct is, why it exists, how it maps to Go, and the rules that keep your schema evolvable.

### 4.1 Why a schema / IDL at all? **[B]**

Before syntax, the *why*. An **Interface Definition Language** is a single, language-neutral description of your data and methods that both sides compile against. The payoff:

- **One source of truth.** The `.proto` is the contract; client and server can't drift because they generate from the same file.
- **Type safety across the wire.** A typo in a field name is a compile error, not a 2 a.m. production surprise.
- **Polyglot by construction.** Generate Go on the server, TypeScript in the browser, Python in a data job — all provably agree.
- **Safe evolution.** The schema language is *designed* so you can add fields without breaking old code (§4.10).

JSON-over-REST can approximate this with OpenAPI, but it is opt-in and rarely fully enforced. With gRPC the schema is *mandatory*, which is a feature.

### 4.2 Anatomy of a `.proto` file **[B]**

```proto
// Every proto3 file MUST declare the syntax on line 1 (comments above it are fine,
// but the syntax statement must be the first non-comment line).
syntax = "proto3";

// The package namespaces messages/services to avoid name clashes across files.
// It becomes part of the fully-qualified name: e.g. user.v1.GetUserRequest.
// Convention: include a version segment (user.v1) — see §4.10.
package user.v1;

// go_package controls the Go import path AND package name of generated code.
// Format: "<import path>;<package name>". The part after ';' is optional but
// recommended for clarity. ALWAYS set this for Go, or codegen guesses badly.
option go_package = "github.com/example/app/gen/user/v1;userv1";

// Pull in well-known types and other protos (§4.8).
import "google/protobuf/timestamp.proto";

// A message is a struct: a typed collection of fields.
message User {
	string id = 1;          // field number 1 — see §4.4
	string email = 2;
	string display_name = 3;
	google.protobuf.Timestamp created_at = 4;
}
```

**Key parameters explained:** `syntax` selects the language dialect (`proto3` is current; `proto2` is legacy). `package` prevents name collisions and is part of the on-wire method path (`/user.v1.UserService/GetUser`). `option go_package` is *the* setting Go developers forget — without it the generated import path is wrong. `import` makes types from other files visible.

### 4.3 Scalar types and their Go mappings **[B]**

```proto
message AllScalars {
	double  d  = 1;   // float64
	float   f  = 2;   // float32
	int32   i  = 3;   // int32   (varint; inefficient for negatives)
	int64   l  = 4;   // int64
	uint32  u  = 5;   // uint32
	uint64  ul = 6;   // uint64
	sint32  si = 7;   // int32   (zig-zag; efficient for negatives)
	sint64  sl = 8;   // int64   (zig-zag)
	fixed32 x  = 9;   // uint32  (always 4 bytes; efficient for large values)
	fixed64 y  = 10;  // uint64  (always 8 bytes)
	bool    b  = 11;  // bool
	string  s  = 12;  // string  (MUST be valid UTF-8)
	bytes   by = 13;  // []byte  (arbitrary binary, no UTF-8 requirement)
}
```

| proto3 type | Go type | When to use it |
|---|---|---|
| `double` / `float` | `float64` / `float32` | Real numbers; prefer `double` unless size matters |
| `int32` / `int64` | `int32` / `int64` | Counts/IDs that are usually positive |
| `uint32` / `uint64` | `uint32` / `uint64` | Never-negative quantities |
| `sint32` / `sint64` | `int32` / `int64` | Numbers that are frequently negative (zig-zag saves bytes) |
| `fixed32`/`fixed64` | `uint32`/`uint64` | Large numbers / hashes where varint wouldn't help |
| `bool` | `bool` | Flags |
| `string` | `string` | Human text (UTF-8 only) |
| `bytes` | `[]byte` | Binary blobs, encrypted data, hashes |

**Default values (proto3) — a critical gotcha:** unset scalar fields take the zero value (`0`, `false`, `""`, empty slice). Crucially, proto3 by default **cannot distinguish "field explicitly set to zero" from "field unset"** for scalars. If that distinction matters (e.g. a PATCH that sets `age` to 0 vs. leaves it untouched), you need `optional` (§4.7).

### 4.4 Field numbers — the heart of wire compatibility **[B]**

```proto
message Example {
	string name = 1;   // The "= 1" is the FIELD NUMBER (a.k.a. tag), NOT a default value.
	int32  age  = 2;
}
```

This is the single most important Protobuf concept for long-lived systems:

- **Field numbers, not names, define the wire format.** You may rename a field freely (it only affects source code); you must **never** reuse or change a field number for an existing field.
- **Valid range:** **1 to 536,870,911**, excluding **19,000–19,999** (reserved for protobuf internals).
- **Numbers 1–15 use a single byte** for the field tag; reserve them for the most frequently set fields. 16–2047 use two bytes.

The logic: because the *number* is the identity, you can rename, reorder, or document fields without touching the bytes, and old/new code interoperate as long as numbers are stable. This is the foundation of §4.10.

### 4.5 Messages, nested messages, and enums **[B/I]**

```proto
syntax = "proto3";
package shop.v1;
option go_package = "github.com/example/app/gen/shop/v1;shopv1";

// Enums: the first value MUST be 0 and is the default. Convention: a zero-valued
// *_UNSPECIFIED member so "unset" is distinguishable from a real, deliberate choice.
enum OrderStatus {
	ORDER_STATUS_UNSPECIFIED = 0;
	ORDER_STATUS_PENDING     = 1;
	ORDER_STATUS_PAID        = 2;
	ORDER_STATUS_SHIPPED     = 3;
	ORDER_STATUS_CANCELLED   = 4;
}

message Order {
	string id = 1;
	OrderStatus status = 2;

	// A nested message type — scoped to Order. In Go this becomes Order_LineItem.
	message LineItem {
		string sku = 1;
		int32  quantity = 2;
		int64  unit_price_cents = 3;
	}

	// repeated = a list/slice. In Go: []*Order_LineItem.
	repeated LineItem items = 3;
}
```

**Why the `*_UNSPECIFIED = 0` convention?** Because proto3 enums default to their zero value, a zero must mean "I didn't say." If `0` were a meaningful state like `PENDING`, you could never tell "explicitly pending" from "field never set." Always make `0` mean "unspecified."

**⚡ Version note:** Enum values in proto3 are **open** by default — an unknown numeric value (e.g. a new status from a newer client) is *preserved on round-trip* rather than rejected. Prefix members with the enum name (`ORDER_STATUS_`) because enum value names share a C++-style scope and would otherwise collide across enums in the same package.

### 4.6 `repeated`, `map`, and `oneof` **[B/I]**

```proto
message Catalog {
	repeated string tags = 1;             // Go: []string
	map<string, int64> stock_by_sku = 2;  // Go: map[string]int64
}
```

- **`repeated`** is a list/slice. Conceptually unordered but preserves insertion order in practice.
- **`map<K,V>`** keys may be any integral or string type (not float/bytes/message); values may be anything except another map. Maps cannot themselves be `repeated`.

**`oneof` — at most one of a set.** Use this for tagged unions / "exactly one of these alternatives":

```proto
message Payment {
	string id = 1;

	// Exactly zero or one of the fields below may be set. Setting one clears the
	// others. Field numbers must be unique within the WHOLE message, not just the oneof.
	oneof method {
		Card   card  = 2;
		Bank   bank  = 3;
		string token = 4;
	}
}

message Card { string last4 = 1; }
message Bank { string iban  = 1; }
```

In Go, a `oneof` generates an interface field plus wrapper types; you `switch` on it:

```go
switch m := payment.Method.(type) {
case *shopv1.Payment_Card:
	fmt.Println("card ending", m.Card.Last4)
case *shopv1.Payment_Bank:
	fmt.Println("bank", m.Bank.Iban)
case *shopv1.Payment_Token:
	fmt.Println("token", m.Token)
case nil:
	fmt.Println("no method set")
}
```

**Best practice:** prefer `oneof` over `Any` (§4.9) whenever the set of alternatives is known and small — it keeps full type safety.

### 4.7 `optional` — explicit presence for scalars **[I]**

```proto
message Profile {
	string name = 1;          // proto3 default: "" is indistinguishable from unset
	optional int32 age = 2;   // generates a *int32 in Go -> nil means "not set"
}
```

`optional` restores **field presence** for scalars. In Go the field becomes a pointer (`*int32`): `nil` means "not provided," `proto.Int32(0)` means "explicitly zero." Use it whenever the difference between *absent* and *zero* matters — PATCH-style partial updates, nullable database columns, "did the client send this flag?".

```go
import "google.golang.org/protobuf/proto"

p := &userv1.Profile{Age: proto.Int32(0)} // explicitly zero
if p.Age != nil {
	fmt.Println("age was provided:", *p.Age)
}
```

**⚡ Version note:** With protobuf "editions," `optional` is fully standard in proto3. Message-typed fields *always* have presence (they're pointers regardless of `optional`), so `optional` only changes behavior for scalars/enums/strings/bytes.

### 4.8 Well-known types (WKT) **[I]**

These are pre-defined messages under `google/protobuf/`. Import and reuse them rather than reinventing time, duration, or "no arguments."

| WKT | Import | Go type | Use |
|---|---|---|---|
| `Timestamp` | `google/protobuf/timestamp.proto` | `*timestamppb.Timestamp` | Points in time |
| `Duration` | `google/protobuf/duration.proto` | `*durationpb.Duration` | Spans of time |
| `Empty` | `google/protobuf/empty.proto` | `*emptypb.Empty` | "no request" / "no response" |
| `StringValue`, `Int32Value`, … | `google/protobuf/wrappers.proto` | `*wrapperspb.StringValue` | Nullable scalars (pre-`optional` idiom) |
| `Struct`, `Value`, `ListValue` | `google/protobuf/struct.proto` | `*structpb.Struct` | Arbitrary JSON-like data |
| `Any` | `google/protobuf/any.proto` | `*anypb.Any` | A packed message of unknown-at-compile-time type |
| `FieldMask` | `google/protobuf/field_mask.proto` | `*fieldmaskpb.FieldMask` | Partial updates (which fields to touch) |

```proto
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";

message Job {
	string id = 1;
	google.protobuf.Timestamp scheduled_at = 2;
	google.protobuf.Duration  timeout      = 3;
}

service Jobs {
	// Empty request: a method that takes no parameters (and/or returns nothing).
	rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

Converting in Go is via helper constructors/accessors:

```go
import (
	"time"

	"google.golang.org/protobuf/types/known/durationpb"
	"google.golang.org/protobuf/types/known/timestamppb"
)

job := &jobsv1.Job{
	ScheduledAt: timestamppb.New(time.Now().Add(time.Hour)), // time.Time -> Timestamp
	Timeout:     durationpb.New(30 * time.Second),           // time.Duration -> Duration
}
t := job.ScheduledAt.AsTime()    // Timestamp -> time.Time
d := job.Timeout.AsDuration()    // Duration  -> time.Duration
```

### 4.9 `Any` — packing arbitrary messages **[A]**

```go
import "google.golang.org/protobuf/types/known/anypb"

// Pack a concrete message into an Any (records its type URL alongside the bytes).
packed, err := anypb.New(&userv1.User{Id: "u1"})

// Unpack: you must know (or check) the target type.
var u userv1.User
if err := packed.UnmarshalTo(&u); err == nil {
	fmt.Println("got user", u.Id)
}
```

`Any` lets a field hold *any* message type, resolved at runtime via a type URL. It is powerful but **erodes type safety** and is harder to evolve — prefer `oneof` when the set of possibilities is known and small. Legitimate uses: plugin systems, the `details` field of error statuses (§9), and generic event envelopes.

### 4.10 Schema evolution — backward & forward compatibility **[I/A]**

This is the single most important *operational* topic in Protobuf. Definitions: **backward compatible** = new code reads old data; **forward compatible** = old code reads new data. Protobuf is *designed* to give you both — if you follow the rules. The mechanism is §4.4: field numbers are the wire identity, and unknown fields are preserved.

**Safe (compatible) changes:**

- ✅ **Add** a new field with a new, unused field number. Old readers ignore it (it is preserved as an "unknown field" and re-serialized intact); new readers see the default if it's absent.
- ✅ **Rename** a field or message (names aren't on the wire — but you'll break *source code* that references the old name).
- ✅ **Add** new enum values (readers treat unknown values as the raw number — that's why enums are "open").
- ✅ **Add** new methods to a service.
- ✅ **Convert** between certain compatible scalar types (`int32`/`int64`/`uint32`/`uint64`/`bool` are interchangeable on the wire — but watch truncation/sign).

**Unsafe (breaking) changes — never do these to a deployed schema:**

- ❌ **Change a field's number.** It becomes a different field entirely; data is lost/misread.
- ❌ **Reuse a field number** previously used for a different field/type. Old data is reinterpreted as the new type — silent corruption.
- ❌ **Change a field's type** incompatibly (e.g. `string` ↔ `int32`).
- ❌ **Move** a field into/out of a `oneof` carelessly, or change `repeated` ↔ scalar.
- ❌ **Delete** a field and later reuse its number.

**When you remove a field, `reserved` its number and name** so nobody can ever reuse them:

```proto
message User {
	reserved 5, 9 to 11;       // reserve removed FIELD NUMBERS
	reserved "legacy_token";   // reserve removed FIELD NAMES
	string id = 1;
	// ... field number 5 is now permanently retired
}
```

**Best practice:** automate enforcement with `buf breaking` (§5) in CI so an incompatible change *cannot* merge. Manual discipline always eventually fails; a CI gate doesn't.

### 4.11 JSON mapping (proto3 ⇄ JSON) **[I]**

Protobuf defines a canonical JSON mapping, used by gRPC-Gateway, ConnectRPC's JSON mode, and the `protojson` package. This is what makes a REST/JSON edge (§13) possible from the same schema.

| proto | JSON | Notes |
|---|---|---|
| field `display_name` | `"displayName"` | Defaults to **lowerCamelCase** (original name also accepted on input) |
| `int64` / `uint64` | string | To avoid JS precision loss: `"9007199254740993"` |
| `bytes` | base64 string | |
| `Timestamp` | RFC 3339 string | `"2026-06-24T10:00:00Z"` |
| `Duration` | string with `s` | `"3.5s"` |
| `enum` | string name | `"ORDER_STATUS_PAID"` (number also accepted on input) |
| unset fields | omitted | unless you ask to emit defaults |

```go
import "google.golang.org/protobuf/encoding/protojson"

b, _ := protojson.Marshal(user)                 // proto -> JSON bytes
var u userv1.User
_ = protojson.Unmarshal(b, &u)                  // JSON bytes -> proto

// Options worth knowing:
m := protojson.MarshalOptions{
	EmitUnpopulated: true, // include zero-valued fields explicitly (predictable JSON)
	UseProtoNames:   true, // snake_case keys instead of camelCase
	Indent:          "  ", // pretty-print
}
pretty, _ := m.Marshal(user)
_ = pretty
```

---

## 5. Toolchain Setup — protoc, Plugins & buf

To turn `.proto` into Go you need three things: (1) a protobuf **compiler**, (2) the Go **plugins**, and (3) a way to **invoke** them. There are two mainstream paths — classic `protoc`, and the modern `buf`. **Use `buf` for new projects**; understand `protoc` because it's everywhere and `buf` runs the same plugins under the hood.

### 5.1 What gets generated **[B]**

Before tooling, know the output so the rest makes sense. From `user.proto` you get **two** Go files:

- **`user.pb.go`** — the **message** types: a Go struct per `message`, with getters (`GetId()`), `oneof` wrappers, enum constants, and the reflection/marshaling machinery. Produced by `protoc-gen-go`.
- **`user_grpc.pb.go`** — the **service** stubs: a `XxxServer` interface (you implement it), a `RegisterXxxServer` function (you call it), a `XxxClient` interface + constructor (you call methods on it), and the stream types. Produced by `protoc-gen-go-grpc`.

You never hand-edit these; you regenerate them whenever the `.proto` changes.

### 5.2 Installing the Go plugins **[B]**

These are Go programs that `protoc`/`buf` invoke as plugins. Install with `go install`:

```bash
# protoc-gen-go: generates message types (*.pb.go) from the protobuf APIv2 runtime.
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# protoc-gen-go-grpc: generates the gRPC service stubs (*_grpc.pb.go).
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Ensure $(go env GOPATH)/bin is on PATH so protoc/buf can find the plugins.
# On Windows (Git Bash): the path is e.g. C:/Users/you/go/bin
export PATH="$PATH:$(go env GOPATH)/bin"
```

**⚡ Version note:** `protoc-gen-go` (from `google.golang.org/protobuf`) and `protoc-gen-go-grpc` (from `google.golang.org/grpc`) are **separate binaries from separate modules**, versioned independently. Pin their versions (via `buf`'s remote plugins, or a `tools.go`, or your CI image) so every developer and CI machine generates *byte-identical* code — otherwise diffs churn meaninglessly.

### 5.3 Installing `protoc` **[B]**

`protoc` is a native C++ binary. Download a release from the protobuf GitHub releases (`protoc-<version>-<os>.zip`), unzip, and put `bin/protoc` on your PATH. On Windows, that's `protoc.exe`. The archive bundles the well-known type `.proto` files under `include/`.

```bash
protoc --version          # e.g. libprotoc 28.x
which protoc-gen-go        # should resolve to your GOPATH/bin
```

### 5.4 The `protoc` command line **[B/I]**

```bash
# Run from your proto root. -I (a.k.a. --proto_path) sets the import search root.
protoc \
  -I proto \
  --go_out=. --go_opt=module=github.com/example/app \
  --go-grpc_out=. --go-grpc_opt=module=github.com/example/app \
  proto/user/v1/user.proto
```

Key flags:
- `-I proto` — search `proto/` for imports (repeatable; add `-I` roots for WKTs if not bundled).
- `--go_out` / `--go-grpc_out` — output directory for each plugin.
- `--go_opt=module=...` — strip this module prefix from `go_package` so files land at the right relative path. (Alternative: `--go_opt=paths=source_relative` to mirror the source tree.)

This is fiddly: paths, `-I` roots, plugin versions, and `go_package` all have to line up, and the error messages are cryptic. That pain is precisely why **buf** exists.

### 5.5 The modern `buf` workflow **[I]**

[`buf`](https://buf.build) wraps `protoc` with sane defaults, dependency management, linting, formatting, and breaking-change detection. Install the single static `buf` binary and configure two files.

**`buf.yaml`** — defines a module (where protos live, lint rules, breaking rules, deps):

```yaml
# buf.yaml  (place at the root of your proto module, often the repo root)
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD          # the recommended rule set (enum naming, versioned packages, ...)
breaking:
  use:
    - FILE              # detect wire-incompatible changes at file granularity
deps:
  # Pull dependencies (e.g. googleapis for HTTP annotations) from the Buf Schema Registry.
  - buf.build/googleapis/googleapis
```

**`buf.gen.yaml`** — defines what to generate and how:

```yaml
# buf.gen.yaml
version: v2
managed:
  enabled: true                 # auto-manage go_package etc. so you don't hand-edit each file
  override:
    - file_option: go_package_prefix
      value: github.com/example/app/gen
plugins:
  # Use locally installed plugins (or swap to remote: buf.build/protocolbuffers/go).
  - local: protoc-gen-go
    out: gen
    opt: paths=source_relative
  - local: protoc-gen-go-grpc
    out: gen
    opt: paths=source_relative
```

Daily commands:

```bash
buf lint          # style/consistency checks (enum naming, package versioning, ...)
buf format -w     # auto-format .proto files in place (like gofmt for protos)
buf breaking --against '.git#branch=main'   # fail CI on incompatible schema change
buf generate      # generate Go code per buf.gen.yaml
buf build         # compile protos to a binary image (for registries/tooling)
```

**⚡ Version note:** `buf` configuration moved to **v2** (`version: v2` with a `modules:` list). Older guides show v1 with a separate `buf.work.yaml` for multi-module workspaces; v2 unifies workspaces into a single `buf.yaml`. Confirm the schema for your installed `buf` version.

### 5.6 Recommended module layout **[I]**

```
myapp/
├── go.mod
├── buf.yaml
├── buf.gen.yaml
├── proto/
│   ├── user/v1/user.proto
│   └── shop/v1/order.proto
├── gen/                      # generated code — checked in OR generated in CI
│   ├── user/v1/
│   │   ├── user.pb.go        # messages
│   │   └── user_grpc.pb.go   # service stubs
│   └── shop/v1/
│       ├── order.pb.go
│       └── order_grpc.pb.go
├── cmd/
│   ├── server/main.go
│   └── client/main.go
└── internal/
    └── service/user.go       # your business logic implementing the generated interface
```

**Check generated code in, or not?** Both are valid. Checking it in makes `go build`/`go install` work without the toolchain (good for `go install` consumers); generating in CI keeps the repo clean but requires every contributor to have `buf`. A common compromise: **check it in, and have CI verify it's up to date** (`buf generate && git diff --exit-code`).

**Security note:** treat generated code as build output, but still review *schema* PRs carefully — a malicious or careless field-number reuse (§4.10) is a data-corruption bug that codegen happily produces. The `buf breaking` gate is your safety net.

---

## 6. Building a gRPC Server in Go

We'll define a complete service exercising all four RPC types, then implement it. This section is the server side end-to-end; §7 adds the matching clients and the conceptual map for each pattern.

### 6.1 The service `.proto` **[I]**

```proto
// proto/user/v1/user.proto
syntax = "proto3";
package user.v1;
option go_package = "github.com/example/app/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";

message User {
	string id = 1;
	string email = 2;
	string display_name = 3;
	google.protobuf.Timestamp created_at = 4;
}

// ---- Unary ----
message GetUserRequest  { string id = 1; }
message GetUserResponse { User user = 1; }

// ---- Server streaming ----
message ListUpdatesRequest { string user_id = 1; }
message Update {
	string message = 1;
	google.protobuf.Timestamp at = 2;
}

// ---- Client streaming ----
message UploadChunk { bytes data = 1; }
message UploadSummary {
	int64 total_bytes = 1;
	int64 chunk_count = 2;
}

// ---- Bidirectional streaming ----
message ChatMessage {
	string from = 1;
	string text = 2;
}

service UserService {
	// Unary: one request, one response.
	rpc GetUser(GetUserRequest) returns (GetUserResponse);

	// Server streaming: one request, a stream of responses.
	rpc ListUpdates(ListUpdatesRequest) returns (stream Update);

	// Client streaming: a stream of requests, one response.
	rpc UploadChunks(stream UploadChunk) returns (UploadSummary);

	// Bidirectional: streams both directions, independently.
	rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

After `buf generate`, you get `userv1.UserServiceServer` (the interface to implement) and `userv1.RegisterUserServiceServer` (the registration helper), plus client types used in §7.

### 6.2 The generated server interface (what you implement) **[I]**

The generated interface looks like this (abridged — *don't write it; it's generated*). Reading it tells you exactly what your implementation must provide:

```go
type UserServiceServer interface {
	GetUser(context.Context, *GetUserRequest) (*GetUserResponse, error)
	ListUpdates(*ListUpdatesRequest, grpc.ServerStreamingServer[Update]) error
	UploadChunks(grpc.ClientStreamingServer[UploadChunk, UploadSummary]) error
	Chat(grpc.BidiStreamingServer[ChatMessage, ChatMessage]) error
	mustEmbedUnimplementedUserServiceServer() // forces forward compatibility (see below)
}
```

Notice the shapes: unary is `(ctx, req) (resp, err)`; the streaming methods receive a *stream object* instead of (or in addition to) a request and return only `error`. The `mustEmbed...` method is unexported, so the *only* way to satisfy the interface is to embed the generated `UnimplementedUserServiceServer` — which is exactly what makes adding methods to the `.proto` non-breaking for existing servers.

**⚡ Version note:** Modern `protoc-gen-go-grpc` uses **generics** for stream types (`grpc.ServerStreamingServer[Update]`, etc.). Older generated code used per-method named interfaces (`UserService_ListUpdatesServer`). Both behave identically; the generic form is current as of 2024+.

### 6.3 Implementing the server — all four methods **[I]**

```go
// internal/service/user.go
package service

import (
	"context"
	"io"
	"log/slog"
	"time"

	userv1 "github.com/example/app/gen/user/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/timestamppb"
)

// UserServer implements userv1.UserServiceServer.
type UserServer struct {
	// Embedding the generated "Unimplemented" struct is REQUIRED for forward
	// compatibility: if a new method is added to the .proto, your server still
	// compiles (the embedded stub returns codes.Unimplemented for the new method
	// until you implement it). Omitting this is a common beginner build error.
	userv1.UnimplementedUserServiceServer

	log   *slog.Logger
	users map[string]*userv1.User // toy in-memory store; real code holds a DB handle
}

func NewUserServer(log *slog.Logger) *UserServer {
	return &UserServer{
		log: log,
		users: map[string]*userv1.User{
			"u1": {Id: "u1", Email: "ada@example.com", DisplayName: "Ada"},
		},
	}
}

// --- 1. Unary ---
func (s *UserServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	// SECURITY/ROBUSTNESS: always validate input before using it. Treat every
	// request as untrusted, even from internal callers (zero trust).
	if req.GetId() == "" {
		// Return a typed gRPC status error (§9), NOT a bare errors.New — bare
		// errors become codes.Unknown and lose machine-actionable meaning.
		return nil, status.Error(codes.InvalidArgument, "id is required")
	}
	// Honor cancellation/deadlines: if the caller gave up, don't do work.
	if err := ctx.Err(); err != nil {
		return nil, status.FromContextError(err).Err()
	}
	u, ok := s.users[req.GetId()]
	if !ok {
		return nil, status.Errorf(codes.NotFound, "user %q not found", req.GetId())
	}
	return &userv1.GetUserResponse{User: u}, nil
}

// --- 2. Server streaming ---
// The response stream is the second parameter. You call Send() repeatedly and
// return nil to close the stream cleanly (the client then observes io.EOF).
func (s *UserServer) ListUpdates(req *userv1.ListUpdatesRequest, stream grpc.ServerStreamingServer[userv1.Update]) error {
	ticker := time.NewTicker(500 * time.Millisecond)
	defer ticker.Stop()

	for i := 0; i < 5; i++ {
		select {
		case <-stream.Context().Done():
			// Client cancelled or deadline exceeded — stop streaming and free work.
			return stream.Context().Err()
		case <-ticker.C:
			up := &userv1.Update{
				Message: "update for " + req.GetUserId(),
				At:      timestamppb.Now(),
			}
			if err := stream.Send(up); err != nil {
				return err // peer went away; stop
			}
		}
	}
	return nil // returning ends the stream with status OK
}

// --- 3. Client streaming ---
// You Recv() in a loop until io.EOF, then SendAndClose() the single response.
func (s *UserServer) UploadChunks(stream grpc.ClientStreamingServer[userv1.UploadChunk, userv1.UploadSummary]) error {
	var total, count int64
	for {
		chunk, err := stream.Recv()
		if err == io.EOF {
			// Client finished sending; reply once with the summary.
			return stream.SendAndClose(&userv1.UploadSummary{
				TotalBytes: total,
				ChunkCount: count,
			})
		}
		if err != nil {
			return err
		}
		// SECURITY: cap accumulated size to avoid unbounded memory from a hostile
		// client (a streaming equivalent of a body-size limit). Tune to your needs.
		total += int64(len(chunk.GetData()))
		count++
		if total > 256<<20 { // 256 MiB ceiling
			return status.Error(codes.ResourceExhausted, "upload too large")
		}
	}
}

// --- 4. Bidirectional streaming ---
// Recv and Send are independent and can interleave freely. Here we echo back.
func (s *UserServer) Chat(stream grpc.BidiStreamingServer[userv1.ChatMessage, userv1.ChatMessage]) error {
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil // client closed its send side; we close ours by returning
		}
		if err != nil {
			return err
		}
		reply := &userv1.ChatMessage{From: "server", Text: "echo: " + msg.GetText()}
		if err := stream.Send(reply); err != nil {
			return err
		}
	}
}
```

### 6.4 Wiring up `main` — listen, register, serve, shut down **[I]**

```go
// cmd/server/main.go
package main

import (
	"log/slog"
	"net"
	"os"
	"os/signal"
	"syscall"
	"time"

	userv1 "github.com/example/app/gen/user/v1"
	"github.com/example/app/internal/service"
	"google.golang.org/grpc"
	"google.golang.org/grpc/keepalive"
)

func main() {
	log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// 1) Open a TCP listener. Bind to a specific interface in prod; :50051 is the
	//    conventional gRPC port.
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Error("listen failed", "err", err)
		os.Exit(1)
	}

	// 2) Construct the server with options. In production add TLS creds (§11) and
	//    interceptors (§10) here.
	srv := grpc.NewServer(
		grpc.MaxRecvMsgSize(16*1024*1024), // raise from the 4 MiB default if needed (§17)
		grpc.KeepaliveParams(keepalive.ServerParameters{
			MaxConnectionIdle: 5 * time.Minute,
			Time:              2 * time.Hour,    // ping idle conns to detect dead peers
			Timeout:           20 * time.Second, // wait this long for a ping ack
		}),
		grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
			MinTime:             10 * time.Second, // reject clients pinging faster than this
			PermitWithoutStream: true,
		}),
		// Add interceptors (§10): grpc.ChainUnaryInterceptor(recovery, logging, auth)
		// Add credentials (§11):  grpc.Creds(tlsCreds)
	)

	// 3) Register your service implementation.
	userv1.RegisterUserServiceServer(srv, service.NewUserServer(log))

	// 4) Serve in a goroutine so main can wait for a shutdown signal.
	go func() {
		log.Info("gRPC server listening", "addr", lis.Addr().String())
		if err := srv.Serve(lis); err != nil {
			log.Error("serve failed", "err", err)
		}
	}()

	// 5) Graceful shutdown on SIGINT/SIGTERM (§18).
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
	<-stop
	log.Info("shutting down...")

	// GracefulStop stops accepting new RPCs and waits for in-flight ones to finish.
	done := make(chan struct{})
	go func() { srv.GracefulStop(); close(done) }()
	select {
	case <-done:
		log.Info("clean shutdown")
	case <-time.After(15 * time.Second):
		srv.Stop() // hard stop if graceful takes too long (e.g. a stuck long stream)
		log.Warn("forced shutdown")
	}
}
```

---

## 7. The Four RPC Types End-to-End (Client + Server)

§6 implemented all four server-side. This section builds the **clients**, then gives the **conceptual map**, **lifecycle**, and **gotchas** for each pattern so it sticks. First, dialing.

### 7.1 Dialing — `grpc.NewClient` (the modern API) **[I]**

**⚡ Version note:** `grpc.Dial` / `grpc.DialContext` are **deprecated** in favor of **`grpc.NewClient`** (stable since grpc-go v1.63, 2024). `NewClient` does **not** connect eagerly and does **not** block — the connection is established lazily on the first RPC (or `conn.Connect()`). It defaults the name resolver to `dns:///`, so plain `host:port` targets work. Drop the old `grpc.WithBlock()` / dial-timeout idioms — connection readiness is now governed by per-RPC deadlines and the wait-for-ready/retry policy (§12).

```go
// cmd/client/main.go
package main

import (
	"log"

	userv1 "github.com/example/app/gen/user/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	// grpc.NewClient returns immediately; connection is lazy. "insecure" is for
	// local/dev ONLY — use TLS credentials in production (§11).
	conn, err := grpc.NewClient(
		"localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		// Optional: a default service config enabling retries/LB (§12).
	)
	if err != nil {
		log.Fatalf("NewClient: %v", err) // only fails on bad args/target syntax
	}
	defer conn.Close()

	// One ClientConn backs many stubs and many concurrent RPCs (HTTP/2 mux).
	// Create it ONCE and share it — never dial per request (§17).
	client := userv1.NewUserServiceClient(conn)

	unaryDemo(client)
	serverStreamDemo(client)
	clientStreamDemo(client)
	bidiDemo(client)
}
```

### 7.2 Unary — `GetUser` (request/response) **[I]**

**What it is / when:** the simplest and most common pattern — one request, one response. Use it for everything that isn't naturally a stream: reads, writes, commands. It is the easiest to reason about, cache, retry, and load-balance per call.

```go
import (
	"context"
	"time"
)

func unaryDemo(client userv1.UserServiceClient) {
	// ALWAYS set a deadline on RPCs (§12). Without one, a hung server hangs you
	// forever — the cardinal gRPC client sin.
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: "u1"})
	if err != nil {
		log.Printf("GetUser error: %v", err) // inspect with status.FromError (§9)
		return
	}
	log.Printf("user: %s <%s>", resp.GetUser().GetDisplayName(), resp.GetUser().GetEmail())
}
```

**Lifecycle:** client calls; server returns `(resp, err)`; done. **Gotchas:** forgetting the deadline; using bare errors on the server so the client can't branch on a code.

### 7.3 Server streaming — `ListUpdates` (feed / pagination / progress) **[I]**

**What it is / when:** one request, a *stream* of responses. Great for live feeds, large result sets you don't want to buffer in one message, progress updates, and server-sent change notifications. The server `Send()`s as many messages as it likes then returns; the client loops `Recv()` until `io.EOF`.

```go
import "io"

func serverStreamDemo(client userv1.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.ListUpdates(ctx, &userv1.ListUpdatesRequest{UserId: "u1"})
	if err != nil {
		log.Printf("ListUpdates: %v", err)
		return
	}
	for {
		up, err := stream.Recv()
		if err == io.EOF {
			break // server closed the stream cleanly
		}
		if err != nil {
			log.Printf("recv: %v", err)
			return
		}
		log.Printf("update: %s @ %s", up.GetMessage(), up.GetAt().AsTime())
	}
}
```

**Gotchas:** the server must honor `stream.Context().Done()` so client cancellation actually stops the work; the client *must* drain to `io.EOF` (or cancel its context) or it leaks the stream.

### 7.4 Client streaming — `UploadChunks` (upload / aggregation) **[I]**

**What it is / when:** a *stream* of requests, one response. Great for file/blob uploads, bulk inserts, and metrics aggregation. The client `Send()`s many messages then `CloseAndRecv()`; the server loops `Recv()` to `io.EOF` then `SendAndClose()` a single response. (Compare with multipart file upload over REST in **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** — gRPC client-streaming gives you back-pressure for free via HTTP/2 flow control.)

```go
func clientStreamDemo(client userv1.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.UploadChunks(ctx)
	if err != nil {
		log.Printf("UploadChunks: %v", err)
		return
	}
	for i := 0; i < 3; i++ {
		if err := stream.Send(&userv1.UploadChunk{Data: []byte("hello-chunk")}); err != nil {
			log.Printf("send: %v", err)
			return
		}
	}
	// CloseAndRecv signals "done sending" and waits for the single response.
	summary, err := stream.CloseAndRecv()
	if err != nil {
		log.Printf("close/recv: %v", err)
		return
	}
	log.Printf("uploaded %d bytes in %d chunks", summary.GetTotalBytes(), summary.GetChunkCount())
}
```

**Gotchas:** the single response only arrives *after* the client closes its send side; don't forget `CloseAndRecv()` (client) and the `io.EOF`-then-`SendAndClose()` shape (server).

### 7.5 Bidirectional streaming — `Chat` (interactive / multiplexed) **[I]**

**What it is / when:** both sides stream, independently. Great for chat, real-time collaboration, request/response multiplexing over one stream, and long-lived control channels. (Compare with WebSockets in **`GO_GORILLA_WEBSOCKETS_GUIDE.md`** — gRPC bidi adds typed messages, deadlines, and the gRPC status model; WebSockets are simpler for browser-native cases.)

On the **client**, run `Recv()` in its own goroutine because it blocks; `Send()` from the main flow; finish with `CloseSend()`.

```go
func bidiDemo(client userv1.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.Chat(ctx)
	if err != nil {
		log.Printf("Chat: %v", err)
		return
	}

	// Receive in a SEPARATE goroutine because Send and Recv run concurrently.
	done := make(chan struct{})
	go func() {
		defer close(done)
		for {
			in, err := stream.Recv()
			if err == io.EOF {
				return
			}
			if err != nil {
				log.Printf("recv: %v", err)
				return
			}
			log.Printf("server says: %s", in.GetText())
		}
	}()

	for _, t := range []string{"hi", "how are you", "bye"} {
		if err := stream.Send(&userv1.ChatMessage{From: "client", Text: t}); err != nil {
			log.Printf("send: %v", err)
			break
		}
	}
	stream.CloseSend() // tell the server we're done sending
	<-done
}
```

**Gotchas:**
- **Never** call `Send()` from two goroutines on the same stream concurrently, and likewise for `Recv()`. Each of `Send` and `Recv` is *not* safe for concurrent use by multiple goroutines — but one goroutine sending while a *different* one receives is the correct, supported pattern.
- Decide and document who closes first. Deadlocks here are usually "both sides waiting to receive."

### 7.6 Lifecycle summary **[I]**

| Mode | Client API | Server API | Ends when |
|---|---|---|---|
| Unary | `resp, err := c.M(ctx, req)` | `return resp, err` | Server returns |
| Server stream | `s,_ := c.M(ctx,req)`; loop `s.Recv()` to `io.EOF` | loop `stream.Send()`; `return nil` | Server returns |
| Client stream | loop `s.Send()`; `s.CloseAndRecv()` | loop `stream.Recv()` to `io.EOF`; `stream.SendAndClose()` | Client closes send + server responds |
| Bidi stream | `s.Send()`/`s.Recv()` concurrently; `s.CloseSend()` | `stream.Recv()`/`stream.Send()` concurrently; `return` | Both sides close |

### 7.7 Connection-management reality **[I]**

- A `*grpc.ClientConn` is **safe for concurrent use** and is meant to be **long-lived and shared**. Create one per backend and reuse it — do **not** dial per request (§17).
- Because HTTP/2 multiplexes, one `ClientConn` already gives you concurrency. You generally do **not** need a "connection pool" of `ClientConn`s.
- `conn.Connect()` forces an eager connection attempt; `conn.GetState()` / `conn.WaitForStateChange(ctx, state)` let you observe connectivity if you really need to.

---

## 8. Metadata — Headers & Trailers

### 8.1 What metadata is and why it exists **[I]**

**Metadata** is gRPC's equivalent of HTTP headers: key/value pairs attached to a call, *separate from* the message body. The logic: some information is *about* the call rather than *part of* it — auth tokens, request/trace IDs, client version, content negotiation. Putting these in metadata keeps your message schema clean (the `.proto` describes the domain, not the plumbing) and lets interceptors (§10) read/modify them generically without parsing message bodies.

**Key rules:** keys are case-insensitive ASCII; values are strings, *except* keys with the `-bin` suffix carry raw bytes (base64-encoded on the wire). The package is `google.golang.org/grpc/metadata`. There are two delivery phases: **headers** (sent before the response body) and **trailers** (sent *after* the body — even for streams — ideal for end-of-call info like row counts or final status detail).

### 8.2 Client → server: sending and reading metadata **[I]**

```go
import "google.golang.org/grpc/metadata"

// Attach OUTGOING metadata to the context before the call.
md := metadata.Pairs(
	"authorization", "Bearer "+token,
	"x-request-id", "abc-123",
)
ctx := metadata.NewOutgoingContext(context.Background(), md)

// Optionally capture the server's response headers and trailers.
var header, trailer metadata.MD
resp, err := client.GetUser(ctx,
	&userv1.GetUserRequest{Id: "u1"},
	grpc.Header(&header),   // populated with response headers
	grpc.Trailer(&trailer), // populated with response trailers
)
_ = resp
_ = err
fmt.Println("server header x-served-by:", header.Get("x-served-by"))
fmt.Println("server trailer x-row-count:", trailer.Get("x-row-count"))
```

### 8.3 Server: reading request metadata, sending headers/trailers **[I]**

```go
func (s *UserServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	// Read INCOMING metadata from the inbound context.
	if md, ok := metadata.FromIncomingContext(ctx); ok {
		if vals := md.Get("authorization"); len(vals) > 0 {
			// inspect vals[0] for a token... (real validation belongs in an interceptor, §10)
		}
		if rid := md.Get("x-request-id"); len(rid) > 0 {
			s.log.Info("handling", "request_id", rid[0])
		}
	}

	// Send response HEADERS (must happen before the first message / return value).
	_ = grpc.SetHeader(ctx, metadata.Pairs("x-served-by", "user-svc-1"))

	// Set TRAILERS (delivered at the end of the call).
	_ = grpc.SetTrailer(ctx, metadata.Pairs("x-row-count", "1"))

	u := s.users[req.GetId()]
	return &userv1.GetUserResponse{User: u}, nil
}
```

For **streaming** handlers, use `stream.SetHeader`/`stream.SendHeader`/`stream.SetTrailer` and read with `metadata.FromIncomingContext(stream.Context())`.

### 8.4 Context propagation (forwarding metadata downstream) **[I/A]**

A common need is forwarding select metadata (trace IDs, request IDs) from an inbound call to a downstream outbound call so a request can be followed across services:

```go
// In a handler/interceptor: copy chosen inbound keys to the outgoing context.
inMD, _ := metadata.FromIncomingContext(ctx)
outCtx := metadata.NewOutgoingContext(ctx,
	metadata.Pairs("x-request-id", first(inMD.Get("x-request-id"))),
)
_, _ = downstream.SomeRPC(outCtx, &pb.Req{})
```

**⚡ Security note:** Do **not** blindly forward *all* inbound metadata downstream — strip auth headers and hop-by-hop values you don't intend to propagate, or you can leak a user's credentials to a service that shouldn't see them, or let a client smuggle headers through. Tracing libraries (OpenTelemetry, §16) provide propagators that handle the right keys correctly; prefer them over hand-rolled copying. Also never log metadata wholesale — it routinely contains bearer tokens.

---

## 9. Error Handling — status, codes & rich errors

### 9.1 The gRPC status model vs Go errors **[I]**

gRPC has a **structured error model**: every RPC ends with a status that carries (1) a **code** — an integer enum from a fixed set, (2) a human-readable **message**, and (3) optional **details** — arbitrary protobuf messages. This is fundamentally richer than a Go `error` (just a string) or an HTTP status (just a number): the code lets clients and retry policies branch programmatically, while details carry machine-actionable specifics.

**The bridge to Go:** use `google.golang.org/grpc/status` and `google.golang.org/grpc/codes`. The cardinal rule: **never return a bare `errors.New` from a handler.** gRPC wraps unrecognized errors as `codes.Unknown`, which is useless to callers (they can't tell a validation failure from a server crash) and breaks retry logic. Always return a `status.Error(codes.X, ...)`.

### 9.2 The standard codes (and their HTTP analogues) **[I]**

| Code | When to use | Roughly like HTTP | Retry-safe? |
|---|---|---|---|
| `OK` | Success | 200 | — |
| `InvalidArgument` | Caller sent bad input (independent of state) | 400 | No |
| `FailedPrecondition` | System not in required state | 400/409 | No |
| `OutOfRange` | Past valid range (e.g. pagination) | 400 | No |
| `Unauthenticated` | No/invalid credentials | 401 | No |
| `PermissionDenied` | Authenticated but not allowed | 403 | No |
| `NotFound` | Entity doesn't exist | 404 | No |
| `AlreadyExists` | Entity already exists | 409 | No |
| `Aborted` | Concurrency conflict (retry the txn) | 409 | Maybe (app-level) |
| `ResourceExhausted` | Quota/rate limit | 429 | With backoff |
| `Cancelled` | Caller cancelled | 499 | No |
| `DeadlineExceeded` | Deadline passed | 504 | Sometimes |
| `Unimplemented` | Method not implemented | 501 | No |
| `Unavailable` | Transient; safe to retry | 503 | **Yes** |
| `Internal` | Server bug / invariant broken | 500 | No |
| `DataLoss` | Unrecoverable data loss | 500 | No |
| `Unknown` | Uncategorized (avoid; usually a leaked plain error) | 500 | No |

**Choosing the right code is a design decision, not an afterthought** — clients and the built-in retry policy (§12) *act* on it. `Unavailable` invites retries; `Internal` must not be retried automatically; `InvalidArgument` tells the client "don't bother retrying, fix your request."

### 9.3 Returning errors from a handler **[I]**

```go
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// Simple coded error:
return nil, status.Error(codes.NotFound, "user not found")

// With formatting:
return nil, status.Errorf(codes.InvalidArgument, "id %q is malformed", req.GetId())
```

### 9.4 Rich errors with `errdetails` **[I/A]**

Attach structured detail messages so clients can react programmatically — which field failed validation, how long to wait before retrying, which quota was exceeded. The standard detail types live in `google.golang.org/genproto/googleapis/rpc/errdetails`.

```go
import (
	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

func invalidEmail() error {
	st := status.New(codes.InvalidArgument, "validation failed")

	// BadRequest details: per-field violations a client UI can highlight.
	br := &errdetails.BadRequest{
		FieldViolations: []*errdetails.BadRequest_FieldViolation{
			{Field: "email", Description: "must be a valid email address"},
		},
	}
	// Others: RetryInfo, QuotaFailure, ErrorInfo, ResourceInfo, LocalizedMessage...
	st, err := st.WithDetails(br)
	if err != nil {
		// WithDetails can fail if a detail can't be marshaled — fall back gracefully.
		return status.Error(codes.InvalidArgument, "validation failed")
	}
	return st.Err()
}
```

### 9.5 Inspecting errors on the client **[I]**

```go
resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: ""})
if err != nil {
	st, ok := status.FromError(err) // ok=false means it wasn't a gRPC status at all
	if !ok {
		log.Printf("non-status error: %v", err)
		return
	}
	switch st.Code() {
	case codes.NotFound:
		log.Println("no such user")
	case codes.InvalidArgument:
		log.Printf("bad request: %s", st.Message())
		// Walk the structured details:
		for _, d := range st.Details() {
			if info, ok := d.(*errdetails.BadRequest); ok {
				for _, v := range info.GetFieldViolations() {
					log.Printf("  field %s: %s", v.GetField(), v.GetDescription())
				}
			}
		}
	default:
		log.Printf("rpc failed [%s]: %s", st.Code(), st.Message())
	}
}
_ = resp
```

### 9.6 Mapping app errors ↔ gRPC status (the boundary pattern) **[I/A]**

**Best practice:** keep your domain/business layer free of gRPC types; translate at the *edge* (often in an interceptor, §10). This keeps the core testable and reusable, and gives you one place to enforce the security rule below.

```go
import "errors"

// Domain sentinel errors (no gRPC import in the domain layer).
var (
	ErrNotFound     = errors.New("not found")
	ErrInvalidInput = errors.New("invalid input")
)

// Translate to a gRPC status at the boundary.
func toStatus(err error) error {
	switch {
	case err == nil:
		return nil
	case errors.Is(err, ErrNotFound):
		return status.Error(codes.NotFound, err.Error())
	case errors.Is(err, ErrInvalidInput):
		return status.Error(codes.InvalidArgument, err.Error())
	default:
		// SECURITY: never leak internal error text / stack traces to clients —
		// it can reveal table names, file paths, library versions. Log the detail
		// server-side; return a generic Internal to the caller.
		return status.Error(codes.Internal, "internal error")
	}
}
```

---

## 10. Interceptors — gRPC's Middleware

### 10.1 What interceptors are and why **[I]**

**Interceptors** are gRPC's middleware: hooks that wrap *every* RPC so you can implement cross-cutting concerns once instead of in every handler. The logic mirrors HTTP middleware (see the REST guides) — logging, authentication, panic recovery, metrics, tracing, rate limiting, validation are *orthogonal* to business logic and belong in a reusable layer.

There are **four kinds**, the cross product of `{Unary, Stream} × {Server, Client}`. Chain multiple with `grpc.ChainUnaryInterceptor` / `grpc.ChainStreamInterceptor` (server) and `grpc.WithChainUnaryInterceptor` / `grpc.WithChainStreamInterceptor` (client). **Order matters: the first listed runs outermost** (wraps all the rest). The conventional ordering on the server is `recovery → logging/metrics/tracing → auth → handler`, so a panic in *any* inner layer is still recovered.

### 10.2 Server-side unary interceptor — logging + timing **[I]**

```go
import (
	"context"
	"log/slog"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/status"
)

func loggingUnary(log *slog.Logger) grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req any,
		info *grpc.UnaryServerInfo, // info.FullMethod = "/user.v1.UserService/GetUser"
		handler grpc.UnaryHandler,  // call this to proceed to the next/real handler
	) (any, error) {
		start := time.Now()
		resp, err := handler(ctx, req) // <-- proceed
		log.Info("rpc",
			"method", info.FullMethod,
			"code", status.Code(err).String(),
			"dur_ms", time.Since(start).Milliseconds(),
		)
		return resp, err
	}
}
```

### 10.3 Server-side recovery (panic) interceptor **[I]**

A panic in a handler would otherwise crash the *whole* server process. Recover and convert it to `codes.Internal`. Put this **outermost** so it catches panics from every inner interceptor and the handler.

```go
import (
	"runtime/debug"

	"google.golang.org/grpc/codes"
)

func recoveryUnary(log *slog.Logger) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
		defer func() {
			if r := recover(); r != nil {
				log.Error("panic recovered", "method", info.FullMethod, "panic", r, "stack", string(debug.Stack()))
				// SECURITY: do not echo the panic value to the client (it may leak internals).
				err = status.Error(codes.Internal, "internal error")
			}
		}()
		return handler(ctx, req)
	}
}
```

### 10.4 Server-side auth interceptor (JWT/Bearer in metadata) **[I/A]**

This is the canonical place to do authentication: validate a token from metadata once, attach the identity to the context, and let handlers stay auth-agnostic. (Token verification primitives — JWT signing/verification, Argon2 password hashing — are covered in **`GO_JWT_ARGON2_GUIDE.md`**.)

```go
import (
	"strings"

	"google.golang.org/grpc/metadata"
)

// authUnary validates a Bearer token and stuffs the user ID into the context.
// `verify` is your JWT/OIDC validation function (returns the subject/user ID).
func authUnary(verify func(token string) (userID string, err error)) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		// Allow-list public methods (e.g. health checks) to skip auth.
		if info.FullMethod == "/grpc.health.v1.Health/Check" {
			return handler(ctx, req)
		}
		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return nil, status.Error(codes.Unauthenticated, "missing metadata")
		}
		auth := md.Get("authorization")
		if len(auth) == 0 || !strings.HasPrefix(auth[0], "Bearer ") {
			return nil, status.Error(codes.Unauthenticated, "missing bearer token")
		}
		userID, err := verify(strings.TrimPrefix(auth[0], "Bearer "))
		if err != nil {
			// Generic message — don't reveal WHY the token failed (expiry vs signature).
			return nil, status.Error(codes.Unauthenticated, "invalid token")
		}
		// Pass identity down via context using an UNEXPORTED key type to avoid collisions.
		ctx = context.WithValue(ctx, userCtxKey{}, userID)
		return handler(ctx, req)
	}
}

type userCtxKey struct{}
```

**Authentication vs authorization:** the interceptor above does *authentication* (who are you). *Authorization* (are you allowed to do this) is usually per-method and often lives in the handler or a dedicated layer, returning `codes.PermissionDenied` on failure.

### 10.5 Stream interceptors **[I/A]**

Streaming RPCs need their own interceptor type. To attach an enriched context (or intercept individual messages), you *wrap* the `grpc.ServerStream`:

```go
// wrappedStream lets us override Context() (and could override RecvMsg/SendMsg).
type wrappedStream struct {
	grpc.ServerStream
	ctx context.Context
}

func (w *wrappedStream) Context() context.Context { return w.ctx }

func authStream(verify func(string) (string, error)) grpc.StreamServerInterceptor {
	return func(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
		md, ok := metadata.FromIncomingContext(ss.Context())
		if !ok {
			return status.Error(codes.Unauthenticated, "missing metadata")
		}
		auth := md.Get("authorization")
		if len(auth) == 0 {
			return status.Error(codes.Unauthenticated, "missing token")
		}
		uid, err := verify(strings.TrimPrefix(auth[0], "Bearer "))
		if err != nil {
			return status.Error(codes.Unauthenticated, "invalid token")
		}
		ctx := context.WithValue(ss.Context(), userCtxKey{}, uid)
		return handler(srv, &wrappedStream{ServerStream: ss, ctx: ctx}) // proceed
	}
}
```

### 10.6 Registering interceptors **[I]**

```go
srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(
		recoveryUnary(log), // OUTERMOST: catches panics from everything inside
		loggingUnary(log),
		authUnary(verifyJWT),
	),
	grpc.ChainStreamInterceptor(
		recoveryStream(log), // (analogous recovery for streams)
		authStream(verifyJWT),
	),
)
```

### 10.7 Client-side interceptors — inject auth + retry **[I/A]**

```go
import "google.golang.org/grpc/credentials/insecure"

// Client unary interceptor that attaches a token to every outgoing call.
func authClientUnary(token string) grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply any,
		cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
		return invoker(ctx, method, req, reply, cc, opts...) // proceed
	}
}

// A tiny manual retry interceptor (prefer the built-in service-config retry, §12).
func retryClientUnary(max int) grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply any,
		cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		var err error
		for attempt := 0; attempt <= max; attempt++ {
			err = invoker(ctx, method, req, reply, cc, opts...)
			if status.Code(err) != codes.Unavailable {
				return err // only retry transient Unavailable
			}
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-time.After(time.Duration(attempt+1) * 100 * time.Millisecond):
			}
		}
		return err
	}
}

conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithChainUnaryInterceptor(authClientUnary(token), retryClientUnary(3)),
)
```

### 10.8 Use `go-grpc-middleware` in production **[I]**

The community package `github.com/grpc-ecosystem/go-grpc-middleware/v2` provides battle-tested interceptors so you don't hand-roll the above: logging (adapters for slog/zap/zerolog), recovery, auth, retry, rate-limiting, validation, and *selective* application (`selector.UnaryServerInterceptor`, to apply auth to some methods but not others). For production, prefer these — the toy versions above are for understanding.

```go
import (
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/logging"
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/recovery"
)

srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(
		recovery.UnaryServerInterceptor(),
		logging.UnaryServerInterceptor(interceptorLogger(slogLogger)),
	),
)
```

---

## 11. Authentication & Security — TLS, mTLS, Tokens

### 11.1 The two-layer credentials model **[I]**

gRPC's `credentials` system has **two distinct layers**, and production setups use *both* together:

1. **Transport credentials** — *channel* security: TLS or mTLS encrypts and (optionally) mutually authenticates the connection itself. This protects the pipe.
2. **Per-RPC credentials** — *call-level* identity: a token (JWT/OIDC/API key) carried in metadata that identifies the caller per request. This identifies who is calling.

The classic, recommended pattern: **TLS for the channel + a token in metadata for identity.** TLS stops eavesdropping/tampering; the token tells the server who the user is. mTLS additionally proves *which service* is calling — ideal for service-to-service.

### 11.2 Server-side TLS **[I]**

```go
import "google.golang.org/grpc/credentials"

// Load the server cert + key (PEM).
creds, err := credentials.NewServerTLSFromFile("certs/server.crt", "certs/server.key")
if err != nil {
	log.Fatal(err)
}
srv := grpc.NewServer(grpc.Creds(creds))
```

### 11.3 Client-side TLS **[I]**

```go
// Trust the server's CA. For public CAs, use credentials.NewTLS with the system pool.
creds, err := credentials.NewClientTLSFromFile("certs/ca.crt", "" /* serverNameOverride */)
if err != nil {
	log.Fatal(err)
}
conn, err := grpc.NewClient("example.com:443", grpc.WithTransportCredentials(creds))
```

### 11.4 Mutual TLS (mTLS) — both sides present certs **[I/A]**

mTLS authenticates the *client* to the server *and* the server to the client, using certificates signed by a trusted CA. This is the gold standard for service-to-service ("east-west") auth inside a mesh: each service has an identity (its cert) and the connection itself proves who's talking, no token needed for service identity.

```go
import (
	"crypto/tls"
	"crypto/x509"
	"os"
)

// ---- Server requiring & verifying client certs ----
func serverMTLS() credentials.TransportCredentials {
	cert, _ := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
	caPEM, _ := os.ReadFile("certs/ca.crt")
	pool := x509.NewCertPool()
	pool.AppendCertsFromPEM(caPEM)
	return credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		ClientCAs:    pool,
		ClientAuth:   tls.RequireAndVerifyClientCert, // <-- DEMAND a valid client cert
		MinVersion:   tls.VersionTLS13,               // prefer TLS 1.3 where possible
	})
}

// ---- Client presenting its cert ----
func clientMTLS() credentials.TransportCredentials {
	cert, _ := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
	caPEM, _ := os.ReadFile("certs/ca.crt")
	pool := x509.NewCertPool()
	pool.AppendCertsFromPEM(caPEM)
	return credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		RootCAs:      pool,
		ServerName:   "user-svc", // MUST match a SAN in the server's cert
		MinVersion:   tls.VersionTLS13,
	})
}
```

### 11.5 Per-RPC credentials (token-based) **[I/A]**

Implement `credentials.PerRPCCredentials` to inject a token on *every* call automatically — cleaner than threading metadata through each call site. gRPC requires transport security for per-RPC creds unless you explicitly opt out (a safety guard so you don't send bearer tokens in cleartext).

```go
type tokenCreds struct {
	token            string
	requireTransport bool
}

func (t tokenCreds) GetRequestMetadata(_ context.Context, _ ...string) (map[string]string, error) {
	return map[string]string{"authorization": "Bearer " + t.token}, nil
}
func (t tokenCreds) RequireTransportSecurity() bool { return t.requireTransport }

// Attach to ALL calls on the connection:
conn, _ := grpc.NewClient("example.com:443",
	grpc.WithTransportCredentials(clientMTLS()),
	grpc.WithPerRPCCredentials(tokenCreds{token: jwt, requireTransport: true}),
)
```

The server validates the token in an **auth interceptor** (§10.4), mapping failures to `codes.Unauthenticated`/`codes.PermissionDenied`.

### 11.6 ALTS (Google-specific) **[A]**

**ALTS** (Application Layer Transport Security) is a mutual-auth scheme used inside Google Cloud (GKE etc.) where identities are managed by the platform rather than by certs you manage. `google.golang.org/grpc/credentials/alts` provides `alts.NewServerCreds()` / `alts.NewClientCreds()`. Outside GCP you'll use TLS/mTLS; this is mentioned so you recognize it in the wild.

### 11.7 Security checklist **[I/A]**

- **Never** run production gRPC with `insecure.NewCredentials()` — that's for local dev/tests only.
- Prefer **mTLS** for east-west (service-to-service); **TLS + token** for north-south (clients).
- Validate tokens in an interceptor; map failures to `Unauthenticated`/`PermissionDenied`; keep error messages generic.
- Set `tls.Config.MinVersion = tls.VersionTLS13` where peers support it.
- **Validate every input** in handlers (§6.3) — zero trust applies even to internal callers.
- **Bound resource usage:** set `MaxRecvMsgSize` (§17), cap streaming accumulation (§6.3), and rate-limit (interceptor) to resist DoS.
- Rotate certs/keys and tokens; never bake long-lived secrets into images or commit them.
- Gate or disable **reflection** (§14) and verbose error details in production.

---

## 12. Deadlines, Cancellation, Retries, Keepalive & Load Balancing

### 12.1 Deadlines & timeouts — do this on every call **[I]**

A **deadline** is an *absolute time* by which the RPC must complete. (A "timeout" is just a deadline expressed relative to now.) The key insight that makes gRPC deadlines powerful: gRPC **propagates the deadline across hops** via metadata, so a downstream service automatically knows how much time is left and can give up early instead of doing doomed work. This is "deadline propagation," and it's why you should thread the *same* `context` through your whole call chain (see `context` in **`GO_GUIDE.md` §12**).

Set it with `context.WithTimeout`/`WithDeadline`:

```go
ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
defer cancel() // ALWAYS cancel to release the timer, even on the success path
_, err := client.GetUser(ctx, req)
if status.Code(err) == codes.DeadlineExceeded {
	// the call ran out of time — handle/fallback
}
```

On the **server**, always `select` on `ctx.Done()` in loops and long operations (as in §6.3's `ListUpdates`) so a blown deadline actually frees the goroutine and any held resources. A handler that ignores `ctx` keeps running after the client has given up — wasted work and a leak vector.

### 12.2 Cancellation **[I]**

Cancelling the client context (explicitly, or by the deadline firing) signals the server: the inbound `ctx.Done()` closes and `ctx.Err()` returns `context.Canceled` / `context.DeadlineExceeded`. **Propagate the same context into downstream calls** so cancellation cascades through the whole tree of work — a user closing a browser tab can free resources all the way down.

### 12.3 Retries via service config (built-in, declarative) **[I/A]**

grpc-go has a **declarative retry policy** configured by a JSON **service config**. This is the recommended way to retry — it understands gRPC semantics (which codes are retryable, exponential backoff with jitter, attempt caps) and integrates with deadlines (it won't retry past the deadline). Prefer it over a hand-rolled retry interceptor.

```go
const serviceConfig = `{
  "methodConfig": [{
    "name": [{"service": "user.v1.UserService"}],
    "waitForReady": true,
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2.0,
      "retryableStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"]
    }
  }]
}`

conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithDefaultServiceConfig(serviceConfig),
)
```

**⚡ Version note & critical safety rule:** Retries are enabled by default in current grpc-go (the old experimental flags are gone). Only retry **idempotent** methods, or methods you've explicitly made safe (e.g. via an idempotency key) — retrying a non-idempotent `CreateOrder` on `Unavailable` can **double-charge a customer**. Choose `retryableStatusCodes` precisely: `Unavailable` is the canonical safe one; never blanket-retry `Internal` or `InvalidArgument`.

### 12.4 `waitForReady` **[I]**

By default, an RPC issued while the channel is in `TRANSIENT_FAILURE` fails fast with `Unavailable`. Setting `waitForReady: true` (or `grpc.WaitForReady(true)` per call) makes the RPC *queue* until the channel becomes ready or the deadline fires — useful at startup when a dependency is still coming up. **Dangerous without a deadline** (it can hang forever), so always pair it with one.

### 12.5 Keepalive **[I/A]**

Keepalive pings detect dead connections and keep idle ones alive through NATs and load balancers that drop quiet TCP sessions.

```go
import "google.golang.org/grpc/keepalive"

// Client side:
conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithKeepaliveParams(keepalive.ClientParameters{
		Time:                30 * time.Second, // ping if no activity for 30s
		Timeout:             10 * time.Second, // wait 10s for a ping ack, else close
		PermitWithoutStream: true,             // ping even with no active RPCs
	}),
)

// Server side ALSO needs an enforcement policy or it may GOAWAY aggressive clients:
srv := grpc.NewServer(
	grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
		MinTime:             10 * time.Second, // reject pings more frequent than this
		PermitWithoutStream: true,
	}),
	grpc.KeepaliveParams(keepalive.ServerParameters{
		Time:    2 * time.Hour,
		Timeout: 20 * time.Second,
	}),
)
```

**⚡ Gotcha:** if the client's keepalive `Time` is *smaller* than the server's `MinTime`, the server closes connections with "too_many_pings". Keep client `Time` ≥ server `MinTime`.

### 12.6 Load balancing basics **[I/A]**

gRPC does **client-side load balancing**: a single `ClientConn` resolves a target to a *list* of backends and balances RPCs across them. Two pieces:

1. **Name resolver** — turns a target URI into addresses. Built-ins: `dns:///user-svc:50051` (re-resolves DNS periodically), `passthrough:///` (use the literal address verbatim). Schemes like `xds:///` integrate with a control plane (Envoy/Istio).
2. **Load-balancing policy** — `pick_first` (default; one backend at a time) or `round_robin` (spread across all resolved backends).

```go
conn, _ := grpc.NewClient(
	"dns:///user-svc.default.svc.cluster.local:50051",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
)
```

**Why this matters (the HTTP/2 trap):** because HTTP/2 multiplexes everything over *one* long-lived connection, a naive L4 (TCP) load balancer pins all of a client's RPCs to a single backend — defeating balancing entirely. **Client-side LB** (or an L7 proxy like Envoy) is what actually spreads load. See §18.

### 12.7 "Connection pooling" reality **[A]**

You usually do **not** pool `ClientConn`s — one multiplexed connection handles many concurrent RPCs (§7.7, §17). The rare exception: extremely high throughput can saturate a single HTTP/2 connection's `MAX_CONCURRENT_STREAMS` or hit single-connection CPU limits; then you might run a few connections (e.g. multiple `ClientConn`s, or `round_robin` over multiple resolved addresses). **Measure before pooling** — premature pooling adds complexity for no gain.

---

## 13. Interoperability — gRPC-Gateway, gRPC-Web, ConnectRPC

gRPC is excellent service-to-service, but **browsers can't speak raw gRPC** — the `fetch`/XHR APIs can't access HTTP/2 framing and trailers the way gRPC needs (recall the final status arrives as a trailer, §2.2). Three bridges solve this, letting you keep one `.proto` contract while serving non-gRPC clients.

### 13.1 gRPC-Gateway — REST/JSON from your gRPC service **[A]**

[`grpc-gateway`](https://github.com/grpc-ecosystem/grpc-gateway) generates a **reverse proxy** that exposes a RESTful JSON API and translates it to gRPC. You annotate methods with HTTP mappings via the `google.api.http` option. This is the "one contract, two surfaces" pattern from §3.3 — internal callers use gRPC, browsers/partners use REST.

```proto
// proto/user/v1/user.proto (add the annotation import + options)
import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"   // maps the URL path var {id} to the request's `id` field
    };
  }
  rpc CreateUser(CreateUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      post: "/v1/users"
      body: "*"               // the whole JSON body maps to the request message
    };
  }
}
```

Generate the gateway stub with `protoc-gen-grpc-gateway`, then run it alongside (or in front of) your gRPC server:

```go
import (
	"context"
	"net/http"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	userv1 "github.com/example/app/gen/user/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func runGateway(ctx context.Context, grpcAddr string) error {
	mux := runtime.NewServeMux() // translates HTTP/JSON <-> gRPC using protojson
	// Register the generated handler, dialing the gRPC backend.
	err := userv1.RegisterUserServiceHandlerFromEndpoint(
		ctx, mux, grpcAddr,
		[]grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())},
	)
	if err != nil {
		return err
	}
	// Now `curl http://localhost:8080/v1/users/u1` proxies to GetUser and returns JSON.
	return http.ListenAndServe(":8080", mux)
}
```

**Security note:** the gateway is a public HTTP edge — apply the same defenses you would to any REST API (see the REST guides): TLS, auth, rate limiting, request-size limits, CORS. It does not magically inherit your gRPC interceptors unless you wire equivalent HTTP middleware.

### 13.2 gRPC-Web — gRPC for browsers via a proxy **[A]**

**gRPC-Web** is a wire protocol the browser *can* speak (it doesn't require raw HTTP/2 trailers the same way). It needs a translating proxy — historically Envoy's gRPC-Web filter, or a standalone wrapper. The browser uses generated TypeScript clients. gRPC-Web supports **unary and server-streaming** but **not** client/bidi streaming (a browser limitation). Increasingly, teams reach for ConnectRPC instead because it needs no separate proxy.

### 13.3 ConnectRPC — the modern, browser-friendly choice **[A]**

[**ConnectRPC**](https://connectrpc.com) (`connectrpc.com/connect` for Go, "connect-go") is a from-scratch implementation that speaks **three** protocols over plain HTTP/1.1 *or* HTTP/2 on a single port:

1. The **Connect protocol** — simple, debuggable, `curl`-able JSON or binary.
2. **gRPC** — fully interoperable with grpc-go servers/clients.
3. **gRPC-Web** — directly, **no proxy needed**.

That means a browser can call your service with no Envoy, and you can `curl` a Connect endpoint like REST, while *the same server still answers grpc-go clients*. Handlers are plain `http.Handler`s, so they slot into any Go HTTP stack (router, middleware, TLS) you already use — the same stacks covered in the REST guides.

```go
// Connect server: generated with protoc-gen-connect-go (often via buf remote plugins).
package main

import (
	"context"
	"errors"
	"net/http"

	"connectrpc.com/connect"
	userv1 "github.com/example/app/gen/user/v1"
	"github.com/example/app/gen/user/v1/userv1connect" // connect-generated
)

type userServer struct{}

// Note the Connect signature: requests/responses are WRAPPED in connect.Request/Response.
func (s *userServer) GetUser(
	ctx context.Context,
	req *connect.Request[userv1.GetUserRequest],
) (*connect.Response[userv1.GetUserResponse], error) {
	id := req.Msg.GetId()
	if id == "" {
		// Connect has its own code/error type, mirroring gRPC codes.
		return nil, connect.NewError(connect.CodeInvalidArgument, errors.New("id required"))
	}
	resp := connect.NewResponse(&userv1.GetUserResponse{
		User: &userv1.User{Id: id, DisplayName: "Ada"},
	})
	resp.Header().Set("X-Served-By", "connect")
	return resp, nil
}

func main() {
	mux := http.NewServeMux()
	// The generated handler returns a (path, http.Handler) pair.
	path, handler := userv1connect.NewUserServiceHandler(&userServer{})
	mux.Handle(path, handler)
	// For local dev without TLS, wrap with h2c (HTTP/2 cleartext): import
	// golang.org/x/net/http2 and golang.org/x/net/http2/h2c. Use real TLS in prod.
	_ = http.ListenAndServe(":8080", mux)
}
```

A Connect **client** is equally simple and works from Go, TypeScript (browser), etc.:

```go
client := userv1connect.NewUserServiceClient(http.DefaultClient, "http://localhost:8080")
resp, err := client.GetUser(ctx, connect.NewRequest(&userv1.GetUserRequest{Id: "u1"}))
```

**When to pick what:**
- Pure internal Go/polyglot microservices, mesh, maximal ecosystem → **grpc-go**.
- Need a REST/JSON edge for the same protos → add **grpc-gateway**.
- Browser clients and/or want `curl`-friendly endpoints with no proxy → **ConnectRPC** (it still speaks gRPC to your other services).

---

## 14. Health Checking & Reflection

### 14.1 The standard health-checking protocol **[I]**

gRPC defines a *standard* health service (`grpc.health.v1.Health`) so load balancers and orchestrators (Kubernetes, Envoy) can probe liveness/readiness **uniformly**, regardless of what your service does. Using the standard means your platform's tooling "just works."

```go
import (
	"google.golang.org/grpc"
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

srv := grpc.NewServer()

hs := health.NewServer()
healthpb.RegisterHealthServer(srv, hs)

// Mark the whole server, and/or specific services, healthy.
hs.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)                      // overall
hs.SetServingStatus("user.v1.UserService", healthpb.HealthCheckResponse_SERVING)   // per-service

// During shutdown, flip to NOT_SERVING so the LB drains you BEFORE you stop (§18).
// hs.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
```

Kubernetes can probe it directly via the built-in `grpc` probe type (no exec wrapper needed):

```yaml
readinessProbe:
  grpc:
    port: 50051
    service: "user.v1.UserService"
```

### 14.2 Server reflection **[I]**

**Reflection** lets tools discover your services and message schemas *at runtime* — no `.proto` file needed by the caller. It's essential for debugging with `grpcurl` and graphical clients (Postman, Kreya, etc.).

```go
import "google.golang.org/grpc/reflection"

srv := grpc.NewServer()
userv1.RegisterUserServiceServer(srv, impl)
reflection.Register(srv) // enable server reflection
```

**⚡ Security note:** reflection exposes your *entire* API surface to anyone who can reach the port. Enable it freely in dev/staging; in production, **gate it behind auth or disable it** on public endpoints — you don't want to hand attackers a map of your methods.

### 14.3 Debugging with `grpcurl` **[I]**

`grpcurl` is `curl` for gRPC. With reflection on:

```bash
# List services
grpcurl -plaintext localhost:50051 list

# List methods of a service
grpcurl -plaintext localhost:50051 list user.v1.UserService

# Describe a message/method
grpcurl -plaintext localhost:50051 describe user.v1.GetUserRequest

# Call a unary method with JSON args
grpcurl -plaintext -d '{"id":"u1"}' localhost:50051 user.v1.UserService/GetUser

# Pass metadata and use TLS
grpcurl -H 'authorization: Bearer TOKEN' -cacert ca.crt \
  -d '{"id":"u1"}' example.com:443 user.v1.UserService/GetUser
```

Without reflection, point grpcurl at the protos: `grpcurl -import-path proto -proto user/v1/user.proto ...`.

---

## 15. Testing gRPC — bufconn, Mocks & Streams

### 15.1 In-memory testing with `bufconn` **[I/A]**

`google.golang.org/grpc/test/bufconn` gives you a real gRPC server and client talking over an **in-memory pipe** — no TCP, no ports, no flakiness. This is the recommended way to test handlers end-to-end: it exercises the *real* serialization, interceptors, and status mapping, unlike a mock. (For Go testing fundamentals — `testing.T`, table tests, subtests — see **`GO_GUIDE.md` §15**.)

```go
// internal/service/user_test.go
package service_test

import (
	"context"
	"net"
	"testing"

	userv1 "github.com/example/app/gen/user/v1"
	"github.com/example/app/internal/service"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
	"google.golang.org/grpc/test/bufconn"
)

func newTestClient(t *testing.T) userv1.UserServiceClient {
	t.Helper()
	const bufSize = 1024 * 1024
	lis := bufconn.Listen(bufSize)

	srv := grpc.NewServer()
	userv1.RegisterUserServiceServer(srv, service.NewUserServer(testLogger()))
	go func() { _ = srv.Serve(lis) }()
	t.Cleanup(srv.Stop)

	// The custom dialer returns the in-memory listener's connection.
	conn, err := grpc.NewClient(
		"passthrough:///bufnet", // passthrough so our dialer is used verbatim
		grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
			return lis.DialContext(ctx)
		}),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		t.Fatalf("NewClient: %v", err)
	}
	t.Cleanup(func() { _ = conn.Close() })
	return userv1.NewUserServiceClient(conn)
}

func TestGetUser(t *testing.T) {
	client := newTestClient(t)

	t.Run("found", func(t *testing.T) {
		resp, err := client.GetUser(context.Background(), &userv1.GetUserRequest{Id: "u1"})
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if got := resp.GetUser().GetDisplayName(); got != "Ada" {
			t.Errorf("display name = %q, want Ada", got)
		}
	})

	t.Run("not found", func(t *testing.T) {
		_, err := client.GetUser(context.Background(), &userv1.GetUserRequest{Id: "nope"})
		if status.Code(err) != codes.NotFound {
			t.Errorf("code = %v, want NotFound", status.Code(err))
		}
	})

	t.Run("invalid", func(t *testing.T) {
		_, err := client.GetUser(context.Background(), &userv1.GetUserRequest{Id: ""})
		if status.Code(err) != codes.InvalidArgument {
			t.Errorf("code = %v, want InvalidArgument", status.Code(err))
		}
	})
}
```

### 15.2 Testing a streaming method **[I]**

```go
func TestUploadChunks(t *testing.T) {
	client := newTestClient(t)
	stream, err := client.UploadChunks(context.Background())
	if err != nil {
		t.Fatal(err)
	}
	for i := 0; i < 3; i++ {
		if err := stream.Send(&userv1.UploadChunk{Data: []byte("abcd")}); err != nil {
			t.Fatal(err)
		}
	}
	summary, err := stream.CloseAndRecv()
	if err != nil {
		t.Fatal(err)
	}
	if summary.GetChunkCount() != 3 || summary.GetTotalBytes() != 12 {
		t.Errorf("summary = %+v, want 3 chunks / 12 bytes", summary)
	}
}
```

### 15.3 Mocking dependencies vs mocking the client **[I/A]**

- **Prefer bufconn** to test the *real* server with *real* generated stubs — it covers serialization, interceptors, and status mapping that a mock would skip.
- To unit-test code that *calls* a gRPC service, mock the **generated client interface** (`userv1.UserServiceClient`), not the concrete stub. Tools: `go.uber.org/mock` (gomock) generating from the interface, or a hand-written fake.
- Stream interfaces (e.g. `grpc.ServerStreamingClient[Update]`) are also interfaces you can fake for table-driven tests of streaming consumers.
- **Test the security paths too:** add cases asserting `Unauthenticated`/`PermissionDenied` for missing/invalid tokens — security regressions are easy to introduce and easy to catch with a test.

---

## 16. Observability — OpenTelemetry, Prometheus, Logging

### 16.1 Why observability is non-optional for RPC **[A]**

In a monolith a stack trace tells the whole story; in a mesh of gRPC services a single user action fans out across many hops, and you need **traces** (the path of one request), **metrics** (rates/errors/durations in aggregate), and **logs** (discrete events) to understand behavior. gRPC's metadata (§8) is the carrier that lets trace context propagate across hops — which is why observability and metadata are intertwined.

### 16.2 OpenTelemetry (the modern default for traces + metrics) **[A]**

**⚡ Version note:** The current way to instrument grpc-go is the official **`google.golang.org/grpc/stats/opentelemetry`** module (a stats-handler), which superseded the older `otelgrpc` *interceptors* for many use cases. It emits standard RPC metrics and propagates trace context automatically.

```go
import (
	"go.opentelemetry.io/otel/sdk/metric"
	"google.golang.org/grpc"
	"google.golang.org/grpc/stats/opentelemetry"
)

mp := metric.NewMeterProvider( /* readers/exporters */ )

// Server: attach the OpenTelemetry stats handler.
srv := grpc.NewServer(
	opentelemetry.ServerOption(opentelemetry.Options{
		MetricsOptions: opentelemetry.MetricsOptions{MeterProvider: mp},
	}),
)

// Client: the dial-option counterpart.
conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	opentelemetry.DialOption(opentelemetry.Options{
		MetricsOptions: opentelemetry.MetricsOptions{MeterProvider: mp},
	}),
)
```

For distributed *tracing* across services, also configure an OTel `TracerProvider` and a propagator (e.g. W3C TraceContext); the stats handler carries trace context in metadata so spans link up across hops.

### 16.3 Prometheus metrics **[A]**

`github.com/grpc-ecosystem/go-grpc-middleware/providers/prometheus` (successor to `go-grpc-prometheus`) provides interceptors that emit RED metrics (Rate, Errors, Duration):

```go
import (
	grpcprom "github.com/grpc-ecosystem/go-grpc-middleware/providers/prometheus"
	"github.com/prometheus/client_golang/prometheus"
)

srvMetrics := grpcprom.NewServerMetrics()
prometheus.MustRegister(srvMetrics)

srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(srvMetrics.UnaryServerInterceptor()),
	grpc.ChainStreamInterceptor(srvMetrics.StreamServerInterceptor()),
)
srvMetrics.InitializeMetrics(srv) // pre-create series for all methods (avoids gaps)

// Expose /metrics via a normal HTTP server with promhttp.Handler().
```

### 16.4 Logging **[A]**

Use the `go-grpc-middleware/v2` logging interceptor with a `slog` adapter (§10.8). Log per call: method, status code, duration, peer, request ID (from metadata). **Security:** do **not** log full request/response bodies by default — they may contain PII or secrets and are large. Never log `authorization` metadata. Redact or sample.

### 16.5 Debugging in practice **[A]**

- `grpcurl` (§14) to poke endpoints by hand.
- Set `GRPC_GO_LOG_VERBOSITY_LEVEL=99` and `GRPC_GO_LOG_SEVERITY_LEVEL=info` env vars for verbose grpc-go internal logs when chasing connection/resolver issues.
- The OTel/Prometheus dashboards above for aggregate latency and error-rate regressions.

---

## 17. Performance — Message Size, Compression, Reuse

### 17.1 Message-size limits **[A]**

grpc-go defaults: **4 MiB max receive** size, and (effectively) unlimited send. Hitting the receive cap returns `ResourceExhausted`. Tune deliberately — large messages hurt latency and memory and risk DoS; prefer *streaming* for big payloads. The receive cap is also a **security control** (it bounds attacker-supplied memory).

```go
// Server:
grpc.NewServer(
	grpc.MaxRecvMsgSize(16*1024*1024),
	grpc.MaxSendMsgSize(16*1024*1024),
)
// Client (default for all calls on the conn, or per-call via CallOptions):
conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(16*1024*1024)),
)
```

### 17.2 Compression **[A]**

gRPC supports per-message compression. `gzip` ships in `google.golang.org/grpc/encoding/gzip`; importing it for its side effect registers it.

```go
import (
	"google.golang.org/grpc"
	"google.golang.org/grpc/encoding/gzip"
	_ "google.golang.org/grpc/encoding/gzip" // registers the "gzip" compressor
)

// Per call:
resp, err := client.GetUser(ctx, req, grpc.UseCompressor(gzip.Name))
_ = resp
_ = err
```

Compression trades CPU for bandwidth. It pays off for large, *compressible* payloads over constrained links; it can *hurt* for small messages or already-compressed data (images, encrypted blobs). **Security caveat:** compressing data that mixes secrets with attacker-influenced content can enable compression side-channel attacks (CRIME/BREACH-style) — be cautious compressing responses that include both secrets and reflected input. Measure before enabling.

### 17.3 Streaming vs unary for throughput **[A]**

- Use **streaming** to avoid buffering huge payloads in memory and to overlap producer/consumer work (HTTP/2 flow control gives you back-pressure for free).
- Use **unary** for simple request/response — easier to reason about, cache, retry, and load-balance per call.
- **Caveat:** streaming RPCs pin to a single connection/backend for their lifetime, which can skew load balancing for very long-lived streams (§18).

### 17.4 Connection reuse — the biggest practical win **[A]**

**Reuse one `ClientConn`** across all calls to a backend. Dialing per request throws away HTTP/2 multiplexing and pays a TCP+TLS handshake every time (often the dominant cost). Create the `ClientConn` once at startup, share it (it's concurrency-safe), and close it at shutdown. Keepalive (§12.5) then keeps that connection healthy so you avoid reconnect storms.

---

## 18. Deployment — Load Balancers, Docker, Graceful Shutdown

### 18.1 Why HTTP/2 changes load balancing **[A]**

gRPC runs on **long-lived, multiplexed HTTP/2 connections**, which breaks the assumption most L4 load balancers make. A **layer-4 (TCP) load balancer** balances *connections*, not *requests* — so once a client connects, *all* its RPCs stick to one backend, defeating balancing. The three fixes:

1. **L7 (HTTP/2-aware) proxy** — Envoy, NGINX (`grpc_pass`), HAProxy, Linkerd, Istio. These balance per-request.
2. **Client-side LB** — `round_robin` over a DNS/xDS resolver (§12.6); each client spreads its own RPCs.
3. **Headless service + client LB** in Kubernetes — a headless `Service` returns all pod IPs; grpc-go's `dns:///` resolver + `round_robin` balances across them.

### 18.2 Dockerizing a gRPC server **[A]**

```dockerfile
# ---- build stage ----
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Static binary so the runtime image can be tiny and have no libc dependency.
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/server ./cmd/server

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/server /server
EXPOSE 50051
USER nonroot:nonroot          # SECURITY: never run as root
ENTRYPOINT ["/server"]
```

### 18.3 Health checks in orchestration **[A]**

Wire the standard health service (§14.1) and use Kubernetes' native `grpc` probes. Crucially, flip the health status to `NOT_SERVING` *before* shutting down so the LB drains traffic away from the pod before it stops accepting.

### 18.4 Graceful shutdown **[A]**

```go
func shutdown(srv *grpc.Server, hs *health.Server) {
	// 1) Tell LBs to stop sending new traffic.
	hs.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
	// 2) (Optional) give the LB a moment to notice before we stop accepting.
	time.Sleep(2 * time.Second)
	// 3) GracefulStop: stop accepting new RPCs; wait for in-flight ones to finish.
	done := make(chan struct{})
	go func() { srv.GracefulStop(); close(done) }()
	select {
	case <-done: // clean
	case <-time.After(30 * time.Second):
		srv.Stop() // force-close if something is stuck (e.g. an endless stream)
	}
}
```

`GracefulStop` blocks until all pending RPCs complete (**including long-lived streams!**), so always bound it with a timeout and fall back to `Stop()`.

### 18.5 Deployment checklist **[A]**

- TLS terminated at the edge *or* end-to-end (prefer end-to-end / mTLS for zero-trust).
- L7 LB or client-side LB — never plain L4 alone.
- Resource limits + `MaxConcurrentStreams` and `MaxRecvMsgSize` tuned.
- Readiness gated on the health service; liveness separate.
- Graceful shutdown bounded by a timeout.
- Reflection disabled or auth-gated in prod; run container as non-root; secrets via env/secret store, not baked into images.

---

## 19. Gotchas & Best Practices

### 19.1 Protobuf / schema **[I/A]**

- **Never** reuse or renumber an existing field. `reserved` removed numbers *and* names (§4.10).
- Always add a `0` `*_UNSPECIFIED` enum member; prefix members with the enum name.
- Version your proto packages (`v1`, `v2`); don't make breaking changes in place.
- Run `buf lint` and `buf breaking` in CI — make incompatible changes impossible to merge.
- Use `optional` (or wrappers) when you must distinguish "unset" from "zero."
- Don't put huge blobs in a single message; stream them (§7.4, §17.3).

### 19.2 Go server/client **[I/A]**

- **Embed `UnimplementedXxxServer`** in every implementation — your forward-compatibility insurance.
- **Always set a deadline** on client calls. No deadline = potential permanent hang.
- **Reuse `ClientConn`** — don't dial per request.
- Use **`grpc.NewClient`**, not the deprecated `grpc.Dial`.
- **Never block indefinitely in a handler** without honoring `ctx.Done()` — it ties up a goroutine and ignores client cancellation.
- On streams, **don't call `Send` (or `Recv`) concurrently** from multiple goroutines; one-sending-one-receiving is fine.
- Drain server streams to `io.EOF` on the client (or cancel) to avoid leaks.

### 19.3 Errors **[I]**

- Return `status.Error(codes.X, ...)`, never bare errors (they become `Unknown`).
- Pick the **right code** — clients and retry policies act on it. `Unavailable` invites retries; `Internal` should not be retried automatically.
- Don't leak internal error text/stack traces to clients; translate at the boundary (§9.6).
- Use `errdetails` for machine-actionable specifics (field violations, retry-after, quota).

### 19.4 Reliability **[A]**

- Only auto-retry **idempotent** methods; configure `retryableStatusCodes` precisely.
- Match client keepalive `Time` with server `EnforcementPolicy.MinTime` to avoid "too_many_pings" GOAWAYs.
- Propagate deadlines and cancellation into downstream calls.
- Use `waitForReady` only with a deadline.

### 19.5 Security **[I/A]**

- TLS/mTLS in production; `insecure` only for local dev/tests.
- Validate auth in interceptors; map failures to `Unauthenticated`/`PermissionDenied`; keep messages generic.
- Validate every input in handlers (zero trust, even internal callers).
- Bound resources: `MaxRecvMsgSize`, streaming caps, rate limits.
- Gate or disable reflection in production; don't log tokens/PII; run containers non-root.

---

## 20. Study Path & Build-to-Learn Projects

A six-stage path from "what is RPC" to "production microservices." **Build the projects — reading alone won't make it stick.** Cross-reference **`GO_GUIDE.md`** for any language concept (context, goroutines, interfaces) you're shaky on, and the REST guides when you compare surfaces.

### Stage 1 — RPC foundations (Week 1)

1. **`net/rpc` round-trip** — implement the `Arith` example from §1 (server + sync + async client). Then swap in the `jsonrpc` codec and inspect the JSON on the wire with `nc`/`socat`. Articulate *why* this doesn't scale across languages.
2. **First proto** — write a `user.v1` proto with a `User` message and a unary `GetUser`. Install `protoc`, `protoc-gen-go`, `protoc-gen-go-grpc`; generate code by hand-running `protoc`. Read the generated `.pb.go` and `_grpc.pb.go`.
3. **Switch to buf** — add `buf.yaml` + `buf.gen.yaml`; run `buf lint`, `buf format`, `buf generate`. Compare ergonomics to raw `protoc`.

### Stage 2 — All four RPC types (Week 2)

4. **Unary server + client** — implement `GetUser` end-to-end with `grpc.NewClient`, deadlines, and proper `status`/`codes` errors. Probe it with `grpcurl` (enable reflection).
5. **Server streaming** — `ListUpdates` that emits 5 ticks; client drains to `io.EOF`; honor `stream.Context().Done()`.
6. **Client streaming** — `UploadChunks` that sums bytes; client `Send`s then `CloseAndRecv`. Compare with a REST multipart upload (`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`).
7. **Bidi streaming** — `Chat` echo server; client uses a goroutine for `Recv` while `Send`ing. Compare with WebSockets (`GO_GORILLA_WEBSOCKETS_GUIDE.md`).

### Stage 3 — Cross-cutting concerns (Week 3)

8. **Interceptors** — add logging, recovery (panic→`Internal`), and a JWT auth interceptor (server) + token-injecting client interceptor. Then replace your hand-rolled ones with `go-grpc-middleware/v2`.
9. **Metadata** — propagate an `x-request-id` from client to server and into downstream calls; return a trailer.
10. **Rich errors** — return `InvalidArgument` with `errdetails.BadRequest` field violations; inspect them on the client.

### Stage 4 — Build Project 1: Chat Service (Week 4)

11. **Bidi chat** — a multi-room chat using bidirectional streaming. Server maintains rooms (`map[room]map[*client]chan msg`); each connected stream gets a goroutine. Broadcast join/leave events.
12. **Auth + metadata** — clients authenticate via a token in metadata; room/user identity comes from an auth interceptor.
13. **Graceful shutdown** — on SIGTERM, flip health to `NOT_SERVING`, notify clients, `GracefulStop`.

### Stage 5 — Build Project 2: Microservice Pair + REST Edge (Week 5)

14. **Two services** — `OrderService` calls `UserService` over gRPC with **mTLS** between them. Propagate deadlines and request IDs across the hop.
15. **Retries + LB** — configure a retry policy via service config; run two `UserService` replicas behind `dns:///` + `round_robin`.
16. **gRPC-Gateway** — add `google.api.http` annotations and stand up a REST/JSON gateway so a browser/curl can hit `OrderService`. One contract, two surfaces.
17. **Observability** — add OpenTelemetry metrics + Prometheus `/metrics`; trace a request across both services.

### Stage 6 — Production hardening + ConnectRPC (Week 6)

18. **ConnectRPC** — re-expose `UserService` with connect-go on a single port; call it from a TypeScript browser client with **no proxy**. Confirm the same server still answers grpc-go clients.
19. **Schema CI** — wire `buf lint` + `buf breaking` into CI; deliberately make a breaking change and watch it fail.
20. **Testing** — full bufconn test suite for all four method types, plus a gomock-based unit test of a caller, plus auth-failure cases. Add a stream test.
21. **Deploy** — Dockerize (distroless, non-root), add Kubernetes `grpc` health probes, headless service + client LB, and validate graceful drain under load.

### Reference bookmarks

| Resource | URL |
|---|---|
| gRPC Go docs | https://pkg.go.dev/google.golang.org/grpc |
| Protobuf Go docs | https://pkg.go.dev/google.golang.org/protobuf |
| Protobuf language guide (proto3) | https://protobuf.dev/programming-guides/proto3/ |
| gRPC official site & guides | https://grpc.io/docs/ |
| buf docs | https://buf.build/docs |
| grpc-gateway | https://grpc-ecosystem.github.io/grpc-gateway/ |
| ConnectRPC (Go) | https://connectrpc.com/docs/go/ |
| go-grpc-middleware v2 | https://github.com/grpc-ecosystem/go-grpc-middleware |
| grpcurl | https://github.com/fullstorydev/grpcurl |
| errdetails | https://pkg.go.dev/google.golang.org/genproto/googleapis/rpc/errdetails |

### Related guides in this library

| Guide | Why read it alongside this one |
|---|---|
| `GO_GUIDE.md` | The Go language itself — context, goroutines, channels, interfaces, generics, errors, testing |
| `GO_NET_HTTP_REST_API_GUIDE.md` | The stdlib REST alternative; REST-vs-gRPC trade-offs (§3) |
| `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md` | Gin REST + file uploads; compare with client-streaming (§7.4) |
| `GO_GORILLA_WEBSOCKETS_GUIDE.md` | WebSockets — the other bidirectional option (compare §7.5) |
| `GO_JWT_ARGON2_GUIDE.md` | JWT/Argon2 primitives behind auth interceptors (§10–§11) |

---

*Guide accurate as of Go 1.23/1.24, proto3, `google.golang.org/grpc` v1.6x+ and `google.golang.org/protobuf` v1.3x+ (June 2026). The codegen toolchain (`protoc`/`buf`, `protoc-gen-go`, `protoc-gen-go-grpc`) and ConnectRPC evolve quickly — always cross-reference pkg.go.dev, protobuf.dev, buf.build, and connectrpc.com for the latest APIs.*
