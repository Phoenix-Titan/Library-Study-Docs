# FTP Server — Go & Node — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have only ever used an FTP *client*" to "I can design, build, secure, and deploy a production file-transfer server" — entirely offline. We treat the **FTP protocol itself** as the foundation (because almost every bug you will ever hit is really a protocol/network misunderstanding), then build working servers in both **Go** (`ftpserverlib`) and **Node.js** (`ftp-srv`), add **FTPS/TLS**, build **SFTP** servers in both languages, and finish with a long, opinionated **security** treatment — because FTP is one of the most security-sensitive things a backend engineer can stand up. Every concept is explained in prose first — *what it is, the logic/why, what it is for and when to use it, how to use it, the key options, best practices, and the security implications* — and only then shown as heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections and sub-topics are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide is current for **2026**. Go examples target **Go 1.25/1.26** and **`github.com/fclairamb/ftpserverlib` v0.21.x–v0.22.x** with **`github.com/spf13/afero`**. Node examples target **Node 20/22 LTS**, **`ftp-srv` v5.x**, and **`ssh2` v1.x**; the Go SFTP examples use **`golang.org/x/crypto/ssh`** + **`github.com/pkg/sftp`**. FTP libraries shift method signatures between minor versions more than most — wherever behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (the built-in `ftp` client, path separators, shells) are called out. Always confirm exact API signatures against the library's `go.mod`/`CHANGELOG.md` or the package README after you install it.
>
> **Why learn to build an FTP server, not just use one?** File-transfer infrastructure is everywhere — customer file-drop portals, automated backup pipelines, EDI/B2B integrations, IoT device uploads, medical/financial data exchange, and a long tail of legacy systems that *only* speak FTP. Understanding FTP from the server side (its two-connection design, NAT traversal, and TLS) makes you the person on the team who can actually debug "the upload hangs after LIST" instead of guessing. Go gives you a single static binary and excellent concurrency; Node gives you rapid iteration and the JavaScript ecosystem. This guide covers both so you can choose deliberately and understand the trade-offs.

---

## Table of Contents

1. [FTP Protocol Fundamentals](#1-ftp-protocol-fundamentals) **[B]**
2. [Active vs Passive Mode & NAT](#2-active-vs-passive-mode--nat) **[B/I]**
3. [FTP vs FTPS vs SFTP — The Crucial Security Distinction](#3-ftp-vs-ftps-vs-sftp) **[B/I]**
4. [FTP Commands & Response Codes — Reference](#4-ftp-commands--response-codes--reference) **[B/I]**
5. [PART 1 — FTP Server in Go (ftpserverlib)](#5-part-1--ftp-server-in-go-ftpserverlib) **[I]**
6. [Adding TLS (FTPS) in Go](#6-adding-tls-ftps-in-go) **[I/A]**
7. [PART 2 — FTP Server in Node.js (ftp-srv)](#7-part-2--ftp-server-in-nodejs-ftp-srv) **[I]**
8. [Adding TLS (FTPS) in Node.js](#8-adding-tls-ftps-in-nodejs) **[I/A]**
9. [PART 3 — SFTP Servers (Go & Node)](#9-part-3--sftp-servers-go--node) **[A]**
10. [Authentication & User Isolation (Chroot / Virtual FS)](#10-authentication--user-isolation-chroot--virtual-fs) **[I/A]**
11. [Testing Your Server](#11-testing-your-server) **[B/I]**
12. [Firewall, Passive Port & NAT Configuration](#12-firewall-passive-port--nat-configuration) **[I/A]**
13. [Docker Deployment Notes](#13-docker-deployment-notes) **[I/A]**
14. [Logging, Observability & Operations](#14-logging-observability--operations) **[I/A]**
15. [Security Best Practices](#15-security-best-practices) **[A]**
16. [Tips, Tricks & Gotchas](#16-tips-tricks--gotchas) **[I/A]**
17. [Study Path & Build-to-Learn Projects](#17-study-path--build-to-learn-projects)

---

## 1. FTP Protocol Fundamentals

### 1.1 What FTP actually is **[B]**

FTP (File Transfer Protocol) is one of the oldest application protocols on the Internet, standardized in **RFC 959** (1985) and later extended by RFC 2228 (security), RFC 2428 (IPv6/EPSV), RFC 3659 (MLST/MDTM/SIZE), and RFC 4217 (FTP over TLS). Despite its age it is far from dead: it survives in managed file-transfer pipelines, ERP/EDI integrations, scientific data mirrors, network equipment, and countless "the vendor only supports FTP" situations. As a backend engineer you will not often *choose* FTP for a greenfield system (you would pick SFTP or HTTPS), but you will frequently have to *operate, secure, or replace* one.

The single most important fact about FTP — the one that explains 90% of its quirks — is that **FTP uses two separate TCP connections**, not one:

| Connection | Default port | Lifetime | Carries |
|---|---|---|---|
| **Control channel** | 21 (server listens) | Entire session | Commands and numeric responses (text) |
| **Data channel** | 20 (active) or dynamic (passive) | One per transfer/listing | File bytes and directory listings |

Almost every other protocol you know (HTTP, SSH, SMTP) multiplexes everything over one connection. FTP's decision to split "commands" from "data" onto two sockets is the historical root of all its firewall and NAT pain. Internalize this split now and the rest of the guide will feel inevitable rather than arbitrary.

> **Why two channels?** In 1985 the design goal was to let the control conversation stay responsive (and even support a *third party* transfer between two remote servers) while a large file streamed independently. The trade-off — that a stateful firewall now has to correlate two connections it sees as unrelated — was not a concern on the trusting early Internet. Today it is FTP's defining weakness.

### 1.2 The control channel (port 21) **[B]**

The client opens a TCP connection to the server on **port 21**. This connection stays open for the whole session. Every command the client sends and every response the server returns travels over it as line-oriented text (CRLF-terminated, UTF-8 in modern servers). In *plain* FTP this text — including your password — is completely unencrypted (see §3, and take it seriously).

The protocol is a strict **command → numeric-response** loop. The client sends a command like `USER alice`; the server replies with a three-digit code and a human-readable message like `331 Password required`. The client never has to guess what happened — the leading digit tells it the category of outcome (see §4).

A complete minimal session over the control channel looks like this:

```
Client                              Server (port 21)
  |------- TCP connect ------------->|
  |<------ 220 Welcome banner -------|
  |------- USER alice -------------->|
  |<------ 331 Password required ----|
  |------- PASS s3cr3t ------------->|
  |<------ 230 User logged in -------|
  |------- TYPE I ------------------>|   ← switch to binary mode
  |<------ 200 Type set to I --------|
  |------- PASV -------------------->|   ← ask for a passive data port
  |<------ 227 (10,0,0,5,195,80) ----|   ← server: "connect to me at 10.0.0.5:50000"
  |------- RETR report.pdf --------->|
  |<------ 150 Opening data conn ----|
  |   (client opens a SECOND TCP conn to 10.0.0.5:50000 — bytes flow there)
  |<------ 226 Transfer complete ----|
  |------- QUIT -------------------->|
  |<------ 221 Goodbye --------------|
```

Note that `RETR report.pdf` is sent on the control channel, but the *file bytes* arrive on a different connection entirely. The `150` response means "I am about to start; watch the data channel," and `226` means "the data channel finished and closed cleanly." Understanding which messages mean "watch the other socket" is the key to reading an FTP trace.

### 1.3 The data channel **[B]**

Every time the client lists a directory (`LIST`, `NLST`, `MLSD`) or transfers a file (`RETR` download, `STOR` upload), a **brand-new TCP connection** is created just for that operation and then closed. The control channel only *coordinates* it. How that data connection is established — who connects to whom — is decided by **active vs passive mode**, which is important enough to get its own section (§2).

A crucial consequence: the control connection can succeed (you log in fine, `PWD` works) while *every* data operation hangs or fails, because data uses different ports that may be blocked. "I can connect and log in but `ls` just sits there" is the canonical symptom of a misconfigured data channel — and it is the bug you will see most often.

### 1.4 ASCII vs binary transfer type **[B]**

Before transferring, the client sets a transfer **type** with the `TYPE` command:

- **`TYPE I`** (Image/binary) — bytes are sent verbatim. Use this for everything that is not plain text: images, PDFs, ZIPs, executables, databases. This is what you almost always want; configure your server to default to binary.
- **`TYPE A`** (ASCII) — the server may translate line endings (LF ↔ CRLF) between platforms. This was meant to make text files portable between Unix and Windows, but it will silently **corrupt binary files** (a single byte `0x0A` inside a JPEG becomes `0x0D 0x0A`).

> **Gotcha & best practice:** Always default to binary mode on both server and client. ASCII mode corruption is insidious — the file transfers "successfully" (226) but is broken. If a downloaded image or zip is mysteriously truncated/corrupt and a few bytes larger/smaller than the original, suspect ASCII mode.

### 1.5 The stateful session **[B/I]**

Unlike HTTP (stateless request/response), an FTP control connection is **stateful and long-lived**. The server remembers, per connection: who you are (after `USER`/`PASS`), your current working directory (changed by `CWD`/`CDUP`), your transfer type, your data-channel mode, and any partial rename in progress (`RNFR` then `RNTO`). This statefulness is why servers implement an **idle timeout** to reap abandoned sessions, and why long-running automated clients send periodic `NOOP` "keep-alive" pings so the server does not disconnect them mid-job.

---

## 2. Active vs Passive Mode & NAT

This is the single most misunderstood part of FTP and the number-one cause of "it works on localhost but not in production." Read it slowly. The question both modes answer is simply: **for the data connection, who dials whom?**

### 2.1 Active mode (the `PORT` command) **[B/I]**

In **active mode** the *server* opens the data connection back *to the client*. The flow:

1. The client picks a local port, starts listening on it, and tells the server its IP and port via the `PORT h1,h2,h3,h4,p1,p2` command (the IP is `h1.h2.h3.h4`, the port is `p1*256 + p2`).
2. When data is needed, the **server connects out** — typically *from* its port 20 — *to* the address the client announced.

```
Client (behind NAT/firewall)          Server
  |--- PORT 203,0,113,5,200,100 ---->|   ← "open a connection back to me at 203.0.113.5:51300"
  |--- RETR file.txt --------------->|
  |<-- server dials client:51300 ----|   ← server INITIATES inbound-to-client connection
  |    (data flows)                   |
```

**Why active mode is broken in the modern world:** the client usually sits behind NAT, so the IP it announces in `PORT` is a *private* address (`192.168.x.x`) the server cannot route to. Even when the address is public, the client's firewall blocks *inbound* connections by default — and a server reaching *into* a client is exactly the pattern firewalls exist to stop. Active mode is effectively legacy; only use it in tightly controlled networks where you own both ends and have explicitly allowed it.

### 2.2 Passive mode (the `PASV` / `EPSV` commands) **[B/I]**

In **passive mode** the *client* opens both connections; the server merely opens a port and waits.

1. The client sends `PASV` (IPv4) or `EPSV` (IPv4/IPv6).
2. The server picks a port from its configured passive range, listens on it, and replies with the address: `227 Entering Passive Mode (h1,h2,h3,h4,p1,p2)` for `PASV`, or `229 Entering Extended Passive Mode (|||port|)` for `EPSV`.
3. The **client connects out** to that address for the data.

```
Client (behind NAT/firewall)          Server
  |--- PASV ------------------------>|
  |<-- 227 (10,0,0,5,195,80) --------|   ← "connect to ME at 10.0.0.5:50000"
  |--- client dials server:50000 --->|   ← client INITIATES (outbound — firewalls allow this)
  |--- RETR file.txt --------------->|
  |<-- data flows ------------------|
```

**Why passive mode works behind NAT:** all connections are *outbound from the client*, and outbound connections are allowed by virtually every firewall. The server only needs port 21 plus a known inbound data-port range open. This is why **every modern FTP client defaults to passive mode** and why you should always configure and test it.

- **`PASV`** — classic passive, IPv4 only, encodes IP and port as six comma-separated bytes.
- **`EPSV`** — extended passive (RFC 2428), works for IPv4 and IPv6, returns only a port number (the client reuses the control-connection IP). Prefer EPSV where supported; it is also more NAT-friendly because the server does not have to embed an IP it may not know.

> **Rule of thumb:** Configure servers to support passive mode and test it from a *different network*, not just localhost. On localhost active mode often appears to work, hiding the bug until a real client connects.

### 2.3 The passive port range — what it is and how to size it **[I]**

Because the server opens a new TCP port for *every* concurrent data transfer/listing, you must pre-declare a **range** of ports that (a) your firewall allows inbound, (b) the server process may bind, and (c) no other service uses. A common choice is **50000–50100** (101 ports → up to 101 simultaneous data operations).

Sizing logic and best practice:

- One port is consumed for the *duration of a single transfer or listing*, then freed. You need roughly as many ports as your peak concurrent transfers, plus headroom.
- **Smaller is more secure** — fewer open ports means a smaller attack surface and (in Docker) fewer proxy processes (§13). Do not open thousands "just in case."
- The range in your **server config**, your **firewall**, and your **Docker `-p` mapping** must all match exactly. A mismatch is the classic "login works, transfers hang" bug.

### 2.4 The public-host problem (PASV behind NAT) **[I/A]**

When your server is itself behind NAT (a cloud box with a separate public IP, or a home router doing port-forwarding), the IP it knows about itself is *private*. If it naively puts that private IP into the `227` response, an external client will try to connect to `192.168.x.x` and hang. The fix is to explicitly configure the server's **public host** (`PublicHost` in Go, `pasv_url` in Node) so the PASV response advertises a *reachable* address.

> **Gotcha:** If a `lftp debug` or Wireshark trace shows the `227` response containing a `10.x`/`192.168.x`/`172.16–31.x` address while the client is on the public Internet, your public-host setting is wrong. This is the most common production FTP failure. (EPSV sidesteps it because it returns no IP.)

---

## 3. FTP vs FTPS vs SFTP

This distinction is the most important thing in the entire guide. Beginners constantly confuse the three, and choosing wrong is a real security incident. Memorize this table, then read the prose under it.

| Protocol | Transport / basis | Port(s) | Authentication | Encryption | One-line verdict |
|---|---|---|---|---|---|
| **FTP** | Raw TCP (RFC 959) | 21 control + dynamic data | Username/password **in cleartext** | **None** | Never over any untrusted network |
| **FTPS** | FTP **+ TLS** (RFC 4217) | 21 (explicit) or 990 (implicit) + data range | Username/password, TLS-protected | TLS/SSL (same as HTTPS) | FTP-compatible *and* encrypted |
| **SFTP** | A subsystem of **SSH** | 22 only | SSH key or password, SSH-protected | SSH (end-to-end) | Different protocol; usually the best choice |

### 3.1 Plain FTP — the cleartext warning **[B]**

> **⚠ WARNING:** Plain FTP sends your **username, password, and every byte of every file in cleartext**. Anyone able to observe the traffic — someone on the same Wi-Fi/LAN, a compromised router, an ISP, a malicious hop — can read all of it, credentials included. **Never run plain FTP over the public Internet or any network you do not fully control.**

Plain FTP is acceptable only in: a fully isolated, audited private LAN; or a throwaway dev/lab environment with no real credentials or data. In every other case use FTPS or SFTP.

### 3.2 FTPS — FTP wrapped in TLS **[I]**

FTPS adds TLS (the same encryption as HTTPS) around FTP's control *and* data channels. It keeps FTP's command set and two-channel model, so it remains compatible with FTP tooling and your mental model — it just encrypts the wire. Two flavours:

- **Explicit FTPS (FTPES / `AUTH TLS`)** — the client connects normally on port 21, then issues `AUTH TLS` to upgrade the connection to TLS before logging in. This is the modern default. The data channel is also protected once the client sends `PBSZ 0` and `PROT P`.
- **Implicit FTPS** — the connection is TLS from the first byte, on a dedicated port **990**. Older and less common; avoid for new deployments.

FTPS needs a TLS certificate (self-signed for testing; Let's Encrypt/your CA for production). Its downside versus SFTP is that you *still* have the two-channel passive-port/NAT complexity — now with the added subtlety that some NAT helpers cannot inspect the encrypted control channel to fix up `PORT`/`PASV`, making correct `PublicHost`/`pasv_url` configuration even more important.

### 3.3 SFTP — SSH File Transfer Protocol **[I]**

The name causes endless confusion, so be precise: **SFTP is not "FTP over SSH" and shares no code or commands with FTP.** It is an entirely separate protocol that runs as a *subsystem of SSH* on port 22. There is no control/data split, no `PASV`, no passive port range, no `PORT` command — a single encrypted SSH connection carries everything.

**Prefer SFTP whenever you control the client.** It is simpler to operate (one port), more secure (strong public-key auth, encrypted by construction), firewall-friendly (no data-port range), and already built into OpenSSH on every Linux/macOS box. The main reasons to use FTPS instead are hard compatibility constraints: a partner or device that only speaks the FTP command set.

> **Decision guide:** New system, you control the client → **SFTP**. Must speak the FTP protocol (legacy partner/device) → **FTPS, mandatory TLS**. Internet-facing → **never plain FTP**. Fully isolated audited LAN → plain FTP is *technically* fine but still discouraged.

---

## 4. FTP Commands & Response Codes — Reference

You do not memorize these; you skim them now and return when reading a trace or implementing a server. The libraries handle the wire format for you, but knowing the commands lets you debug with `lftp debug` or Wireshark.

### 4.1 Client-to-server commands **[B/I]**

| Command | Arguments | What it does |
|---|---|---|
| `USER` | username | Begin authentication with a username |
| `PASS` | password | Supply the password |
| `ACCT` | account | Account info (rarely used) |
| `QUIT` | — | End the session gracefully |
| `PWD` / `XPWD` | — | Print working directory |
| `CWD` | path | Change working directory |
| `CDUP` | — | Change to parent directory |
| `LIST` | [path] | Human-readable listing (like `ls -l`) — uses data channel |
| `NLST` | [path] | Bare names only — uses data channel |
| `MLSD` | [path] | Machine-parseable listing (RFC 3659) — preferred by modern clients |
| `MLST` | [path] | Machine-parseable facts for one path |
| `RETR` | filename | Download a file — uses data channel |
| `STOR` | filename | Upload a file — uses data channel |
| `STOU` | — | Upload with a unique server-chosen name |
| `APPE` | filename | Append to a file (upload) |
| `REST` | offset | Restart a transfer at a byte offset (resume) |
| `DELE` | filename | Delete a file |
| `MKD` / `XMKD` | dirname | Create a directory |
| `RMD` / `XRMD` | dirname | Remove a directory |
| `RNFR` | oldname | Rename: specify source (step 1) |
| `RNTO` | newname | Rename: specify destination (step 2) |
| `TYPE` | `A` or `I` | Set transfer type: ASCII or Image(binary) |
| `MODE` | `S` | Transfer mode (Stream is universal) |
| `STRU` | `F` | File structure (File is universal) |
| `PASV` | — | Enter passive mode (IPv4) |
| `EPSV` | [proto] | Enter extended passive mode (IPv4/IPv6) |
| `PORT` | h1,..,p2 | Enter active mode (IPv4) |
| `EPRT` | proto/addr/port | Extended active mode (IPv4/IPv6) |
| `SYST` | — | Report the server's OS family |
| `FEAT` | — | List supported extensions (UTF8, MLST, AUTH, REST, etc.) |
| `OPTS` | feature value | Set an option, e.g. `OPTS UTF8 ON` |
| `AUTH` | `TLS` | Upgrade the control channel to TLS (FTPS) |
| `PBSZ` | `0` | Protection buffer size (must precede `PROT` in FTPS) |
| `PROT` | `P` / `C` | Data-channel protection: `P`=Private(TLS), `C`=Clear |
| `SIZE` | filename | File size in bytes (RFC 3659) |
| `MDTM` | filename | Last-modified time (RFC 3659) |
| `NOOP` | — | No-op keep-alive (server replies `200`) |
| `STAT` | [path] | Status/listing over the control channel |
| `ABOR` | — | Abort the in-progress transfer |

### 4.2 Response-code structure **[B/I]**

Every response is a three-digit code plus text. The **first digit** is the outcome category and the **second digit** narrows the subject; you can react correctly knowing only the first digit:

| First digit | Category | Meaning for the client |
|---|---|---|
| **1xx** | Positive Preliminary | Action started; another reply is coming (e.g. watch the data channel) |
| **2xx** | Positive Completion | Done successfully |
| **3xx** | Positive Intermediate | Accepted so far; send the next piece (e.g. password after username) |
| **4xx** | Transient Negative | Failed *temporarily*; retrying later may work |
| **5xx** | Permanent Negative | Failed; do not retry without changing the request |

| Second digit | Subject |
|---|---|
| x0x | Syntax |
| x1x | Information (help, status) |
| x2x | Connections (control/data channel) |
| x3x | Authentication/accounts |
| x5x | File system |

### 4.3 Common specific codes **[B/I]**

| Code | Meaning |
|---|---|
| `150` | File status OK; about to open the data connection |
| `200` | Command OK |
| `211/213` | System status / file status (e.g. `SIZE` reply) |
| `220` | Service ready — the welcome banner |
| `221` | Goodbye (closing control connection) |
| `226` | Data connection closed; transfer succeeded |
| `227` | Entering Passive Mode `(h1,h2,h3,h4,p1,p2)` |
| `229` | Entering Extended Passive Mode `(\|\|\|port\|)` |
| `230` | User logged in |
| `234` | `AUTH TLS` accepted; begin TLS handshake |
| `250` | Requested file action completed (e.g. `CWD` OK) |
| `257` | Path created / `PWD` reply `"/path"` |
| `331` | Username OK, password required |
| `332` | Need account for login |
| `350` | Pending further info (`RNFR` got, send `RNTO`; `REST` got, send `RETR`) |
| `421` | Service unavailable; closing control connection (e.g. idle timeout) |
| `425` | Cannot open data connection (firewall/passive-port problem) |
| `426` | Connection closed; transfer aborted |
| `430` | Invalid username or password |
| `450` | File action not taken; file busy/unavailable |
| `451` | Local processing error |
| `500/501` | Syntax error / invalid parameters |
| `502` | Command not implemented |
| `503` | Bad command sequence (e.g. `PASS` before `USER`) |
| `530` | Not logged in |
| `534` | Request denied for policy reasons (often "TLS required") |
| `550` | File unavailable — not found, or no permission |
| `552` | Storage allocation exceeded (quota) |
| `553` | File name not allowed |

> **Debugging tip:** `425` and `426` almost always mean *data channel / passive port / firewall*, not auth. `530`/`534` mean *auth or TLS policy*. `550` means *path or permission*. Reading the first failing code immediately narrows the cause.

---

## 5. PART 1 — FTP Server in Go (ftpserverlib)

### 5.1 Library choice and why **[I]**

The Go ecosystem has two notable FTP-server libraries:

| Library | Import path | Status (2026) |
|---|---|---|
| **ftpserverlib** | `github.com/fclairamb/ftpserverlib` | Actively maintained; clean *driver* interface; FTPS, EPSV, per-user chroot, pluggable filesystem |
| goftp/server | `github.com/goftp/server` | Older, less active |

We use **`ftpserverlib`** because it cleanly separates the *protocol engine* (which you never touch) from your *policy* — authentication and which filesystem each user sees — behind a small **driver interface**. It also delegates the actual filesystem to **`afero`** (`github.com/spf13/afero`), an abstraction that lets you back a user with the real OS filesystem, an in-memory filesystem, a chrooted subtree, or your own implementation, without the engine knowing the difference. That separation is exactly what you want for testability and for per-user isolation (§10).

> **⚡ Version note:** `ftpserverlib`'s driver interface has been revised across minor versions (notably around v0.21). In some versions TLS is provided by a `GetTLSConfig()` method on the driver; in others a `*tls.Config` is passed directly in `Settings`. The examples below show both wiring styles and call out which is which — if the compiler complains about a missing/extra method, open the library's `driver.go`/`CHANGELOG.md` for your exact version. The *concepts* (settings, auth, per-user FS) never change.

### 5.2 Project setup **[I]**

```bash
mkdir go-ftp-server && cd go-ftp-server
go mod init github.com/yourname/go-ftp-server

# The server engine + the filesystem abstraction it uses.
go get github.com/fclairamb/ftpserverlib@latest
go get github.com/spf13/afero@latest
```

### 5.3 The driver model — the mental picture **[I]**

You implement two interfaces. The engine calls into them; you never call the engine except to start it.

```
ftpserverlib engine  (handles sockets, PASV/EPSV, the command loop, TLS)
        │
        ├──► MainDriver.GetSettings()            → ports, public host, passive range, TLS, banner
        ├──► MainDriver.ClientConnected(cc)       → fires on TCP connect (pre-auth) — gate/log/rate-limit here
        ├──► MainDriver.AuthUser(cc, user, pass)  → validate creds; on success return a ClientDriver
        ├──► MainDriver.ClientDisconnected(cc)    → fires on disconnect — clean up / audit
        └──► ClientDriver  (one per authenticated session)
                └──► GetFS() afero.Fs            → the (ideally chrooted) filesystem this user may touch
```

The genius of this design is that **isolation is just "return the right afero.Fs."** To jail a user to their own folder you hand back an `afero.NewBasePathFs` rooted at their directory; the engine can only ever see paths inside it.

### 5.4 Complete minimal working server **[I]**

This runs on port **2121** (a non-privileged port, so no root needed), with two users, a passive range of 50000–50100, and per-user chrooted folders under `./ftp-root`. Read the comments — they explain *why*, not just *what*.

```go
// main.go — a minimal but correct multi-user FTP server.
package main

import (
	"errors"
	"fmt"
	"log"
	"os"

	ftpserver "github.com/fclairamb/ftpserverlib"
	"github.com/spf13/afero"
)

// ─────────────────────────────────────────────────────────────
// 1.  User "database". In real systems this is a DB or directory
//     service, and passwords are HASHED (see §15), never plaintext.
// ─────────────────────────────────────────────────────────────

type User struct {
	Password string // DEMO ONLY — store a bcrypt/argon2 hash in production.
	RootDir  string // The folder this user is chrooted into.
}

var users = map[string]User{
	"alice": {Password: "s3cr3t", RootDir: "./ftp-root/alice"},
	"bob":   {Password: "hunter2", RootDir: "./ftp-root/bob"},
}

// ─────────────────────────────────────────────────────────────
// 2.  ClientDriver — created per authenticated session. Its only
//     job is to expose the filesystem the engine may operate on.
// ─────────────────────────────────────────────────────────────

type ClientDriver struct {
	fs afero.Fs
}

// GetFS returns the per-user filesystem. Because we hand back a
// BasePathFs (created in AuthUser), the engine physically cannot
// resolve a path outside this user's RootDir — that IS the jail.
func (d *ClientDriver) GetFS(_ ftpserver.ClientContext) (afero.Fs, error) {
	return d.fs, nil
}

// ─────────────────────────────────────────────────────────────
// 3.  MainDriver — global server policy.
// ─────────────────────────────────────────────────────────────

type MainDriver struct{}

// GetSettings is called once at startup to configure the engine.
func (d *MainDriver) GetSettings() (*ftpserver.Settings, error) {
	return &ftpserver.Settings{
		// "host:port" to bind. Use ":21" in production (needs root or
		// the cap_net_bind_service capability). 2121 runs unprivileged.
		ListenAddr: "0.0.0.0:2121",

		// The address the server PUTS INTO the PASV/227 response so the
		// client knows where to open the data connection. MUST be an IP
		// the client can reach: 127.0.0.1 for localhost tests, your LAN
		// IP on a LAN, your PUBLIC IP for an Internet-facing server. A
		// wrong value here is the #1 production FTP failure (see §2.4).
		PublicHost: "127.0.0.1",

		// The inbound passive data-port range. This MUST match what you
		// open in the firewall and (in Docker) what you publish.
		PassiveTransferPortRange: &ftpserver.PortRange{
			Start: 50000,
			End:   50100,
		},

		// 220 banner shown on connect. Keep it generic — do not leak the
		// software name/version (helps attackers fingerprint you).
		Banner: "Service ready.",

		// Disconnect idle sessions after N seconds (0 = never). Reaping
		// abandoned sessions frees passive ports and limits exposure.
		IdleTimeout: 900,

		// Default to binary so we never corrupt non-text files (§1.4).
		DefaultTransferType: ftpserver.TransferTypeBinary,
	}, nil
}

// ClientConnected fires on a NEW TCP connection, BEFORE login. Return
// an error to reject the connection outright — this is the right place
// for IP allow/deny lists and connection rate limiting (§15).
func (d *MainDriver) ClientConnected(cc ftpserver.ClientContext) (string, error) {
	log.Printf("connect id=%d remote=%s", cc.ID(), cc.RemoteAddr())
	return "Service ready.", nil
}

// ClientDisconnected fires when a client goes away — audit/cleanup here.
func (d *MainDriver) ClientDisconnected(cc ftpserver.ClientContext) {
	log.Printf("disconnect id=%d", cc.ID())
}

// AuthUser validates USER+PASS. On success it returns the ClientDriver
// that exposes the user's chrooted filesystem; on failure it returns a
// GENERIC error (never reveal whether the username exists — that helps
// account enumeration).
func (d *MainDriver) AuthUser(cc ftpserver.ClientContext, username, password string) (ftpserver.ClientDriver, error) {
	user, ok := users[username]
	// Compare even when the user is missing to keep timing uniform.
	if !ok || user.Password != password {
		log.Printf("auth FAILED user=%q remote=%s", username, cc.RemoteAddr())
		return nil, errors.New("authentication failed")
	}

	// Ensure the user's root exists with restrictive perms (owner rwx,
	// group r-x, others none).
	if err := os.MkdirAll(user.RootDir, 0o750); err != nil {
		return nil, fmt.Errorf("preparing user root: %w", err)
	}

	log.Printf("auth OK user=%q id=%d", username, cc.ID())

	// BasePathFs is the chroot: every path the engine resolves is
	// rewritten to live under user.RootDir, so ".." can never escape.
	jailed := afero.NewBasePathFs(afero.NewOsFs(), user.RootDir)
	return &ClientDriver{fs: jailed}, nil
}

// GetTLSConfig — return nil for plain FTP. See §6 to enable FTPS.
// (In versions that take TLS via Settings instead, omit this method.)
func (d *MainDriver) GetTLSConfig() (*tls.Config, error) {
	return nil, nil
}

// ─────────────────────────────────────────────────────────────
// 4.  Wire it up and run.
// ─────────────────────────────────────────────────────────────

func main() {
	// Pre-create user roots so first login does not race on MkdirAll.
	for _, u := range users {
		if err := os.MkdirAll(u.RootDir, 0o750); err != nil {
			log.Fatalf("cannot create %s: %v", u.RootDir, err)
		}
	}

	server := ftpserver.NewFtpServer(&MainDriver{})

	log.Println("FTP server starting on :2121 (passive 50000-50100)")
	// ListenAndServe blocks until the server stops or errors.
	if err := server.ListenAndServe(); err != nil {
		log.Fatalf("server error: %v", err)
	}
}
```

> **⚡ Version note:** The `GetTLSConfig` method (and the exact `GetFS` signature — some versions are `GetFS() afero.Fs`, others `GetFS(ClientContext) (afero.Fs, error)`) varies by release. If the build fails on these, adjust the signature to match your installed version; the logic inside is identical. To find the truth, run `go doc github.com/fclairamb/ftpserverlib.ClientDriver` and `... .MainDriver`.

### 5.5 Running it **[I]**

```bash
mkdir -p ftp-root/alice ftp-root/bob
echo "Hello from alice" > ftp-root/alice/hello.txt
echo "Bob's data"       > ftp-root/bob/data.csv

go run main.go
# FTP server starting on :2121 (passive 50000-50100)
```

Test it with `lftp`, FileZilla, or the smoke-test script in §11.

### 5.6 Swapping the backend: in-memory filesystem **[I/A]**

Because the user is just an `afero.Fs`, you can hand back an **in-memory** filesystem instead of the OS one — nothing ever touches disk. This is perfect for integration tests (fast, isolated, no cleanup) and for sandboxed/ephemeral upload endpoints.

```go
// Build an in-memory FS, pre-populate it, and jail to it. When the
// process exits, everything is gone — ideal for tests and sandboxes.
mem := afero.NewMemMapFs()
_ = mem.MkdirAll("/uploads", 0o755)
_ = afero.WriteFile(mem, "/welcome.txt", []byte("Hello!\n"), 0o644)

jailed := afero.NewBasePathFs(mem, "/")
return &ClientDriver{fs: jailed}, nil
```

You can go further and implement the `afero.Fs` interface yourself to add quotas, audit logging on every `OpenFile`, virus scanning hooks, or an S3/database backend — the engine cannot tell the difference (§10).

---

## 6. Adding TLS (FTPS) in Go

FTPS = the §5 server, plus a TLS certificate and a couple of settings. The engine handles the `AUTH TLS`/`PBSZ`/`PROT` dance; you supply the cert and declare your policy.

### 6.1 Generate a certificate **[I]**

```bash
# DEVELOPMENT ONLY — a self-signed cert. Clients will warn; you accept it.
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt \
  -days 365 -nodes -subj "/CN=localhost"
```

For production use a CA-issued certificate — Let's Encrypt is free and auto-renewable (see §15.2). Self-signed certs train users to click through warnings, which is itself a security risk.

### 6.2 Provide the TLS config and require encryption **[I/A]**

```go
import (
	"crypto/tls"
	// ...existing imports
)

// GetTLSConfig is called by the engine when a client requests AUTH TLS.
// It loads the keypair and returns a hardened *tls.Config.
func (d *MainDriver) GetTLSConfig() (*tls.Config, error) {
	cert, err := tls.LoadX509KeyPair("server.crt", "server.key")
	if err != nil {
		return nil, fmt.Errorf("loading TLS keypair: %w", err)
	}
	return &tls.Config{
		Certificates: []tls.Certificate{cert},
		// Refuse the long-broken TLS 1.0/1.1. 1.2 is the safe floor;
		// allow 1.3 (the default MaxVersion) for the best clients.
		MinVersion: tls.VersionTLS12,
	}, nil
}
```

Then in `GetSettings`, enable TLS and decide whether it is *mandatory*. Mandatory is what you want on the Internet — it refuses any client that tries to log in without first upgrading to TLS, so credentials can never cross the wire in clear.

```go
func (d *MainDriver) GetSettings() (*ftpserver.Settings, error) {
	tlsCfg, err := d.GetTLSConfig()
	if err != nil {
		return nil, err
	}
	return &ftpserver.Settings{
		ListenAddr: "0.0.0.0:2121",
		PublicHost: "127.0.0.1",
		PassiveTransferPortRange: &ftpserver.PortRange{Start: 50000, End: 50100},
		Banner:     "Service ready.",

		// Some versions accept the TLS config here in Settings instead of
		// (or in addition to) the GetTLSConfig method — see the version note.
		TLSConfig: tlsCfg,

		// MandatoryEncryption: clients MUST do AUTH TLS before login.
		// Use ClearOrEncrypted only during a migration window when you
		// still have legacy plaintext clients to move off.
		TLSRequired: ftpserver.MandatoryEncryption,
	}, nil
}
```

> **⚡ Version note:** Depending on your `ftpserverlib` version, TLS is wired either via the `GetTLSConfig()` driver method *or* the `TLSConfig` field in `Settings` (newer releases). Use whichever your version exposes — do not set both unless the docs say to. The `TLSRequired` constant names (`MandatoryEncryption`, `ClearOrEncrypted`, `ImplicitEncryption`) are also version-sensitive.

> **Testing FTPS:** The built-in `ftp` client does **not** support TLS. Use `lftp` or FileZilla. With `lftp` and a self-signed cert: `lftp -e "set ssl:verify-certificate false; set ftp:ssl-force true; set ftp:ssl-protect-data true; open ftp://alice:s3cr3t@localhost:2121"`. Disable `verify-certificate` *only* for dev self-signed certs — never in production.

---

## 7. PART 2 — FTP Server in Node.js (ftp-srv)

### 7.1 Library choice **[I]**

**`ftp-srv`** is the most actively maintained Node FTP server (2024–2026). It gives you PASV/EPSV, explicit FTPS (`AUTH TLS`), per-connection filesystem roots, a promise-based `login` event for auth, and a pluggable `FileSystem` class for virtual backends. Its model is event-driven and idiomatic Node: you create a server, listen for `login`, and resolve/reject.

> **⚡ Version note:** Examples target **`ftp-srv` v5.x**. v4.x had slightly different option and event names. Confirm with `npm show ftp-srv version` after installing, and skim the README — the `login` resolution shape (`{ root }` vs `{ fs }`) and the `tls` option are the things most likely to have shifted.

### 7.2 Project setup **[I]**

```bash
mkdir node-ftp-server && cd node-ftp-server
npm init -y
npm install ftp-srv
```

### 7.3 Complete minimal working server **[I]**

```javascript
// server.js — minimal multi-user FTP server with per-user chroot.
'use strict';

const FtpSrv = require('ftp-srv');
const path = require('path');
const fs = require('fs');

// ── 1. User "database". Production: a real store, with HASHED passwords. ──
const USERS = {
  alice: { password: 's3cr3t', root: path.resolve('./ftp-root/alice') },
  bob:   { password: 'hunter2', root: path.resolve('./ftp-root/bob') },
};
// Make sure each user's root exists before anyone logs in.
Object.values(USERS).forEach((u) => fs.mkdirSync(u.root, { recursive: true }));

// ── 2. Create the server. ──
const ftpServer = new FtpSrv({
  // The control-channel bind URL. 'ftp://0.0.0.0:21' in production needs
  // root/cap_net_bind_service; 2121 runs unprivileged.
  url: 'ftp://0.0.0.0:2121',

  // The IP advertised to clients in the PASV (227) response. MUST be
  // reachable by the client: 127.0.0.1 locally, your PUBLIC IP on the
  // Internet. Wrong value here = transfers hang for external clients.
  pasv_url: '127.0.0.1',

  // The passive data-port range — must match firewall/Docker publishing.
  pasv_min: 50000,
  pasv_max: 50100,

  // The 220 banner. Keep it generic; do not leak software/version.
  greeting: ['Service ready.'],

  // NEVER true in production unless this is a deliberate public, sensitive-
  // data-free share. Anonymous = no credentials required at all.
  anonymous: false,

  // Optional: a pino-style logger. Omit for default.
  // log: require('pino')({ level: 'info' }),
});

// ── 3. Authentication: the 'login' event fires per USER+PASS attempt. ──
// resolve({ root }) grants access and chroots to that folder;
// reject(err) denies. ftp-srv jails the session to `root` automatically.
ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];
  if (!user || user.password !== password) {
    // Generic message — do not reveal whether the username exists.
    return reject(new Error('Authentication failed'));
  }
  console.log(`[FTP] login ok user=${username} ip=${connection.ip}`);
  return resolve({ root: user.root });
});

// ── 4. Per-connection error events (bad TLS handshake, abrupt drops). ──
ftpServer.on('client-error', ({ context, error }) => {
  console.error(`[FTP] client-error context=${context}: ${error.message}`);
});

// ── 5. Start, with graceful shutdown. ──
ftpServer
  .listen()
  .then(() => console.log('[FTP] listening on 2121, passive 50000-50100'))
  .catch((err) => {
    console.error('[FTP] failed to start:', err);
    process.exit(1);
  });

const shutdown = () => ftpServer.close().then(() => process.exit(0));
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);
```

### 7.4 Running it **[I]**

```bash
mkdir -p ftp-root/alice ftp-root/bob
echo "Hello from alice" > ftp-root/alice/hello.txt
node server.js
# [FTP] listening on 2121, passive 50000-50100
```

### 7.5 Custom virtual filesystem per connection **[A]**

Instead of a plain `{ root }`, you can resolve with `{ fs }` — your own subclass of `ftp-srv`'s `FileSystem`. This lets you add audit logging, per-user quotas, upload hooks (antivirus, webhooks), or a non-disk backend. Every protocol operation routes through your overridden methods.

```javascript
const { FileSystem } = require('ftp-srv');

// Logs every upload; otherwise behaves exactly like the default FS.
class AuditedFileSystem extends FileSystem {
  // write() is invoked on STOR (upload). Returning the parent stream keeps
  // normal behaviour while letting us observe/meter it.
  write(fileName, options = {}) {
    console.log(`[AUDIT] upload root=${this.root} file=${fileName}`);
    return super.write(fileName, options);
  }
  // You could likewise override read (RETR), list, delete, mkdir, rename,
  // chmod, get/stat — e.g. to enforce a quota before allowing write().
}

ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];
  if (!user || user.password !== password) {
    return reject(new Error('Authentication failed'));
  }
  // Pass an FS INSTANCE instead of a path string.
  const customFs = new AuditedFileSystem(connection, { root: user.root, cwd: '/' });
  return resolve({ fs: customFs });
});
```

---

## 8. Adding TLS (FTPS) in Node.js

`ftp-srv` enables explicit FTPS simply by passing a standard Node `tls` options object. The library then advertises `AUTH TLS` in `FEAT` and upgrades connections on request.

### 8.1 Certificate **[I]**

```bash
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt \
  -days 365 -nodes -subj "/CN=localhost"
```

### 8.2 Code **[I/A]**

```javascript
// server-tls.js — FTPS (explicit AUTH TLS).
'use strict';

const FtpSrv = require('ftp-srv');
const path = require('path');
const fs = require('fs');

const USERS = {
  alice: { password: 's3cr3t', root: path.resolve('./ftp-root/alice') },
};
Object.values(USERS).forEach((u) => fs.mkdirSync(u.root, { recursive: true }));

// Standard Node TLS options. ftp-srv passes these straight to tls.
const tlsOptions = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),
  // TLS 1.2 is the safe floor; Node negotiates 1.3 when both sides support it.
  minVersion: 'TLSv1.2',
  // honorCipherOrder lets the server pick strong ciphers over client prefs.
  honorCipherOrder: true,
};

const ftpServer = new FtpSrv({
  url: 'ftp://0.0.0.0:2121',
  pasv_url: '127.0.0.1',
  pasv_min: 50000,
  pasv_max: 50100,
  greeting: 'Service ready.',
  anonymous: false,

  // Passing `tls` turns on explicit FTPS (AUTH TLS on the same port).
  // For IMPLICIT FTPS you would instead listen with an ftps:// url on 990.
  tls: tlsOptions,
});

ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];
  if (!user || user.password !== password) {
    return reject(new Error('Authentication failed'));
  }
  console.log(`[FTPS] login ok user=${username} ip=${connection.ip}`);
  return resolve({ root: user.root });
});

ftpServer
  .listen()
  .then(() => console.log('[FTPS] listening on 2121 (explicit TLS)'));
```

> **Enforcing TLS:** To *require* encryption (refuse cleartext logins), reject any `login` where the connection is not yet secured. `ftp-srv` exposes the secure state on the connection (e.g. `connection.secure`); check it in the `login` handler and `reject(new Error('TLS required'))` if false. This is the Node equivalent of Go's `MandatoryEncryption`. Always do this on Internet-facing servers.

> **Testing:** `lftp -e "set ssl:verify-certificate false; set ftp:ssl-force true; open ftp://alice:s3cr3t@localhost:2121"`, or FileZilla with "Require explicit FTP over TLS." Accept the self-signed warning in dev only.

---

## 9. PART 3 — SFTP Servers (Go & Node)

SFTP is usually the better choice (§3.3): one encrypted port, strong key auth, no passive-range headaches, built into OpenSSH. The skeletons below teach how the SSH+SFTP layering works and are useful for embedded/custom scenarios. **For most production needs, configuring OpenSSH's built-in SFTP subsystem (with `ChrootDirectory` and an `sftp`-only group) is simpler and more battle-tested than hand-rolling a server** — but knowing the internals makes you able to debug and customize either path.

How SFTP layers on SSH, conceptually:

```
TCP :22  →  SSH transport (encryption, host-key verification)
          →  SSH authentication (password or, better, public key)
          →  SSH "session" channel
          →  "subsystem" request == "sftp"  →  SFTP protocol (OPEN/READ/WRITE/READDIR/...)
```

### 9.1 SFTP server in Go (`x/crypto/ssh` + `pkg/sftp`) **[A]**

`pkg/sftp` implements the SFTP subsystem on top of an SSH channel from `golang.org/x/crypto/ssh`. The library even gives you a one-call server that serves a directory — and `sftp.WithServerWorkingDirectory` plus a chrooted handler does the user jailing.

```bash
go get golang.org/x/crypto/ssh@latest
go get github.com/pkg/sftp@latest
```

```go
// sftp_server.go — minimal SFTP server with password auth and per-user roots.
package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"os"

	"github.com/pkg/sftp"
	"golang.org/x/crypto/ssh"
)

// Demo creds — production stores HASHES and strongly prefers public keys.
var validUsers = map[string]string{
	"alice": "s3cr3t",
	"bob":   "hunter2",
}

func main() {
	// ── 1. Host key. Generate ONCE and persist; clients pin it on first
	//      connect (trust-on-first-use), so a changing host key triggers
	//      a scary MITM warning. Never regenerate it in production. ──
	hostKeyBytes, err := os.ReadFile("host_key")
	if err != nil {
		log.Fatalf("read host key (run: ssh-keygen -t ed25519 -f host_key -N \"\"): %v", err)
	}
	hostKey, err := ssh.ParsePrivateKey(hostKeyBytes)
	if err != nil {
		log.Fatalf("parse host key: %v", err)
	}

	// ── 2. SSH server config: how to authenticate. ──
	sshConfig := &ssh.ServerConfig{
		// PasswordCallback returns nil to allow, an error to deny.
		PasswordCallback: func(c ssh.ConnMetadata, pass []byte) (*ssh.Permissions, error) {
			if want, ok := validUsers[c.User()]; ok && want == string(pass) {
				log.Printf("[SFTP] auth ok user=%s remote=%s", c.User(), c.RemoteAddr())
				// Stash the username so the session handler can find the root.
				return &ssh.Permissions{Extensions: map[string]string{"user": c.User()}}, nil
			}
			return nil, fmt.Errorf("auth failed for %q", c.User())
		},
		// In production, ALSO/INSTEAD implement PublicKeyCallback and prefer
		// keys over passwords. MaxAuthTries limits brute force.
		MaxAuthTries: 3,
	}
	sshConfig.AddHostKey(hostKey)

	// ── 3. Accept TCP connections; one goroutine per connection. ──
	ln, err := net.Listen("tcp", "0.0.0.0:2222")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	log.Println("[SFTP] listening on :2222")

	for {
		conn, err := ln.Accept()
		if err != nil {
			log.Printf("accept: %v", err)
			continue
		}
		go handleConn(conn, sshConfig)
	}
}

func handleConn(conn net.Conn, cfg *ssh.ServerConfig) {
	defer conn.Close()

	// SSH handshake + authentication happen here.
	sshConn, chans, reqs, err := ssh.NewServerConn(conn, cfg)
	if err != nil {
		log.Printf("[SFTP] handshake failed: %v", err)
		return
	}
	defer sshConn.Close()
	log.Printf("[SFTP] connected remote=%s client=%s", sshConn.RemoteAddr(), sshConn.ClientVersion())

	// Drain global out-of-band requests (keep-alives, etc.).
	go ssh.DiscardRequests(reqs)

	// Each "session" channel can carry one SFTP subsystem.
	for newChan := range chans {
		if newChan.ChannelType() != "session" {
			_ = newChan.Reject(ssh.UnknownChannelType, "only session channels")
			continue
		}
		go handleSession(newChan, sshConn)
	}
}

func handleSession(newChan ssh.NewChannel, sshConn *ssh.ServerConn) {
	channel, requests, err := newChan.Accept()
	if err != nil {
		log.Printf("[SFTP] accept channel: %v", err)
		return
	}
	defer channel.Close()

	for req := range requests {
		// We only honour the "sftp" subsystem. We deliberately do NOT honour
		// "exec"/"shell"/"pty" — this is a file server, not a shell account.
		ok := false
		if req.Type == "subsystem" && len(req.Payload) >= 4 && string(req.Payload[4:]) == "sftp" {
			ok = true
		}
		_ = req.Reply(ok, nil)
		if !ok {
			continue
		}

		// Resolve and create this user's chroot root.
		user := sshConn.Permissions.Extensions["user"]
		root := fmt.Sprintf("./sftp-root/%s", user)
		_ = os.MkdirAll(root, 0o750)

		// WithServerWorkingDirectory both sets the start dir AND restricts
		// path resolution to it — this is the user jail. (For a stricter
		// jail you can also pass a custom Handlers implementation.)
		srv, err := sftp.NewServer(channel, sftp.WithServerWorkingDirectory(root))
		if err != nil {
			log.Printf("[SFTP] new server: %v", err)
			return
		}
		// Serve blocks until the client disconnects.
		if err := srv.Serve(); err != nil && err != io.EOF {
			log.Printf("[SFTP] serve: %v", err)
		}
		return
	}
}
```

```bash
ssh-keygen -t ed25519 -f host_key -N ""     # generate the host key ONCE
mkdir -p sftp-root/alice
echo "Alice's file" > sftp-root/alice/hello.txt
go run sftp_server.go

# Test:
sftp -P 2222 alice@localhost   # password: s3cr3t
# sftp> ls
# sftp> get hello.txt
```

> **Why Go is the stronger choice for a hand-rolled SFTP server:** `golang.org/x/crypto/ssh` is a mature, widely deployed SSH implementation, and `pkg/sftp` gives you a complete subsystem server in a couple of calls. The Node path (below) requires you to wire up the SFTP operations yourself.

### 9.2 SFTP server in Go with public-key auth **[A]**

Public keys beat passwords for SFTP: no shared secret crosses the wire, and you can revoke per-key. Replace/augment the password callback with a `PublicKeyCallback` that checks the offered key against an `authorized_keys`-style allowlist.

```go
// authorizedKeys maps username -> set of accepted public keys (by marshalled form).
// Load these from each user's authorized_keys file at startup.
func buildAuthorizedKeys() map[string]map[string]bool {
	out := map[string]map[string]bool{}
	for user := range validUsers {
		data, err := os.ReadFile(fmt.Sprintf("./keys/%s.pub", user))
		if err != nil {
			continue // user has no key configured
		}
		set := map[string]bool{}
		rest := data
		for len(rest) > 0 {
			pk, _, _, r, err := ssh.ParseAuthorizedKey(rest)
			if err != nil {
				break
			}
			set[string(pk.Marshal())] = true // index by exact wire bytes
			rest = r
		}
		out[user] = set
	}
	return out
}

// Wire it into the ServerConfig (alongside or instead of PasswordCallback):
//
//	authKeys := buildAuthorizedKeys()
//	sshConfig.PublicKeyCallback = func(c ssh.ConnMetadata, key ssh.PublicKey) (*ssh.Permissions, error) {
//		if set := authKeys[c.User()]; set != nil && set[string(key.Marshal())] {
//			return &ssh.Permissions{Extensions: map[string]string{"user": c.User()}}, nil
//		}
//		return nil, fmt.Errorf("unauthorized key for %q", c.User())
//	}
```

### 9.3 SFTP server in Node (`ssh2`) **[A]**

`ssh2` provides both an SSH client and server plus an SFTP subsystem. Unlike Go's `pkg/sftp`, you handle each SFTP request (`OPEN`, `READ`, `WRITE`, `READDIR`, ...) yourself — which is more code but shows the protocol in full.

```bash
npm install ssh2
```

> **⚡ Version note:** Examples target **`ssh2` v1.x**, whose API differs significantly from v0.x (the `SFTP` handler events, `STATUS_CODE`, and key generation all changed). Re-check the `ssh2` README after installing.

```javascript
// sftp-server.js — SFTP server with password auth and per-user chroot.
'use strict';

const { Server, utils } = require('ssh2');
const path = require('path');
const fs = require('fs');

// ── Users. Production: hashed passwords and/or public keys. ──
const USERS = {
  alice: { password: 's3cr3t', root: path.resolve('./sftp-root/alice') },
  bob:   { password: 'hunter2', root: path.resolve('./sftp-root/bob') },
};
Object.values(USERS).forEach((u) => fs.mkdirSync(u.root, { recursive: true }));

// ── Host key: persist it (TOFU pinning). Generate ephemeral only in dev. ──
let hostKey;
if (fs.existsSync('host_key')) {
  hostKey = fs.readFileSync('host_key');
} else {
  const keys = utils.generateKeyPairSync('ed25519');
  fs.writeFileSync('host_key', keys.private, { mode: 0o600 });
  hostKey = keys.private;
}

// ── Per-session SFTP handler. `userRoot` is this user's jail. ──
function handleSftp(sftp, userRoot) {
  const { STATUS_CODE } = utils.sftp;

  // Translate a client path to a real path UNDER userRoot. We normalize to
  // strip ".." sequences so a client cannot escape the jail with traversal.
  const real = (p) => path.join(userRoot, path.normalize('/' + p));

  const openFiles = new Map(); // handle-id -> fd
  const openDirs = new Map(); // handle-id -> { entries, index, rp }
  let counter = 0;
  const newHandle = (store, value) => {
    const id = counter++;
    const h = Buffer.alloc(4);
    h.writeUInt32BE(id, 0);
    store.set(id, value);
    return h;
  };
  const idOf = (h) => h.readUInt32BE(0);

  // OPEN — open/create a file; reply with a handle.
  sftp.on('OPEN', (reqId, filename, flags) => {
    try {
      const fd = fs.openSync(real(filename), utils.sftp.flagsToString(flags));
      sftp.handle(reqId, newHandle(openFiles, fd));
    } catch {
      sftp.status(reqId, STATUS_CODE.FAILURE);
    }
  });

  // READ — read `length` bytes at `offset`.
  sftp.on('READ', (reqId, handle, offset, length) => {
    const fd = openFiles.get(idOf(handle));
    if (fd === undefined) return sftp.status(reqId, STATUS_CODE.FAILURE);
    const buf = Buffer.alloc(length);
    try {
      const n = fs.readSync(fd, buf, 0, length, offset);
      if (n === 0) return sftp.status(reqId, STATUS_CODE.EOF);
      sftp.data(reqId, buf.subarray(0, n));
    } catch {
      sftp.status(reqId, STATUS_CODE.FAILURE);
    }
  });

  // WRITE — upload bytes at `offset`.
  sftp.on('WRITE', (reqId, handle, offset, data) => {
    const fd = openFiles.get(idOf(handle));
    if (fd === undefined) return sftp.status(reqId, STATUS_CODE.FAILURE);
    try {
      fs.writeSync(fd, data, 0, data.length, offset);
      sftp.status(reqId, STATUS_CODE.OK);
    } catch {
      sftp.status(reqId, STATUS_CODE.FAILURE);
    }
  });

  // CLOSE — close a file or dir handle.
  sftp.on('CLOSE', (reqId, handle) => {
    const id = idOf(handle);
    if (openFiles.has(id)) {
      try { fs.closeSync(openFiles.get(id)); } catch {}
      openFiles.delete(id);
    }
    openDirs.delete(id);
    sftp.status(reqId, STATUS_CODE.OK);
  });

  // STAT / LSTAT — file metadata.
  const sendStat = (reqId, p) => {
    try {
      const s = fs.statSync(p);
      sftp.attrs(reqId, {
        mode: s.mode, uid: s.uid, gid: s.gid, size: s.size,
        atime: Math.floor(s.atimeMs / 1000),
        mtime: Math.floor(s.mtimeMs / 1000),
      });
    } catch {
      sftp.status(reqId, STATUS_CODE.NO_SUCH_FILE);
    }
  };
  sftp.on('STAT', (reqId, p) => sendStat(reqId, real(p)));
  sftp.on('LSTAT', (reqId, p) => sendStat(reqId, real(p)));

  // OPENDIR / READDIR — directory listing (paged).
  sftp.on('OPENDIR', (reqId, dir) => {
    const rp = real(dir);
    try {
      const entries = fs.readdirSync(rp, { withFileTypes: true });
      sftp.handle(reqId, newHandle(openDirs, { entries, index: 0, rp }));
    } catch {
      sftp.status(reqId, STATUS_CODE.NO_SUCH_FILE);
    }
  });
  sftp.on('READDIR', (reqId, handle) => {
    const d = openDirs.get(idOf(handle));
    if (!d || d.index >= d.entries.length) return sftp.status(reqId, STATUS_CODE.EOF);
    const batch = d.entries.slice(d.index, d.index + 20);
    d.index += batch.length;
    const names = batch.map((e) => {
      let attrs = { mode: 0o644, size: 0, uid: 0, gid: 0, atime: 0, mtime: 0 };
      try {
        const s = fs.statSync(path.join(d.rp, e.name));
        attrs = {
          mode: s.mode, size: s.size, uid: s.uid, gid: s.gid,
          atime: Math.floor(s.atimeMs / 1000), mtime: Math.floor(s.mtimeMs / 1000),
        };
      } catch {}
      return { filename: e.name, longname: e.name, attrs };
    });
    sftp.name(reqId, names);
  });

  // Mutations.
  sftp.on('REMOVE', (reqId, p) => {
    try { fs.unlinkSync(real(p)); sftp.status(reqId, STATUS_CODE.OK); }
    catch { sftp.status(reqId, STATUS_CODE.FAILURE); }
  });
  sftp.on('RMDIR', (reqId, p) => {
    try { fs.rmdirSync(real(p)); sftp.status(reqId, STATUS_CODE.OK); }
    catch { sftp.status(reqId, STATUS_CODE.FAILURE); }
  });
  sftp.on('MKDIR', (reqId, p) => {
    try { fs.mkdirSync(real(p), { recursive: true }); sftp.status(reqId, STATUS_CODE.OK); }
    catch { sftp.status(reqId, STATUS_CODE.FAILURE); }
  });
  sftp.on('RENAME', (reqId, oldP, newP) => {
    try { fs.renameSync(real(oldP), real(newP)); sftp.status(reqId, STATUS_CODE.OK); }
    catch { sftp.status(reqId, STATUS_CODE.FAILURE); }
  });
}

// ── SSH server. ──
const server = new Server({ hostKeys: [hostKey] }, (client) => {
  let authedUser = null; // remember who authenticated, to pick their root.

  client.on('authentication', (ctx) => {
    const user = USERS[ctx.username];
    if (ctx.method === 'password' && user && user.password === ctx.password) {
      authedUser = ctx.username;
      return ctx.accept();
    }
    // Tell the client which methods we support; reject this attempt.
    return ctx.reject(['password']);
  });

  client.on('ready', () => {
    client.on('session', (acceptSession) => {
      const session = acceptSession();
      session.on('sftp', (acceptSftp) => {
        const root = USERS[authedUser]?.root ?? path.resolve('./sftp-root/_default');
        handleSftp(acceptSftp(), root);
      });
    });
  });

  client.on('error', (err) => console.error('[SFTP] client error:', err.message));
  client.on('close', () => console.log('[SFTP] client disconnected'));
});

server.listen(2222, '0.0.0.0', () => {
  console.log('[SFTP] listening on 2222');
});
```

```bash
node sftp-server.js
sftp -P 2222 alice@localhost   # password: s3cr3t
```

> **Production recommendation:** For a real Node SFTP service, prefer wrapping OpenSSH (`Subsystem sftp internal-sftp`, `Match Group sftponly`, `ChrootDirectory`) over maintaining a hand-written request handler — it is far more thoroughly audited. Use these skeletons to *understand* the protocol and for niche embedded/virtual-backend cases.

---

## 10. Authentication & User Isolation (Chroot / Virtual FS)

Isolation is the heart of a safe file server: user A must never read, write, list, or even *learn the existence of* user B's files, and no user may escape to the host filesystem. There are two layers — **authentication** (who are you) and **authorization/jailing** (what can you touch).

### 10.1 Authentication options, weakest to strongest **[I]**

- **Static cleartext map (demo only).** What our minimal examples use. Never ship it — passwords in source/config are a breach waiting to happen.
- **Hashed passwords in a store.** Store a bcrypt/argon2 hash; verify with a constant-time comparison. Acceptable for FTPS; mandatory if you must use passwords (see §15.4 for code).
- **Public-key auth (SFTP).** No shared secret crosses the wire; revoke per key; the gold standard for automated SFTP. (§9.2.)
- **Mutual TLS / client certificates (FTPS).** Require *and verify* a client certificate in addition to a password — defense in depth for high-value FTPS endpoints (set `ClientAuth: tls.RequireAndVerifyClientCert` in the Go `tls.Config`).

Whichever you use, follow these rules: return a **generic** failure ("authentication failed") so attackers cannot enumerate valid usernames; keep failure timing uniform (compare even when the user is missing); cap attempts (`MaxAuthTries`/rate limiting); and log every failure with the source IP.

### 10.2 Jailing: chroot vs virtual filesystem **[I/A]**

A user must be confined to a subtree. There are two ways:

- **Real chroot / base-path jail.** Map the user onto a real directory such that paths can never resolve above it. In Go this is `afero.NewBasePathFs(osFs, "/srv/ftp/alice")`; in `ftp-srv` it is `resolve({ root: '/srv/ftp/alice' })`; in OpenSSH it is `ChrootDirectory`. The engine literally cannot form a path outside the root, so `cd ..` at the top stays put.
- **Virtual filesystem.** Implement the filesystem interface yourself (Go: `afero.Fs`; Node: a `FileSystem` subclass) and serve from anywhere — a database, S3, an in-memory map — while enforcing your own boundaries. This is how you add quotas, audit trails, on-upload scanning, or a non-disk backend.

> **Security must-do — path-traversal defense:** Even with a jail, normalize and validate every client-supplied path. A malicious client may send `../../etc/passwd` or absolute paths. `afero.BasePathFs` and OpenSSH `ChrootDirectory` handle this for you; in the hand-rolled Node SFTP server we deliberately do `path.join(root, path.normalize('/' + p))` so traversal sequences collapse *before* being joined to the root. Test it explicitly (§10.3). Also beware **symlink escape**: a symlink inside the jail pointing outside it can defeat a naive base-path jail — disallow symlink creation, or resolve and verify the real path stays within the root, or use OS-level chroot which follows links inside the new root only.

### 10.3 Verifying isolation **[I]**

Always prove the jail holds before trusting it:

```bash
# As alice, try to escape upward and to read another user's data.
sftp -P 2222 alice@localhost
sftp> pwd            # expect "/" (her root), not the host path
sftp> cd ..          # expect to stay at "/" or get an error
sftp> ls ../bob      # expect "No such file" / permission denied
sftp> get ../../etc/passwd   # expect failure

# As FTP:
ftp> cd /etc         # expect to stay inside the jail (no host /etc)
```

If any of these succeed, your jail is broken — stop and fix it before deploying.

### 10.4 OS-level hardening around the jail **[I/A]**

- Run the server as a **dedicated unprivileged user** (`ftpuser`), never root. If it must bind port 21, grant `cap_net_bind_service` instead of running as root.
- Set restrictive permissions on the roots: `chmod 750` on user dirs, `750`/`770` only where writes are needed, `chown` to the service user.
- For true chroot (OpenSSH), the chroot directory itself must be **owned by root and not group/other-writable**, with the user's writable area as a *subdirectory* — a classic OpenSSH requirement people miss.

---

## 11. Testing Your Server

### 11.1 The built-in `ftp` client **[B]**

Available on most systems (on Windows it is `ftp.exe`; in Git Bash you also get a Unix-style one). It supports plain FTP only — **no TLS**.

```bash
ftp -p localhost 2121      # -p = passive mode (essential behind any firewall)
# At the ftp> prompt:
ftp> user alice            # then enter the password when prompted
ftp> pwd                   # print working directory
ftp> ls                    # directory listing (exercises the DATA channel)
ftp> binary                # TYPE I — do this before transferring non-text!
ftp> get hello.txt         # download
ftp> put localfile.txt     # upload
ftp> mkdir uploads         # create a directory
ftp> bye                   # quit
```

> If `ls`/`get` hang while login works, it is a data-channel/passive-port/public-host problem (§2, §12), not auth.

### 11.2 `lftp` — the debugging power tool (Linux/macOS) **[I]**

```bash
# Install: sudo apt install lftp   |   brew install lftp

# Plain FTP:
lftp -u alice,s3cr3t ftp://localhost:2121

# FTPS (explicit TLS), self-signed cert in dev:
lftp -e "set ssl:verify-certificate false; set ftp:ssl-force true; set ftp:ssl-protect-data true; open ftp://alice:s3cr3t@localhost:2121"

# THE killer debugging feature — see every command and response, incl. the
# 227/229 passive reply so you can confirm the advertised IP is reachable:
lftp -e "debug 5; open ftp://alice:s3cr3t@localhost:2121"

# Inside lftp:
lftp> ls
lftp> get hello.txt
lftp> put localfile.txt
lftp> mirror remote/ local/        # download a whole tree
lftp> mirror -R local/ remote/     # upload a whole tree
lftp> bye
```

### 11.3 FileZilla (GUI, all platforms) **[B]**

1. **File → Site Manager → New Site.**
2. Host `localhost`, Port `2121`, Protocol `FTP - File Transfer Protocol`.
3. Encryption: `Use explicit FTP over TLS if available` for FTPS, or `Only use plain FTP (insecure)` for plain.
4. Logon Type `Normal`, User `alice`, Password `s3cr3t` → **Connect**.
5. Enable **View → Message Log** to watch the raw command/response conversation — the best way to *see* PASV/EPSV and TLS upgrades happen.

For a self-signed cert, accept FileZilla's certificate warning (dev only).

### 11.4 Testing SFTP **[B]**

```bash
sftp -P 2222 alice@localhost          # note: capital -P for the port
sftp> ls
sftp> put localfile.txt
sftp> get hello.txt
sftp> exit

# Non-interactive one-liner (dev only — skips host-key checking):
echo "ls" | sftp -P 2222 -o StrictHostKeyChecking=no alice@localhost
```

### 11.5 Automated smoke test with `basic-ftp` (Node) **[I]**

A scripted client is invaluable in CI: start the server, run this, assert. `basic-ftp` is a clean, promise-based FTP/FTPS client.

```javascript
// test-ftp.js — npm install basic-ftp
const ftp = require('basic-ftp');
const { Readable, Writable } = require('stream');

async function main() {
  const client = new ftp.Client();
  client.ftp.verbose = true; // print the full command/response trace

  try {
    await client.access({
      host: 'localhost',
      port: 2121,
      user: 'alice',
      password: 's3cr3t',
      secure: false, // true for FTPS
      // secureOptions: { rejectUnauthorized: false }, // dev self-signed certs
    });

    console.log('files:', (await client.list('/')).map((f) => f.name));

    // Upload from an in-memory stream, then download and verify.
    await client.uploadFrom(Readable.from(['hello from CI\n']), 'ci.txt');
    const chunks = [];
    await client.downloadTo(
      new Writable({ write(c, _e, cb) { chunks.push(c); cb(); } }),
      'ci.txt',
    );
    console.log('roundtrip:', Buffer.concat(chunks).toString());
  } finally {
    client.close();
  }
}

main().catch((e) => { console.error(e); process.exit(1); });
```

---

## 12. Firewall, Passive Port & NAT Configuration

### 12.1 Why this matters **[I]**

The data channel is a *separate* TCP connection on a *different* port (§1.3, §2). If that port is blocked, every listing and transfer fails even though login works. Getting the firewall, the passive range, and the advertised public host to agree is most of the operational work of running FTP.

### 12.2 The ports you must open **[I]**

| Port / range | Direction | Purpose |
|---|---|---|
| 21 (or 2121 in dev) | Inbound | FTP control channel |
| 50000–50100 (your range) | Inbound | FTP **passive** data connections |
| 990 | Inbound | Implicit FTPS (only if you use it) |
| 20 | Outbound from server | Active-mode data (only if you must support active) |
| 22 (or 2222 in dev) | Inbound | SFTP (SSH) — *the only port SFTP needs* |

Notice how much simpler SFTP is: one port, no range. That alone is a reason to prefer it.

### 12.3 Linux firewall (`ufw` and `iptables`) **[I]**

```bash
# ufw (Ubuntu/Debian) — simplest:
sudo ufw allow 21/tcp
sudo ufw allow 50000:50100/tcp
sudo ufw allow 22/tcp
sudo ufw reload

# iptables — note the ESTABLISHED,RELATED rule lets data return traffic flow:
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 50000:50100 -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### 12.4 Cloud security groups **[I]**

- **AWS Security Group:** inbound TCP 21 and TCP 50000–50100 (and 22 for SFTP) from your allowed sources. Prefer locking the source CIDR to known clients rather than `0.0.0.0/0`.
- **DigitalOcean Cloud Firewall / Azure NSG / GCP firewall:** add the same inbound rules. On all of them, the passive range and the control port must both be open or transfers hang.

### 12.5 Behind a home/office router (NAT) **[I/A]**

1. **Port-forward** port 21 *and the entire passive range* (50000–50100) on the router to your server's LAN IP. (For SFTP, forward only 22.)
2. Set the server's advertised host to your **public IP**, not the LAN IP — `PublicHost` (Go) / `pasv_url` (Node).
3. Find your public IP with `curl ifconfig.me` or `curl icanhazip.com`.

```go
PublicHost: "203.0.113.42", // Go: your real public IP
```

```javascript
pasv_url: '203.0.113.42',   // Node: your real public IP
```

> **Gotcha:** Advertising a LAN IP (`192.168.x.x`) to Internet clients makes passive mode silently fail — the client dials an unroutable private address and hangs until timeout. Using **EPSV** avoids embedding an IP at all and is more robust through NAT; ensure your clients use it.

### 12.6 Verifying a passive port from outside **[I]**

```bash
# From a DIFFERENT machine/network — test one port in the range:
nc -zv YOUR_PUBLIC_IP 50000        # "succeeded" = open; "refused/timed out" = blocked
nmap -p 50000-50010 YOUR_PUBLIC_IP # scan a slice of the range
```

Always test from a different network — localhost and same-LAN tests hide exactly the NAT/firewall problems you are trying to find.

---

## 13. Docker Deployment Notes

FTP is awkward in Docker precisely because of the passive range — you must publish many ports, and the advertised host must be the *Docker host's* reachable IP, not the container's internal IP.

### 13.1 `docker run` **[I/A]**

```bash
docker run -d \
  --name ftp-server \
  -p 21:21 \
  -p 50000-50100:50000-50100 \
  -v /data/ftp:/ftp-root \
  -e FTP_PUBLIC_HOST=203.0.113.42 \
  -e FTP_PASSIVE_MIN=50000 \
  -e FTP_PASSIVE_MAX=50100 \
  your-ftp-image:latest
```

> **Gotcha:** Publishing a range like `50000-50100` (101 ports) is valid, but Docker spawns one `docker-proxy` process **per port**, which is slow to start and memory-heavy. Keep the range as small as your concurrency allows. On Linux, `--network host` avoids the proxy entirely (see §13.5).

### 13.2 Docker Compose **[I]**

```yaml
# docker-compose.yml
services:
  ftp:
    build: .
    ports:
      - "21:21"
      - "50000-50100:50000-50100"   # MUST match the server's passive range
    volumes:
      - ftp_data:/ftp-root
    environment:
      FTP_PUBLIC_HOST: "203.0.113.42"  # the Docker HOST's reachable IP
      FTP_PASSIVE_MIN: "50000"
      FTP_PASSIVE_MAX: "50100"
    restart: unless-stopped

  sftp:
    build:
      context: .
      dockerfile: Dockerfile.sftp
    ports:
      - "22:22"                     # SFTP needs only one port
    volumes:
      - sftp_data:/sftp-root
      - ./host_key:/app/host_key:ro # persist the host key across restarts!
    restart: unless-stopped

volumes:
  ftp_data:
  sftp_data:
```

> Persist the SFTP **host key** via a volume/secret. If it regenerates on every restart, every client gets a MITM-style "host key changed" warning and many will refuse to connect.

### 13.3 Dockerfile (Go FTP server) **[I]**

```dockerfile
# ── Build stage ──
FROM golang:1.24-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Static binary, stripped, no CGO.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o ftp-server ./main.go

# ── Runtime stage (minimal, non-root) ──
FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
RUN addgroup -S ftpgroup && adduser -S ftpuser -G ftpgroup
WORKDIR /app
COPY --from=builder /build/ftp-server .
RUN mkdir -p /ftp-root && chown -R ftpuser:ftpgroup /ftp-root
USER ftpuser
EXPOSE 21 50000-50100
CMD ["./ftp-server"]
```

### 13.4 Dockerfile (Node FTP server) **[I]**

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY server.js ./
RUN addgroup -S ftpgroup && adduser -S ftpuser -G ftpgroup \
    && mkdir -p /ftp-root && chown -R ftpuser:ftpgroup /ftp-root /app
USER ftpuser
EXPOSE 2121 50000-50100
CMD ["node", "server.js"]
```

### 13.5 `--network host` (Linux only) **[A]**

```bash
docker run -d --network host your-ftp-image
```

The container shares the host network stack, so no port mapping and no `docker-proxy` swarm — handy when a large passive range is impractical.

> **Warning:** `--network host` works only on Linux Docker (not Docker Desktop on Mac/Windows) and removes network isolation between container and host. Use it only in environments you trust.

---

## 14. Logging, Observability & Operations

You cannot secure or debug what you cannot see. A file server is a high-value target, so treat its logs as a security control, not an afterthought.

### 14.1 What to log **[I]**

- **Connections:** every connect/disconnect with source IP and session ID (Go `ClientConnected`/`ClientDisconnected`; Node `connection`/`disconnect`/`client-error` events).
- **Auth outcomes:** every success *and failure* with username and IP — failures feed your brute-force detection. Never log passwords.
- **File operations:** uploads, downloads, deletes, renames — who, what path, how many bytes, success/failure. This is your audit trail for "who touched this file."
- **TLS:** handshake failures and the negotiated version/cipher, to catch misconfigured or downgraded clients.

> **Privacy/security:** never log credentials, full file contents, or (often) full personal paths. Treat logs as sensitive; restrict access and rotate them.

### 14.2 Structured logging in Go (`log/slog`) **[I/A]**

```go
import "log/slog"

// JSON logs to stdout — easy to ship to a log aggregator.
var logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))

func (d *MainDriver) AuthUser(cc ftpserver.ClientContext, user, pass string) (ftpserver.ClientDriver, error) {
	u, ok := users[user]
	if !ok || u.Password != pass {
		// Structured key/value fields are filterable/alertable downstream.
		logger.Warn("auth failed", "user", user, "remote", cc.RemoteAddr().String())
		return nil, errors.New("authentication failed")
	}
	logger.Info("auth ok", "user", user, "session", cc.ID())
	// ...return ClientDriver
	return &ClientDriver{fs: afero.NewBasePathFs(afero.NewOsFs(), u.RootDir)}, nil
}
```

### 14.3 Structured logging in Node (`pino`) **[I/A]**

```javascript
// npm install pino
const logger = require('pino')({ level: 'info' });

ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];
  if (!user || user.password !== password) {
    logger.warn({ user: username, ip: connection.ip }, 'auth failed');
    return reject(new Error('Authentication failed'));
  }
  logger.info({ user: username, ip: connection.ip }, 'auth ok');
  return resolve({ root: user.root });
});

// ftp-srv accepts a logger directly: new FtpSrv({ ..., log: logger })
```

### 14.4 Health checks and metrics **[A]**

- Expose a tiny HTTP `/healthz` on a side port so your orchestrator can liveness/readiness-probe the process (FTP itself has no easy health endpoint).
- Track counters: active sessions, transfers/sec, bytes in/out, auth failures/min, passive-port-pool exhaustion. Auth-failures-per-minute and passive-pool-exhaustion are the two metrics that most directly signal an attack or capacity problem.

---

## 15. Security Best Practices

FTP is one of the easiest services to deploy *insecurely*, and a file server is a juicy target (credentials, data exfiltration, and a foothold for lateral movement). Treat this section as the most important in the guide.

### 15.1 Use FTPS or SFTP — never plain FTP in production **[A]**

This cannot be overstated. Plain FTP transmits credentials and data in cleartext; assume any network you do not fully control is hostile.

| Scenario | Use |
|---|---|
| New system, you control the client | **SFTP** (simplest, strongest, one port) |
| Must speak the FTP protocol (legacy partner/device) | **FTPS with mandatory TLS** |
| Internet-facing, any kind | **FTPS or SFTP — never plain FTP** |
| Fully isolated, audited private LAN | Plain FTP is *technically* tolerable but still discouraged |

### 15.2 TLS certificate management **[A]**

```bash
# Production: Let's Encrypt — free and auto-renewable.
sudo certbot certonly --standalone -d ftp.example.com
# Certs land at:
#   /etc/letsencrypt/live/ftp.example.com/fullchain.pem
#   /etc/letsencrypt/live/ftp.example.com/privkey.pem

# Reload the server after renewal (renewal hook or cron):
# 0 3 * * * certbot renew --quiet --deploy-hook "systemctl restart my-ftp"
```

Enforce **TLS 1.2+** (`MinVersion: tls.VersionTLS12` in Go; `minVersion: 'TLSv1.2'` in Node), prefer TLS 1.3, and keep private keys `chmod 600` owned by the service user. Do not use self-signed certs in production — they normalize click-through warnings.

### 15.3 Mandatory encryption, no plaintext fallback **[A]**

Require TLS before login so credentials can never cross in clear: Go `TLSRequired: ftpserver.MandatoryEncryption`; Node — reject any `login` where `connection.secure` is false. Use a `ClearOrEncrypted` window *only* during a migration off legacy plaintext clients, with a deadline.

### 15.4 Strong authentication & hashed passwords **[A]**

Prefer SSH public keys (SFTP) or client certificates (FTPS). If you must use passwords, store **hashes** (bcrypt or argon2), never plaintext, and compare with a constant-time function.

```go
// Go — bcrypt. import "golang.org/x/crypto/bcrypt"
// Store at user-creation time:
hash, _ := bcrypt.GenerateFromPassword([]byte(plain), bcrypt.DefaultCost)
// Verify at login (constant-time inside CompareHashAndPassword):
if err := bcrypt.CompareHashAndPassword(storedHash, []byte(input)); err != nil {
	return nil, errors.New("authentication failed") // generic message
}
```

```javascript
// Node — bcrypt. npm install bcrypt
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(plain, 12);          // at user creation
const ok = await bcrypt.compare(input, storedHash); // at login
```

Other auth hardening: generic failure messages (no username enumeration), uniform failure timing, `MaxAuthTries`/attempt caps, and disabling anonymous access.

### 15.5 No anonymous access in production **[A]**

```go
// Go: if the username is not in your store, FAIL. Never fall through to anon.
```

```javascript
// Node: always set anonymous: false in the FtpSrv options.
anonymous: false,
```

### 15.6 Jail every user (chroot / virtual FS) **[A]**

Confine each user to their own subtree and *verify* it (§10.3). Go `afero.NewBasePathFs`; Node `resolve({ root })`; OpenSSH `ChrootDirectory`. Defend against path traversal (normalize paths) and symlink escape (disallow or resolve-and-check). Run the server as a dedicated non-root user.

### 15.7 Keep the passive range small **[A]**

Fewer open ports = smaller attack surface and fewer Docker proxies. Size to peak concurrency plus headroom (100–200 ports is usually plenty); do not open thousands "just in case."

### 15.8 Rate limiting & brute-force defense **[A]**

```go
// Go: count failures per IP in ClientConnected/AuthUser; block after N.
var (
	mu         sync.Mutex
	failCounts = map[string]int{}
)

func (d *MainDriver) ClientConnected(cc ftpserver.ClientContext) (string, error) {
	ip, _, _ := net.SplitHostPort(cc.RemoteAddr().String())
	mu.Lock()
	defer mu.Unlock()
	if failCounts[ip] > 10 {
		return "", errors.New("temporarily blocked") // refuse before login
	}
	return "Service ready.", nil
}
// Increment failCounts[ip] in AuthUser on failure; reset to 0 on success.
// For production, use a TTL store (Redis) so blocks expire, and combine with
// fail2ban watching your auth-failure logs.
```

```javascript
// Node: in-memory counter for illustration; use Redis + TTL in production.
const failCounts = new Map();
ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const ip = connection.ip;
  const user = USERS[username];
  if (!user || user.password !== password) {
    const n = (failCounts.get(ip) || 0) + 1;
    failCounts.set(ip, n);
    return reject(new Error(n > 5 ? 'Too many attempts' : 'Authentication failed'));
  }
  failCounts.set(ip, 0); // reset on success
  return resolve({ root: user.root });
});
```

Also restrict source IPs at the firewall/security group where you can — many FTP/SFTP endpoints serve a known, small set of partners.

### 15.9 File-permission & OS hardening **[A]**

```bash
chmod 750 /ftp-root /ftp-root/alice        # owner rwx, group r-x, others none
chmod 770 /ftp-root/alice/uploads          # group-writable only where needed
chown -R ftpuser:ftpgroup /ftp-root
```

Run as a dedicated unprivileged user; use `cap_net_bind_service` instead of root to bind low ports; keep the OS, the runtime, and the FTP/SSH libraries patched (subscribe to their advisories).

### 15.10 Defense in depth (high-value endpoints) **[A]**

- **FTPS + client certificates:** `ClientAuth: tls.RequireAndVerifyClientCert` so only holders of an issued client cert can even start a session — on top of password/key auth.
- **Per-tenant isolation:** separate Unix users/containers per tenant for hard kernel-enforced boundaries.
- **Antivirus / DLP on upload:** scan in an on-upload hook (Go custom `afero.Fs`, Node `FileSystem.write` override) and quarantine on detection.
- **Quotas:** enforce per-user storage/transfer limits in the virtual FS to blunt abuse and DoS.

---

## 16. Tips, Tricks & Gotchas

**Do:**
- Always configure **and test passive mode** — active mode fails behind NAT/firewalls. Test from a *different network*, not just localhost.
- Set `PublicHost`/`pasv_url` to the server's **reachable public IP** (or prefer EPSV, which embeds no IP).
- Use **FTPS or SFTP** in any production environment; require TLS (no plaintext fallback).
- Open **both** the control port and the **entire passive range** in the firewall *and* in Docker `-p`.
- **Jail** every user to their own directory and verify the jail holds (`cd ..`, traversal, symlink tests).
- Default to **binary** transfer type so non-text files are never corrupted.
- Keep the passive range **small** (100–200 ports). Set an **idle timeout** to reap abandoned sessions.
- **Log** connections, auth failures, and file operations with the source IP; alert on auth-failure spikes and passive-pool exhaustion.
- **Persist** the SFTP host key across restarts.

**Don't / common bugs:**
- Never advertise a LAN IP (`192.168.x.x`) for an Internet-facing server — passive mode silently fails for external clients.
- Never omit the passive range and then wonder why `LIST`/`RETR` hang while login works (the classic `425`).
- Don't use ASCII transfer mode for binary files (images, PDFs, zips, executables) — it corrupts them.
- Don't rely on the built-in `ftp` client for FTPS — it has no TLS; use `lftp` or FileZilla.
- Don't forget the passive range in Docker `-p`: just publishing port 21 lets clients log in but all transfers fail.
- Don't store passwords in plaintext, don't allow anonymous access in production, and don't use self-signed certs in production.
- Don't let the SFTP host key regenerate on restart — clients will see a host-key-changed (MITM) warning.

**Tricks:**
- `lftp -e "debug 5; open ..."` shows every command/response — the fastest way to see the actual `227`/`229` reply and prove the advertised IP/port.
- In FileZilla, **View → Message Log** reveals the raw FTP conversation, including TLS upgrades.
- `nc -zv HOST PORT` is the quickest way to check whether a passive port is reachable from outside.
- For Go integration tests, back users with `afero.NewMemMapFs()` — no disk I/O, fully isolated, instant cleanup.
- `basic-ftp` (Node) is an excellent lightweight client for automated server tests.
- For long-running automated clients, send periodic `NOOP` to dodge idle timeouts.
- `sftp.WithServerWorkingDirectory()` in Go both sets the start directory and jails the session.
- Combine `fail2ban` watching your auth-failure logs with in-app rate limiting for layered brute-force defense.

---

## 17. Study Path & Build-to-Learn Projects

Learn in this order for fastest offline mastery:

1. **Protocol fundamentals** (§1) and **active vs passive + NAT** (§2). This is the hardest *conceptual* part; everything else is code. Do not move on until you can explain, out loud, why passive mode exists and why the server must advertise its public IP.
2. **FTP vs FTPS vs SFTP** (§3). Memorize the table — it comes up in every real file-transfer security conversation.
3. **Commands & response codes** (§4). Skim now; return when reading a trace. Learn that `425/426` = data channel, `530/534` = auth/TLS, `550` = path/permission.
4. **Run a plain-FTP server** in Go (§5) or Node (§7). Get it running locally, connect with `lftp`/FileZilla, upload and download — watch the message log.
5. **Add FTPS** (§6 / §8). Generate a self-signed cert, enable TLS, make it mandatory, test with `lftp`.
6. **Build an SFTP server** (§9) and feel how much simpler one encrypted port is than two-channel FTP. Add public-key auth.
7. **Prove isolation** (§10) — try to escape the jail and confirm you cannot.
8. **Test from a second machine** (§11) — passive/NAT bugs often appear only over a real network hop.
9. **Deploy behind a firewall / in Docker** (§12, §13) — open the passive range, set the public host, persist the SFTP host key.
10. **Harden** (§14, §15) — structured logging, bcrypt/keys, rate limiting, mandatory TLS, Let's Encrypt, small passive range.

### Build-to-Learn Projects

1. **Local FTP drop box.** Run the Go or Node server, connect with FileZilla, drag files between two machines. *Reinforces:* passive mode, chroot, basic auth.
2. **Multi-user FTPS server on a VPS.** Deploy to a small cloud box, add Let's Encrypt, create three isolated users, and *prove* user A cannot see user B's files. *Reinforces:* TLS, isolation, firewall/passive config.
3. **SFTP-backed backup system.** Write a small Go or Node client that mirrors a local folder to your SFTP server on a schedule using SSH key auth. *Reinforces:* SFTP client usage, key auth, delta uploads.
4. **FTP server with upload webhooks.** Extend the Node `ftp-srv` server (or a custom `afero.Fs` in Go) to fire a webhook/email when a file is uploaded. *Reinforces:* event-driven design, virtual filesystems, integrating FTP with the rest of your stack.
5. **Drop-in replacement for a legacy FTP integration.** Take a system that uploads to a vendor FTP server and point it at your own (same credentials/paths), then add FTPS. *Reinforces:* protocol compliance, debugging with `lftp debug`/Wireshark, real-world compatibility.
6. **Hardened SFTP service via OpenSSH.** Configure OpenSSH with `internal-sftp`, a `Match Group sftponly`, `ChrootDirectory`, and key-only auth — then compare its robustness to your hand-rolled `ssh2`/`pkg-sftp` server. *Reinforces:* production SFTP, OS-level chroot, when to build vs. configure.

> **The one idea to lock in above all others:** FTP's complexity exists because it uses *two* connections, and passive mode exists because the client is behind NAT while the server is on the public Internet — which is why the server must announce its **public, reachable IP/port** in the PASV/EPSV response so the client can open the data connection itself. Get that rock-solid and every firewall, NAT, and Docker quirk in this guide becomes obvious. And when you get to choose: prefer **SFTP**.
