# Go WebSockets with Gorilla WebSocket — Complete Reference Guide

A full, offline-study reference for building **real-time Go applications** using the **Gorilla WebSocket library** (`github.com/gorilla/websocket`). Covers the WebSocket protocol, every Gorilla API, the concurrency model, a complete chat-server example, browser and Go clients, security, scaling, and the patterns that separate toy demos from production code.

> **Version note:** This guide targets **Go 1.22+** and **gorilla/websocket v1.5.x** (2024–2026). The Gorilla project was dormant for a time but is **actively maintained again** under community stewardship as of 2023. The v1 API is stable and unlikely to change. Where something is version-sensitive it is flagged with **⚡ Version note**.

---

## Table of Contents

1. [What WebSockets Are & When to Use Them](#1-what-websockets-are--when-to-use-them)
2. [Project Setup](#2-project-setup)
3. [The Upgrader — HTTP to WebSocket](#3-the-upgrader)
4. [Reading & Writing Messages](#4-reading--writing-messages)
5. [The Critical Concurrency Model](#5-the-critical-concurrency-model)
6. [Ping/Pong, Deadlines & Connection Health](#6-pingpong-deadlines--connection-health)
7. [Handling Disconnects & Close Codes](#7-handling-disconnects--close-codes)
8. [The Hub / Broadcast Pattern — Full Chat Server](#8-the-hub--broadcast-pattern)
9. [Browser JS Client & Go Dialer Client](#9-browser-js-client--go-dialer-client)
10. [Origin Checking & Security](#10-origin-checking--security)
11. [Scaling — Beyond a Single Process](#11-scaling--beyond-a-single-process)
12. [Tips, Tricks & Gotchas](#12-tips-tricks--gotchas)
13. [Study Path](#13-study-path)

---

## 1. What WebSockets Are & When to Use Them

### HTTP vs WebSocket

HTTP is a **request-response** protocol. The client sends a request; the server sends a response; the connection closes (or is reused for the *next* request in HTTP keep-alive, but the server still cannot push unprompted).

WebSocket is a **persistent, full-duplex** protocol. After a one-time HTTP handshake, both sides can send frames to each other at any time, with very low overhead (~2–14 bytes of framing per message vs hundreds of bytes for an HTTP header).

| Property | HTTP | WebSocket |
|---|---|---|
| Connection lifetime | Short-lived per request | Persistent until closed |
| Who initiates messages | Client only | Either side |
| Overhead per message | High (headers each time) | Low (small frame header) |
| Latency | Round-trip per message | Minimal (persistent TCP) |
| Streaming data | Polling / SSE (one-way) | Full-duplex |
| Use case | CRUD, page loads | Real-time, live data |

### When to Use WebSockets

**Reach for WebSockets when:**
- Messages flow in both directions and timing matters (chat, multiplayer games)
- The server needs to push data without the client asking (live dashboards, notifications, stock tickers)
- High message rate makes polling expensive (sports scores, live sensor data)
- You need sub-second latency

**Stick with HTTP/SSE when:**
- The server only pushes (use **Server-Sent Events** — simpler, HTTP/2 friendly)
- Low-frequency updates where polling every 30 s is acceptable
- The client is a simple script/cron that does not need to receive pushes

### The WebSocket Handshake

The connection begins as a standard HTTP/1.1 request with special upgrade headers:

```
Client → Server:
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client (101 Switching Protocols):
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the `101` response the TCP socket is "promoted" — both sides speak the WebSocket framing protocol from that point on. Gorilla handles all of this automatically when you call `upgrader.Upgrade()`.

---

## 2. Project Setup

### Prerequisites

- Go 1.18+ (generics are available but not required here; 1.22+ recommended)
- A working Go module (`go.mod`)

### Installing Gorilla WebSocket

```bash
# Inside your module directory
go get github.com/gorilla/websocket@latest
```

This adds `github.com/gorilla/websocket v1.5.x` to your `go.mod` and `go.sum`.

> **⚡ Version note:** `gorilla/websocket` follows semantic versioning. v1.x is the only stable major version as of 2026. There is no v2 migration needed.

### Minimal project layout

```
myapp/
├── go.mod
├── go.sum
├── main.go          ← HTTP server entry point
├── hub.go           ← Hub broadcast logic
├── client.go        ← Client read/write pumps
└── static/
    └── index.html   ← Browser client
```

### go.mod after installation

```go
module github.com/yourname/myapp

go 1.22

require github.com/gorilla/websocket v1.5.3
```

### Hello-world sanity check

```go
// main.go — bare-minimum WebSocket echo server
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/gorilla/websocket"
)

// upgrader converts an HTTP connection into a WebSocket connection.
// AllowAllOrigins is fine for local development — tighten this in production!
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func echoHandler(w http.ResponseWriter, r *http.Request) {
    // Upgrade the HTTP connection to WebSocket.
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("upgrade error:", err)
        return
    }
    defer conn.Close()

    for {
        // Read a message from the client.
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            log.Println("read error:", err)
            break
        }
        fmt.Printf("Received: %s\n", msg)

        // Echo the same message back.
        if err := conn.WriteMessage(msgType, msg); err != nil {
            log.Println("write error:", err)
            break
        }
    }
}

func main() {
    http.HandleFunc("/ws", echoHandler)
    http.Handle("/", http.FileServer(http.Dir("./static")))
    log.Println("Listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Run with `go run main.go` and test with `wscat -c ws://localhost:8080/ws` or the browser console.

---

## 3. The Upgrader

`websocket.Upgrader` is the bridge between Go's `net/http` and the WebSocket protocol. You almost always declare one as a package-level variable and reuse it.

```go
var upgrader = websocket.Upgrader{
    // ReadBufferSize is the size of the internal read buffer (bytes).
    // Messages larger than this are still handled — the buffer is just
    // for I/O efficiency. Default 4096.
    ReadBufferSize: 1024,

    // WriteBufferSize is the same for writes.
    WriteBufferSize: 1024,

    // HandshakeTimeout limits how long the initial upgrade may take.
    // Zero means no timeout (not recommended for production).
    HandshakeTimeout: 10 * time.Second,

    // CheckOrigin is called for every upgrade request.
    // Return true → allow, false → reject with 403.
    // Returning true for all origins is ONLY acceptable for local dev.
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        return origin == "https://myapp.example.com"
    },

    // Subprotocols the server supports (optional).
    // The client's Sec-WebSocket-Protocol header is matched against this list.
    Subprotocols: []string{"chat", "v2"},

    // Error is called when the upgrade fails; defaults to http.Error.
    Error: func(w http.ResponseWriter, r *http.Request, status int, reason error) {
        http.Error(w, reason.Error(), status)
    },
}
```

### Upgrading inside a handler

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    // You can inspect r before upgrading — check auth tokens, session
    // cookies, query params, etc. Once upgraded, you cannot send HTTP
    // responses, so validate BEFORE calling Upgrade.
    if !isAuthenticated(r) {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    // responseHeader can add custom headers to the 101 response (e.g., cookies).
    responseHeader := http.Header{}
    responseHeader.Set("X-Server", "myapp")

    conn, err := upgrader.Upgrade(w, r, responseHeader)
    if err != nil {
        // upgrader already wrote an HTTP error to w; just log and return.
        log.Printf("upgrade failed for %s: %v", r.RemoteAddr, err)
        return
    }
    defer conn.Close() // always close to release OS resources

    // conn is a *websocket.Conn — use it to read/write messages.
    handleConnection(conn)
}
```

### Buffer size guidelines

| Situation | ReadBuffer | WriteBuffer |
|---|---|---|
| Small JSON messages (chat, events) | 1024–4096 | 1024–4096 |
| Medium payloads (dashboards) | 4096–16384 | 4096–16384 |
| Binary / file transfer | 32768–65536 | 32768–65536 |
| Thousands of idle connections | 512 (minimise RSS) | 512 |

Gorilla allocates one buffer per connection from the heap, so smaller buffers mean less memory when you have many connections.

---

## 4. Reading & Writing Messages

### Message types

WebSocket frames have a numeric opcode. Gorilla exposes them as constants:

| Constant | Value | Meaning |
|---|---|---|
| `websocket.TextMessage` | 1 | UTF-8 text (JSON, plain text) |
| `websocket.BinaryMessage` | 2 | Raw bytes (protobuf, images) |
| `websocket.CloseMessage` | 8 | Initiate close handshake |
| `websocket.PingMessage` | 9 | Keepalive ping |
| `websocket.PongMessage` | 10 | Keepalive pong (reply to ping) |

### ReadMessage / WriteMessage

```go
// Reading — blocks until a full message arrives or an error occurs.
messageType, p, err := conn.ReadMessage()
// messageType is websocket.TextMessage or websocket.BinaryMessage
// p is []byte — the full message payload
// err is non-nil on close, timeout, or network error

// Writing — sends one complete message.
err = conn.WriteMessage(websocket.TextMessage, []byte(`{"hello":"world"}`))
```

### ReadJSON / WriteJSON

Gorilla provides helpers that wrap encoding/json:

```go
// WriteJSON marshals v to JSON then sends as a TextMessage.
type Event struct {
    Type    string `json:"type"`
    Payload string `json:"payload"`
}

err := conn.WriteJSON(Event{Type: "msg", Payload: "hello"})

// ReadJSON unmarshals the next message into v.
var incoming Event
err = conn.ReadJSON(&incoming)
if err != nil {
    log.Println("readJSON error:", err)
    return
}
fmt.Println(incoming.Type, incoming.Payload)
```

> **Note:** `ReadJSON` calls `ReadMessage` under the hood and then `json.Unmarshal`. If the payload is not valid JSON you get a JSON decode error, not a WebSocket error.

### Streaming large messages with NextReader / NextWriter

For large payloads, avoid loading the entire message into memory at once:

```go
// NextReader returns an io.Reader that streams the next message.
msgType, r, err := conn.NextReader()
if err != nil {
    return
}
// Read from r like any io.Reader — useful for proxying or streaming decode.
io.Copy(destination, r)

// NextWriter returns an io.WriteCloser for writing a message incrementally.
w, err := conn.NextWriter(websocket.BinaryMessage)
if err != nil {
    return
}
io.Copy(w, source)
w.Close() // MUST call Close() to flush and send the final frame
```

### Complete read/write example

```go
// handleConnection demonstrates the basic message loop.
// Run this in a dedicated goroutine per connection.
func handleConnection(conn *websocket.Conn) {
    defer conn.Close()

    for {
        // ReadMessage blocks until a message arrives or the connection closes.
        msgType, data, err := conn.ReadMessage()
        if err != nil {
            // websocket.IsCloseError checks for expected close conditions.
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                log.Printf("unexpected close: %v", err)
            }
            return // exit the loop; deferred conn.Close() fires
        }

        switch msgType {
        case websocket.TextMessage:
            log.Printf("text: %s", data)
            // Process and respond.
            conn.WriteMessage(websocket.TextMessage, data)
        case websocket.BinaryMessage:
            log.Printf("binary: %d bytes", len(data))
        }
    }
}
```

---

## 5. The Critical Concurrency Model

This is **the most important section**. Get this wrong and your server panics under load.

### The rule

> **A `*websocket.Conn` is NOT goroutine-safe for concurrent writes (or concurrent reads).**
>
> - At most **ONE goroutine** may call read methods at a time.
> - At most **ONE goroutine** may call write methods at a time.
> - Reads and writes may happen concurrently with each other (one goroutine each).

### Why concurrent writes panic

Gorilla's `WriteMessage` writes a header frame then a data frame. If two goroutines call `WriteMessage` simultaneously, the frames interleave on the wire — corrupting the stream. Gorilla does **not** add internal locking; instead it documents this constraint so you design correctly.

```go
// BAD — two goroutines writing concurrently → PANIC or corrupted stream
go func() { conn.WriteMessage(websocket.TextMessage, []byte("a")) }()
go func() { conn.WriteMessage(websocket.TextMessage, []byte("b")) }() // RACE!
```

### The canonical solution: a send channel + writePump goroutine

```go
// Client represents a single WebSocket connection.
type Client struct {
    conn *websocket.Conn
    send chan []byte // buffered channel of outbound messages
}

// writePump is the ONLY goroutine that writes to conn.
// It drains the send channel and writes each message.
func (c *Client) writePump() {
    // ticker sends periodic pings to keep the connection alive.
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            // Set a write deadline so a slow/stuck client cannot block forever.
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))

            if !ok {
                // The hub closed the channel — send a WebSocket close frame.
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            // Get a writer for a text message — enables batching multiple
            // queued messages into one WebSocket frame (efficiency win).
            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // Drain any additional queued messages into the same frame.
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte{'\n'}) // separator
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            // Send a ping to check if the client is still alive.
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// readPump is the ONLY goroutine that reads from conn.
func (c *Client) readPump() {
    defer func() {
        c.conn.Close()
    }()

    // Cap the maximum message size to prevent memory exhaustion.
    c.conn.SetReadLimit(512 * 1024) // 512 KB

    // Initial read deadline — the client must send its first message within this time.
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))

    // Every time a pong arrives, reset the read deadline.
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseAbnormalClosure,
            ) {
                log.Printf("readPump error: %v", err)
            }
            break
        }
        // Hand the message off to application logic — do NOT write to conn here!
        c.handleIncomingMessage(message)
    }
}
```

### Summary diagram

```
                          ┌─────────────┐
                          │  net/http   │
                          │  goroutine  │ ← calls Upgrade(), then spawns:
                          └──────┬──────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                                     ▼
      ┌───────────────┐                  ┌───────────────────┐
      │  readPump()   │                  │   writePump()     │
      │  goroutine    │ ───send chan───▶  │   goroutine       │
      │               │                  │                   │
      │ conn.Read*()  │                  │ conn.Write*()     │
      │ ONLY          │                  │ ONLY              │
      └───────────────┘                  └───────────────────┘
             ▲                                     │
             │                                     ▼
        WebSocket                            WebSocket
         frames in                           frames out
```

---

## 6. Ping/Pong, Deadlines & Connection Health

TCP connections can silently die (NAT timeout, mobile switching networks, crashed client). Without keepalive you would hold open goroutines and memory for ghost connections indefinitely.

### How Ping/Pong works

The WebSocket protocol defines ping (opcode 9) and pong (opcode 10) control frames. The spec requires that any endpoint receiving a ping reply immediately with a pong containing the same payload. Gorilla handles incoming pings automatically — it sends the pong for you unless you override the handler.

### SetReadDeadline + SetWriteDeadline

```go
// Deadlines are absolute times. Extend them after each successful activity.

// writePump — set before every write operation:
conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
conn.WriteMessage(websocket.PingMessage, nil)

// readPump — set initially, then reset in the PongHandler:
const (
    pongWait   = 60 * time.Second  // wait this long for a pong
    pingPeriod = (pongWait * 9) / 10 // ping slightly before deadline
    writeWait  = 10 * time.Second  // max time for a write to complete
)

conn.SetReadDeadline(time.Now().Add(pongWait))

conn.SetPongHandler(func(appData string) error {
    // Client is alive — extend the read deadline.
    conn.SetReadDeadline(time.Now().Add(pongWait))
    return nil
})
```

### Full keepalive setup inside writePump

```go
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod) // e.g., 54s if pongWait = 60s
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case msg, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            if err := c.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                return // write deadline exceeded or connection broken
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return // client did not pong in time or network is gone
            }
        }
    }
}
```

### SetPingHandler / SetCloseHandler

```go
// Override the default ping handler (usually not needed):
conn.SetPingHandler(func(appData string) error {
    log.Println("received ping")
    // Must send pong — the default handler does this for you.
    err := conn.WriteControl(websocket.PongMessage, []byte(appData),
        time.Now().Add(writeWait))
    if err == websocket.ErrCloseSent {
        return nil
    }
    return err
})

// Override the close handler (fires when the peer sends a close frame):
conn.SetCloseHandler(func(code int, text string) error {
    log.Printf("close: code=%d text=%s", code, text)
    // The default sends an echo close frame back — preserve that behaviour:
    msg := websocket.FormatCloseMessage(code, "")
    conn.WriteControl(websocket.CloseMessage, msg, time.Now().Add(writeWait))
    return nil
})
```

### SetReadLimit

```go
// Reject messages larger than maxMessageSize bytes.
// Protects against malicious or buggy clients sending huge payloads.
const maxMessageSize = 512 * 1024 // 512 KB
conn.SetReadLimit(maxMessageSize)
```

---

## 7. Handling Disconnects & Close Codes

### The close handshake

A clean disconnect involves a close frame exchange:

1. Either side sends a close frame (with optional code + reason string).
2. The peer echoes back a close frame.
3. Both sides close the TCP connection.

Gorilla handles the echo automatically inside `ReadMessage` — when it receives a close frame it sends the reply and returns an error of type `*websocket.CloseError`.

### WebSocket close codes (common subset)

| Code | Constant | Meaning |
|---|---|---|
| 1000 | `CloseNormalClosure` | Clean shutdown |
| 1001 | `CloseGoingAway` | Browser tab closed, server restart |
| 1002 | `CloseProtocolError` | Protocol violation |
| 1003 | `CloseUnsupportedData` | Wrong message type |
| 1008 | `ClosePolicyViolation` | App-level rejection |
| 1009 | `CloseMessageTooBig` | Payload too large |
| 1011 | `CloseInternalServerErr` | Server-side error |

### Detecting and handling close errors

```go
func (c *Client) readPump() {
    defer c.conn.Close()

    for {
        _, msg, err := c.conn.ReadMessage()
        if err != nil {
            var closeErr *websocket.CloseError
            if errors.As(err, &closeErr) {
                // Expected close — log code and reason.
                log.Printf("client closed: code=%d reason=%s",
                    closeErr.Code, closeErr.Text)
            } else if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
                websocket.CloseAbnormalClosure,
            ) {
                // Unexpected: network reset, crash, timeout.
                log.Printf("unexpected close: %v", err)
            }
            return // always exit the loop
        }
        c.processMessage(msg)
    }
}
```

### Initiating a clean close from the server

```go
func closeClient(conn *websocket.Conn, code int, reason string) {
    msg := websocket.FormatCloseMessage(code, reason)
    // Send the close frame with a short deadline.
    conn.WriteControl(websocket.CloseMessage, msg, time.Now().Add(5*time.Second))
    // Give the client a moment to echo back, then close the TCP socket.
    time.Sleep(500 * time.Millisecond)
    conn.Close()
}

// Usage:
closeClient(conn, websocket.CloseNormalClosure, "server restarting")
```

---

## 8. The Hub / Broadcast Pattern

This is the standard architecture for any multi-client real-time server (chat rooms, live dashboards, notification systems).

The key insight: **a Hub is a single goroutine that owns a map of clients**. No mutex needed because only the Hub goroutine touches the map. Clients communicate with the Hub over channels.

### Architecture overview

```
 HTTP handler
      │
      │ upgrade → *websocket.Conn
      │
      ▼
  ┌────────┐   register/unregister/broadcast
  │ Client │ ───────────────────────────────▶ ┌─────┐
  │ struct │                                   │ Hub │  ── map[*Client]bool
  └────────┘ ◀─────────── send chan ─────────── └─────┘
  readPump│                                        ▲
  writePump│                              other clients' readPumps
```

### Full implementation

#### hub.go

```go
package main

import "log"

// Hub maintains the set of active clients and broadcasts messages.
type Hub struct {
    // clients is the registry of connected clients.
    // Only the Hub goroutine reads/writes this map (no mutex needed).
    clients map[*Client]bool

    // broadcast receives messages that should be sent to every client.
    broadcast chan []byte

    // register receives new clients that just connected.
    register chan *Client

    // unregister receives clients that are disconnecting.
    unregister chan *Client
}

func newHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}

// run is the Hub's event loop. Run it in exactly one goroutine.
func (h *Hub) run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true
            log.Printf("client registered; total=%d", len(h.clients))

        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send) // signal writePump to exit
                log.Printf("client unregistered; total=%d", len(h.clients))
            }

        case message := <-h.broadcast:
            // Send to every connected client.
            for client := range h.clients {
                select {
                case client.send <- message:
                    // Message queued successfully.
                default:
                    // The client's send buffer is full — it's too slow.
                    // Unregister it to prevent the Hub from blocking.
                    delete(h.clients, client)
                    close(client.send)
                    log.Println("dropped slow client")
                }
            }
        }
    }
}
```

#### client.go

```go
package main

import (
    "bytes"
    "log"
    "net/http"
    "time"

    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10 // 54s
    maxMessageSize = 512                  // bytes — small for chat
)

// Client is a middleman between a WebSocket connection and the Hub.
type Client struct {
    hub  *Hub
    conn *websocket.Conn
    // send is a buffered channel of outbound messages.
    // writePump is the ONLY goroutine that reads from send and writes to conn.
    send chan []byte
}

// readPump pumps messages from the WebSocket connection to the Hub.
//
// The application runs readPump in a per-connection goroutine. The application
// ensures that there is at most one reader on a connection by running all reads
// from this goroutine.
func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseAbnormalClosure,
            ) {
                log.Printf("readPump: %v", err)
            }
            break
        }

        // Trim whitespace and forward to the Hub for broadcast.
        message = bytes.TrimSpace(bytes.Replace(message, []byte{'\n'}, []byte{' '}, -1))
        c.hub.broadcast <- message
    }
}

// writePump pumps messages from the Hub to the WebSocket connection.
//
// A goroutine running writePump is started for each connection. The
// application ensures that there is at most one writer to a connection by
// running all writes from this goroutine.
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Hub closed the channel.
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // Batch-write any additional queued messages (efficiency).
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte{'\n'})
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

#### main.go — wiring it together

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
        // TODO: restrict to your domain in production
        return true
    },
}

// serveWs handles WebSocket upgrade requests from the peer.
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("upgrade:", err)
        return
    }

    client := &Client{
        hub:  hub,
        conn: conn,
        send: make(chan []byte, 256),
    }

    // Register the client with the Hub.
    client.hub.register <- client

    // Run pumps in their own goroutines so that the HTTP handler can return.
    go client.writePump()
    go client.readPump()
}

func main() {
    hub := newHub()
    go hub.run() // Hub event loop runs in a single goroutine

    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        serveWs(hub, w, r)
    })
    http.Handle("/", http.FileServer(http.Dir("./static")))

    log.Println("chat server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### JSON message types

In real apps you usually send typed JSON envelopes rather than raw strings:

```go
// Message is the JSON envelope for all chat messages.
type Message struct {
    Type    string `json:"type"`             // "chat", "join", "leave", "system"
    User    string `json:"user,omitempty"`
    Content string `json:"content,omitempty"`
    Time    int64  `json:"time"`             // Unix timestamp ms
}

// In readPump — decode the JSON and re-encode with server-set fields:
var incoming Message
if err := json.Unmarshal(message, &incoming); err != nil {
    log.Println("bad JSON:", err)
    continue
}
incoming.Time = time.Now().UnixMilli()
outgoing, _ := json.Marshal(incoming)
c.hub.broadcast <- outgoing
```

---

## 9. Browser JS Client & Go Dialer Client

### Browser JavaScript client

```html
<!-- static/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Go Chat</title>
</head>
<body>
  <div id="log" style="height:400px;overflow:auto;border:1px solid #ccc;padding:8px;font-family:monospace;"></div>
  <input id="msg" type="text" placeholder="Type a message…" style="width:80%">
  <button id="send">Send</button>

  <script>
    const log  = document.getElementById('log');
    const input = document.getElementById('msg');

    // Connect to the WebSocket server.
    // Use wss:// in production (TLS).
    const ws = new WebSocket('ws://localhost:8080/ws');

    ws.onopen = () => {
      appendLog('Connected');
    };

    ws.onmessage = (event) => {
      // Each line in the event.data is a batched message from Go's NextWriter.
      const lines = event.data.split('\n');
      lines.forEach(line => {
        try {
          const msg = JSON.parse(line);
          appendLog(`[${msg.user}] ${msg.content}`);
        } catch {
          appendLog(line); // plain text fallback
        }
      });
    };

    ws.onclose = (event) => {
      appendLog(`Disconnected: code=${event.code} reason=${event.reason}`);
    };

    ws.onerror = (err) => {
      appendLog(`Error: ${err.message ?? 'unknown'}`);
    };

    document.getElementById('send').addEventListener('click', sendMessage);
    input.addEventListener('keydown', e => { if (e.key === 'Enter') sendMessage(); });

    function sendMessage() {
      const text = input.value.trim();
      if (!text || ws.readyState !== WebSocket.OPEN) return;
      const msg = JSON.stringify({ type: 'chat', user: 'Alice', content: text });
      ws.send(msg);
      input.value = '';
    }

    function appendLog(text) {
      const p = document.createElement('p');
      p.textContent = text;
      log.appendChild(p);
      log.scrollTop = log.scrollHeight;
    }
  </script>
</body>
</html>
```

### WebSocket.readyState values (reference)

| Value | Constant | Meaning |
|---|---|---|
| 0 | CONNECTING | Handshake in progress |
| 1 | OPEN | Connected; can send/receive |
| 2 | CLOSING | Close handshake in progress |
| 3 | CLOSED | Connection closed |

### Go client using websocket.Dialer

The Gorilla library also provides a **client-side** WebSocket dialer, useful for testing, microservice communication, or CLI tools:

```go
// goclient/main.go — connects to a WebSocket server from Go
package main

import (
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
    // Construct the server URL.
    u := url.URL{Scheme: "ws", Host: "localhost:8080", Path: "/ws"}
    log.Printf("connecting to %s", u.String())

    // Dialer lets you customise TLS, proxy, timeouts, headers.
    dialer := websocket.Dialer{
        HandshakeTimeout: 10 * time.Second,
    }

    // RequestHeader is your chance to send auth tokens, cookies, etc.
    header := make(map[string][]string)
    header["Authorization"] = []string{"Bearer my-token"}

    conn, _, err := dialer.Dial(u.String(), header)
    if err != nil {
        log.Fatal("dial error:", err)
    }
    defer conn.Close()

    // Handle OS interrupt for clean shutdown.
    interrupt := make(chan os.Signal, 1)
    signal.Notify(interrupt, os.Interrupt)

    // Goroutine to read messages from the server.
    done := make(chan struct{})
    go func() {
        defer close(done)
        for {
            _, data, err := conn.ReadMessage()
            if err != nil {
                log.Println("read:", err)
                return
            }
            var msg Message
            if err := json.Unmarshal(data, &msg); err == nil {
                log.Printf("[%s] %s", msg.User, msg.Content)
            } else {
                log.Printf("raw: %s", data)
            }
        }
    }()

    // Send a message every 3 seconds until interrupted.
    ticker := time.NewTicker(3 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            msg := Message{Type: "chat", User: "GoClient", Content: "ping from Go"}
            data, _ := json.Marshal(msg)
            if err := conn.WriteMessage(websocket.TextMessage, data); err != nil {
                log.Println("write:", err)
                return
            }

        case <-interrupt:
            log.Println("interrupt — closing cleanly")
            // Send a close frame and wait for the server to echo it.
            err := conn.WriteMessage(
                websocket.CloseMessage,
                websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""),
            )
            if err != nil {
                log.Println("write close:", err)
                return
            }
            select {
            case <-done:     // server echoed close
            case <-time.After(time.Second): // timeout
            }
            return

        case <-done:
            return // server closed first
        }
    }
}
```

### Connecting with TLS (wss://)

```go
import "crypto/tls"

dialer := websocket.Dialer{
    TLSClientConfig: &tls.Config{
        InsecureSkipVerify: false, // NEVER true in production
        // RootCAs: customCertPool, // for self-signed certs
    },
}
conn, _, err := dialer.Dial("wss://myapp.example.com/ws", nil)
```

---

## 10. Origin Checking & Security

### Why CheckOrigin matters

Browsers automatically include the `Origin` header on WebSocket upgrade requests. Without checking it, a malicious site could open a WebSocket connection to your server while the victim is logged in, and send messages as that user (cross-site WebSocket hijacking).

### Never use this in production

```go
// DANGEROUS — accepts any origin, including attacker.example.com
CheckOrigin: func(r *http.Request) bool { return true }
```

### Correct origin checking

```go
import (
    "net/http"
    "strings"
)

var allowedOrigins = map[string]bool{
    "https://myapp.example.com":  true,
    "https://www.example.com":    true,
    "http://localhost:3000":       true, // dev only — remove in prod
}

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        return allowedOrigins[origin]
    },
}
```

### Authenticate BEFORE upgrading

Once the connection is upgraded you can no longer send HTTP 401 responses. Put all auth logic before the `Upgrade` call:

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    // 1. Check a session cookie or JWT query param.
    userID, err := validateToken(r)
    if err != nil {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return // never reaches Upgrade
    }

    // 2. Now it is safe to upgrade.
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("upgrade:", err)
        return
    }

    // 3. Attach userID to the client struct for later use.
    client := &Client{conn: conn, userID: userID, send: make(chan []byte, 256)}
    hub.register <- client
    go client.writePump()
    go client.readPump()
}
```

### JWT via query parameter (common pattern)

```go
// Client connects to: wss://api.example.com/ws?token=eyJ...
token := r.URL.Query().Get("token")
if token == "" {
    http.Error(w, "missing token", http.StatusUnauthorized)
    return
}
claims, err := jwtVerify(token)
if err != nil {
    http.Error(w, "invalid token", http.StatusUnauthorized)
    return
}
```

> **Note:** Sending JWTs in query params means they appear in server logs. Prefer a short-lived ticket exchanged for the WS connection, or send auth as the first WebSocket message.

### SetReadLimit protects against large payloads

```go
conn.SetReadLimit(64 * 1024) // 64 KB per message
// Clients sending larger messages get disconnected automatically.
```

### Security checklist

| Item | Action |
|---|---|
| Origin checking | Allowlist known origins only |
| Authentication | Validate before `Upgrade()` |
| Message size | Always set `SetReadLimit` |
| TLS | Use `wss://` — never `ws://` on the internet |
| Rate limiting | Implement per-client message rate limits |
| Input validation | Validate/sanitise all incoming message content |
| Goroutine leak | Ensure readPump/writePump always exit on disconnect |

---

## 11. Scaling — Beyond a Single Process

### Single-process limits

A single Go process can realistically hold **10,000–100,000 concurrent WebSocket connections** depending on:
- Message rate per connection (idle connections are cheap)
- Message payload size
- Server RAM (each connection has buffers + two goroutines)
- OS file descriptor limit (`ulimit -n`)

For small/medium apps (< 50k concurrent users) a single well-tuned Go process is usually sufficient.

### The multi-instance problem

When you scale horizontally (multiple app servers behind a load balancer), a message broadcast by a client on Server A must also reach clients on Servers B and C. Your in-process Hub only knows about local clients.

### Redis Pub/Sub pattern (conceptual)

```
Client A (on Server 1)
    │ sends message
    ▼
Server 1 Hub → PUBLISH to Redis channel "chat:room1"
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
     Server 1       Server 2       Server 3
    SUBSCRIBE       SUBSCRIBE      SUBSCRIBE
     ↓                ↓              ↓
  local clients    local clients  local clients
  receive msg      receive msg    receive msg
```

```go
// Pseudocode — Redis fan-out skeleton
import "github.com/redis/go-redis/v9"

func (h *Hub) publishToRedis(msg []byte) {
    h.redisClient.Publish(ctx, "chat:broadcast", msg)
}

func (h *Hub) subscribeFromRedis() {
    pubsub := h.redisClient.Subscribe(ctx, "chat:broadcast")
    ch := pubsub.Channel()
    for msg := range ch {
        // Broadcast to all local clients.
        h.broadcast <- []byte(msg.Payload)
    }
}
```

### Other scaling options

| Approach | Use case |
|---|---|
| Redis Pub/Sub | Most common; simple fan-out across instances |
| NATS | Lower latency than Redis, built for messaging |
| Kafka | High-throughput event streaming, durable message log |
| Single large instance | Often the right call — profile before scaling |
| Sticky sessions on LB | Avoids fan-out but creates uneven distribution |

### OS-level tuning for many connections

```bash
# Increase file descriptor limit (each WS connection = 1 fd)
ulimit -n 1000000

# /etc/sysctl.conf for persistent settings
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
fs.file-max = 1000000
```

---

## 12. Tips, Tricks & Gotchas

### 1. Concurrent write panic (the #1 mistake)

```go
// WRONG — two goroutines writing at the same time
go func() { conn.WriteJSON(msg1) }()
go func() { conn.WriteJSON(msg2) }() // concurrent write → panic or corruption
```

**Fix:** All writes go through a single `writePump` goroutine. Use a send channel. See Section 5.

---

### 2. Forgetting deadlines = goroutine leak factory

If a client silently drops (mobile network, killed process), `ReadMessage` blocks forever. Your goroutine never exits. Memory and goroutine count grow until the server crashes.

**Fix:** Always set `SetReadDeadline` and reset it in `SetPongHandler`. Always set `SetWriteDeadline` before every write. See Section 6.

---

### 3. Goroutine leaks from early returns

```go
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
    conn, _ := upgrader.Upgrade(w, r, nil)
    client := &Client{conn: conn, send: make(chan []byte, 256)}
    hub.register <- client
    go client.writePump()
    go client.readPump()
    // CORRECT: HTTP handler returns; both goroutines are running independently
}
```

If you block the HTTP handler goroutine instead of spawning dedicated goroutines, you starve the server's goroutine pool.

---

### 4. Backpressure — slow clients block the Hub

```go
// In the Hub's broadcast loop:
select {
case client.send <- message:
    // fast path — queued
default:
    // slow client — drop them to prevent the Hub from blocking
    delete(h.clients, client)
    close(client.send)
}
```

Without the `default` case, a slow/stuck client causes `hub.broadcast` to block, which means the `readPump` of *every other* client blocks while trying to send to the hub. The whole server stalls.

---

### 5. Not checking `ok` on the send channel

```go
case message, ok := <-c.send:
    if !ok {
        // Hub closed the channel — MUST send a close frame and exit
        c.conn.WriteMessage(websocket.CloseMessage, []byte{})
        return
    }
```

If you don't check `ok`, you write a zero-value `[]byte(nil)` to the connection on every tick after the channel is closed.

---

### 6. Using WriteMessage after NextWriter

```go
w, err := conn.NextWriter(websocket.TextMessage)
// ...
w.Write(data)
w.Close() // send

// Do NOT call conn.WriteMessage while NextWriter is open!
// NextWriter holds the connection; WriteMessage would try to start
// a new frame while the previous one is incomplete.
```

---

### 7. ReadJSON does not limit message size

`conn.ReadJSON(&v)` reads the *entire* message into memory before decoding. If you haven't set `SetReadLimit`, a 100 MB message allocates 100 MB on your heap per connection.

Always set `conn.SetReadLimit(maxBytes)` before your first read.

---

### 8. Binary vs Text message type with JSON

You *can* send JSON as a `BinaryMessage` and browsers will still receive it, but `websocket.TextMessage` is semantically correct and some proxy/firewall middleboxes only allow text frames through.

---

### 9. Connection errors during server shutdown

Use Go's `http.Server.Shutdown` + context to drain in-flight requests:

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

// Shutdown on SIGTERM / Ctrl-C
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGTERM, os.Interrupt)
go func() {
    <-c
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}()

srv.ListenAndServe()
```

For WebSocket connections, you should also close all `Hub` clients gracefully — send them a `CloseGoingAway` frame before shutting down.

---

### 10. wss:// behind a reverse proxy (nginx)

Nginx must explicitly proxy WebSocket upgrade headers:

```nginx
location /ws {
    proxy_pass         http://localhost:8080;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "Upgrade";
    proxy_set_header   Host $host;
    proxy_read_timeout 86400s; # keep-alive — must be longer than ping period
}
```

Without `proxy_read_timeout` being long enough, nginx kills idle WebSocket connections after 60 s.

---

## 13. Study Path

Work through these in order. Each builds on the last. **Implement, don't just read.**

### Stage 1 — Foundation (Week 1)

1. **Echo server** — write the simplest possible handler: upgrade, read one message, echo it back, close. Understand the upgrade lifecycle.
2. **Text & binary** — extend the echo server to handle both `TextMessage` and `BinaryMessage`, log the type.
3. **Ping/pong** — add `SetReadDeadline`, `SetPongHandler`, and a periodic ping in `writePump`. Kill the client process and verify the server detects it within `pongWait`.
4. **Close codes** — send `CloseNormalClosure` from the server after 10 seconds. Verify the browser reconnects cleanly.

**Resources:** Gorilla examples repo (`github.com/gorilla/websocket/tree/main/examples`), Go `net/http` docs.

---

### Stage 2 — Concurrency Model (Week 2)

5. **readPump + writePump** — refactor your echo server into the canonical two-goroutine model with a send channel. Verify no panics under 100 concurrent connections (`goroutine-race: go test -race`).
6. **Hub pattern** — build the broadcast Hub. Connect 5 browser tabs and verify a message from one appears in all.
7. **Slow client test** — set the send channel buffer to 1 and add a 10 ms sleep in `writePump`. Send 100 messages rapidly and verify the slow client is dropped gracefully, not blocking others.

---

### Stage 3 — Build Project 1: Chat Server (Week 3)

8. **Rooms** — extend the Hub to support multiple rooms. Clients join a room; broadcasts are room-scoped. Use `map[string]map[*Client]bool`.
9. **JSON protocol** — define a `Message` struct with `type`, `room`, `user`, `content`, `timestamp`. All messages are JSON envelopes.
10. **Join/leave events** — when a client connects or disconnects, broadcast a `join`/`leave` system message to the room.
11. **Browser UI** — build a minimal React or vanilla-JS chat client. Multi-room tabs.

---

### Stage 4 — Build Project 2: Live Dashboard (Week 4)

12. **Server-push only** — server publishes metrics (CPU, memory, request rate) every second. Clients only receive.
13. **Per-client subscriptions** — clients send a `subscribe` message with a list of metric names. Hub filters broadcasts per subscription.
14. **Charts** — plot data in the browser with Chart.js or uPlot (lightweight).

---

### Stage 5 — Build Project 3: Notifications Service (Week 5)

15. **User-targeted messages** — Hub maps `userID → *Client` instead of storing all clients indiscriminately.
16. **REST API trigger** — an internal HTTP endpoint (`POST /notify`) accepts `{userID, message}` and sends it to the target client if connected, or queues it for later.
17. **Persistence** — store unsent notifications in SQLite/Postgres. On client connect, flush pending notifications.

---

### Stage 6 — Production Readiness (Week 6)

18. **TLS** — add `ListenAndServeTLS` or put the server behind nginx with `wss://`.
19. **Auth** — JWT validation before upgrade. Short-lived WS tickets.
20. **Metrics** — expose Prometheus metrics: active connections, messages per second, goroutine count.
21. **Graceful shutdown** — on SIGTERM, send `CloseGoingAway` to all clients, wait for pumps to exit, then shut down.
22. **Redis fan-out** — run two instances of your chat server. Use Redis Pub/Sub to fan messages across instances. Load-balance with nginx.

---

### Reference bookmarks

| Resource | URL |
|---|---|
| Gorilla WebSocket docs | https://pkg.go.dev/github.com/gorilla/websocket |
| Official examples | https://github.com/gorilla/websocket/tree/main/examples |
| WebSocket RFC 6455 | https://datatracker.ietf.org/doc/html/rfc6455 |
| Go net/http docs | https://pkg.go.dev/net/http |
| wscat (CLI test tool) | `npm install -g wscat` then `wscat -c ws://localhost:8080/ws` |

---

*Guide accurate as of Go 1.22 and gorilla/websocket v1.5.x (June 2026). Always cross-reference with pkg.go.dev for the latest API details.*
