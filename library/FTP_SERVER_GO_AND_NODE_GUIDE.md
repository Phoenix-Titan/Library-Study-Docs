# Building an FTP Server — with Go and with Node.js

A thorough, offline-study reference for understanding the **FTP protocol** and building working FTP (and SFTP) servers in both **Go** and **Node.js** — covering the protocol fundamentals, active vs passive mode, TLS (FTPS), security, Docker deployment, and real runnable code.

> **Why learn to build an FTP server?** File-transfer infrastructure is everywhere — client file-drop portals, automated backup systems, legacy integrations, IoT device uploads. Understanding FTP from the inside (not just the client side) makes you a stronger backend engineer. Go and Node.js are both excellent choices: Go for performance and a single compiled binary, Node for rapid iteration and the JavaScript ecosystem. This guide covers both so you can pick the right tool and understand the trade-offs.

---

## Table of Contents

1. [FTP Protocol Fundamentals](#1-ftp-protocol-fundamentals)
2. [FTP vs FTPS vs SFTP — The Crucial Distinction](#2-ftp-vs-ftps-vs-sftp)
3. [Common FTP Commands & Response Codes](#3-common-ftp-commands--response-codes)
4. [PART 1 — FTP Server in Go](#4-part-1--ftp-server-in-go)
5. [Adding TLS (FTPS) in Go](#5-adding-tls-ftps-in-go)
6. [PART 2 — FTP Server in Node.js](#6-part-2--ftp-server-in-nodejs)
7. [Adding TLS (FTPS) in Node.js](#7-adding-tls-ftps-in-nodejs)
8. [PART 3 — SFTP Alternative (Recommended for Security)](#8-part-3--sftp-alternative-recommended-for-security)
9. [Testing Your Server](#9-testing-your-server)
10. [Firewall, Passive Port & NAT Configuration](#10-firewall-passive-port--nat-configuration)
11. [Docker Deployment Notes](#11-docker-deployment-notes)
12. [Security Best Practices](#12-security-best-practices)
13. [Tips, Tricks & Gotchas](#13-tips-tricks--gotchas)
14. [Study Path](#14-study-path)

---

## 1. FTP Protocol Fundamentals

### What FTP Is

FTP (File Transfer Protocol) is one of the oldest Internet protocols, defined in **RFC 959** (1985). It was designed for transferring files between a client and a server over a TCP/IP network. Despite its age, it remains in use today in legacy systems, managed file-transfer pipelines, and industrial equipment.

FTP is unique among common protocols because it uses **two separate TCP connections**:

| Connection | Port | Purpose |
|---|---|---|
| **Control channel** | 21 (server listens) | Commands and responses — authentication, navigation, file requests |
| **Data channel** | varies (see below) | Actual file contents, directory listings |

This dual-connection design is the root of most FTP complexity — especially around firewalls and NAT.

---

### The Control Channel (Port 21)

The client connects to the server on **TCP port 21**. This channel stays open for the entire session. Every FTP command the client sends and every response the server returns travels over this channel as plain UTF-8 text (unless FTPS is used).

```
Client                             Server (port 21)
  |------- TCP connect ------------>|
  |<------ 220 Welcome banner ------|
  |------- USER alice ------------->|
  |<------ 331 Password required ---|
  |------- PASS secret ------------>|
  |<------ 230 Logged in ------------|
  |------- PASV -------------------->|
  |<------ 227 (192,168,1,10,195,80) |   ← server tells client where to connect for data
  |------- RETR file.txt ----------->|
  |<------ 150 Opening data conn ----|
  |   (client opens a NEW TCP conn to 192.168.1.10:50000 for the actual file bytes)
  |<------ 226 Transfer complete ----|
  |------- QUIT -------------------->|
  |<------ 221 Goodbye --------------|
```

---

### The Data Channel — Active vs Passive Mode

This is the most misunderstood part of FTP and the #1 source of connectivity failures behind firewalls and NAT.

#### Active Mode (PORT command)

In **active mode**, the server initiates the data connection back to the client.

```
Client (behind NAT/firewall)          Server
  |--- PORT 203,0,113,5,200,100 -->|   ← client says: "connect to me on this IP:port"
  |--- RETR file.txt ------------->|
  |<-- Server opens NEW TCP conn --|   ← server connects FROM port 20 TO client
  |    (DATA arrives on client port 51300)
```

**Problem:** The client announces its LAN IP address (e.g. `192.168.1.5`), but the server is on the public internet and cannot reach that private IP. Even if the IP were public, most home/corporate firewalls block **inbound** connections by default. Active mode is practically broken behind NAT.

#### Passive Mode (PASV / EPSV commands)

In **passive mode**, the client initiates both connections — the server just opens a port and waits.

```
Client (behind NAT/firewall)          Server
  |--- PASV ----------------------->|
  |<-- 227 Entering Passive Mode ---|   ← server says: "connect to ME on port 50001"
  |--- (client opens data conn) ---->|
  |--- RETR file.txt -------------->|
  |<-- File data flows <------------|
```

**Why passive mode works behind NAT:** the client makes all outbound connections. Firewalls allow outbound connections by default. The server only needs a static range of ports open inbound for data (and port 21 for control).

- **PASV** — classic passive, IPv4, returns IP:port as `(h1,h2,h3,h4,p1,p2)`.
- **EPSV** — extended passive, works with both IPv4 and IPv6, returns just a port number.

> **Rule of thumb:** Always configure servers to support passive mode. Almost all modern FTP clients default to passive mode. Active mode is legacy and should only be used in tightly controlled network environments where you control both sides.

---

### Passive Port Range

Since the server must open a new TCP port for every data transfer, you must pre-configure a range of ports that:

- The server's firewall allows inbound.
- The server process is permitted to bind.
- Are not used by other services.

A common range is **50000–51000** (1001 ports = 1001 simultaneous transfers). You configure this in your server code and open the same range in your firewall.

---

## 2. FTP vs FTPS vs SFTP

This distinction is critical. Beginners frequently confuse these three protocols.

| Protocol | Transport | Port(s) | Authentication | Encryption |
|---|---|---|---|---|
| **FTP** | Raw TCP | 21 (control), dynamic (data) | Username/password in plaintext | None — all data is cleartext |
| **FTPS** | FTP over TLS | 21 (implicit: 990) | Username/password (TLS-protected) | TLS/SSL (same as HTTPS) |
| **SFTP** | SSH subsystem | 22 | SSH key or password (SSH-protected) | SSH encryption end-to-end |

### Plain FTP — Security Warning

> **WARNING:** Plain FTP sends your username, password, and all file contents in cleartext over the network. Anyone who can intercept traffic between client and server (on the same LAN, or a compromised router) can read everything, including credentials. **Never use plain FTP over the public internet or any untrusted network.** Use FTPS or SFTP instead.

Plain FTP is only acceptable in:
- A fully isolated, trusted private LAN where you have audited every device.
- A development/lab environment where no real credentials or data are used.

### FTPS — FTP + TLS

FTPS wraps the FTP control channel and data channel in TLS, the same encryption used by HTTPS. There are two modes:

- **Explicit FTPS (FTPES / AUTH TLS):** The client connects on port 21 and then sends the `AUTH TLS` command to upgrade the connection to TLS. Most common.
- **Implicit FTPS:** The entire connection is TLS from the start, on port **990**. Less common today.

FTPS requires the server to have a TLS certificate (self-signed works for testing; use Let's Encrypt for production).

### SFTP — SSH File Transfer Protocol

SFTP is **not** FTP over SSH — it is an entirely different protocol that happens to run as an SSH subsystem on port 22. It has none of FTP's dual-channel complexity; it uses a single encrypted SSH connection for everything.

**Prefer SFTP whenever you have a choice.** It is simpler, more secure, works through firewalls without any passive port range configuration, and is supported by every modern SSH server (OpenSSH includes it by default).

---

## 3. Common FTP Commands & Response Codes

### Client-to-Server Commands

| Command | Arguments | What it does |
|---|---|---|
| `USER` | username | Send the username |
| `PASS` | password | Send the password |
| `QUIT` | — | End the session |
| `PWD` | — | Print working directory |
| `CWD` | path | Change working directory |
| `CDUP` | — | Change to parent directory |
| `LIST` | [path] | List directory (like `ls -l`) |
| `NLST` | [path] | List names only |
| `RETR` | filename | Retrieve (download) a file |
| `STOR` | filename | Store (upload) a file |
| `DELE` | filename | Delete a file |
| `MKD` | dirname | Make directory |
| `RMD` | dirname | Remove directory |
| `RNFR` | oldname | Rename from (part 1) |
| `RNTO` | newname | Rename to (part 2) |
| `TYPE` | A or I | Transfer type: ASCII or Image (binary) |
| `PASV` | — | Enter passive mode (IPv4) |
| `EPSV` | — | Enter extended passive mode (IPv4/IPv6) |
| `PORT` | h1,h2,h3,h4,p1,p2 | Enter active mode |
| `SYST` | — | Get OS type |
| `FEAT` | — | List server features |
| `AUTH` | TLS | Upgrade control channel to TLS (FTPS) |
| `PBSZ` | 0 | Protection buffer size (FTPS) |
| `PROT` | P | Data channel protection: P=Private (TLS) |
| `SIZE` | filename | Get file size |
| `MDTM` | filename | Get file modification time |
| `NOOP` | — | Keep-alive (server responds 200) |

### Server Response Codes

FTP uses three-digit numeric response codes with a text message. The first digit indicates the category:

| Code Range | Category | Meaning |
|---|---|---|
| 1xx | Positive Preliminary | Action started; expect another reply |
| 2xx | Positive Completion | Action completed successfully |
| 3xx | Positive Intermediate | Command accepted; send more info |
| 4xx | Transient Negative | Action failed; may succeed if retried |
| 5xx | Permanent Negative | Action failed; do not retry without changing something |

### Common Specific Codes

| Code | Meaning |
|---|---|
| `220` | Service ready / welcome banner |
| `221` | Goodbye |
| `226` | Closing data connection; transfer complete |
| `227` | Entering passive mode `(h1,h2,h3,h4,p1,p2)` |
| `229` | Entering extended passive mode `(\|\|\|port\|)` |
| `230` | User logged in |
| `331` | Username OK, need password |
| `350` | Pending further information (RNFR, REST) |
| `421` | Service unavailable (closing connection) |
| `425` | Can't open data connection |
| `426` | Connection closed; transfer aborted |
| `430` | Invalid username or password |
| `450` | File unavailable (busy) |
| `451` | Local error in processing |
| `500` | Syntax error / unrecognized command |
| `502` | Command not implemented |
| `530` | Not logged in |
| `550` | File unavailable (not found / no permission) |
| `553` | File name not allowed |

---

## 4. PART 1 — FTP Server in Go

### Library Choice

The Go ecosystem has two main maintained options for building FTP servers:

| Library | GitHub | Notes |
|---|---|---|
| `github.com/fclairamb/ftpserverlib` | fclairamb/ftpserverlib | Most actively maintained (2024-2026); clean driver interface; supports FTPS, EPSV, per-user chroot |
| `github.com/goftp/server` | goftp/server | Older, less active maintenance |

This guide uses **`ftpserverlib`** — it separates the server engine from the filesystem/auth logic through a driver interface, which makes it very flexible.

> **⚡ Version note:** `ftpserverlib` v0.21+ uses a revised driver interface. The examples below target **v0.21.x / v0.22.x** (current as of 2026). Always check the module's `go.mod` and `CHANGELOG.md` when you `go get` it.

### Project Setup

```bash
mkdir go-ftp-server && cd go-ftp-server
go mod init github.com/yourname/go-ftp-server

# Get the library and afero (for filesystem abstraction)
go get github.com/fclairamb/ftpserverlib@latest
go get github.com/spf13/afero@latest
```

`afero` is a filesystem abstraction library. `ftpserverlib` uses afero internally, making it easy to swap between a real OS filesystem and in-memory or virtual filesystems.

---

### The Driver Interface

`ftpserverlib` requires you to implement two interfaces:

1. **`server.MainDriver`** — top-level: provides server settings and authenticates clients.
2. **`server.ClientDriver`** — per-client: provides the filesystem root for that user.

```
ftpserverlib engine
        │
        ├──► MainDriver.GetSettings()          → server config (ports, TLS, banner)
        ├──► MainDriver.ClientConnected(cc)    → called when client connects
        ├──► MainDriver.AuthUser(cc, user, pass) → return ClientDriver if auth OK
        └──► ClientDriver (per session)
                └──► afero.Fs                  → the filesystem this user can access
```

---

### Complete Minimal Working Example

This example creates an FTP server on port 2121 (non-root port for testing) with a single user `alice` and a passive port range of 50000–50100. Files are served from a local `./ftp-root` folder.

```go
// main.go
package main

import (
	"errors"
	"fmt"
	"log"
	"os"

	ftpserver "github.com/fclairamb/ftpserverlib"
	"github.com/spf13/afero"
)

// ─────────────────────────────────────────────
// 1.  In-memory user database
// ─────────────────────────────────────────────

type User struct {
	Password string
	RootDir  string // which folder on disk this user is chrooted to
}

var users = map[string]User{
	"alice": {Password: "s3cr3t", RootDir: "./ftp-root/alice"},
	"bob":   {Password: "hunter2", RootDir: "./ftp-root/bob"},
}

// ─────────────────────────────────────────────
// 2.  ClientDriver — one per authenticated session
// ─────────────────────────────────────────────

// ClientDriver wraps an afero.Fs and satisfies ftpserverlib.ClientDriver.
type ClientDriver struct {
	fs afero.Fs
}

// GetFS returns the filesystem root for this client.
// ftpserverlib calls this once per session to get the afero filesystem.
func (d *ClientDriver) GetFS() afero.Fs {
	return d.fs
}

// ─────────────────────────────────────────────
// 3.  MainDriver — global server logic
// ─────────────────────────────────────────────

type MainDriver struct {
	// tlsConfig *tls.Config  // uncomment when adding FTPS (see §5)
}

// GetSettings returns the server configuration.
func (d *MainDriver) GetSettings() (*ftpserver.Settings, error) {
	return &ftpserver.Settings{
		// ListenAddr is "host:port"; use ":21" in production (requires root or capability).
		// We use 2121 here so you can run the example without sudo.
		ListenAddr: "0.0.0.0:2121",

		// The public IP/hostname that the server tells clients to connect to
		// for passive mode data connections. Must be reachable from the client.
		// In development, use your LAN IP or "127.0.0.1".
		// In production, use your public IP or hostname.
		PublicHost: "127.0.0.1",

		// Passive port range — open these in your firewall too.
		PassiveTransferPortRange: &ftpserver.PortRange{
			Start: 50000,
			End:   50100,
		},

		// Banner displayed to clients on connect.
		Banner: "Welcome to Go FTP Server",

		// Idle timeout in seconds (0 = no timeout).
		IdleTimeout: 900,

		// Default transfer type: true = binary, false = ASCII.
		DefaultTransferType: ftpserver.TransferTypeBinary,
	}, nil
}

// ClientConnected is called when a new TCP connection arrives (before auth).
// Return an error to reject the connection before login.
func (d *MainDriver) ClientConnected(cc ftpserver.ClientContext) (string, error) {
	log.Printf("Client connected: ID=%d RemoteAddr=%s", cc.ID(), cc.RemoteAddr())
	return "Welcome!", nil
}

// ClientDisconnected is called when a client disconnects.
func (d *MainDriver) ClientDisconnected(cc ftpserver.ClientContext) {
	log.Printf("Client disconnected: ID=%d", cc.ID())
}

// AuthUser validates credentials and, if valid, returns a ClientDriver
// that provides the per-user filesystem.
func (d *MainDriver) AuthUser(cc ftpserver.ClientContext, username, password string) (ftpserver.ClientDriver, error) {
	user, ok := users[username]
	if !ok || user.Password != password {
		// Return a generic error; never reveal whether the username exists.
		return nil, errors.New("invalid credentials")
	}

	// Make sure the user's root directory exists.
	if err := os.MkdirAll(user.RootDir, 0750); err != nil {
		return nil, fmt.Errorf("cannot create user root: %w", err)
	}

	log.Printf("User %q authenticated (session %d)", username, cc.ID())

	// afero.NewBasePathFs creates a chrooted view of the filesystem:
	// the user cannot navigate above their RootDir.
	fs := afero.NewBasePathFs(afero.NewOsFs(), user.RootDir)

	return &ClientDriver{fs: fs}, nil
}

// GetTLSConfig returns nil here (plain FTP).
// See §5 to add FTPS.
func (d *MainDriver) GetTLSConfig() (*tls.Config, error) {
	return nil, nil
}

// ─────────────────────────────────────────────
// 4.  main — wire it all up and start
// ─────────────────────────────────────────────

func main() {
	// Create the top-level user root directories.
	for _, u := range users {
		if err := os.MkdirAll(u.RootDir, 0750); err != nil {
			log.Fatalf("Cannot create user directory: %v", err)
		}
	}

	driver := &MainDriver{}

	// ftpserver.NewFtpServer takes your MainDriver and returns the server.
	server := ftpserver.NewFtpServer(driver)

	log.Println("Starting FTP server on :2121 ...")
	log.Println("Passive port range: 50000-50100")
	log.Println("Users: alice / s3cr3t, bob / hunter2")

	// ListenAndServe blocks until the server exits.
	if err := server.ListenAndServe(); err != nil {
		log.Fatalf("FTP server error: %v", err)
	}
}
```

> **⚡ Version note:** `ftpserverlib` v0.21 removed the `GetTLSConfig` method from `MainDriver` in favor of passing a `*tls.Config` directly in `Settings`. If your version complains, check the library's `FtpDriver` interface in its source — method signatures sometimes shift between minor versions. The approach is identical; only the wiring changes.

### Running the Go Server

```bash
# Create test files so you have something to download
mkdir -p ftp-root/alice ftp-root/bob
echo "Hello from alice's folder" > ftp-root/alice/hello.txt
echo "Bob's secret data" > ftp-root/bob/data.csv

go run main.go
# Output:
# Starting FTP server on :2121 ...
# Passive port range: 50000-50100
```

Test with `ftp` or FileZilla (see §9 for testing details).

---

### Using an In-Memory Filesystem (Testing / Sandboxing)

Replace the `afero.NewOsFs()` with `afero.NewMemMapFs()` for a completely in-memory filesystem — useful for tests or sandboxed environments where you do not want any disk writes:

```go
// In-memory filesystem — nothing touches disk.
// Pre-populate it with test files if needed.
memFs := afero.NewMemMapFs()
_ = memFs.MkdirAll("/uploads", 0755)
_ = afero.WriteFile(memFs, "/welcome.txt", []byte("Hello!\n"), 0644)

// Chroot is optional with in-memory fs but still recommended.
fs := afero.NewBasePathFs(memFs, "/")
return &ClientDriver{fs: fs}, nil
```

---

## 5. Adding TLS (FTPS) in Go

FTPS requires a TLS certificate. For development, generate a self-signed certificate. For production, use Let's Encrypt or your CA.

### Generate a Self-Signed Certificate (development only)

```bash
# Generates server.key (private key) and server.crt (certificate), valid 365 days.
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt \
  -days 365 -nodes \
  -subj "/CN=localhost"
```

### Updated Go Code for FTPS

```go
// Add this import at the top of main.go
import (
	"crypto/tls"
	// ... other imports
)

// GetTLSConfig — return a TLS config to enable FTPS (Explicit TLS / AUTH TLS).
// ftpserverlib will advertise AUTH TLS in its FEAT response and upgrade connections.
func (d *MainDriver) GetTLSConfig() (*tls.Config, error) {
	// Load the certificate and private key.
	cert, err := tls.LoadX509KeyPair("server.crt", "server.key")
	if err != nil {
		return nil, fmt.Errorf("loading TLS cert: %w", err)
	}

	return &tls.Config{
		Certificates: []tls.Certificate{cert},
		// MinVersion ensures TLS 1.2+ (TLS 1.0/1.1 are deprecated).
		MinVersion: tls.VersionTLS12,
		// Prefer server cipher suites for better security.
		PreferServerCipherSuites: true,
	}, nil
}
```

Update `GetSettings()` to tell the server that TLS is required:

```go
func (d *MainDriver) GetSettings() (*ftpserver.Settings, error) {
	tlsCfg, err := d.GetTLSConfig()
	if err != nil {
		return nil, err
	}

	return &ftpserver.Settings{
		ListenAddr: "0.0.0.0:2121",
		PublicHost: "127.0.0.1",
		PassiveTransferPortRange: &ftpserver.PortRange{
			Start: 50000,
			End:   50100,
		},
		Banner: "Welcome to Go FTPS Server",

		// TLSConfig enables FTPS — both control and data channels are TLS-protected.
		TLSConfig: tlsCfg,

		// TLSRequired=true means clients MUST use AUTH TLS (no plaintext fallback).
		// Set to ftpserver.ClearOrEncrypted to allow both (useful in migration).
		TLSRequired: ftpserver.MandatoryEncryption,
	}, nil
}
```

> When testing FTPS with FileZilla, use "Require explicit FTP over TLS" and accept the self-signed certificate. The `ftp` command-line client does not support FTPS — use `lftp` instead (`lftp -e "set ftp:ssl-force true; open ftp://alice:s3cr3t@localhost:2121"`).

---

## 6. PART 2 — FTP Server in Node.js

### Library Choice

**`ftp-srv`** is the most actively maintained Node.js FTP server library (2024–2026). It supports:

- PASV and EPSV (passive mode)
- FTPS (explicit TLS via `AUTH TLS`)
- Customizable per-user filesystem roots
- Promise-based `login` event API

> **⚡ Version note:** `ftp-srv` v5.x is the current API as of 2026. v4.x had slightly different event naming. The examples below target **v5.x**. Run `npm show ftp-srv version` to confirm what you are installing.

### Project Setup

```bash
mkdir node-ftp-server && cd node-ftp-server
npm init -y
npm install ftp-srv
```

---

### Complete Minimal Working Example

```javascript
// server.js
'use strict';

const FtpSrv = require('ftp-srv');
const path   = require('path');
const fs     = require('fs');

// ─────────────────────────────────────────────
// 1.  User database (in production, use a DB or env-based config)
// ─────────────────────────────────────────────

const USERS = {
  alice: { password: 's3cr3t', root: path.resolve('./ftp-root/alice') },
  bob:   { password: 'hunter2', root: path.resolve('./ftp-root/bob')  },
};

// Ensure user root directories exist
Object.values(USERS).forEach(u => fs.mkdirSync(u.root, { recursive: true }));

// ─────────────────────────────────────────────
// 2.  Create the FTP server
// ─────────────────────────────────────────────

const ftpServer = new FtpSrv({
  // url: the control channel bind address.
  // Use 'ftp://0.0.0.0:21' in production (requires root/cap_net_bind_service).
  // We use port 2121 here to run without elevated privileges.
  url: 'ftp://0.0.0.0:2121',

  // pasv_url: the IP address that the server advertises to clients in the
  // PASV response. Must be the server's public IP or hostname.
  // In development, use '127.0.0.1'. In production, use your public IP.
  pasv_url: '127.0.0.1',

  // pasv_min / pasv_max: the passive port range.
  // Open these ports in your firewall!
  pasv_min: 50000,
  pasv_max: 50100,

  // greeting: the 220 banner sent to clients on connect.
  greeting: ['Welcome to Node.js FTP Server', 'Authorized users only.'],

  // anonymous: set to true only for fully public file shares with no sensitive data.
  anonymous: false,

  // log: pass a pino/bunyan compatible logger, or leave undefined for no logging.
  // Example with basic console: { info: console.log, error: console.error, debug: ()=>{} }
});

// ─────────────────────────────────────────────
// 3.  Authentication — the 'login' event
// ─────────────────────────────────────────────

// ftp-srv fires 'login' for every USER+PASS attempt.
// You resolve() with a { root } object to grant access,
// or reject() with an error to deny.
ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];

  if (!user || user.password !== password) {
    // Use a generic message — never reveal whether the username exists.
    return reject(new Error('Invalid credentials'));
  }

  console.log(`[FTP] User "${username}" authenticated from ${connection.ip}`);

  // Resolve with the filesystem root for this user.
  // ftp-srv automatically chroots the session to this directory —
  // the user cannot navigate above it.
  return resolve({ root: user.root });
});

// ─────────────────────────────────────────────
// 4.  Optional: listen for additional events
// ─────────────────────────────────────────────

// 'client-error' fires for per-connection errors (e.g. bad TLS handshake).
ftpServer.on('client-error', ({ connection, context, error }) => {
  console.error(`[FTP] Client error (${context}):`, error.message);
});

// ─────────────────────────────────────────────
// 5.  Start the server
// ─────────────────────────────────────────────

ftpServer.listen()
  .then(() => {
    console.log('[FTP] Server listening on port 2121');
    console.log('[FTP] Passive ports: 50000-50100');
    console.log('[FTP] Users: alice / s3cr3t, bob / hunter2');
  })
  .catch(err => {
    console.error('[FTP] Failed to start:', err);
    process.exit(1);
  });

// Graceful shutdown on SIGINT / SIGTERM
process.on('SIGINT',  () => ftpServer.close().then(() => process.exit(0)));
process.on('SIGTERM', () => ftpServer.close().then(() => process.exit(0)));
```

### Running the Node Server

```bash
# Create test files
mkdir -p ftp-root/alice ftp-root/bob
echo "Hello from Alice" > ftp-root/alice/hello.txt
echo "Bob's data"       > ftp-root/bob/data.csv

node server.js
# [FTP] Server listening on port 2121
# [FTP] Passive ports: 50000-50100
```

---

### Advanced: Custom Filesystem Root Per Connection

`ftp-srv` supports returning a `FileSystem` instance instead of a plain path, letting you implement virtual filesystems, per-user upload quotas, or audit logging:

```javascript
const { FileSystem } = require('ftp-srv');

// Extend the built-in FileSystem to add upload logging.
class AuditedFileSystem extends FileSystem {
  // write() is called on STOR (file upload).
  write(fileName, { append, start } = {}) {
    console.log(`[AUDIT] Upload: ${this.root}/${fileName}`);
    // Delegate to the parent implementation.
    return super.write(fileName, { append, start });
  }
}

// In the login handler:
ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];
  if (!user || user.password !== password) return reject(new Error('Invalid credentials'));

  // Pass an instance instead of a path string.
  const customFs = new AuditedFileSystem(connection, { root: user.root, cwd: '/' });
  return resolve({ fs: customFs });
});
```

---

## 7. Adding TLS (FTPS) in Node.js

### Generate a Self-Signed Certificate

```bash
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt \
  -days 365 -nodes -subj "/CN=localhost"
```

### Updated Node.js Code for FTPS

```javascript
// server-tls.js
'use strict';

const FtpSrv = require('ftp-srv');
const path   = require('path');
const fs     = require('fs');
const tls    = require('tls');

const USERS = {
  alice: { password: 's3cr3t', root: path.resolve('./ftp-root/alice') },
};
Object.values(USERS).forEach(u => fs.mkdirSync(u.root, { recursive: true }));

// Build a TLS context. ftp-srv accepts a standard Node.js tls options object.
const tlsOptions = {
  key:  fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),

  // Minimum TLS version — TLS 1.2 is the current safe minimum.
  minVersion: 'TLSv1.2',

  // For explicit FTPS (AUTH TLS), ftp-srv negotiates TLS on the existing
  // port 21 connection after the client sends AUTH TLS.
  // For implicit FTPS (always-TLS), you'd use a separate port 990.
};

const ftpServer = new FtpSrv({
  url:      'ftp://0.0.0.0:2121',
  pasv_url: '127.0.0.1',
  pasv_min: 50000,
  pasv_max: 50100,
  greeting: 'Welcome to Secure Node.js FTPS Server',
  anonymous: false,

  // tls: passing this object enables FTPS support.
  // The server will advertise AUTH TLS in FEAT and handle TLS upgrades.
  tls: tlsOptions,
});

ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const user = USERS[username];
  if (!user || user.password !== password) return reject(new Error('Invalid credentials'));
  console.log(`[FTPS] User "${username}" authenticated`);
  return resolve({ root: user.root });
});

ftpServer.listen().then(() => {
  console.log('[FTPS] Server listening on port 2121 with explicit TLS');
});
```

> **Testing FTPS in Node:** Use `lftp` (`lftp -e "set ftp:ssl-force true; open ftp://alice:s3cr3t@localhost:2121"`) or FileZilla with "Require explicit FTP over TLS". Accept the self-signed certificate warning in dev.

---

## 8. PART 3 — SFTP Alternative (Recommended for Security)

SFTP (SSH File Transfer Protocol) is almost always preferable to FTP/FTPS in new systems:

- Single encrypted connection (no passive port range headaches).
- SSH key authentication (much stronger than passwords).
- Firewall-friendly (only port 22).
- Built into OpenSSH (zero extra software on Linux servers).

The examples below show minimal server skeletons. For production use, wrapping OpenSSH is usually better than implementing your own SSH server — but these skeletons teach you how the protocol works and are useful for embedded/custom scenarios.

---

### SFTP Server in Go

Uses `golang.org/x/crypto/ssh` (SSH server) + `github.com/pkg/sftp` (SFTP subsystem handler).

```bash
go get golang.org/x/crypto/ssh@latest
go get github.com/pkg/sftp@latest
```

```go
// sftp_server.go
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

func main() {
	// ── 1. Load or generate the host key ──────────────────────────────
	// In production, generate once and persist: ssh-keygen -t ed25519 -f host_key
	// For this example we load a pre-existing key file.
	hostKeyBytes, err := os.ReadFile("host_key")
	if err != nil {
		log.Fatalf("Read host key: %v", err)
	}
	hostKey, err := ssh.ParsePrivateKey(hostKeyBytes)
	if err != nil {
		log.Fatalf("Parse host key: %v", err)
	}

	// ── 2. SSH server configuration ───────────────────────────────────
	sshConfig := &ssh.ServerConfig{
		// PasswordCallback validates username+password.
		// Return nil to allow, non-nil error to deny.
		PasswordCallback: func(conn ssh.ConnMetadata, password []byte) (*ssh.Permissions, error) {
			validUsers := map[string]string{
				"alice": "s3cr3t",
				"bob":   "hunter2",
			}
			if p, ok := validUsers[conn.User()]; ok && p == string(password) {
				log.Printf("[SFTP] Authenticated: %s from %s", conn.User(), conn.RemoteAddr())
				return &ssh.Permissions{
					// Store the username so the SFTP handler can use it later.
					Extensions: map[string]string{"user": conn.User()},
				}, nil
			}
			return nil, fmt.Errorf("invalid credentials for user %q", conn.User())
		},

		// PublicKeyCallback can also be implemented for SSH key auth.
		// Best practice: prefer keys over passwords.
	}
	sshConfig.AddHostKey(hostKey)

	// ── 3. Listen for TCP connections ─────────────────────────────────
	listener, err := net.Listen("tcp", "0.0.0.0:2222")
	if err != nil {
		log.Fatalf("Listen: %v", err)
	}
	log.Println("[SFTP] Listening on :2222")

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Printf("Accept error: %v", err)
			continue
		}
		// Handle each connection in its own goroutine.
		go handleSSHConnection(conn, sshConfig)
	}
}

func handleSSHConnection(conn net.Conn, config *ssh.ServerConfig) {
	defer conn.Close()

	// Perform the SSH handshake.
	sshConn, chans, reqs, err := ssh.NewServerConn(conn, config)
	if err != nil {
		log.Printf("[SFTP] SSH handshake failed: %v", err)
		return
	}
	defer sshConn.Close()
	log.Printf("[SFTP] New SSH connection: %s (%s)", sshConn.RemoteAddr(), sshConn.ClientVersion())

	// Discard global requests (keep-alive pings etc.)
	go ssh.DiscardRequests(reqs)

	// Handle channel requests (one channel per SFTP session).
	for newChan := range chans {
		if newChan.ChannelType() != "session" {
			_ = newChan.Reject(ssh.UnknownChannelType, "unknown channel type")
			continue
		}
		go handleSession(newChan, sshConn)
	}
}

func handleSession(newChan ssh.NewChannel, sshConn *ssh.ServerConn) {
	channel, requests, err := newChan.Accept()
	if err != nil {
		log.Printf("[SFTP] Accept channel: %v", err)
		return
	}
	defer channel.Close()

	// Look for a "subsystem" request for "sftp".
	for req := range requests {
		if req.Type == "subsystem" && len(req.Payload) > 4 {
			subsystem := string(req.Payload[4:])
			if subsystem == "sftp" {
				_ = req.Reply(true, nil)

				// Determine the root directory for this user.
				username := sshConn.Permissions.Extensions["user"]
				root := fmt.Sprintf("./sftp-root/%s", username)
				_ = os.MkdirAll(root, 0750)

				// Create an sftp.Server using the channel as its read/write transport.
				// sftp.WithServerWorkingDirectory sets the starting directory.
				sftpServer, err := sftp.NewServer(
					channel,
					sftp.WithServerWorkingDirectory(root),
				)
				if err != nil {
					log.Printf("[SFTP] Server error: %v", err)
					return
				}
				// Serve blocks until the client disconnects.
				if err := sftpServer.Serve(); err != nil && err != io.EOF {
					log.Printf("[SFTP] Serve error: %v", err)
				}
				return
			}
		}
		// Reject any other subsystem / exec / pty requests.
		_ = req.Reply(false, nil)
	}
}
```

```bash
# Generate a host key (do this once; commit host_key to your secrets store, not to git)
ssh-keygen -t ed25519 -f host_key -N ""

# Create user directories and test files
mkdir -p sftp-root/alice sftp-root/bob
echo "Alice's file" > sftp-root/alice/hello.txt

go run sftp_server.go
# [SFTP] Listening on :2222

# Test with the sftp client
sftp -P 2222 alice@localhost
# Enter password: s3cr3t
# sftp> ls
# sftp> get hello.txt
```

---

### SFTP Server in Node.js

Uses the `ssh2` package, which provides both an SSH client and server, plus an SFTP subsystem handler.

```bash
npm install ssh2
```

> **⚡ Version note:** `ssh2` v1.x (current as of 2026) has a revised API compared to v0.x. The example below uses **v1.x**. The `SFTP` event/handler system changed significantly between versions — always check the `ssh2` README after installing.

```javascript
// sftp-server.js
'use strict';

const { Server, utils: { generateKeyPairSync } } = require('ssh2');
const path = require('path');
const fs   = require('fs');

// ── User database ──────────────────────────────────────────────────────────
const USERS = {
  alice: { password: 's3cr3t', root: path.resolve('./sftp-root/alice') },
  bob:   { password: 'hunter2', root: path.resolve('./sftp-root/bob')  },
};
Object.values(USERS).forEach(u => fs.mkdirSync(u.root, { recursive: true }));

// ── Load or generate host key ──────────────────────────────────────────────
// In production: generate once with ssh-keygen and load from a secrets store.
// Here we generate an ephemeral key on startup (fine for development).
const hostKey = fs.existsSync('host_key')
  ? fs.readFileSync('host_key')
  : (() => {
      const { private: priv } = generateKeyPairSync('ed25519');
      fs.writeFileSync('host_key', priv, { mode: 0o600 });
      return priv;
    })();

// ── SFTP request handler ───────────────────────────────────────────────────
// This function handles all SFTP operations (read, write, stat, list, etc.)
// using Node's fs module, chrooted to userRoot.
function handleSftpSession(accept, userRoot) {
  const sftp = accept(); // accept the SFTP subsystem request

  // Helper: translate a client path (always absolute from their root)
  // to a real filesystem path under userRoot.
  const realPath = (p) => path.join(userRoot, path.normalize('/' + p));

  // Map of open file descriptors: handle → fs.FileHandle
  const openFiles = new Map();
  let handleCounter = 0;

  // OPEN — client wants to open/create a file
  sftp.on('OPEN', (reqId, filename, flags, attrs) => {
    const fpath = realPath(filename);
    // flags is a bitmask: sftp.OPEN_MODE.READ, WRITE, CREAT, TRUNC, APPEND
    const fsFlags = sftp.flagsToString(flags); // e.g. 'r', 'w', 'a'
    try {
      const fd = fs.openSync(fpath, fsFlags);
      const handle = Buffer.alloc(4);
      handle.writeUInt32BE(handleCounter++, 0);
      openFiles.set(handle.readUInt32BE(0), fd);
      sftp.handle(reqId, handle);
    } catch (e) {
      sftp.status(reqId, sftp.STATUS_CODE.FAILURE);
    }
  });

  // READ — client wants to read bytes from an open file
  sftp.on('READ', (reqId, handle, offset, length) => {
    const fd = openFiles.get(handle.readUInt32BE(0));
    if (fd === undefined) return sftp.status(reqId, sftp.STATUS_CODE.FAILURE);
    const buf = Buffer.alloc(length);
    try {
      const bytesRead = fs.readSync(fd, buf, 0, length, offset);
      if (bytesRead === 0) return sftp.status(reqId, sftp.STATUS_CODE.EOF);
      sftp.data(reqId, buf.slice(0, bytesRead));
    } catch (e) {
      sftp.status(reqId, sftp.STATUS_CODE.FAILURE);
    }
  });

  // WRITE — client is uploading data
  sftp.on('WRITE', (reqId, handle, offset, data) => {
    const fd = openFiles.get(handle.readUInt32BE(0));
    if (fd === undefined) return sftp.status(reqId, sftp.STATUS_CODE.FAILURE);
    try {
      fs.writeSync(fd, data, 0, data.length, offset);
      sftp.status(reqId, sftp.STATUS_CODE.OK);
    } catch (e) {
      sftp.status(reqId, sftp.STATUS_CODE.FAILURE);
    }
  });

  // CLOSE — client closes a file handle
  sftp.on('CLOSE', (reqId, handle) => {
    const key = handle.readUInt32BE(0);
    const fd  = openFiles.get(key);
    if (fd !== undefined) {
      fs.closeSync(fd);
      openFiles.delete(key);
    }
    sftp.status(reqId, sftp.STATUS_CODE.OK);
  });

  // STAT / LSTAT — get file metadata
  const sendStat = (reqId, fpath) => {
    try {
      const stat = fs.statSync(fpath);
      sftp.attrs(reqId, {
        mode:  stat.mode,
        uid:   stat.uid,
        gid:   stat.gid,
        size:  stat.size,
        atime: Math.floor(stat.atimeMs / 1000),
        mtime: Math.floor(stat.mtimeMs / 1000),
      });
    } catch (e) {
      sftp.status(reqId, sftp.STATUS_CODE.NO_SUCH_FILE);
    }
  };
  sftp.on('STAT',  (reqId, fpath) => sendStat(reqId, realPath(fpath)));
  sftp.on('LSTAT', (reqId, fpath) => sendStat(reqId, realPath(fpath)));

  // OPENDIR / READDIR / CLOSEDIR — directory listing
  const openDirs = new Map();
  sftp.on('OPENDIR', (reqId, dirpath) => {
    const rp = realPath(dirpath);
    try {
      const entries = fs.readdirSync(rp, { withFileTypes: true });
      const handle  = Buffer.alloc(4);
      handle.writeUInt32BE(handleCounter++, 0);
      openDirs.set(handle.readUInt32BE(0), { entries, index: 0, rp });
      sftp.handle(reqId, handle);
    } catch (e) {
      sftp.status(reqId, sftp.STATUS_CODE.NO_SUCH_FILE);
    }
  });

  sftp.on('READDIR', (reqId, handle) => {
    const dir = openDirs.get(handle.readUInt32BE(0));
    if (!dir || dir.index >= dir.entries.length) {
      return sftp.status(reqId, sftp.STATUS_CODE.EOF);
    }
    // Return up to 20 entries at a time
    const batch = dir.entries.slice(dir.index, dir.index + 20);
    dir.index += batch.length;
    const names = batch.map(entry => {
      const fullPath = path.join(dir.rp, entry.name);
      let stat = { mode: 0o644, size: 0, atime: 0, mtime: 0, uid: 0, gid: 0 };
      try { const s = fs.statSync(fullPath); stat = { mode: s.mode, size: s.size, atime: Math.floor(s.atimeMs/1000), mtime: Math.floor(s.mtimeMs/1000), uid: s.uid, gid: s.gid }; } catch {}
      return { filename: entry.name, longname: entry.name, attrs: stat };
    });
    sftp.name(reqId, names);
  });

  sftp.on('CLOSEDIR', (reqId, handle) => {
    openDirs.delete(handle.readUInt32BE(0));
    sftp.status(reqId, sftp.STATUS_CODE.OK);
  });

  // REMOVE / RMDIR / MKDIR / RENAME
  sftp.on('REMOVE', (reqId, fpath) => {
    try { fs.unlinkSync(realPath(fpath)); sftp.status(reqId, sftp.STATUS_CODE.OK); }
    catch { sftp.status(reqId, sftp.STATUS_CODE.FAILURE); }
  });
  sftp.on('RMDIR', (reqId, fpath) => {
    try { fs.rmdirSync(realPath(fpath)); sftp.status(reqId, sftp.STATUS_CODE.OK); }
    catch { sftp.status(reqId, sftp.STATUS_CODE.FAILURE); }
  });
  sftp.on('MKDIR', (reqId, fpath, attrs) => {
    try { fs.mkdirSync(realPath(fpath), { recursive: true }); sftp.status(reqId, sftp.STATUS_CODE.OK); }
    catch { sftp.status(reqId, sftp.STATUS_CODE.FAILURE); }
  });
  sftp.on('RENAME', (reqId, oldPath, newPath) => {
    try { fs.renameSync(realPath(oldPath), realPath(newPath)); sftp.status(reqId, sftp.STATUS_CODE.OK); }
    catch { sftp.status(reqId, sftp.STATUS_CODE.FAILURE); }
  });
}

// ── SSH Server ─────────────────────────────────────────────────────────────
const server = new Server({ hostKeys: [hostKey] }, (client) => {
  console.log('[SFTP] Client connected:', client._sock?.remoteAddress);

  client.on('authentication', (ctx) => {
    const user = USERS[ctx.username];
    if (ctx.method === 'password' && user && user.password === ctx.password) {
      ctx.accept();
    } else {
      ctx.reject(['password']); // tell client: only password auth supported
    }
  });

  client.on('ready', () => {
    client.on('session', (accept, reject) => {
      const session = accept();
      session.on('sftp', (accept) => {
        const userRoot = USERS[client._state?.authsQueue?.[0] ?? '']?.root
          ?? path.resolve('./sftp-root/default');
        handleSftpSession(accept, userRoot);
      });
    });
  });

  client.on('error', (err) => console.error('[SFTP] Client error:', err.message));
  client.on('close', () => console.log('[SFTP] Client disconnected'));
});

server.listen(2222, '0.0.0.0', () => {
  console.log('[SFTP] Server listening on port 2222');
  console.log('[SFTP] Users: alice / s3cr3t, bob / hunter2');
});
```

```bash
node sftp-server.js
# [SFTP] Server listening on port 2222

# Test
sftp -P 2222 alice@localhost
# alice@localhost's password: s3cr3t
# sftp> ls
# sftp> put localfile.txt
# sftp> get hello.txt
```

> **Production recommendation:** For a production SFTP server in Node, consider using a well-tested wrapper around OpenSSH via `child_process`, or use Go with `golang.org/x/crypto/ssh` which has a much more complete and battle-tested SSH implementation.

---

## 9. Testing Your Server

### Command-Line `ftp` Client (built into most OS)

```bash
# Connect to your server (replace 2121 with your port)
ftp -p localhost 2121     # -p flag = passive mode (essential!)
# or on some systems:
ftp localhost 2121

# At the ftp> prompt:
ftp> user alice           # enter username
ftp> pass s3cr3t          # enter password (or it will prompt)
ftp> pwd                  # print working directory
ftp> ls                   # list files
ftp> get hello.txt        # download a file
ftp> put myfile.txt       # upload a file
ftp> mkdir uploads        # create a directory
ftp> bye                  # exit
```

> The built-in `ftp` command on Windows and macOS does not support FTPS. For FTPS testing, use `lftp` or FileZilla.

### `lftp` — Advanced Command-Line Client (Linux/macOS)

```bash
# Install: sudo apt install lftp  OR  brew install lftp

# Connect (plain FTP)
lftp -u alice,s3cr3t ftp://localhost:2121

# Connect with FTPS (explicit TLS)
lftp -e "set ftp:ssl-force true; set ssl:verify-certificate false; open ftp://alice:s3cr3t@localhost:2121"
# set ssl:verify-certificate false — only for self-signed certs in development!

# Useful lftp commands
lftp> ls                      # list remote files
lftp> get hello.txt           # download
lftp> put localfile.txt       # upload
lftp> mirror remote/ local/   # mirror entire directory tree
lftp> mirror -R local/ remote/  # reverse mirror (upload tree)
lftp> bye
```

### FileZilla (GUI — Windows/macOS/Linux)

1. Open FileZilla → **File → Site Manager → New Site**.
2. **Host:** `localhost` | **Port:** `2121` | **Protocol:** `FTP - File Transfer Protocol`.
3. **Encryption:** `Use explicit FTP over TLS if available` (for FTPS) or `Use plain FTP`.
4. **Logon Type:** Normal | **User:** `alice` | **Password:** `s3cr3t`.
5. Click **Connect**.

For FTPS with a self-signed certificate: FileZilla will show a certificate warning — click "OK" / "Trust" to proceed (development only).

### Testing SFTP

```bash
# Built-in sftp client (available on Linux/macOS; also in Git Bash on Windows)
sftp -P 2222 alice@localhost
# Enter password when prompted

sftp> ls
sftp> put localfile.txt
sftp> get hello.txt
sftp> exit

# Or in one command (non-interactive):
echo "ls" | sftp -P 2222 -o StrictHostKeyChecking=no alice@localhost
```

### Quick Smoke Test Script (Node.js)

```javascript
// test-ftp.js — simple client test using basic-ftp (npm install basic-ftp)
const ftp = require('basic-ftp');

async function main() {
  const client = new ftp.Client();
  client.ftp.verbose = true; // log all commands and responses

  try {
    await client.access({
      host:     'localhost',
      port:     2121,
      user:     'alice',
      password: 's3cr3t',
      secure:   false, // set to true for FTPS
    });

    console.log('Connected!');
    const list = await client.list('/');
    console.log('Files:', list.map(f => f.name));

    // Upload a test file
    const { Readable } = require('stream');
    const content = Readable.from(['Hello from test script!\n']);
    await client.uploadFrom(content, 'test-upload.txt');
    console.log('Upload successful');

    // Download it back
    const { Writable } = require('stream');
    const chunks = [];
    const dest = new Writable({ write(c,_,cb) { chunks.push(c); cb(); } });
    await client.downloadTo(dest, 'test-upload.txt');
    console.log('Downloaded:', Buffer.concat(chunks).toString());
  } finally {
    client.close();
  }
}

main().catch(console.error);
```

```bash
npm install basic-ftp
node test-ftp.js
```

---

## 10. Firewall, Passive Port & NAT Configuration

### Why This Matters

The FTP data channel creates a separate TCP connection. In passive mode, the server opens a port in your configured range and tells the client to connect to it. If that port is blocked, every file transfer fails even if the control connection works fine.

### The Ports You Must Open

| Port / Range | Direction | Purpose |
|---|---|---|
| 21 (or 2121 in dev) | Inbound | FTP control channel |
| 50000–50100 (your range) | Inbound | FTP passive data connections |
| 22 (or 2222 in dev) | Inbound | SFTP (SSH) |
| 990 | Inbound | Implicit FTPS (if used) |

### Linux (iptables / ufw)

```bash
# Using ufw (simpler, Ubuntu/Debian)
sudo ufw allow 21/tcp
sudo ufw allow 50000:50100/tcp
sudo ufw allow 22/tcp
sudo ufw reload

# Using iptables directly
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 50000:50100 -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### Cloud Providers (Security Groups / Firewall Rules)

**AWS (Security Group):**
- Inbound rule: TCP 21, source: 0.0.0.0/0
- Inbound rule: TCP 50000-50100, source: 0.0.0.0/0

**DigitalOcean (Cloud Firewall):**
- Add inbound rules for TCP 21 and TCP 50000-50100.

**Azure (Network Security Group):**
- Add inbound security rules for ports 21 and 50000-50100.

### NAT / Behind a Router

If your server is behind a home or office router (NAT):

1. Set up **port forwarding** in the router for port 21 and the entire passive range (50000–50100) → your server's local IP.
2. In your server code, set `PublicHost` / `pasv_url` to your **public IP** (not the LAN IP). The server tells the client to connect to this IP for data.
3. To find your public IP: `curl ifconfig.me` or `curl icanhazip.com`.

```go
// Go — use your public IP in GetSettings()
PublicHost: "203.0.113.42",  // your real public IP
```

```javascript
// Node — use your public IP
pasv_url: '203.0.113.42',   // your real public IP
```

> **Gotcha:** If you set `PublicHost`/`pasv_url` to a LAN IP (e.g. `192.168.x.x`) but clients are connecting from the internet, passive mode will completely fail — the client tries to open a data connection to a private IP that is unreachable from outside your network.

### Checking Your Passive Port Range from Outside

```bash
# From a different machine (or use an online port checker), test one port in your range:
nc -zv YOUR_PUBLIC_IP 50000
# "Connection succeeded" means the port is open. "Connection refused/timed out" means it's blocked.

# Test the whole range with nmap:
nmap -p 50000-50010 YOUR_PUBLIC_IP
```

---

## 11. Docker Deployment Notes

FTP is notoriously tricky with Docker because of the passive port range — you must expose many ports, not just port 21.

### Docker Run (FTP server)

```bash
# Expose the control port AND the entire passive range.
# The passive range should match what you configured in your server code.
docker run -d \
  --name ftp-server \
  -p 21:21 \
  -p 50000-50100:50000-50100 \
  -v /data/ftp:/ftp-root \
  -e FTP_PASSIVE_MIN=50000 \
  -e FTP_PASSIVE_MAX=50100 \
  -e FTP_PUBLIC_HOST=YOUR_PUBLIC_IP \
  your-ftp-image:latest
```

> Exposing a range like `50000-50100` (101 ports) is perfectly valid Docker syntax but will create 101 `docker-proxy` processes, one per port. This is a known Docker limitation with FTP. Keep the range as small as practical.

### Docker Compose (FTP server)

```yaml
# docker-compose.yml
version: "3.9"
services:
  ftp:
    build: .
    ports:
      - "21:21"
      # Passive port range — must match server config.
      # Docker Compose supports ranges directly.
      - "50000-50100:50000-50100"
    volumes:
      - ftp_data:/ftp-root
    environment:
      FTP_PASSIVE_MIN: "50000"
      FTP_PASSIVE_MAX: "50100"
      # IMPORTANT: set this to the Docker host's public IP
      # so the PASV response gives clients a reachable address.
      FTP_PUBLIC_HOST: "203.0.113.42"
    restart: unless-stopped

  sftp:
    build:
      context: .
      dockerfile: Dockerfile.sftp
    ports:
      - "22:22"      # SFTP only needs one port
    volumes:
      - sftp_data:/sftp-root
      - ./host_key:/app/host_key:ro
    restart: unless-stopped

volumes:
  ftp_data:
  sftp_data:
```

### Dockerfile for the Go FTP Server

```dockerfile
# Dockerfile
# ── Build stage ──────────────────────────────────────────────────────
FROM golang:1.23-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o ftp-server ./main.go

# ── Runtime stage (minimal) ──────────────────────────────────────────
FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
# Create a non-root user
RUN addgroup -S ftpgroup && adduser -S ftpuser -G ftpgroup
WORKDIR /app
COPY --from=builder /build/ftp-server .
RUN mkdir -p /ftp-root && chown -R ftpuser:ftpgroup /ftp-root
USER ftpuser
EXPOSE 21 50000-50100
CMD ["./ftp-server"]
```

### Dockerfile for the Node.js FTP Server

```dockerfile
# Dockerfile
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

### Docker Network Mode Host (Alternative for FTP)

For development or environments where exposing a large port range is impractical, you can use `--network host` on Linux. The container shares the host's network stack, so no port mapping is needed:

```bash
docker run -d --network host your-ftp-image
```

> **Warning:** `--network host` only works on Linux Docker (not Docker Desktop for Mac/Windows) and removes all network isolation. Use only in trusted environments.

---

## 12. Security Best Practices

### Use FTPS or SFTP (Never Plain FTP in Production)

This cannot be overstated. Plain FTP sends credentials and data as cleartext. On any network where you do not control every device, assume someone is listening.

| Scenario | Recommendation |
|---|---|
| New project, full control over client | Use SFTP (simpler, more secure, works through firewalls) |
| Must use FTP protocol (legacy client compatibility) | Use FTPS with mandatory TLS |
| Completely isolated private LAN | Plain FTP is technically acceptable but still not recommended |
| Internet-facing | FTPS or SFTP, never plain FTP |

### TLS Certificate Management

```bash
# Production: use Let's Encrypt (free, auto-renewed)
# certbot is the official Let's Encrypt client
sudo certbot certonly --standalone -d ftp.yourdomain.com

# The cert files are at:
# /etc/letsencrypt/live/ftp.yourdomain.com/fullchain.pem  (certificate)
# /etc/letsencrypt/live/ftp.yourdomain.com/privkey.pem    (private key)

# Add a cron job to reload the server after renewal:
# 0 3 * * * certbot renew --quiet && systemctl restart your-ftp-service
```

### Chroot / Jail Per User

Always chroot users to their own directory. Neither the Go nor Node examples above allow a user to navigate above their root. Verify this by testing:

```bash
ftp> cd ..     # should stay in / (root of their chroot) or return an error
ftp> pwd       # should still show /
```

### No Anonymous Access in Production

```go
// Go: only authenticate users that are in your user map.
// If username is not found, return an error — never fall through to anonymous.
```

```javascript
// Node: always set anonymous: false in FtpSrv config.
anonymous: false,
```

### Strong Authentication

- Use long, random passwords or — better — implement public-key authentication (SFTP).
- Store passwords hashed with bcrypt or Argon2, never in plaintext:

```go
// Go — hashed password comparison
import "golang.org/x/crypto/bcrypt"

// When storing:
hash, _ := bcrypt.GenerateFromPassword([]byte(plainPassword), bcrypt.DefaultCost)

// When authenticating:
err := bcrypt.CompareHashAndPassword(storedHash, []byte(inputPassword)
// err == nil means the password matches
```

```javascript
// Node — hashed password comparison
const bcrypt = require('bcrypt'); // npm install bcrypt

// When storing:
const hash = await bcrypt.hash(plainPassword, 12);

// When authenticating:
const ok = await bcrypt.compare(inputPassword, storedHash);
```

### Limit the Passive Port Range

The smaller your passive port range, the smaller your attack surface. Size it for your expected maximum concurrent transfers:

- 100 ports → 100 simultaneous file transfers (usually plenty).
- Do not open thousands of ports just in case.

### Rate Limiting and IP Blocking

```go
// Go: implement rate limiting in ClientConnected()
// Track connection attempts per IP and reject after N failures.
var (
	mu         sync.Mutex
	failCounts = map[string]int{}
)

func (d *MainDriver) ClientConnected(cc ftpserver.ClientContext) (string, error) {
	ip := cc.RemoteAddr().String()
	mu.Lock()
	defer mu.Unlock()
	if failCounts[ip] > 10 {
		return "", errors.New("too many failed attempts — IP temporarily blocked")
	}
	return "Welcome", nil
}
```

```javascript
// Node: use a simple in-memory counter; for production use Redis + a proper rate limiter.
const failCounts = {};

ftpServer.on('login', ({ connection, username, password }, resolve, reject) => {
  const ip = connection.ip;
  failCounts[ip] = (failCounts[ip] || 0);

  const user = USERS[username];
  if (!user || user.password !== password) {
    failCounts[ip]++;
    if (failCounts[ip] > 5) {
      return reject(new Error('Too many failed attempts'));
    }
    return reject(new Error('Invalid credentials'));
  }

  failCounts[ip] = 0; // reset on successful login
  return resolve({ root: user.root });
});
```

### File Permission Hardening

```bash
# FTP root directories should not be world-writable.
chmod 750 /ftp-root
chmod 750 /ftp-root/alice
chown -R ftpuser:ftpgroup /ftp-root

# Upload directories can be 770 (group-writable) if needed.
chmod 770 /ftp-root/alice/uploads
```

---

## 13. Tips, Tricks & Gotchas

**Do:**
- Always configure and test **passive mode** — active mode fails behind NAT/firewalls.
- Set `PublicHost`/`pasv_url` to the **public IP** of your server, not the LAN IP.
- Use **FTPS or SFTP** in any production environment.
- Open **both** port 21 and the passive port range in your firewall.
- Chroot every user to their own directory (`afero.NewBasePathFs` in Go, `root` in ftp-srv).
- Use **binary transfer mode** (TYPE I) for non-text files — ASCII mode can corrupt binary files by translating line endings.
- Test from a **different machine** on a different network, not just localhost.
- Keep the passive port range **small** (100-200 ports is usually enough).
- Set an **idle timeout** to disconnect abandoned sessions.
- Log connection events including the remote IP for security auditing.

**Don't / Common Bugs:**
- Never set `PublicHost` to a LAN IP (`192.168.x.x`) for an internet-facing server — passive mode will silently fail for external clients.
- Never omit the passive port range configuration and then wonder why `LIST` and `RETR` hang.
- Do not expose port 20 (active mode data) unless you specifically need active mode.
- Do not use ASCII transfer mode for binary files (images, PDFs, ZIPs, executables).
- Do not rely on the built-in `ftp` command for FTPS testing — it does not support TLS; use `lftp` or FileZilla.
- Do not expose the server on port 21 if you can avoid it (use a non-standard port in dev to avoid needing root).
- Do not store passwords in plaintext — hash them with bcrypt/Argon2.
- Do not allow anonymous access on production servers.
- Do not forget to open the passive port range in Docker's `-p` flags — just exposing port 21 will allow login but all transfers will fail.
- Do not use a self-signed certificate in production — use Let's Encrypt.
- Passive mode and NAT: if the PASV response returns a local IP address (visible in `lftp` verbose mode or Wireshark), your `PublicHost`/`pasv_url` is misconfigured.

**Tricks:**
- Use `lftp` with `set ftp:passive-mode on` and `debug 5` to see every FTP command and response — invaluable for debugging passive mode issues.
- In FileZilla, enable `View → Message Log` to see the raw FTP conversation.
- `nc -zv HOST PORT` is the fastest way to check if a passive port is reachable.
- For the SFTP server in Go, `sftp.WithServerWorkingDirectory()` automatically chroots the session and handles path translations.
- For integration tests, use `github.com/fclairamb/ftpserverlib` in test mode with an in-memory `afero.MemMapFs` — no disk I/O, fully isolated, fast.
- `basic-ftp` (Node) is an excellent lightweight FTP client library for writing automated tests against your FTP server.
- FTP sessions are stateful and long-lived — implement a NOOP heartbeat in long-running clients to prevent idle timeouts.
- When building a multi-tenant FTP service, generate per-user TLS client certificates and use `tls.RequireAndVerifyClientCert` in addition to password auth for defense in depth.

---

## 14. Study Path

Learn in this order for fastest offline mastery:

1. **Protocol fundamentals** — understand control vs data channel, active vs passive mode (§1). This is the hardest conceptual part; everything else is code.
2. **FTP vs FTPS vs SFTP** — memorize the distinction (§2). It will come up in every real-world conversation about file transfer security.
3. **Commands and response codes** — skim the tables (§3); return to them when debugging.
4. **Run the Go or Node example** with plain FTP (§4 or §6) — get a server running locally, connect with `ftp` or FileZilla, upload and download a file.
5. **Add FTPS** (§5 or §7) — generate a self-signed cert, enable TLS, test with `lftp`.
6. **Try the SFTP skeleton** (§8) — compare the simplicity of SFTP vs the two-channel FTP setup.
7. **Test from a second machine** (§9) — connects theory to reality; passive port issues often only appear with real network hops.
8. **Deploy to a VPS behind a firewall** (§10, §11) — configure passive ports in the firewall, set the public IP, use Docker Compose.
9. **Add security hardening** (§12) — bcrypt passwords, rate limiting, chroot verification, Let's Encrypt cert.

### Build-to-Learn Projects

1. **Local FTP drop box** — run the Go or Node server on your machine, connect with FileZilla, drag-and-drop files between your laptop and a friend's. Reinforce: passive mode, chroot, basic auth.

2. **Multi-user FTPS server on a VPS** — deploy to a $5 DigitalOcean droplet, add Let's Encrypt, create 3 users with isolated directories, verify that user A cannot see user B's files.

3. **SFTP-based backup system** — write a small Go or Node client that rsync-style mirrors a local folder to the SFTP server on a schedule. Reinforce: SFTP client usage, SSH key auth, delta uploads.

4. **FTP server with upload notifications** — extend the Node `ftp-srv` server to fire a webhook (or send an email) when a file is uploaded. Reinforce: event-driven architecture, integrating FTP with the rest of your stack.

5. **Drop-in replacement for an existing FTP integration** — take a legacy system that currently uploads to a vendor FTP server and redirect it to your own server (using the same credentials/paths). Reinforce: FTP compatibility, real-world protocol compliance, debugging with Wireshark/lftp verbose.

> The single most important skill from this guide: understanding that passive mode exists because the client is behind NAT and the server is on the public internet — and that the server must therefore announce its **public IP** in the PASV response so the client can connect to it. Get that concept rock-solid and the rest of the configuration falls into place.
