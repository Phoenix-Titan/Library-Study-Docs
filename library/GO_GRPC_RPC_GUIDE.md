# gRPC & RPC with Go — Complete Offline Reference

> **Who this is for:** Go developers (beginner to intermediate) who want to understand Remote Procedure Calls from first principles — starting with the standard library `net/rpc`, then mastering **gRPC** and **Protocol Buffers** for building production microservices. Every concept ships with runnable, heavily commented `.proto` and Go code you can paste into a module and run. No prior gRPC experience assumed; no internet required to learn from this document.
>
> **Version note:** This guide targets **Go 1.23 / 1.24** (2025–2026), **Protocol Buffers proto3**, and the current generation of modules: `google.golang.org/grpc` (v1.6x+) and `google.golang.org/protobuf` (v1.3x+, the "APIv2" runtime). It uses the modern codegen toolchain — `protoc` **or** [**buf**](https://buf.build), plus `protoc-gen-go` and `protoc-gen-go-grpc`. It also covers **ConnectRPC** (`connectrpc.com/connect`) as a modern, browser-friendly alternative that speaks gRPC, gRPC-Web, and its own Connect protocol. Fast-moving details are flagged with **⚡ Version note**. Always confirm exact APIs at pkg.go.dev and protobuf.dev.

---

## Table of Contents

1. [RPC Fundamentals & the Standard Library `net/rpc`](#1-rpc-fundamentals--the-standard-library-netrpc)
2. [gRPC & Protocol Buffers Overview](#2-grpc--protocol-buffers-overview)
3. [Protocol Buffers (proto3) In Depth](#3-protocol-buffers-proto3-in-depth)
4. [Toolchain Setup — protoc, Plugins & buf](#4-toolchain-setup--protoc-plugins--buf)
5. [Building a gRPC Server in Go](#5-building-a-grpc-server-in-go)
6. [Building a gRPC Client in Go](#6-building-a-grpc-client-in-go)
7. [The Four Streaming Patterns End-to-End](#7-the-four-streaming-patterns-end-to-end)
8. [Metadata — Headers & Trailers](#8-metadata--headers--trailers)
9. [Error Handling — status, codes & rich errors](#9-error-handling--status-codes--rich-errors)
10. [Interceptors — gRPC's Middleware](#10-interceptors--grpcs-middleware)
11. [Authentication & Security — TLS, mTLS, Tokens](#11-authentication--security--tls-mtls-tokens)
12. [Deadlines, Retries, Keepalive & Load Balancing](#12-deadlines-retries-keepalive--load-balancing)
13. [Interoperability — gRPC-Gateway, gRPC-Web, ConnectRPC](#13-interoperability--grpc-gateway-grpc-web-connectrpc)
14. [Health Checking & Reflection](#14-health-checking--reflection)
15. [Testing gRPC — bufconn, Mocks & Streams](#15-testing-grpc--bufconn-mocks--streams)
16. [Observability — OpenTelemetry, Prometheus, Logging](#16-observability--opentelemetry-prometheus-logging)
17. [Performance — Message Size, Compression, Reuse](#17-performance--message-size-compression-reuse)
18. [Deployment — Load Balancers, Docker, Graceful Shutdown](#18-deployment--load-balancers-docker-graceful-shutdown)
19. [Gotchas & Best Practices](#19-gotchas--best-practices)
20. [Study Path & Build-to-Learn Projects](#20-study-path--build-to-learn-projects)

---

## 1. RPC Fundamentals & the Standard Library `net/rpc`

### What is RPC?

**Remote Procedure Call (RPC)** is a programming model where calling a function on a *remote* machine looks (almost) like calling a *local* function. You invoke `client.GetUser(ctx, id)` and the framework handles the messy parts: serializing arguments, sending bytes over the network, dispatching to the right function on the server, serializing the return value, and handing it back to you.

The core idea is old (1980s) but enduring: **make the network disappear behind a function call.** The four jobs every RPC system must do:

1. **Serialization (marshaling):** turn in-memory data structures into bytes (and back).
2. **Transport:** move those bytes across a connection (TCP, HTTP/2, etc.).
3. **Dispatch:** on the server, route an incoming request to the correct handler.
4. **Contract:** both sides must agree on method names, argument shapes, and return shapes.

The "almost" in "almost like a local call" is where engineering judgement lives. Remote calls can be **slow**, can **fail partway**, can **time out**, and can be **reordered**. A good RPC framework gives you tools (deadlines, retries, error codes) to deal with these realities. A naive one pretends the network is reliable — and your service falls over in production.

### RPC vs REST vs GraphQL

| Dimension | RPC (gRPC) | REST | GraphQL |
|---|---|---|---|
| Mental model | Call a function | Manipulate a resource (CRUD over HTTP verbs) | Query a graph; ask for exactly the fields you want |
| Contract | Strongly typed schema (`.proto`) | Convention + OpenAPI (optional) | Strongly typed schema (SDL) |
| Payload | Binary (Protocol Buffers) | Usually JSON | JSON |
| Transport | HTTP/2 (multiplexed, streaming) | HTTP/1.1 or HTTP/2 | Usually HTTP/1.1 POST |
| Streaming | First-class (4 modes) | SSE / chunked (awkward) | Subscriptions (over WS) |
| Browser support | Needs gRPC-Web proxy / Connect | Native | Native |
| Discoverability | Reflection / generated stubs | URLs + docs | Introspection |
| Best for | Internal microservices, low latency, polyglot | Public APIs, simple CRUD, caching | Aggregating many data sources, client-driven field selection |
| Caching | Manual | HTTP caching is built-in | Hard |

**Rules of thumb:**
- **gRPC** when you control both ends, need performance, want streaming, and work across languages — i.e. **service-to-service** inside a system.
- **REST** for public-facing APIs where ubiquity, cacheability, and "just curl it" matter.
- **GraphQL** when clients have wildly different data needs and you want to avoid endpoint sprawl (typically a gateway, not a leaf service).

These aren't mutually exclusive. A very common topology: gRPC between internal services, with a **gRPC-Gateway** (Section 13) exposing a REST/JSON edge for browsers and third parties.

### The Go standard library: `net/rpc`

Before gRPC, Go shipped its own RPC package: `net/rpc`. It is **still in the standard library**, still works, and is worth understanding because it demystifies what gRPC does under the hood. It is, however, **frozen** (no new features) and **Go-specific** by default (its native wire format is `encoding/gob`, which only Go speaks). There is also `net/rpc/jsonrpc` for a language-neutral JSON-RPC 1.0 codec.

#### Rules for `net/rpc` methods

A method is eligible to be registered for RPC only if it satisfies a rigid signature:

```
func (t *T) MethodName(argType T1, replyType *T2) error
```

1. The method's type (`T`) is **exported**.
2. The method is **exported**.
3. It has exactly **two arguments**, both **exported** (or builtin) types.
4. The **second argument is a pointer** (the reply, filled in by the method).
5. The method returns an `error`.

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
// encoding/gob can serialize them across the wire.
type Args struct {
	A, B int
}

// Quotient is a richer reply type to show structs flow both ways.
type Quotient struct {
	Quo, Rem int
}

// Arith is the receiver type whose methods we expose over RPC.
// It carries no state here, but it could hold dependencies (DB handles, etc.).
type Arith int

// Multiply matches the required signature:
//   (receiver) Method(args T1, reply *T2) error
// args is passed by value; reply is a pointer we fill in.
func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil // returning a non-nil error sends that error text to the client
}

// Divide demonstrates returning an application error.
func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		// This error string is transmitted to the client and surfaces there
		// as a plain *errors.errorString — there is no error *code*, just text.
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

	// Two common transports:
	//  (a) Over raw TCP with the default gob codec.
	//  (b) Over HTTP (handy for sharing a port / debugging).
	// We use HTTP here. HandleHTTP wires RPC onto the default HTTP mux.
	rpc.HandleHTTP()

	ln, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("listen:", err)
	}
	log.Println("net/rpc server listening on :1234")

	// http.Serve drives the accept loop; each connection is handled concurrently.
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
	// "Arith.Multiply" is "<registered name>.<method>".
	args := &Args{A: 7, B: 8}
	var product int
	if err := client.Call("Arith.Multiply", args, &product); err != nil {
		log.Fatal("Arith.Multiply error:", err)
	}
	fmt.Printf("7 * 8 = %d\n", product) // → 56

	// --- Asynchronous call ---
	// Go() returns immediately; the result lands on the Done channel.
	quot := new(Quotient)
	divCall := client.Go("Arith.Divide", &Args{A: 17, B: 5}, quot, nil)
	replyCall := <-divCall.Done // block until the async call completes
	if replyCall.Error != nil {
		log.Fatal("Arith.Divide error:", replyCall.Error)
	}
	fmt.Printf("17 / 5 = %d remainder %d\n", quot.Quo, quot.Rem) // → 3 r 2

	// --- Error path ---
	err = client.Call("Arith.Divide", &Args{A: 1, B: 0}, new(Quotient))
	fmt.Println("divide-by-zero call returned:", err) // → divide by zero
}
```

#### JSON-RPC variant (`net/rpc/jsonrpc`)

If you want a language-neutral wire format, swap the codec. The server registers identically but serves with `jsonrpc.NewServerCodec`:

```go
// Server side: accept raw TCP and serve each conn with the JSON codec.
ln, _ := net.Listen("tcp", ":1234")
for {
	conn, err := ln.Accept()
	if err != nil {
		continue
	}
	// One goroutine per connection, JSON-RPC 1.0 framing.
	go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
}

// Client side:
conn, _ := net.Dial("tcp", "localhost:1234")
client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
var product int
_ = client.Call("Arith.Multiply", &Args{A: 7, B: 8}, &product)
```

The JSON on the wire looks like:

```json
{"method":"Arith.Multiply","params":[{"A":7,"B":8}],"id":0}
{"id":0,"result":56,"error":null}
```

#### Why gRPC superseded `net/rpc`

`net/rpc` is elegant and tiny, but for modern distributed systems it falls short on almost every production axis:

| Concern | `net/rpc` | gRPC |
|---|---|---|
| Cross-language | No (gob); JSON-RPC is weakly typed | Yes — generated stubs for 10+ languages |
| Schema / contract | Implicit (Go structs) | Explicit, versioned `.proto` |
| Streaming | No | 4 first-class modes |
| Deadlines / cancellation | No `context` support | `context.Context` everywhere |
| Error model | Plain string | Rich `status` + `codes` + details |
| Metadata / headers | No | First-class metadata |
| Interceptors / middleware | No | Unary + stream interceptors |
| Transport | gob over TCP/HTTP | HTTP/2, multiplexed, flow-controlled |
| Auth / TLS | DIY | Built-in credentials, mTLS, per-RPC creds |
| Tooling | None | reflection, grpcurl, gateway, codegen |
| Maintenance | Frozen | Actively developed |

In short: `net/rpc` teaches you the *concept*; gRPC is what you *ship*. The rest of this guide is gRPC.

---

## 2. gRPC & Protocol Buffers Overview

### What gRPC is

**gRPC** (a recursive-ish acronym; officially "gRPC Remote Procedure Calls") is a high-performance, open-source RPC framework originally from Google, now a CNCF project. Its three pillars:

1. **Protocol Buffers** as the Interface Definition Language (IDL) and serialization format.
2. **HTTP/2** as the transport.
3. **Code generation** that produces strongly typed client and server stubs from your `.proto` files.

### Why HTTP/2?

HTTP/2 is what makes gRPC fast and streaming-capable:

- **Multiplexing:** many concurrent RPCs share one TCP connection without head-of-line blocking at the HTTP layer. No need for a connection pool to get parallelism.
- **Binary framing:** the protocol is binary, not text, so it's compact and fast to parse.
- **Header compression (HPACK):** repeated headers (metadata) cost far fewer bytes.
- **Bidirectional streams:** HTTP/2 streams map naturally onto gRPC's streaming RPC modes.
- **Flow control:** per-stream and per-connection back-pressure keeps a fast sender from drowning a slow receiver.

A gRPC call is, on the wire, an HTTP/2 `POST` to a path like `/package.Service/Method`, with length-prefixed protobuf messages in the body and gRPC-specific headers/trailers (e.g. `grpc-status`, `grpc-encoding`).

### Why Protocol Buffers?

Protobuf gives you:

- **Compactness:** binary encoding is typically 3–10x smaller than equivalent JSON.
- **Speed:** parsing is fast and allocation-light.
- **A strict, evolvable schema:** field numbers (not names) define the wire format, enabling safe schema evolution (Section 3).
- **Polyglot codegen:** one `.proto`, stubs in Go, Java, Python, C++, Rust, TypeScript, etc.

The trade-off: payloads aren't human-readable, and you need codegen tooling. For internal services that's a great deal; for a public "curl-friendly" API it's why people add a JSON edge.

### The four RPC types

This is the conceptual core of gRPC. Every method you define is one of four kinds:

| Type | Client sends | Server sends | Go signature shape | Example |
|---|---|---|---|---|
| **Unary** | 1 message | 1 message | `Get(ctx, req) (resp, err)` | `GetUser` |
| **Server streaming** | 1 message | stream | `List(ctx, req) (stream, err)` | `ListUpdates` |
| **Client streaming** | stream | 1 message | `Upload(ctx) (stream, err)` | `UploadChunks` |
| **Bidirectional streaming** | stream | stream | `Chat(ctx) (stream, err)` | `Chat` |

In proto, the `stream` keyword on the request and/or response side selects the mode:

```proto
service Demo {
  rpc Unary(Req) returns (Resp);                       // unary
  rpc ServerStream(Req) returns (stream Resp);         // server streaming
  rpc ClientStream(stream Req) returns (Resp);         // client streaming
  rpc BidiStream(stream Req) returns (stream Resp);    // bidirectional
}
```

We implement all four fully in Sections 5 and 7.

### The contract-first workflow

gRPC is **schema-first**. The canonical loop:

1. **Write `.proto`** — define messages and services. This is the source of truth and the contract between teams.
2. **Generate code** — `protoc`/`buf` produces Go types (`*.pb.go`) and service stubs (`*_grpc.pb.go`).
3. **Implement the server** — fill in the generated service interface.
4. **Use the client stub** — call methods like local functions.
5. **Evolve safely** — change the `.proto` following compatibility rules; regenerate.

The `.proto` file lives in version control and is often shared via a **schema registry** (e.g. the Buf Schema Registry) or a shared repo so multiple services and languages consume the same contract.

---

## 3. Protocol Buffers (proto3) In Depth

### Anatomy of a `.proto` file

```proto
// Every proto3 file MUST declare the syntax on line 1 (before comments count).
syntax = "proto3";

// The package namespaces messages/services to avoid name clashes across files.
// It becomes part of the fully-qualified name: e.g. user.v1.GetUserRequest.
package user.v1;

// go_package controls the Go import path AND package name of generated code.
// Format: "<import path>;<package name>". The part after ';' is optional but
// recommended for clarity. ALWAYS set this for Go.
option go_package = "github.com/example/app/gen/user/v1;userv1";

// Pull in well-known types and other protos.
import "google/protobuf/timestamp.proto";

// A message is a struct: a typed collection of fields.
message User {
  string id = 1;          // field number 1 — see "field numbers" below
  string email = 2;
  string display_name = 3;
  google.protobuf.Timestamp created_at = 4;
}
```

### Scalar types and their Go mappings

| proto3 type | Go type | Notes |
|---|---|---|
| `double` | `float64` | |
| `float` | `float32` | |
| `int32` | `int32` | Variable-length; inefficient for negatives |
| `int64` | `int64` | |
| `uint32` / `uint64` | `uint32` / `uint64` | |
| `sint32` / `sint64` | `int32` / `int64` | Zig-zag encoded — efficient for negatives |
| `fixed32` / `fixed64` | `uint32` / `uint64` | Always 4/8 bytes; efficient for large numbers |
| `sfixed32` / `sfixed64` | `int32` / `int64` | Signed fixed-width |
| `bool` | `bool` | |
| `string` | `string` | Must be valid UTF-8 |
| `bytes` | `[]byte` | Arbitrary binary |

**Default values (proto3):** unset scalar fields take the zero value — `0`, `false`, `""`, empty slice. Crucially, proto3 (by default) **does not distinguish "field set to zero" from "field unset"** for scalars. That's what `optional` (below) fixes.

### Field numbers — the heart of wire compatibility

```proto
message Example {
  string name = 1;   // The "= 1" is the FIELD NUMBER (a.k.a. tag), NOT a default.
  int32  age  = 2;
}
```

- Field numbers, **not names**, are what's written on the wire. You can rename a field freely; you must **never** reuse or change a field number for an existing field.
- Valid range: **1 to 536,870,911**, excluding **19,000–19,999** (reserved for protobuf internals).
- **1–15** use a single byte for the field tag; reserve them for the most frequently set fields. 16+ use two bytes.

### Messages, nested messages, enums

```proto
syntax = "proto3";
package shop.v1;
option go_package = "github.com/example/app/gen/shop/v1;shopv1";

// Enums: the first value MUST be 0 and is the default. Convention: a zero-valued
// UNSPECIFIED member so "unset" is distinguishable from a real choice.
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

  // A nested message type — scoped to Order (Go: Order_LineItem).
  message LineItem {
    string sku = 1;
    int32  quantity = 2;
    int64  unit_price_cents = 3;
  }

  // repeated = a list/slice (Go: []*Order_LineItem).
  repeated LineItem items = 3;
}
```

⚡ **Version note:** Enum values in proto3 are **open** by default — an unknown numeric value is preserved on round-trip rather than rejected. Prefix members with the enum name (`ORDER_STATUS_`) because enum value names share a C++-style scope and would otherwise collide across enums.

### `repeated` and `map`

```proto
message Catalog {
  repeated string tags = 1;             // Go: []string
  map<string, int64> stock_by_sku = 2;  // Go: map[string]int64
}
```

- `repeated` fields are unordered conceptually but preserve insertion order in practice.
- `map<K,V>`: keys may be any integral or string type (not float/bytes/message); values may be anything except another map. Maps cannot be `repeated`.

### `oneof` — at most one of a set

```proto
message Payment {
  string id = 1;

  // Exactly zero or one of the fields below may be set. Setting one clears
  // the others. Field numbers must be unique within the whole message.
  oneof method {
    Card   card   = 2;
    Bank   bank   = 3;
    string token  = 4;
  }
}

message Card { string last4 = 1; }
message Bank { string iban  = 1; }
```

In Go, a `oneof` generates an interface field plus wrapper types; you switch on it:

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

### `optional` — explicit presence for scalars

```proto
message Profile {
  string name = 1;            // proto3 default: "" indistinguishable from unset
  optional int32 age = 2;     // generates a *int32 in Go → nil means "not set"
}
```

`optional` brings back **field presence** for scalars. In Go the field becomes a pointer (`*int32`), so `nil` means "not provided" and `proto.Int32(0)` means "explicitly zero." Use it whenever the difference between "absent" and "zero" matters (PATCH-style updates, nullable columns, etc.).

⚡ **Version note:** Since protobuf adopted "editions," `optional` is fully standard in proto3. Message-typed fields *always* have presence (they're pointers regardless of `optional`).

### Well-known types (WKT)

These are pre-defined messages in `google/protobuf/`. Import and use them rather than reinventing.

| WKT | Import | Go type | Use |
|---|---|---|---|
| `Timestamp` | `google/protobuf/timestamp.proto` | `*timestamppb.Timestamp` | Points in time |
| `Duration` | `google/protobuf/duration.proto` | `*durationpb.Duration` | Spans of time |
| `Empty` | `google/protobuf/empty.proto` | `*emptypb.Empty` | "no request/response" |
| `StringValue`, `Int32Value`, … (wrappers) | `google/protobuf/wrappers.proto` | `*wrapperspb.StringValue` | Nullable scalars (pre-`optional` idiom) |
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
  // Empty request: a method that takes no parameters.
  rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

Converting in Go:

```go
import (
	"time"
	"google.golang.org/protobuf/types/known/timestamppb"
	"google.golang.org/protobuf/types/known/durationpb"
)

job := &jobsv1.Job{
	ScheduledAt: timestamppb.New(time.Now().Add(time.Hour)),
	Timeout:     durationpb.New(30 * time.Second),
}
t := job.ScheduledAt.AsTime()        // back to time.Time
d := job.Timeout.AsDuration()        // back to time.Duration
```

### `Any` — packing arbitrary messages

```go
import "google.golang.org/protobuf/types/known/anypb"

// Pack a concrete message into an Any (records its type URL).
packed, err := anypb.New(&userv1.User{Id: "u1"})

// Unpack: you must know (or check) the target type.
var u userv1.User
if err := packed.UnmarshalTo(&u); err == nil {
	fmt.Println("got user", u.Id)
}
```

`Any` is powerful but erodes type safety — prefer `oneof` when the set of possibilities is known and small.

### Packages, imports & file organization

```proto
// common/v1/money.proto
syntax = "proto3";
package common.v1;
option go_package = "github.com/example/app/gen/common/v1;commonv1";

message Money {
  string currency_code = 1; // ISO 4217, e.g. "USD"
  int64  units = 2;
  int32  nanos = 3;
}
```

```proto
// shop/v1/order.proto
syntax = "proto3";
package shop.v1;
option go_package = "github.com/example/app/gen/shop/v1;shopv1";

import "common/v1/money.proto"; // import path is relative to the proto root(s)

message Order {
  string id = 1;
  common.v1.Money total = 2; // reference imported type by fully-qualified name
}
```

**Convention:** version your packages (`user.v1`, `user.v2`) so you can run two major versions side by side. Put each `.proto` in a directory matching its package path.

### Schema evolution — backward & forward compatibility

This is the single most important operational topic. **Backward compatible** = new code reads old data. **Forward compatible** = old code reads new data. Protobuf is designed to give you both if you follow the rules.

**Safe (compatible) changes:**

- ✅ **Add** a new field with a new, unused field number. Old readers ignore it (stored as "unknown fields" and preserved on re-serialize); new readers see the default if it's absent.
- ✅ **Rename** a field or message (names aren't on the wire — but you'll break source code that references the old name).
- ✅ **Add** new enum values (readers treat unknown values as the raw number).
- ✅ **Add** new methods to a service.
- ✅ **Convert** between compatible scalar types in limited cases (`int32`/`int64`/`uint32`/`uint64`/`bool` are interchangeable on the wire — but watch for truncation/sign).
- ✅ **Convert** a single optional field into a member of a **new** `oneof` (with care).

**Unsafe (breaking) changes — never do these to a deployed schema:**

- ❌ **Change a field's number.** It's a different field entirely.
- ❌ **Reuse a field number** that was previously used for a different field/type. Old data will be misinterpreted.
- ❌ **Change a field's type** incompatibly (e.g. `string` ↔ `int32`).
- ❌ **Move** a field into or out of a `oneof` carelessly, or change `repeated` ↔ scalar.
- ❌ **Delete** a field and later reuse its number.

**When you remove a field, reserve its number and name** so nobody reuses them:

```proto
message User {
  reserved 5, 9 to 11;          // reserve removed field numbers
  reserved "legacy_token";       // reserve removed field names
  string id = 1;
  // ... field 5 is gone forever
}
```

Automate enforcement with `buf breaking` (Section 4) in CI so you can't accidentally ship an incompatible change.

### JSON mapping (proto3 ⇄ JSON)

Protobuf defines a canonical JSON mapping (used by gRPC-Gateway, ConnectRPC's JSON mode, and `protojson`):

| proto | JSON | Notes |
|---|---|---|
| field `display_name` | `"displayName"` | Defaults to **lowerCamelCase** (original name also accepted on input) |
| `int64` / `uint64` | string | To avoid JS precision loss: `"9007199254740993"` |
| `bytes` | base64 string | |
| `Timestamp` | RFC 3339 string | `"2026-06-21T10:00:00Z"` |
| `Duration` | string with `s` | `"3.5s"` |
| `enum` | string name | `"ORDER_STATUS_PAID"` (number also accepted) |
| unset fields | omitted | unless you ask to emit defaults |

```go
import "google.golang.org/protobuf/encoding/protojson"

b, _ := protojson.Marshal(user)                 // proto → JSON bytes
var u userv1.User
_ = protojson.Unmarshal(b, &u)                  // JSON bytes → proto

// Options worth knowing:
m := protojson.MarshalOptions{
	EmitUnpopulated: true,  // include zero-valued fields explicitly
	UseProtoNames:   true,  // snake_case keys instead of camelCase
	Indent:          "  ",  // pretty-print
}
pretty, _ := m.Marshal(user)
```

---

## 4. Toolchain Setup — protoc, Plugins & buf

You need: (1) a protobuf compiler, (2) the Go plugins, and (3) a way to invoke them. There are two mainstream paths — classic `protoc`, and the modern `buf`. Use **buf** for new projects; understand `protoc` because it's everywhere.

### Installing the Go plugins

These are Go programs that `protoc`/`buf` invoke as plugins. Install with `go install`:

```bash
# protoc-gen-go: generates message types (*.pb.go) from the protobuf APIv2 runtime.
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# protoc-gen-go-grpc: generates the gRPC service stubs (*_grpc.pb.go).
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Make sure $(go env GOPATH)/bin is on your PATH so protoc/buf can find them.
export PATH="$PATH:$(go env GOPATH)/bin"
```

⚡ **Version note:** `protoc-gen-go` (from `google.golang.org/protobuf`) and `protoc-gen-go-grpc` (from `google.golang.org/grpc`) are **separate** binaries from **separate** modules. They're versioned independently. Pin them in a `tools.go` (or use `buf`'s remote plugins) so every developer and CI generates identical code.

### Installing `protoc` (the C++ compiler)

`protoc` is a native binary. Download a release from the protobuf GitHub releases (`protoc-<version>-<os>.zip`), unzip, and put `bin/protoc` on your PATH. It bundles the well-known type `.proto` files under `include/`.

Verify:

```bash
protoc --version          # e.g. libprotoc 28.x
which protoc-gen-go        # should resolve to your GOPATH/bin
```

### The `protoc` command line

```bash
# Run from your proto root. -I (a.k.a. --proto_path) sets the import search root.
protoc \
  -I proto \
  --go_out=. --go_opt=module=github.com/example/app \
  --go-grpc_out=. --go-grpc_opt=module=github.com/example/app \
  proto/user/v1/user.proto
```

Key flags:
- `-I proto` — search `proto/` for imports (repeatable).
- `--go_out` / `--go-grpc_out` — output directories for each plugin.
- `--go_opt=module=...` — strip this module prefix from `go_package` so files land at the right relative path. (Alternative: `--go_opt=paths=source_relative` to mirror the source tree.)

This is fiddly: paths, `-I` roots, plugin versions, and `go_package` all have to line up. That pain is precisely why **buf** exists.

### The modern `buf` workflow

[`buf`](https://buf.build) wraps `protoc` with sane defaults, dependency management, linting, and breaking-change detection. Install the `buf` binary (single static binary) and configure two files.

**`buf.yaml`** — defines a module (where your protos live, lint rules, breaking rules):

```yaml
# buf.yaml  (place at the root of your proto module, often proto/)
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD          # the recommended rule set
breaking:
  use:
    - FILE              # detect wire-incompatible changes at file granularity
deps:
  # Pull dependencies (e.g. googleapis) from the Buf Schema Registry.
  - buf.build/googleapis/googleapis
```

**`buf.gen.yaml`** — defines what to generate and how:

```yaml
# buf.gen.yaml
version: v2
managed:
  enabled: true                 # auto-manage go_package etc. so you don't hand-edit
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
buf format -w     # auto-format .proto files in place
buf breaking --against '.git#branch=main'   # fail CI on incompatible schema change
buf generate      # generate code per buf.gen.yaml
buf build         # compile protos to a binary image (useful for tooling/registries)
```

⚡ **Version note:** `buf` configuration moved to **v2** (`version: v2` with a `modules:` list) — older guides show v1 with `buf.work.yaml` for multi-module workspaces. v2 unifies workspaces into a single `buf.yaml`. Confirm the schema for your installed `buf` version.

### Module layout for generated code

A clean, conventional Go layout:

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
    └── service/user.go       # your business logic implementing the interface
```

**Check generated code in or not?** Both are valid. Checking it in makes `go build`/`go install` work without the toolchain (good for `go install` consumers); generating in CI keeps the repo clean but requires every contributor to have `buf`. A common compromise: check it in, and have CI verify it's up to date (`buf generate && git diff --exit-code`).

---

## 5. Building a gRPC Server in Go

Let's define a complete service exercising all four RPC types, then implement it.

### The service `.proto`

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

After `buf generate`, you get `userv1.UserServiceServer` (the interface to implement) and `userv1.RegisterUserServiceServer` (the registration helper).

### The generated server interface (what you implement)

The generated interface looks like this (abridged — don't write it; it's generated):

```go
type UserServiceServer interface {
	GetUser(context.Context, *GetUserRequest) (*GetUserResponse, error)
	ListUpdates(*ListUpdatesRequest, grpc.ServerStreamingServer[Update]) error
	UploadChunks(grpc.ClientStreamingServer[UploadChunk, UploadSummary]) error
	Chat(grpc.BidiStreamingServer[ChatMessage, ChatMessage]) error
	mustEmbedUnimplementedUserServiceServer() // forces forward compatibility
}
```

⚡ **Version note:** Modern `protoc-gen-go-grpc` uses **generics** for stream types (`grpc.ServerStreamingServer[Update]`, etc.). Older generated code used per-method named interfaces (`UserService_ListUpdatesServer`). Both behave the same; the generic form is current as of 2024+.

### Implementing the server — all four methods

```go
// internal/service/user.go
package service

import (
	"context"
	"io"
	"log/slog"
	"time"

	userv1 "github.com/example/app/gen/user/v1"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/timestamppb"
)

// UserServer implements userv1.UserServiceServer.
type UserServer struct {
	// Embedding the generated "Unimplemented" struct is REQUIRED for forward
	// compatibility: if a new method is added to the .proto, your server still
	// compiles (the embedded stub returns codes.Unimplemented for it).
	userv1.UnimplementedUserServiceServer

	log   *slog.Logger
	users map[string]*userv1.User // toy in-memory store
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
	// Always honor cancellation/deadlines from ctx in real handlers.
	if req.GetId() == "" {
		// Return a gRPC status error (Section 9), not a bare error.
		return nil, status.Error(codes.InvalidArgument, "id is required")
	}
	u, ok := s.users[req.GetId()]
	if !ok {
		return nil, status.Errorf(codes.NotFound, "user %q not found", req.GetId())
	}
	return &userv1.GetUserResponse{User: u}, nil
}

// --- 2. Server streaming ---
// The response stream is the second parameter; you Send() multiple messages
// and return nil to close the stream cleanly.
func (s *UserServer) ListUpdates(req *userv1.ListUpdatesRequest, stream grpc.ServerStreamingServer[userv1.Update]) error {
	ticker := time.NewTicker(500 * time.Millisecond)
	defer ticker.Stop()

	for i := 0; i < 5; i++ {
		select {
		case <-stream.Context().Done():
			// Client cancelled or deadline exceeded — stop streaming.
			return stream.Context().Err()
		case <-ticker.C:
			up := &userv1.Update{
				Message: "update for " + req.GetUserId(),
				At:      timestamppb.Now(),
			}
			if err := stream.Send(up); err != nil {
				return err // peer went away
			}
		}
	}
	return nil // closing the function ends the stream with OK
}

// --- 3. Client streaming ---
// You Recv() in a loop until io.EOF, then SendAndClose() the single response.
func (s *UserServer) UploadChunks(stream grpc.ClientStreamingServer[userv1.UploadChunk, userv1.UploadSummary]) error {
	var total, count int64
	for {
		chunk, err := stream.Recv()
		if err == io.EOF {
			// Client finished sending; send the single summary response.
			return stream.SendAndClose(&userv1.UploadSummary{
				TotalBytes: total,
				ChunkCount: count,
			})
		}
		if err != nil {
			return err
		}
		total += int64(len(chunk.GetData()))
		count++
	}
}

// --- 4. Bidirectional streaming ---
// Recv and Send are independent and can interleave freely. Here we echo.
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

> The `grpc` package must be imported in this file for the stream types; add `"google.golang.org/grpc"` to the import block.

### Wiring up `main` — listen, register, serve

```go
// cmd/server/main.go
package main

import (
	"context"
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

	// 1) Open a TCP listener.
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Error("listen failed", "err", err)
		os.Exit(1)
	}

	// 2) Construct the gRPC server with options (interceptors, keepalive, limits).
	srv := grpc.NewServer(
		grpc.MaxRecvMsgSize(16*1024*1024), // raise from the 4 MiB default if needed
		grpc.KeepaliveParams(keepalive.ServerParameters{
			MaxConnectionIdle: 5 * time.Minute,
			Time:              2 * time.Hour, // ping idle conns to detect dead peers
			Timeout:           20 * time.Second,
		}),
		// chain interceptors here (Section 10):
		// grpc.ChainUnaryInterceptor(loggingUnary(log), recoveryUnary(log)),
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

	// 5) Graceful shutdown on SIGINT/SIGTERM (Section 18).
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
		srv.Stop() // hard stop if graceful takes too long
		log.Warn("forced shutdown")
	}
	_ = context.Background()
}
```

---

## 6. Building a gRPC Client in Go

### Dialing — `grpc.NewClient` (the modern API)

⚡ **Version note:** `grpc.Dial` / `grpc.DialContext` are **deprecated** in favor of **`grpc.NewClient`** (stable since grpc-go v1.63, 2024). `NewClient` does **not** connect eagerly and does **not** block; the connection is established lazily on the first RPC (or when you call `conn.Connect()`). It also defaults the name resolver to `dns:///`, so plain `host:port` targets work. Drop the old `grpc.WithBlock()` / `context.WithTimeout` around dialing — connection readiness is now governed by per-RPC deadlines and the retry/wait-for-ready policy.

```go
// cmd/client/main.go
package main

import (
	"context"
	"io"
	"log"
	"time"

	userv1 "github.com/example/app/gen/user/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	// grpc.NewClient: returns immediately; connection is lazy. "insecure" is for
	// local/dev only — use TLS credentials in production (Section 11).
	conn, err := grpc.NewClient(
		"localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		// Optional: a default service config enabling retries, etc. (Section 12).
	)
	if err != nil {
		log.Fatalf("NewClient: %v", err) // only fails on bad args/target syntax
	}
	defer conn.Close()

	// One ClientConn can back many stubs and many concurrent RPCs (HTTP/2 mux).
	client := userv1.NewUserServiceClient(conn)

	unaryDemo(client)
	serverStreamDemo(client)
	clientStreamDemo(client)
	bidiDemo(client)
}
```

### Calling a unary method (with a deadline)

```go
func unaryDemo(client userv1.UserServiceClient) {
	// ALWAYS set a deadline on RPCs. Without one, a hung server hangs you forever.
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: "u1"})
	if err != nil {
		log.Printf("GetUser error: %v", err) // inspect with status.FromError (Section 9)
		return
	}
	log.Printf("user: %s <%s>", resp.GetUser().GetDisplayName(), resp.GetUser().GetEmail())
}
```

### Calling a server-streaming method

```go
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

### Calling a client-streaming method

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

### Calling a bidirectional-streaming method

```go
func bidiDemo(client userv1.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.Chat(ctx)
	if err != nil {
		log.Printf("Chat: %v", err)
		return
	}

	// Receive in a separate goroutine because Send and Recv run concurrently.
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

### Connection management reality

- A `*grpc.ClientConn` is **safe for concurrent use** and is meant to be **long-lived and shared**. Create one per backend and reuse it — do **not** dial per request.
- Because HTTP/2 multiplexes, one `ClientConn` already gives you concurrency. You generally do **not** need a "connection pool" of `ClientConn`s. (Section 17 covers the rare exceptions.)
- `conn.Connect()` forces an eager connection attempt; `conn.WaitForStateChange(ctx, state)` and `conn.GetState()` let you observe connectivity if you really need to.

---

## 7. The Four Streaming Patterns End-to-End

Sections 5 and 6 already show all four in production form. This section adds the **conceptual map**, the **lifecycle**, and the **gotchas** for each so the patterns stick.

### Lifecycle summary

| Mode | Client API | Server API | Ends when |
|---|---|---|---|
| Unary | `resp, err := c.M(ctx, req)` | `return resp, err` | Server returns |
| Server stream | `s,_ := c.M(ctx,req)`; loop `s.Recv()` until `io.EOF` | loop `stream.Send()`; `return nil` | Server returns |
| Client stream | loop `s.Send()`; `s.CloseAndRecv()` | loop `stream.Recv()` until `io.EOF`; `stream.SendAndClose()` | Client closes send + server responds |
| Bidi stream | `s.Send()` / `s.Recv()` concurrently; `s.CloseSend()` | `stream.Recv()` / `stream.Send()` concurrently; `return` | Both sides close |

### Unary — `GetUser` (request/response)

The simplest and most common. Use it for everything that isn't naturally a stream. Implementation: Section 5 (`GetUser`). Call: Section 6 (`unaryDemo`).

### Server streaming — `ListUpdates` (feed / pagination / progress)

Great for: live feeds, large result sets you don't want to buffer, progress updates, server-sent change notifications. The server `Send()`s as many messages as it likes, then returns. The client loops `Recv()` until `io.EOF`.

**Gotchas:**
- Honor `stream.Context().Done()` on the server so a client cancellation/deadline actually stops the work.
- The client *must* drain the stream (loop to `io.EOF`) or cancel its context — otherwise resources leak.

### Client streaming — `UploadChunks` (upload / aggregation)

Great for: file/blob uploads, bulk inserts, metrics aggregation. The client `Send()`s many messages then `CloseAndRecv()`; the server loops `Recv()` to `io.EOF` then `SendAndClose()` a single response.

**Gotchas:**
- The single response only comes after the client closes its send side.
- Don't forget `CloseAndRecv()` (client) and the `io.EOF`-then-`SendAndClose()` shape (server).

### Bidirectional streaming — `Chat` (interactive / multiplexed)

Great for: chat, real-time collaboration, request/response multiplexing over one stream, long-lived control channels. Both sides read and write independently. On the **client**, run `Recv()` in its own goroutine because it blocks; `Send()` from the main flow; finish with `CloseSend()`.

**Gotchas:**
- **Never** call `Send()` from two goroutines on the same stream concurrently, and likewise for `Recv()`. The stream's `Send` and `Recv` are each *not* safe for concurrent use by multiple goroutines — but one goroutine sending while a *different* one receives is the correct, supported pattern.
- Decide and document who closes first. Deadlocks here are usually "both sides waiting to receive."

---

## 8. Metadata — Headers & Trailers

**Metadata** is gRPC's equivalent of HTTP headers: key/value pairs attached to a call, separate from the message body. Keys are case-insensitive ASCII; values are strings (or, for `-bin` suffixed keys, raw bytes). Use it for auth tokens, request IDs, tracing context, etc. The package is `google.golang.org/grpc/metadata`.

There are two phases: **headers** (sent before the response body) and **trailers** (sent after, even on streams — useful for end-of-call status info like row counts).

### Client → server: sending metadata

```go
import "google.golang.org/grpc/metadata"

// Attach outgoing metadata to the context.
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
_ = resp; _ = err
fmt.Println("server header x-served-by:", header.Get("x-served-by"))
fmt.Println("server trailer x-row-count:", trailer.Get("x-row-count"))
```

### Server: reading request metadata, sending headers/trailers

```go
func (s *UserServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	// Read incoming metadata from the inbound context.
	if md, ok := metadata.FromIncomingContext(ctx); ok {
		if vals := md.Get("authorization"); len(vals) > 0 {
			// inspect vals[0] for a token...
		}
		if rid := md.Get("x-request-id"); len(rid) > 0 {
			s.log.Info("handling", "request_id", rid[0])
		}
	}

	// Send response headers (must happen before the first message / return value).
	_ = grpc.SetHeader(ctx, metadata.Pairs("x-served-by", "user-svc-1"))

	// Set trailers (delivered at the end of the call).
	_ = grpc.SetTrailer(ctx, metadata.Pairs("x-row-count", "1"))

	u := s.users[req.GetId()]
	return &userv1.GetUserResponse{User: u}, nil
}
```

For **streaming** handlers, use `stream.SetHeader`/`stream.SendHeader`/`stream.SetTrailer` and read with `metadata.FromIncomingContext(stream.Context())`.

### Context propagation

A common need is forwarding select metadata (trace IDs, auth) from an inbound call to a downstream outbound call:

```go
// In a handler/interceptor: copy chosen inbound keys to the outgoing context.
inMD, _ := metadata.FromIncomingContext(ctx)
outCtx := metadata.NewOutgoingContext(ctx, metadata.Join(metadata.New(nil),
	metadata.Pairs("x-request-id", first(inMD.Get("x-request-id"))),
))
_, _ = downstream.SomeRPC(outCtx, &pb.Req{})
```

⚡ **Version note:** Don't blindly forward *all* inbound metadata downstream — strip hop-by-hop and auth headers you don't intend to propagate. Tracing libraries (OpenTelemetry) provide propagators that handle this correctly; prefer them over hand-rolled copying.

---

## 9. Error Handling — status, codes & rich errors

gRPC has a **structured error model**: every RPC ends with a status **code** (an integer enum), a human-readable **message**, and optional **details** (arbitrary protobuf messages). Use `google.golang.org/grpc/status` and `google.golang.org/grpc/codes`. **Never** return a bare `errors.New` from a handler if you can help it — it becomes `codes.Unknown`.

### The standard codes

| Code | When to use | Roughly like HTTP |
|---|---|---|
| `OK` | Success | 200 |
| `InvalidArgument` | Caller sent bad input (independent of state) | 400 |
| `FailedPrecondition` | System not in required state | 400/409 |
| `OutOfRange` | Past valid range (e.g. pagination) | 400 |
| `Unauthenticated` | No/invalid credentials | 401 |
| `PermissionDenied` | Authenticated but not allowed | 403 |
| `NotFound` | Entity doesn't exist | 404 |
| `AlreadyExists` | Entity already exists | 409 |
| `Aborted` | Concurrency conflict (retry the txn) | 409 |
| `ResourceExhausted` | Quota/rate limit | 429 |
| `Cancelled` | Caller cancelled | 499 |
| `DeadlineExceeded` | Deadline passed | 504 |
| `Unimplemented` | Method not implemented | 501 |
| `Unavailable` | Transient; safe to retry | 503 |
| `Internal` | Server bug/invariant broken | 500 |
| `DataLoss` | Unrecoverable data loss | 500 |
| `Unknown` | Uncategorized (avoid; usually a leaked plain error) | 500 |

### Returning errors from a handler

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

### Rich errors with `errdetails`

You can attach structured detail messages so clients can react programmatically (e.g. which field failed validation, retry timing, quota info). The standard detail types live in `google.golang.org/genproto/googleapis/rpc/errdetails`.

```go
import (
	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

func invalidEmail() error {
	st := status.New(codes.InvalidArgument, "validation failed")

	// BadRequest details: per-field violations.
	br := &errdetails.BadRequest{
		FieldViolations: []*errdetails.BadRequest_FieldViolation{
			{Field: "email", Description: "must be a valid email address"},
		},
	}
	// RetryInfo, QuotaFailure, ErrorInfo, ResourceInfo, etc. are also available.
	st, err := st.WithDetails(br)
	if err != nil {
		// WithDetails can fail if a detail can't be marshaled — fall back.
		return status.Error(codes.InvalidArgument, "validation failed")
	}
	return st.Err()
}
```

### Inspecting errors on the client

```go
resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: ""})
if err != nil {
	st, ok := status.FromError(err) // ok=false means it wasn't a gRPC status
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
			switch info := d.(type) {
			case *errdetails.BadRequest:
				for _, v := range info.GetFieldViolations() {
					log.Printf("  field %s: %s", v.GetField(), v.GetDescription())
				}
			}
		}
	default:
		log.Printf("rpc failed [%s]: %s", st.Code(), st.Message())
	}
}
```

### Mapping app errors ↔ gRPC status

Keep your domain layer free of gRPC types; translate at the edge (often in an interceptor):

```go
// Domain sentinel errors.
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
		// Don't leak internal details to clients.
		return status.Error(codes.Internal, "internal error")
	}
}
```

---

## 10. Interceptors — gRPC's Middleware

**Interceptors** are gRPC's middleware: hooks that wrap every RPC. There are four kinds — `{Unary, Stream} × {Server, Client}`. Chain them with `grpc.ChainUnaryInterceptor` / `grpc.ChainStreamInterceptor` (server) and `grpc.WithChainUnaryInterceptor` / `grpc.WithChainStreamInterceptor` (client). Order matters: the first listed runs outermost.

### Server-side unary interceptor — logging + timing

```go
import (
	"context"
	"time"
	"log/slog"
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
		code := status.Code(err)
		log.Info("rpc",
			"method", info.FullMethod,
			"code", code.String(),
			"dur_ms", time.Since(start).Milliseconds(),
		)
		return resp, err
	}
}
```

### Server-side recovery (panic) interceptor

A panic in a handler would otherwise crash the whole server. Recover and convert to `codes.Internal`.

```go
import "runtime/debug"

func recoveryUnary(log *slog.Logger) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
		defer func() {
			if r := recover(); r != nil {
				log.Error("panic recovered", "method", info.FullMethod, "panic", r, "stack", string(debug.Stack()))
				err = status.Error(codes.Internal, "internal error")
			}
		}()
		return handler(ctx, req)
	}
}
```

### Server-side auth interceptor (JWT in metadata)

```go
import (
	"strings"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

// authUnary validates a Bearer token and stuffs the user into the context.
func authUnary(verify func(token string) (userID string, err error)) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		// Optionally skip auth for public methods:
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
			return nil, status.Error(codes.Unauthenticated, "invalid token")
		}
		// Pass identity down via context (use an unexported key type in real code).
		ctx = context.WithValue(ctx, userCtxKey{}, userID)
		return handler(ctx, req)
	}
}

type userCtxKey struct{}
```

### Stream interceptors

Streaming RPCs need their own interceptor type. To intercept messages you wrap the `grpc.ServerStream`:

```go
// wrappedStream lets us override Context()/RecvMsg()/SendMsg().
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

### Registering interceptors

```go
srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(
		recoveryUnary(log),  // outermost: catches panics from everything inside
		loggingUnary(log),
		authUnary(verifyJWT),
	),
	grpc.ChainStreamInterceptor(
		authStream(verifyJWT),
	),
)
```

### Client-side interceptors — inject auth + retry

```go
// Client unary interceptor that attaches a token to every outgoing call.
func authClientUnary(token string) grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply any,
		cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
		return invoker(ctx, method, req, reply, cc, opts...) // proceed
	}
}

// A tiny manual retry interceptor (prefer the built-in service-config retry, Section 12).
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

### `go-grpc-middleware`

The community package `github.com/grpc-ecosystem/go-grpc-middleware/v2` provides battle-tested interceptors so you don't hand-roll them: logging (with adapters for slog/zap/zerolog), recovery, auth, retry, rate-limiting, validation, and selective application (`selector.UnaryServerInterceptor`). For production, prefer these over the toy versions above.

```go
import (
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/recovery"
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/logging"
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

gRPC's `credentials` system has two layers: **transport credentials** (channel security — TLS/mTLS) and **per-RPC credentials** (call-level — tokens). Use both together: TLS protects the pipe; tokens identify the caller.

### Server-side TLS

```go
import "google.golang.org/grpc/credentials"

// Load the server cert + key (PEM).
creds, err := credentials.NewServerTLSFromFile("certs/server.crt", "certs/server.key")
if err != nil {
	log.Fatal(err)
}
srv := grpc.NewServer(grpc.Creds(creds))
```

### Client-side TLS

```go
// Trust the server's CA. For public CAs use credentials.NewTLS with the system pool.
creds, err := credentials.NewClientTLSFromFile("certs/ca.crt", "" /* serverNameOverride */)
if err != nil {
	log.Fatal(err)
}
conn, err := grpc.NewClient("example.com:443", grpc.WithTransportCredentials(creds))
```

### Mutual TLS (mTLS) — both sides present certs

mTLS authenticates the *client* to the server (and vice versa) using certificates. This is the gold standard for service-to-service auth inside a mesh.

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
		ClientAuth:   tls.RequireAndVerifyClientCert, // <-- demand a valid client cert
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
		ServerName:   "user-svc", // must match the server cert's SAN
	})
}
```

### Per-RPC credentials (token-based)

Implement `credentials.PerRPCCredentials` to inject a token on every call automatically. gRPC requires transport security for per-RPC creds unless you opt out.

```go
type tokenCreds struct {
	token            string
	requireTransport bool
}

func (t tokenCreds) GetRequestMetadata(_ context.Context, _ ...string) (map[string]string, error) {
	return map[string]string{"authorization": "Bearer " + t.token}, nil
}
func (t tokenCreds) RequireTransportSecurity() bool { return t.requireTransport }

// Attach to all calls on the connection:
conn, _ := grpc.NewClient("example.com:443",
	grpc.WithTransportCredentials(clientMTLS()),
	grpc.WithPerRPCCredentials(tokenCreds{token: jwt, requireTransport: true}),
)
```

The server validates the token in an **auth interceptor** (Section 10). This "TLS for the channel + JWT/OIDC token in metadata" pattern is the most common real-world setup.

### ALTS (Google-specific)

**ALTS** (Application Layer Transport Security) is a mutual-auth scheme used inside Google Cloud (GKE etc.) where identities are managed by the platform. `google.golang.org/grpc/credentials/alts` provides `alts.NewServerCreds()` / `alts.NewClientCreds()`. Outside GCP you'll use TLS/mTLS; mention it so you recognize it.

### Security checklist

- Never run production gRPC with `insecure.NewCredentials()`.
- Prefer mTLS for east-west (service-to-service); TLS + token for north-south (clients).
- Validate tokens in an interceptor; map failures to `codes.Unauthenticated` / `codes.PermissionDenied`.
- Set `tls.Config.MinVersion = tls.VersionTLS13` where possible.
- Rotate certs/keys; don't bake long-lived secrets into images.

---

## 12. Deadlines, Retries, Keepalive & Load Balancing

### Deadlines & timeouts (do this on every call)

A **deadline** is an absolute time by which the RPC must complete; gRPC propagates it across hops via metadata, so a downstream service knows how much time is left. Set it with `context.WithTimeout`/`WithDeadline`:

```go
ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
defer cancel()
_, err := client.GetUser(ctx, req)
if status.Code(err) == codes.DeadlineExceeded { /* handle */ }
```

On the server, always select on `ctx.Done()` in loops and long operations so a blown deadline actually frees resources.

### Cancellation

Cancelling the client context (or the deadline firing) signals the server: the inbound `ctx.Done()` closes and `ctx.Err()` returns `context.Canceled`/`DeadlineExceeded`. Propagate the **same** context into downstream calls so cancellation cascades.

### Retries via service config (built-in, no interceptor needed)

grpc-go has a **declarative retry policy** configured by a JSON **service config**. This is the recommended way to retry — it understands gRPC semantics (retryable codes, exponential backoff with jitter, attempt caps) and integrates with deadlines.

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

⚡ **Version note:** Retries are enabled by default in current grpc-go (the old `grpc.WithDefaultCallOptions(grpc.WaitForReady(...))` and experimental retry flags are gone). Only retry **idempotent** methods, or methods you've made safe — retrying a non-idempotent `CreateOrder` on `Unavailable` can double-charge a customer. Mark intent clearly and choose `retryableStatusCodes` carefully.

### `waitForReady`

By default, an RPC made while the channel is in `TRANSIENT_FAILURE` fails fast with `Unavailable`. Setting `waitForReady: true` (or `grpc.WaitForReady(true)` per call) makes the RPC queue until the channel is ready or the deadline fires — useful at startup, dangerous without a deadline.

### Keepalive

Keepalive pings detect dead connections and keep idle ones alive through NAT/load balancers.

```go
import "google.golang.org/grpc/keepalive"

// Client side:
conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithKeepaliveParams(keepalive.ClientParameters{
		Time:                30 * time.Second, // ping if no activity for 30s
		Timeout:             10 * time.Second, // wait 10s for ping ack, else close
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

⚡ **Gotcha:** If the client's keepalive `Time` is smaller than the server's `MinTime`, the server will close connections with "too_many_pings". Keep them consistent.

### Load balancing basics

gRPC does **client-side load balancing**: a single `ClientConn` resolves a target to a list of backends and balances across them. Two pieces:

1. **Name resolver** — turns a target URI into addresses. Built-ins: `dns:///user-svc:50051` (re-resolves DNS), `passthrough:///` (use the literal address). Schemes like `xds:///` integrate with a control plane (Envoy/Istio).
2. **Load balancing policy** — `pick_first` (default; one backend at a time) or `round_robin` (spread across all resolved backends).

```go
conn, _ := grpc.NewClient(
	"dns:///user-svc.default.svc.cluster.local:50051",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
)
```

**Why this matters:** because HTTP/2 multiplexes over one connection, a naive L4 (TCP) load balancer will pin all of a client's RPCs to one backend. Client-side LB (or an L7 proxy like Envoy) is what actually spreads load. See Section 18.

### "Connection pooling" reality

You usually do **not** pool `ClientConn`s — one multiplexed connection handles many concurrent RPCs. The rare exception: extremely high throughput can saturate a single HTTP/2 connection's `MAX_CONCURRENT_STREAMS` or hit single-connection CPU limits; then you might run a small number of connections (e.g. via multiple `ClientConn`s, or the gRPC `round_robin` policy over multiple resolved addresses). Measure before pooling.

---

## 13. Interoperability — gRPC-Gateway, gRPC-Web, ConnectRPC

gRPC is great service-to-service, but browsers can't speak raw gRPC (no access to HTTP/2 trailers/framing from `fetch`). Three bridges solve this.

### gRPC-Gateway — REST/JSON from your gRPC service

[`grpc-gateway`](https://github.com/grpc-ecosystem/grpc-gateway) generates a **reverse-proxy** that exposes a RESTful JSON API and translates to gRPC. You annotate methods with HTTP mappings via `google.api.http` options.

```proto
// proto/user/v1/user.proto (add the annotation import + options)
import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"   // maps the URL path var {id} to the request field
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

Generate the gateway stub with the `protoc-gen-grpc-gateway` plugin, then run it alongside (or in front of) your gRPC server:

```go
import (
	"context"
	"net/http"
	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	userv1 "github.com/example/app/gen/user/v1"
)

func runGateway(ctx context.Context, grpcAddr string) error {
	mux := runtime.NewServeMux() // translates HTTP/JSON ⇄ gRPC
	// Register the generated handler, dialing the gRPC backend.
	err := userv1.RegisterUserServiceHandlerFromEndpoint(
		ctx, mux, grpcAddr,
		[]grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())},
	)
	if err != nil {
		return err
	}
	// Now `curl http://localhost:8080/v1/users/u1` proxies to GetUser.
	return http.ListenAndServe(":8080", mux)
}
```

Now `curl http://localhost:8080/v1/users/u1` returns JSON, while internal callers still use gRPC. You get one contract, two surfaces.

### gRPC-Web — gRPC for browsers via a proxy

**gRPC-Web** is a wire protocol the browser *can* speak (it doesn't require HTTP/2 trailers in the same way). It needs a translating proxy — historically Envoy's gRPC-Web filter, or the standalone `grpcweb` wrapper. The browser uses generated TypeScript clients (`@grpc/grpc-web` or, increasingly, Connect's client). gRPC-Web supports unary and **server-streaming** but **not** client/bidi streaming (a browser limitation).

### ConnectRPC — the modern, browser-friendly choice

[**ConnectRPC**](https://connectrpc.com) (`connectrpc.com/connect` for Go, "connect-go") is a from-scratch implementation that speaks **three** protocols over plain HTTP/1.1 *or* HTTP/2 on a single port:

1. The **Connect protocol** — simple, debuggable, `curl`-able JSON or binary.
2. **gRPC** — fully interoperable with grpc-go servers/clients.
3. **gRPC-Web** — directly, **no proxy needed**.

That means a browser can call your service with no Envoy, and you can `curl` a Connect endpoint like REST. Handlers are plain `http.Handler`s, so they slot into any Go HTTP stack (mux, middleware, TLS) you already use.

```go
// Connect server: generated with protoc-gen-connect-go (often via buf remote plugins).
package main

import (
	"context"
	"net/http"

	"connectrpc.com/connect"
	userv1 "github.com/example/app/gen/user/v1"
	"github.com/example/app/gen/user/v1/userv1connect" // connect-generated
)

type userServer struct{}

// Note the Connect signature: requests/responses are wrapped in connect.Request/Response.
func (s *userServer) GetUser(
	ctx context.Context,
	req *connect.Request[userv1.GetUserRequest],
) (*connect.Response[userv1.GetUserResponse], error) {
	id := req.Msg.GetId()
	if id == "" {
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
	// The generated handler returns a path + http.Handler.
	path, handler := userv1connect.NewUserServiceHandler(&userServer{})
	mux.Handle(path, handler)
	// h2c lets HTTP/2 work without TLS for local dev (use real TLS in prod).
	// import "golang.org/x/net/http2/h2c" and "golang.org/x/net/http2"
	http.ListenAndServe(":8080", mux)
}
```

A Connect **client** is equally simple and works from Go, TypeScript (browser), etc.:

```go
client := userv1connect.NewUserServiceClient(http.DefaultClient, "http://localhost:8080")
resp, err := client.GetUser(ctx, connect.NewRequest(&userv1.GetUserRequest{Id: "u1"}))
```

**When to pick what:**
- Pure internal Go/polyglot microservices, mesh, max ecosystem → **grpc-go**.
- Need a REST/JSON edge for the same protos → add **grpc-gateway**.
- Browser clients and/or want `curl`-friendly endpoints with no proxy → **ConnectRPC** (it still speaks gRPC to your other services).

---

## 14. Health Checking & Reflection

### The standard health checking protocol

gRPC defines a standard health service (`grpc.health.v1.Health`) so load balancers and orchestrators (Kubernetes, Envoy) can probe liveness/readiness uniformly.

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
hs.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)                 // overall
hs.SetServingStatus("user.v1.UserService", healthpb.HealthCheckResponse_SERVING)

// During shutdown, flip to NOT_SERVING so the LB drains you before you stop.
// hs.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
```

Kubernetes can probe it directly via the built-in `grpc` probe type (no exec needed):

```yaml
readinessProbe:
  grpc:
    port: 50051
    service: "user.v1.UserService"
```

### Server reflection

**Reflection** lets tools discover your services and message schemas at runtime — no `.proto` file needed by the caller. Essential for debugging with `grpcurl` and graphical clients.

```go
import "google.golang.org/grpc/reflection"

srv := grpc.NewServer()
userv1.RegisterUserServiceServer(srv, impl)
reflection.Register(srv) // enable server reflection
```

⚡ **Security note:** reflection exposes your full API surface. Enable it in dev/staging freely; in production, gate it behind auth or disable it on public endpoints.

### Debugging with `grpcurl`

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

### In-memory testing with `bufconn`

`google.golang.org/grpc/test/bufconn` gives you a real gRPC server and client talking over an **in-memory** pipe — no TCP, no ports, fast and deterministic. This is the recommended way to test handlers end-to-end.

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
		"passthrough:///bufnet", // passthrough so the dialer is used verbatim
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

### Testing a streaming method

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

### Mocking dependencies vs mocking the client

- **Prefer bufconn** to test the *real* server with *real* generated stubs — it exercises serialization, interceptors, and status mapping.
- To unit-test code that *calls* a gRPC service, mock the **generated client interface** (`userv1.UserServiceClient`). Tools: `go.uber.org/mock` (gomock) generating from the interface, or a hand-written fake. Don't mock the concrete stub; mock the interface.
- Stream interfaces (e.g. `grpc.ServerStreamingClient[Update]`) are also interfaces you can fake for table tests.

---

## 16. Observability — OpenTelemetry, Prometheus, Logging

### OpenTelemetry (the modern default for traces + metrics)

⚡ **Version note:** The current way to instrument grpc-go is the official **`google.golang.org/grpc/stats/opentelemetry`** module (stats-handler based), which superseded the older `otelgrpc` interceptors for many use cases. It produces standard RPC metrics and propagates trace context.

```go
import (
	"google.golang.org/grpc"
	"google.golang.org/grpc/stats/opentelemetry"
	"go.opentelemetry.io/otel/sdk/metric"
)

mp := metric.NewMeterProvider(/* readers/exporters */)

// Server: attach the OpenTelemetry stats handler.
srv := grpc.NewServer(
	opentelemetry.ServerOption(opentelemetry.Options{
		MetricsOptions: opentelemetry.MetricsOptions{MeterProvider: mp},
	}),
)

// Client: dial option counterpart.
conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	opentelemetry.DialOption(opentelemetry.Options{
		MetricsOptions: opentelemetry.MetricsOptions{MeterProvider: mp},
	}),
)
```

For distributed tracing across services, also configure an OTel `TracerProvider` and a propagator (e.g. W3C TraceContext); the stats handler / `otelgrpc` carries the trace context in metadata so spans link up across hops.

### Prometheus metrics

`github.com/grpc-ecosystem/go-grpc-middleware/providers/prometheus` (the successor to `go-grpc-prometheus`) provides interceptors that emit RED metrics (Rate, Errors, Duration):

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
srvMetrics.InitializeMetrics(srv) // pre-create series for all methods

// Expose /metrics via a normal HTTP server with promhttp.Handler().
```

### Logging

Use the `go-grpc-middleware/v2` logging interceptor with a `slog` adapter (Section 10). Log: method, status code, duration, peer, request ID (from metadata). **Do not** log full request/response bodies by default — they may contain PII and are large.

### Debugging in practice

- `grpcurl` (Section 14) to poke endpoints by hand.
- Set `GRPC_GO_LOG_VERBOSITY_LEVEL=99` and `GRPC_GO_LOG_SEVERITY_LEVEL=info` env vars for verbose grpc-go internal logs when chasing connection/resolver issues.
- `grpc.EnableTracing` and the `/debug/requests` endpoint (golang.org/x/net/trace) for in-process request tracing during development.

---

## 17. Performance — Message Size, Compression, Reuse

### Message size limits

grpc-go defaults: **4 MiB max receive** size, and (effectively) unlimited send. Hitting the cap returns `ResourceExhausted`. Tune deliberately — large messages hurt latency and memory; prefer streaming for big payloads.

```go
// Server:
grpc.NewServer(
	grpc.MaxRecvMsgSize(16*1024*1024),
	grpc.MaxSendMsgSize(16*1024*1024),
)
// Client (per-call or default):
conn, _ := grpc.NewClient(addr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(16*1024*1024)),
)
```

### Compression

gRPC supports per-message compression. `gzip` ships in `google.golang.org/grpc/encoding/gzip`; importing it for its side effect registers it.

```go
import _ "google.golang.org/grpc/encoding/gzip"
// Per call:
resp, err := client.GetUser(ctx, req, grpc.UseCompressor(gzip.Name))
```

Compression trades CPU for bandwidth. It pays off for large, compressible payloads over constrained links; it can *hurt* for small messages or already-compressed data (images, encrypted blobs). Measure.

### Streaming vs unary

- Use **streaming** to avoid buffering huge payloads in memory and to overlap producer/consumer work (back-pressure via HTTP/2 flow control comes free).
- Use **unary** for simple request/response — it's easier to reason about, cache, retry, and load-balance per call.
- Streaming RPCs pin to a single connection/backend for their lifetime, which can skew load balancing for very long-lived streams.

### Keepalive tuning

Covered in Section 12. The key point for performance: keepalive avoids reconnect storms (TCP + TLS handshakes are expensive) by keeping idle connections healthy through NATs/LBs.

### Connection reuse

The biggest practical win: **reuse one `ClientConn`** across all calls to a backend. Dialing per request throws away HTTP/2 multiplexing and pays handshake cost every time. Create the `ClientConn` once at startup, share it (it's concurrency-safe), close it at shutdown.

---

## 18. Deployment — Load Balancers, Docker, Graceful Shutdown

### Why HTTP/2 changes load balancing

gRPC runs on **long-lived, multiplexed HTTP/2 connections**. A **layer-4 (TCP) load balancer** balances *connections*, not *requests* — so once a client connects, all its RPCs stick to one backend, defeating balancing. Solutions:

1. **L7 (HTTP/2-aware) proxy** — Envoy, NGINX (with `grpc_pass`), HAProxy, Linkerd, Istio. These balance per-request.
2. **Client-side LB** — `round_robin` over a DNS/xDS resolver (Section 12); each client spreads its own RPCs.
3. **Headless service + client LB** in Kubernetes — a headless `Service` returns all pod IPs; grpc-go's `dns:///` resolver + `round_robin` balances across them.

### Dockerizing a gRPC server

```dockerfile
# ---- build stage ----
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Static binary so the runtime image can be tiny.
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/server ./cmd/server

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/server /server
EXPOSE 50051
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

### Health checks in orchestration

Wire the standard health service (Section 14) and use Kubernetes' native `grpc` probes. Flip the health status to `NOT_SERVING` *before* shutting down so the LB drains traffic.

### Graceful shutdown

```go
func shutdown(srv *grpc.Server, hs *health.Server) {
	// 1) Tell LBs to stop sending new traffic.
	hs.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
	// 2) (Optional) give the LB a moment to notice before we stop accepting.
	time.Sleep(2 * time.Second)
	// 3) GracefulStop: stop accepting new RPCs; wait for in-flight to finish.
	done := make(chan struct{})
	go func() { srv.GracefulStop(); close(done) }()
	select {
	case <-done: // clean
	case <-time.After(30 * time.Second):
		srv.Stop() // force-close if something is stuck
	}
}
```

`GracefulStop` blocks until all pending RPCs complete (including long streams!), so always bound it with a timeout and fall back to `Stop()`.

### Deployment checklist

- TLS terminated at the edge *or* end-to-end (prefer end-to-end / mTLS for zero-trust).
- L7 LB or client-side LB — never plain L4 alone.
- Resource limits + `MaxConcurrentStreams` tuned.
- Readiness gated on the health service; liveness separate.
- Graceful shutdown bounded by a timeout.
- Reflection disabled or auth-gated in prod.

---

## 19. Gotchas & Best Practices

### Protobuf / schema

- **Never** reuse or renumber an existing field. `reserved` removed numbers and names.
- Always add a `0` `*_UNSPECIFIED` enum member; prefix members with the enum name.
- Version your proto packages (`v1`, `v2`); don't make breaking changes in place.
- Run `buf lint` and `buf breaking` in CI — make incompatible changes impossible to merge.
- Use `optional` (or wrappers) when you must distinguish "unset" from "zero."
- Don't put huge blobs in a single message; stream them.

### Go server/client

- **Embed `UnimplementedXxxServer`** in every implementation — it's your forward-compatibility insurance.
- **Always set a deadline** on client calls. No deadline = potential permanent hang.
- **Reuse `ClientConn`** — don't dial per request.
- Use **`grpc.NewClient`**, not the deprecated `grpc.Dial`.
- **Never block indefinitely in a handler** without honoring `ctx.Done()` — it ties up a goroutine and ignores client cancellation.
- On streams, **don't call `Send` (or `Recv`) concurrently** from multiple goroutines; one-sending-one-receiving is fine.
- Drain server streams to `io.EOF` on the client (or cancel) to avoid leaks.

### Errors

- Return `status.Error(codes.X, ...)`, never bare errors (they become `Unknown`).
- Pick the **right code** — clients (and retry policies) act on it. `Unavailable` invites retries; `Internal` should not be retried automatically.
- Don't leak internal error text/stack traces to clients; translate at the boundary.
- Use `errdetails` for machine-actionable specifics (field violations, retry-after, quota).

### Reliability

- Only auto-retry **idempotent** methods; configure `retryableStatusCodes` precisely.
- Match client keepalive `Time` with server `EnforcementPolicy.MinTime` to avoid "too_many_pings" GOAWAYs.
- Propagate deadlines and cancellation into downstream calls.
- Use `waitForReady` only with a deadline.

### Security

- TLS/mTLS in production; `insecure` only for local dev/tests.
- Validate auth in interceptors; map to `Unauthenticated`/`PermissionDenied`.
- Gate or disable reflection in production.

---

## 20. Study Path & Build-to-Learn Projects

A six-stage path from "what is RPC" to "production microservices." Build the projects — reading alone won't make it stick.

### Stage 1 — RPC foundations (Week 1)

1. **`net/rpc` round-trip** — implement the `Arith` example from Section 1 (server + sync + async client). Then swap in the `jsonrpc` codec and inspect the JSON on the wire with `nc`/`socat`. Articulate why this approach doesn't scale across languages.
2. **First proto** — write a `user.v1` proto with a `User` message and a unary `GetUser`. Install `protoc`, `protoc-gen-go`, `protoc-gen-go-grpc`; generate code by hand-running `protoc`. Read the generated `.pb.go` and `_grpc.pb.go`.
3. **Switch to buf** — add `buf.yaml` + `buf.gen.yaml`; run `buf lint`, `buf format`, `buf generate`. Compare ergonomics to raw `protoc`.

### Stage 2 — All four RPC types (Week 2)

4. **Unary server + client** — implement `GetUser` end-to-end with `grpc.NewClient`, deadlines, and proper `status`/`codes` errors. Probe it with `grpcurl` (enable reflection).
5. **Server streaming** — `ListUpdates` that emits 5 ticks; client drains to `io.EOF`; honor `stream.Context().Done()`.
6. **Client streaming** — `UploadChunks` that sums bytes; client `Send`s then `CloseAndRecv`.
7. **Bidi streaming** — `Chat` echo server; client uses a goroutine for `Recv` while `Send`ing.

### Stage 3 — Cross-cutting concerns (Week 3)

8. **Interceptors** — add logging, recovery (panic→`Internal`), and a JWT auth interceptor (server) + token-injecting client interceptor. Then replace your hand-rolled ones with `go-grpc-middleware/v2`.
9. **Metadata** — propagate an `x-request-id` from client to server and into downstream calls; return a trailer.
10. **Rich errors** — return `InvalidArgument` with `errdetails.BadRequest` field violations; inspect them on the client.

### Stage 4 — Build Project 1: Chat Service (Week 4)

11. **Bidi chat** — a multi-room chat using bidirectional streaming. Server maintains rooms (`map[room]map[*client]chan msg`); each connected stream gets a goroutine. Broadcast join/leave events.
12. **Auth + metadata** — clients authenticate via a token in metadata; the room/user identity comes from an auth interceptor.
13. **Graceful shutdown** — on SIGTERM, flip health to `NOT_SERVING`, notify clients, `GracefulStop`.

### Stage 5 — Build Project 2: Microservice Pair + REST Edge (Week 5)

14. **Two services** — `OrderService` calls `UserService` over gRPC (mTLS between them). Propagate deadlines and request IDs across the hop.
15. **Retries + LB** — configure a retry policy via service config; run two `UserService` replicas behind `dns:///` + `round_robin`.
16. **gRPC-Gateway** — add `google.api.http` annotations and stand up a REST/JSON gateway so a browser/curl can hit `OrderService`. One contract, two surfaces.
17. **Observability** — add OpenTelemetry metrics + Prometheus `/metrics`; trace a request across both services.

### Stage 6 — Production hardening + ConnectRPC (Week 6)

18. **ConnectRPC** — re-expose `UserService` with connect-go on a single port; call it from a TypeScript browser client with **no proxy**. Confirm the same server still answers grpc-go clients.
19. **Schema CI** — wire `buf lint` + `buf breaking` into CI; deliberately make a breaking change and watch it fail.
20. **Testing** — full bufconn test suite for all four method types, plus a gomock-based unit test of a caller. Add a stream test.
21. **Deploy** — Dockerize (distroless), add Kubernetes `grpc` health probes, headless service + client LB, and validate graceful drain under load.

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

---

*Guide accurate as of Go 1.23/1.24, proto3, `google.golang.org/grpc` v1.6x+ and `google.golang.org/protobuf` v1.3x+ (June 2026). The codegen toolchain (`protoc`/`buf`, `protoc-gen-go`, `protoc-gen-go-grpc`) and ConnectRPC evolve quickly — always cross-reference pkg.go.dev, protobuf.dev, buf.build, and connectrpc.com for the latest APIs.*
