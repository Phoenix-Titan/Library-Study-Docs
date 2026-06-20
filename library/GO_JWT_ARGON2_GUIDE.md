# Go Authentication: JWT & Argon2 Complete Guide

> **Who this is for:** Go developers who want to build secure, production-ready authentication from scratch — password hashing with Argon2id and stateless sessions with JSON Web Tokens. Everything here is self-contained and offline-friendly. Libraries used: `github.com/golang-jwt/jwt/v5` and `golang.org/x/crypto/argon2`. Where an API changed recently or has version-specific behaviour, it is flagged with **⚡ Version note**.

---

## Table of Contents

**PART A — Argon2 Password Hashing**
1. [Why Hash Passwords & Why Argon2id](#1-why-hash-passwords--why-argon2id)
2. [Generating a Salt & Calling argon2.IDKey](#2-generating-a-salt--calling-argon2idkey)
3. [PHC String Encoding](#3-phc-string-encoding)
4. [Verifying a Password (Constant-Time Compare)](#4-verifying-a-password-constant-time-compare)
5. [Full Reusable HashPassword / VerifyPassword](#5-full-reusable-hashpassword--verifypassword)

**PART B — JSON Web Tokens**
6. [What a JWT Is & Signing Algorithms](#6-what-a-jwt-is--signing-algorithms)
7. [Creating & Signing a Token](#7-creating--signing-a-token)
8. [Parsing & Validating a Token](#8-parsing--validating-a-token)
9. [Access + Refresh Token Pattern](#9-access--refresh-token-pattern)

**PART C — Putting It Together**
10. [Full Auth Flow with net/http](#10-full-auth-flow-with-nethttp)
11. [Role-Based Authorization in Claims](#11-role-based-authorization-in-claims)
12. [Security Best Practices & Gotchas](#12-security-best-practices--gotchas)
13. [Study Path](#13-study-path)

---

## PART A — Argon2 Password Hashing

---

## 1. Why Hash Passwords & Why Argon2id

### The Core Problem

When a user registers, you must **never store their plaintext password**. If your database leaks, attackers must not be able to recover passwords. The solution is a one-way hash — but not just any hash.

**Why plain SHA-256/MD5 is catastrophically wrong:**
- They are designed to be *fast* — a modern GPU can test billions of SHA-256 guesses per second.
- Pre-built "rainbow tables" map common passwords to their hashes.
- No salt means identical passwords produce identical hashes.

**Why bcrypt is better but not ideal:**
- bcrypt is intentionally slow and includes a built-in salt.
- However, it is limited to 72-byte inputs and is CPU-bound — modern ASICs and GPUs still crack it faster than it was designed for.
- Memory usage is fixed at ~4 KB, making GPU attacks cheap.

**Why Argon2id is the current gold standard:**
- Winner of the 2015 Password Hashing Competition (PHC).
- **Memory-hard**: forces attackers to use large amounts of RAM per attempt, pricing out GPU/ASIC farms.
- Three variants: Argon2d (GPU-resistant), Argon2i (side-channel-resistant), **Argon2id** (hybrid — recommended by OWASP for general password hashing).
- Parameters are tunable: you can make it harder as hardware improves without changing the API.

### OWASP Recommended Parameters (2025/2026)

| Parameter | Minimum | Recommended |
|-----------|---------|-------------|
| Memory (`m`) | 19 MiB (19456 KB) | 64 MiB (65536 KB) |
| Iterations (`t`) | 2 | 3 |
| Parallelism (`p`) | 1 | 4 |
| Hash output length | 32 bytes | 32 bytes |
| Salt length | 16 bytes | 16 bytes |

> **Source:** OWASP Password Storage Cheat Sheet (2025). The "recommended" column targets ~0.5–1 second compute time on a modern server. **Always benchmark on your own hardware** and tune so hashing takes 200–500 ms — long enough to deter attackers, short enough for users.

### Go module setup

```bash
# Initialize your module
go mod init github.com/yourname/authexample

# Add the crypto package (provides argon2)
go get golang.org/x/crypto

# Add the JWT library
go get github.com/golang-jwt/jwt/v5
```

---

## 2. Generating a Salt & Calling argon2.IDKey

A **salt** is a random value mixed into the hash so that:
1. Two users with the same password get different hashes.
2. Pre-computed rainbow tables are useless.

**Always generate the salt with `crypto/rand`**, not `math/rand`. The `crypto/rand` reader is a CSPRNG (cryptographically secure pseudo-random number generator) backed by the OS entropy pool.

```go
package main

import (
	"crypto/rand"
	"fmt"
	"log"

	"golang.org/x/crypto/argon2"
)

// Argon2Params holds the tunable parameters for Argon2id.
// Store these alongside the hash so you can adjust them over time
// without breaking existing password verification.
type Argon2Params struct {
	Memory      uint32 // KiB of memory to use
	Iterations  uint32 // number of passes over the memory
	Parallelism uint8  // degree of parallelism (threads)
	SaltLength  uint32 // bytes of random salt to generate
	KeyLength   uint32 // bytes in the output hash
}

// DefaultParams are OWASP-recommended settings for 2025/2026.
// Tune Memory and Iterations upward if your server can handle it.
var DefaultParams = &Argon2Params{
	Memory:      64 * 1024, // 64 MiB expressed in KiB
	Iterations:  3,
	Parallelism: 4,
	SaltLength:  16,
	KeyLength:   32,
}

// generateSalt returns cryptographically random bytes of the given length.
// It panics only if the OS random source is broken — that should never
// happen on a healthy server, but we log.Fatal here for visibility.
func generateSalt(n uint32) ([]byte, error) {
	salt := make([]byte, n)
	if _, err := rand.Read(salt); err != nil {
		return nil, fmt.Errorf("generateSalt: %w", err)
	}
	return salt, nil
}

func main() {
	password := "correct-horse-battery-staple"
	params := DefaultParams

	// Step 1 — generate a fresh random salt
	salt, err := generateSalt(params.SaltLength)
	if err != nil {
		log.Fatal(err)
	}

	// Step 2 — derive the hash key with Argon2id
	// argon2.IDKey(password, salt, time, memory, threads, keyLen)
	//   time      → iterations (t in the PHC string)
	//   memory    → KiB of RAM to use (m in the PHC string)
	//   threads   → degree of parallelism (p in the PHC string)
	//   keyLen    → output length in bytes
	hash := argon2.IDKey(
		[]byte(password),
		salt,
		params.Iterations,
		params.Memory,
		params.Parallelism,
		params.KeyLength,
	)

	fmt.Printf("Salt (hex): %x\n", salt)
	fmt.Printf("Hash (hex): %x\n", hash)
	// ⚠️  Never store raw hex like this — see Section 3 for the PHC format.
}
```

> **⚡ Version note:** `golang.org/x/crypto/argon2` is part of the extended standard library. The `IDKey` function signature has been stable since Go 1.13. Always import `golang.org/x/crypto` from the module proxy — do not vendor a fork.

**Why these exact parameter names matter:**

| `argon2.IDKey` argument | PHC field | What it controls |
|-------------------------|-----------|-----------------|
| `time` (uint32) | `t=` | Number of passes over the memory buffer — increases CPU cost |
| `memory` (uint32) | `m=` | KiB of RAM allocated — the key differentiator vs bcrypt |
| `threads` (uint8) | `p=` | Parallelism — use number of CPU cores available to the hash |
| `keyLen` (uint32) | output | Bytes of derived key — 32 bytes = 256 bits, more than enough |

---

## 3. PHC String Encoding

A raw 32-byte hash blob is useless without knowing the algorithm and parameters used to create it. The **PHC string format** bundles everything into a single human-readable, self-describing string:

```
$argon2id$v=19$m=65536,t=3,p=4$<base64-salt>$<base64-hash>
```

This means you can:
- Store a single column in your database.
- Change parameters in the future — old hashes keep working because the params are embedded.
- Identify the algorithm in case you later support multiple algorithms.

```go
package main

import (
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"log"

	"golang.org/x/crypto/argon2"
)

type Argon2Params struct {
	Memory      uint32
	Iterations  uint32
	Parallelism uint8
	SaltLength  uint32
	KeyLength   uint32
}

var DefaultParams = &Argon2Params{
	Memory:      64 * 1024,
	Iterations:  3,
	Parallelism: 4,
	SaltLength:  16,
	KeyLength:   32,
}

// encodeHash formats the salt and hash into a PHC string.
// The format is:
//   $argon2id$v=<version>$m=<memory>,t=<iterations>,p=<parallelism>$<b64salt>$<b64hash>
//
// We use base64.RawStdEncoding (no padding, standard alphabet) which is
// the encoding specified by the PHC draft.
func encodeHash(params *Argon2Params, salt, hash []byte) string {
	// base64.RawStdEncoding omits '=' padding characters — required by PHC spec.
	b64Salt := base64.RawStdEncoding.EncodeToString(salt)
	b64Hash := base64.RawStdEncoding.EncodeToString(hash)

	// argon2.Version is the constant 0x13 == 19, which is the current Argon2 spec version.
	return fmt.Sprintf(
		"$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version,   // always 19
		params.Memory,
		params.Iterations,
		params.Parallelism,
		b64Salt,
		b64Hash,
	)
}

func main() {
	params := DefaultParams
	password := "correct-horse-battery-staple"

	salt := make([]byte, params.SaltLength)
	if _, err := rand.Read(salt); err != nil {
		log.Fatal(err)
	}

	hash := argon2.IDKey(
		[]byte(password),
		salt,
		params.Iterations,
		params.Memory,
		params.Parallelism,
		params.KeyLength,
	)

	encoded := encodeHash(params, salt, hash)
	fmt.Println(encoded)
	// Example output:
	// $argon2id$v=19$m=65536,t=3,p=4$FNk/...==$2X3N...==
}
```

**What to store in your database:**

```sql
-- Users table: one column holds the entire encoded hash.
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,  -- stores the full PHC string
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. Verifying a Password (Constant-Time Compare)

Verification requires:
1. **Parsing** the stored PHC string back into params, salt, and hash.
2. **Re-running** Argon2id with the *same* params and salt on the candidate password.
3. **Comparing** the two hashes with `subtle.ConstantTimeCompare`.

Step 3 is critical. A regular `bytes.Equal` or `==` comparison short-circuits as soon as it finds a mismatch — this leaks timing information that attackers can use to mount a timing attack. `subtle.ConstantTimeCompare` always takes the same time regardless of where the mismatch is.

```go
package main

import (
	"crypto/subtle"
	"encoding/base64"
	"errors"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

// Sentinel errors — use these to distinguish "wrong password" from
// "the stored hash is corrupted/invalid".
var (
	ErrInvalidHash         = errors.New("argon2: the encoded hash is not in the correct format")
	ErrIncompatibleVersion = errors.New("argon2: incompatible version of argon2")
)

type Argon2Params struct {
	Memory      uint32
	Iterations  uint32
	Parallelism uint8
	SaltLength  uint32
	KeyLength   uint32
}

// decodeHash parses a PHC-format Argon2id string into its component parts.
// It validates the algorithm identifier and version before returning.
func decodeHash(encodedHash string) (params *Argon2Params, salt, hash []byte, err error) {
	// Expected format:
	// $argon2id$v=19$m=65536,t=3,p=4$<b64salt>$<b64hash>
	// Split on '$'; first element is always empty (leading '$')
	vals := strings.Split(encodedHash, "$")
	if len(vals) != 6 {
		return nil, nil, nil, ErrInvalidHash
	}

	// vals[0] = ""       (before the first $)
	// vals[1] = "argon2id"
	// vals[2] = "v=19"
	// vals[3] = "m=65536,t=3,p=4"
	// vals[4] = "<b64salt>"
	// vals[5] = "<b64hash>"

	if vals[1] != "argon2id" {
		return nil, nil, nil, ErrInvalidHash
	}

	// Parse the version
	var version int
	if _, scanErr := fmt.Sscanf(vals[2], "v=%d", &version); scanErr != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	if version != argon2.Version {
		return nil, nil, nil, ErrIncompatibleVersion
	}

	// Parse m, t, p
	params = &Argon2Params{}
	if _, scanErr := fmt.Sscanf(
		vals[3], "m=%d,t=%d,p=%d",
		&params.Memory, &params.Iterations, &params.Parallelism,
	); scanErr != nil {
		return nil, nil, nil, ErrInvalidHash
	}

	// Decode the base64 salt (no padding)
	salt, err = base64.RawStdEncoding.DecodeString(vals[4])
	if err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	params.SaltLength = uint32(len(salt))

	// Decode the base64 hash (no padding)
	hash, err = base64.RawStdEncoding.DecodeString(vals[5])
	if err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	params.KeyLength = uint32(len(hash))

	return params, salt, hash, nil
}

// verifyPassword checks a plaintext password against a PHC-encoded Argon2id hash.
// Returns true if the password matches, false otherwise.
// Returns an error only for malformed/incompatible hashes — NOT for wrong passwords.
func verifyPassword(password, encodedHash string) (bool, error) {
	params, salt, hash, err := decodeHash(encodedHash)
	if err != nil {
		return false, err
	}

	// Re-derive the key using the SAME parameters extracted from the stored hash.
	// This is the key insight: the parameters travel with the hash.
	candidateHash := argon2.IDKey(
		[]byte(password),
		salt,
		params.Iterations,
		params.Memory,
		params.Parallelism,
		params.KeyLength,
	)

	// subtle.ConstantTimeCompare returns 1 if equal, 0 if not.
	// It ALWAYS reads both slices fully — no early exit on mismatch.
	// This prevents timing attacks that could reveal partial hash information.
	if subtle.ConstantTimeCompare(hash, candidateHash) == 1 {
		return true, nil
	}

	return false, nil
}
```

**Timing attack explained:**

```
// WRONG — leaks information via timing:
if bytes.Equal(storedHash, candidateHash) { ... }

// RIGHT — constant time regardless of where they differ:
if subtle.ConstantTimeCompare(storedHash, candidateHash) == 1 { ... }

// The attacker's strategy with a timing oracle:
// Send thousands of slightly different passwords.
// Measure response times.
// Longer time = matched more bytes = narrowing in on the hash.
// subtle.ConstantTimeCompare eliminates this channel.
```

---

## 5. Full Reusable HashPassword / VerifyPassword

Here is the complete, production-ready `password` package you can drop into any project:

```go
// package password provides Argon2id password hashing and verification.
// It implements the PHC string format so parameters are stored with the hash,
// allowing future parameter upgrades without breaking existing passwords.
package password

import (
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
	"errors"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

// Sentinel errors
var (
	ErrInvalidHash         = errors.New("password: encoded hash is not a valid PHC Argon2id string")
	ErrIncompatibleVersion = errors.New("password: argon2 version mismatch — re-hash required")
	ErrMismatch            = errors.New("password: password does not match hash")
)

// Params configures the Argon2id key derivation function.
// Use DefaultParams() for OWASP-recommended settings.
// Tune on your own hardware so Hash() takes 200–500 ms.
type Params struct {
	Memory      uint32 // KiB of RAM (e.g. 65536 = 64 MiB)
	Iterations  uint32 // passes over the memory buffer
	Parallelism uint8  // concurrent threads
	SaltLength  uint32 // random bytes for the salt
	KeyLength   uint32 // output hash size in bytes
}

// DefaultParams returns OWASP-recommended Argon2id parameters for 2025/2026.
// Always call this — do not hard-code params at call sites so they stay in sync.
func DefaultParams() *Params {
	return &Params{
		Memory:      64 * 1024, // 64 MiB
		Iterations:  3,
		Parallelism: 4,
		SaltLength:  16,
		KeyLength:   32,
	}
}

// Hash derives an Argon2id hash from password using params and returns
// a self-describing PHC string safe to store directly in a database column.
//
// Example output:
//   $argon2id$v=19$m=65536,t=3,p=4$3Z7....$Xb4....
func Hash(password string, params *Params) (string, error) {
	if params == nil {
		params = DefaultParams()
	}

	// Generate cryptographically random salt.
	salt := make([]byte, params.SaltLength)
	if _, err := rand.Read(salt); err != nil {
		return "", fmt.Errorf("password.Hash: generating salt: %w", err)
	}

	// Derive the key.
	key := argon2.IDKey(
		[]byte(password),
		salt,
		params.Iterations,
		params.Memory,
		params.Parallelism,
		params.KeyLength,
	)

	// Encode to PHC string.
	encoded := fmt.Sprintf(
		"$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version,
		params.Memory,
		params.Iterations,
		params.Parallelism,
		base64.RawStdEncoding.EncodeToString(salt),
		base64.RawStdEncoding.EncodeToString(key),
	)

	return encoded, nil
}

// Verify checks password against an encoded PHC Argon2id hash.
//
// Returns:
//   - nil if the password is correct.
//   - ErrMismatch if the password is wrong.
//   - ErrInvalidHash or ErrIncompatibleVersion if the stored hash is malformed.
//
// IMPORTANT: This function deliberately does not distinguish between
// ErrMismatch and "user not found" at the HTTP layer — callers should
// return the same generic error to the client in both cases.
func Verify(password, encodedHash string) error {
	params, salt, storedKey, err := decode(encodedHash)
	if err != nil {
		return err
	}

	candidateKey := argon2.IDKey(
		[]byte(password),
		salt,
		params.Iterations,
		params.Memory,
		params.Parallelism,
		params.KeyLength,
	)

	// Constant-time comparison — mandatory to prevent timing attacks.
	if subtle.ConstantTimeCompare(storedKey, candidateKey) != 1 {
		return ErrMismatch
	}

	return nil
}

// NeedsRehash reports whether the stored hash was produced with different
// parameters than p. Use this after a successful login to silently upgrade
// old hashes when you raise the security parameters.
//
//	ok, err := password.Verify(plaintext, stored)
//	if ok == nil && password.NeedsRehash(stored, password.DefaultParams()) {
//	    newHash, _ := password.Hash(plaintext, password.DefaultParams())
//	    db.UpdatePasswordHash(userID, newHash)
//	}
func NeedsRehash(encodedHash string, p *Params) bool {
	stored, _, _, err := decode(encodedHash)
	if err != nil {
		return true // malformed — definitely re-hash
	}
	return stored.Memory != p.Memory ||
		stored.Iterations != p.Iterations ||
		stored.Parallelism != p.Parallelism ||
		stored.KeyLength != p.KeyLength
}

// decode parses a PHC Argon2id string into its constituent parts.
func decode(encodedHash string) (p *Params, salt, key []byte, err error) {
	parts := strings.Split(encodedHash, "$")
	if len(parts) != 6 {
		return nil, nil, nil, ErrInvalidHash
	}
	// parts: ["", "argon2id", "v=19", "m=65536,t=3,p=4", "<b64salt>", "<b64hash>"]

	if parts[1] != "argon2id" {
		return nil, nil, nil, ErrInvalidHash
	}

	var version int
	if _, e := fmt.Sscanf(parts[2], "v=%d", &version); e != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	if version != argon2.Version {
		return nil, nil, nil, ErrIncompatibleVersion
	}

	p = &Params{}
	if _, e := fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d",
		&p.Memory, &p.Iterations, &p.Parallelism); e != nil {
		return nil, nil, nil, ErrInvalidHash
	}

	salt, err = base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	p.SaltLength = uint32(len(salt))

	key, err = base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	p.KeyLength = uint32(len(key))

	return p, salt, key, nil
}
```

**Usage:**

```go
package main

import (
	"fmt"
	"log"

	"github.com/yourname/authexample/password"
)

func main() {
	plain := "hunter2"

	// Hash on registration
	encoded, err := password.Hash(plain, password.DefaultParams())
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Stored:", encoded)

	// Verify on login
	if err := password.Verify(plain, encoded); err != nil {
		fmt.Println("Login failed:", err)
	} else {
		fmt.Println("Login OK")
	}

	// Verify wrong password
	if err := password.Verify("wrong", encoded); err != nil {
		fmt.Println("Wrong password:", err) // password: password does not match hash
	}
}
```

---

## PART B — JSON Web Tokens

---

## 6. What a JWT Is & Signing Algorithms

### Anatomy of a JWT

A JWT is three base64url-encoded JSON objects joined by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMjM0IiwiZW1haWwiOiJhbGljZUBleGFtcGxlLmNvbSIsImV4cCI6MTc1MDAwMDAwMH0
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Part 1 — Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Part 2 — Payload (Claims):**
```json
{
  "sub": "1234",
  "email": "alice@example.com",
  "exp": 1750000000
}
```

**Part 3 — Signature:**
The server's HMAC-SHA256 (or RSA/ECDSA signature) over `base64url(header) + "." + base64url(payload)`. This prevents tampering — anyone can *read* the payload, but only the server can *produce* a valid signature.

> **Critical misconception:** JWTs are **not encrypted** by default. Never put sensitive data (SSNs, credit cards, passwords) in the payload. The signature only proves authenticity and integrity — the payload is readable by anyone who holds the token.

### Registered Claims (RFC 7519)

| Claim | Name | Type | Purpose |
|-------|------|------|---------|
| `iss` | Issuer | string | Who created the token (e.g., `"auth.example.com"`) |
| `sub` | Subject | string | Who the token is about (usually user ID) |
| `aud` | Audience | string/[]string | Intended recipients |
| `exp` | Expiration | Unix timestamp | Token must be rejected after this time |
| `nbf` | Not Before | Unix timestamp | Token must not be accepted before this time |
| `iat` | Issued At | Unix timestamp | When the token was created |
| `jti` | JWT ID | string | Unique identifier — used for revocation |

### Signing Algorithms: HS256 vs RS256 vs ES256

| Algorithm | Type | Key | When to use |
|-----------|------|-----|-------------|
| **HS256** | Symmetric HMAC-SHA256 | Shared secret ([]byte) | Single-service auth — issuer and verifier are the same server |
| **HS384 / HS512** | Symmetric HMAC | Longer shared secret | Same as HS256, marginally more bits |
| **RS256** | Asymmetric RSA-SHA256 | Private key signs, public key verifies | Microservices — many services verify but only one issues |
| **ES256** | Asymmetric ECDSA-SHA256 | Elliptic curve key pair | Same as RS256 but smaller tokens and faster verification |
| **PS256** | RSA-PSS-SHA256 | RSA key pair | Like RS256 but with probabilistic signature padding |

**Rule of thumb:**
- **Single service** (monolith, one API): HS256 with a 32+ byte random secret.
- **Many services verify tokens** (microservices, third-party APIs need to verify): RS256 or ES256 — publish your public key, keep the private key only on the auth server.

> **⚡ Version note (`github.com/golang-jwt/jwt/v5`):** In v5, the library defaults to validating the `exp` claim automatically when you call `ParseWithClaims`. The old `MapClaims` pattern is still supported but discouraged. Prefer typed claims structs (shown below).

---

## 7. Creating & Signing a Token

```go
// package token provides JWT creation and validation for the auth service.
package token

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// CustomClaims embeds jwt.RegisteredClaims (which contains sub, exp, iat, iss, etc.)
// and adds application-specific fields.
//
// ⚡ Version note (v5): Always embed jwt.RegisteredClaims (not jwt.StandardClaims
// which was removed in v5). The fields are value types, not pointer types.
type CustomClaims struct {
	// Embed the standard registered claims.
	jwt.RegisteredClaims

	// Application-specific claims.
	// Keep this lean — the token is sent with every request.
	// Never include passwords, SSNs, or PII beyond what the client needs.
	Email string `json:"email"`
	Role  string `json:"role"`
}

// TokenConfig holds secrets and durations needed to issue tokens.
// Load these from environment variables — never hard-code in source.
type TokenConfig struct {
	AccessSecret  []byte        // HS256 secret for access tokens — min 32 random bytes
	RefreshSecret []byte        // separate HS256 secret for refresh tokens
	AccessTTL     time.Duration // e.g. 15 * time.Minute
	RefreshTTL    time.Duration // e.g. 7 * 24 * time.Hour
	Issuer        string        // e.g. "auth.example.com"
}

// NewAccessToken creates and signs an access token for the given user.
// The token is valid for cfg.AccessTTL from now.
func NewAccessToken(userID, email, role string, cfg *TokenConfig) (string, error) {
	now := time.Now()

	claims := CustomClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			// Subject is the user identifier — must be stable (use DB primary key, not email)
			Subject: userID,
			// Issuer tells consumers which service signed this token
			Issuer: cfg.Issuer,
			// IssuedAt and ExpiresAt are automatically validated by the parser
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(cfg.AccessTTL)),
			// NotBefore can lock a token to not be usable until a future time
			NotBefore: jwt.NewNumericDate(now),
		},
		Email: email,
		Role:  role,
	}

	// Create the token object with the HS256 signing method and your claims.
	// jwt.SigningMethodHS256 is a *jwt.SigningMethodHMAC value defined by the library.
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

	// Sign the token with the secret — this produces the final "header.payload.signature" string.
	// cfg.AccessSecret must be at least 32 random bytes for HS256.
	signed, err := token.SignedString(cfg.AccessSecret)
	if err != nil {
		return "", fmt.Errorf("token.NewAccessToken: signing: %w", err)
	}

	return signed, nil
}

// Example of creating a token at the call site:
//
//   cfg := &token.TokenConfig{
//       AccessSecret:  []byte(os.Getenv("JWT_ACCESS_SECRET")),
//       RefreshSecret: []byte(os.Getenv("JWT_REFRESH_SECRET")),
//       AccessTTL:     15 * time.Minute,
//       RefreshTTL:    7 * 24 * time.Hour,
//       Issuer:        "auth.example.com",
//   }
//
//   accessToken, err := token.NewAccessToken("42", "alice@example.com", "admin", cfg)
```

**Generating a strong secret (shell):**

```bash
# Generate 32 cryptographically random bytes, hex-encoded (64 hex chars = 256 bits).
# Store this in your environment: JWT_ACCESS_SECRET=<output>
openssl rand -hex 32

# Or with Go:
# import "crypto/rand"; import "encoding/hex"
# b := make([]byte, 32); rand.Read(b); fmt.Println(hex.EncodeToString(b))
```

---

## 8. Parsing & Validating a Token

Parsing is where most security bugs live. The library's `ParseWithClaims` handles expiry, not-before, and signature validation — but **you must provide the key function correctly** to prevent algorithm confusion attacks.

```go
package token

import (
	"errors"
	"fmt"

	"github.com/golang-jwt/jwt/v5"
)

// ParseAccessToken validates a raw JWT string and extracts the claims.
// It enforces:
//   - Valid signature (using the correct HMAC secret)
//   - Algorithm must be HS256 (prevents alg-confusion attacks)
//   - Token not expired (exp claim)
//   - Token not used before its NotBefore time (nbf claim)
//
// Returns the typed *CustomClaims on success, error on any failure.
func ParseAccessToken(tokenString string, cfg *TokenConfig) (*CustomClaims, error) {
	// ParseWithClaims parses, validates signature, and populates your claims struct.
	// The keyFunc is called with the parsed (but not yet verified) token header.
	// This is your chance to enforce the algorithm BEFORE trusting any data.
	token, err := jwt.ParseWithClaims(
		tokenString,
		&CustomClaims{},
		func(t *jwt.Token) (interface{}, error) {
			// ⚠️  ALGORITHM CONFUSION ATTACK PREVENTION — critical security check.
			//
			// Without this check, an attacker could:
			// 1. Take a token signed with HS256.
			// 2. Change the header to "alg": "none".
			// 3. Strip the signature.
			// 4. Some naive implementations would accept it because they didn't
			//    validate which algorithm was actually used.
			//
			// Verify the algorithm is exactly what you expect:
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}

			// Return the secret — the library uses this to verify the signature.
			return cfg.AccessSecret, nil
		},
		// ⚡ Version note (v5): Pass validation options as variadic args.
		// These are applied in addition to the automatic exp/nbf checks.
		jwt.WithIssuer(cfg.Issuer),           // validates the 'iss' claim
		jwt.WithExpirationRequired(),          // explicitly require exp claim to be present
		jwt.WithValidMethods([]string{"HS256"}), // belt-and-suspenders alg whitelist
	)
	if err != nil {
		// The library returns wrapped errors — check for specific causes.
		switch {
		case errors.Is(err, jwt.ErrTokenExpired):
			return nil, fmt.Errorf("token expired: %w", err)
		case errors.Is(err, jwt.ErrTokenNotValidYet):
			return nil, fmt.Errorf("token not yet valid: %w", err)
		case errors.Is(err, jwt.ErrSignatureInvalid):
			return nil, fmt.Errorf("token signature invalid: %w", err)
		default:
			return nil, fmt.Errorf("token invalid: %w", err)
		}
	}

	// Type-assert the claims — ParseWithClaims returns jwt.Claims interface.
	claims, ok := token.Claims.(*CustomClaims)
	if !ok || !token.Valid {
		return nil, fmt.Errorf("token: could not extract claims")
	}

	return claims, nil
}
```

**Algorithm confusion — what it looks like and why it matters:**

```
// An attacker creates a forged "none" algorithm token:
// Header:  {"alg": "none", "typ": "JWT"}
// Payload: {"sub": "admin", "role": "superuser", "exp": 9999999999}
// Signature: (empty)
//
// If the server doesn't check t.Method before trusting, the attacker
// gets admin access. The check:
//   if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok { return nil, err }
// prevents this by rejecting any non-HMAC method before key lookup.
//
// jwt.WithValidMethods([]string{"HS256"}) adds a second layer.
```

**Validate for RS256 (asymmetric):**

```go
// For RS256, the keyFunc returns the *rsa.PublicKey instead of the HMAC []byte.
// The library handles the rest identically from the caller's perspective.

import (
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"os"

	"github.com/golang-jwt/jwt/v5"
)

// LoadPublicKey reads an RSA public key from a PEM file.
func LoadPublicKey(path string) (*rsa.PublicKey, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, fmt.Errorf("failed to decode PEM block")
	}
	pub, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	rsaPub, ok := pub.(*rsa.PublicKey)
	if !ok {
		return nil, fmt.Errorf("not an RSA public key")
	}
	return rsaPub, nil
}

// RS256 keyFunc — note the algorithm check changes to *jwt.SigningMethodRSA
func rs256KeyFunc(pubKey *rsa.PublicKey) jwt.Keyfunc {
	return func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return pubKey, nil
	}
}
```

---

## 9. Access + Refresh Token Pattern

### Why Two Tokens?

A single long-lived token is a liability — if stolen, it works for days or weeks. The access + refresh pattern limits exposure:

| Token | TTL | Storage | Purpose |
|-------|-----|---------|---------|
| **Access token** | 5–15 minutes | Memory / JS variable | Sent with every API request |
| **Refresh token** | 7–30 days | httpOnly cookie / secure storage | Exchanges for a new access token when the access token expires |

### The Flow

```
1. User logs in → server issues access_token (15m) + refresh_token (7d)
2. Client stores refresh_token in httpOnly cookie, access_token in memory
3. Client sends access_token in Authorization: Bearer <token> header
4. When access_token expires (401 response) → client hits /auth/refresh
5. Server validates refresh_token, issues new access_token (+optionally rotates refresh_token)
6. User logs out → server invalidates the refresh_token (blocklist or delete from DB)
```

### Storage Considerations

| Location | XSS risk | CSRF risk | Notes |
|----------|----------|-----------|-------|
| `localStorage` | **HIGH** — JS can read it | Low | **Avoid for sensitive tokens** |
| `sessionStorage` | HIGH | Low | Lost on tab close, still XSS-vulnerable |
| **httpOnly cookie** | **None** — JS cannot read | Moderate (mitigate with SameSite=Strict) | Best for refresh tokens |
| Memory (JS var) | Low — lost on refresh | None | Best for short-lived access tokens in SPAs |

```go
package token

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// RefreshClaims is intentionally minimal — the refresh token only needs
// to identify the user and the token itself (for revocation).
type RefreshClaims struct {
	jwt.RegisteredClaims
	// TokenID (jti) lets you invalidate individual refresh tokens.
	// Store issued token IDs in Redis/DB and check on each refresh.
}

// NewRefreshToken creates a long-lived refresh token.
// The 'jti' (JWT ID) should be a random UUID stored in the DB so the
// token can be revoked by deleting that row.
func NewRefreshToken(userID, tokenID string, cfg *TokenConfig) (string, error) {
	now := time.Now()

	claims := RefreshClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    cfg.Issuer,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(cfg.RefreshTTL)),
			ID:        tokenID, // jti — for revocation
		},
	}

	t := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signed, err := t.SignedString(cfg.RefreshSecret)
	if err != nil {
		return "", fmt.Errorf("token.NewRefreshToken: %w", err)
	}
	return signed, nil
}

// ParseRefreshToken validates a refresh token and returns its claims.
func ParseRefreshToken(tokenString string, cfg *TokenConfig) (*RefreshClaims, error) {
	t, err := jwt.ParseWithClaims(
		tokenString,
		&RefreshClaims{},
		func(tok *jwt.Token) (interface{}, error) {
			if _, ok := tok.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", tok.Header["alg"])
			}
			// Use the refresh secret — different from the access secret.
			// This means a stolen access secret cannot forge refresh tokens and vice-versa.
			return cfg.RefreshSecret, nil
		},
		jwt.WithIssuer(cfg.Issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"HS256"}),
	)
	if err != nil {
		return nil, fmt.Errorf("refresh token invalid: %w", err)
	}

	claims, ok := t.Claims.(*RefreshClaims)
	if !ok || !t.Valid {
		return nil, fmt.Errorf("could not extract refresh claims")
	}
	return claims, nil
}
```

### Token Rotation

```go
// On /auth/refresh:
// 1. Parse and validate the old refresh token.
// 2. Check its jti against the DB (if revoked, reject).
// 3. Invalidate the old jti in the DB.
// 4. Issue a new access token AND a new refresh token with a new jti.
// 5. Store the new jti in the DB.
// 6. Return the new access token; set the new refresh token in the httpOnly cookie.
//
// This is called "refresh token rotation" and means a leaked refresh token
// can only be used once before it is invalidated. If the attacker uses it
// first, the legitimate user's next refresh will fail (old jti gone) — alerting
// them to a breach.
```

---

## PART C — Putting It Together

---

## 10. Full Auth Flow with net/http

This is a self-contained example: register, login, and an authenticated endpoint using only the Go standard library plus the two packages above.

```go
// main.go — complete auth server example
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"
	"strings"
	"sync"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"golang.org/x/crypto/argon2"
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
)

// ─── In-memory "database" (replace with PostgreSQL/SQLite in production) ────

type User struct {
	ID           string
	Email        string
	PasswordHash string
	Role         string
}

var (
	mu    sync.RWMutex
	users = map[string]*User{} // keyed by email
)

// ─── Config ─────────────────────────────────────────────────────────────────

var cfg = struct {
	AccessSecret  []byte
	RefreshSecret []byte
	AccessTTL     time.Duration
	RefreshTTL    time.Duration
	Issuer        string
}{
	// In production: load from environment variables, never hard-code.
	AccessSecret:  []byte(getenv("JWT_ACCESS_SECRET", "change-me-to-32-random-bytes!!")),
	RefreshSecret: []byte(getenv("JWT_REFRESH_SECRET", "also-change-me-32-random-bytes!")),
	AccessTTL:     15 * time.Minute,
	RefreshTTL:    7 * 24 * time.Hour,
	Issuer:        "auth.example.com",
}

func getenv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

// ─── Argon2id helpers (from the password package in Section 5) ──────────────

type argon2Params struct {
	memory      uint32
	iterations  uint32
	parallelism uint8
	saltLen     uint32
	keyLen      uint32
}

var defaultArgonParams = argon2Params{
	memory: 64 * 1024, iterations: 3, parallelism: 4, saltLen: 16, keyLen: 32,
}

func hashPassword(password string) (string, error) {
	p := defaultArgonParams
	salt := make([]byte, p.saltLen)
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}
	key := argon2.IDKey([]byte(password), salt, p.iterations, p.memory, p.parallelism, p.keyLen)
	return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version, p.memory, p.iterations, p.parallelism,
		base64.RawStdEncoding.EncodeToString(salt),
		base64.RawStdEncoding.EncodeToString(key),
	), nil
}

func verifyPassword(password, encoded string) error {
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 || parts[1] != "argon2id" {
		return errors.New("invalid hash format")
	}
	var p argon2Params
	var version int
	fmt.Sscanf(parts[2], "v=%d", &version)
	fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &p.memory, &p.iterations, &p.parallelism)
	salt, _ := base64.RawStdEncoding.DecodeString(parts[4])
	storedKey, _ := base64.RawStdEncoding.DecodeString(parts[5])
	p.keyLen = uint32(len(storedKey))
	candidate := argon2.IDKey([]byte(password), salt, p.iterations, p.memory, p.parallelism, p.keyLen)
	if subtle.ConstantTimeCompare(storedKey, candidate) != 1 {
		return errors.New("password mismatch")
	}
	return nil
}

// ─── JWT Claims & helpers ────────────────────────────────────────────────────

type Claims struct {
	jwt.RegisteredClaims
	Email string `json:"email"`
	Role  string `json:"role"`
}

func issueAccessToken(u *User) (string, error) {
	now := time.Now()
	claims := Claims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   u.ID,
			Issuer:    cfg.Issuer,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(cfg.AccessTTL)),
		},
		Email: u.Email,
		Role:  u.Role,
	}
	t := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return t.SignedString(cfg.AccessSecret)
}

func parseAccessToken(raw string) (*Claims, error) {
	t, err := jwt.ParseWithClaims(raw, &Claims{},
		func(tok *jwt.Token) (interface{}, error) {
			if _, ok := tok.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected alg: %v", tok.Header["alg"])
			}
			return cfg.AccessSecret, nil
		},
		jwt.WithIssuer(cfg.Issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"HS256"}),
	)
	if err != nil {
		return nil, err
	}
	c, ok := t.Claims.(*Claims)
	if !ok || !t.Valid {
		return nil, errors.New("invalid token")
	}
	return c, nil
}

// ─── Context key for injecting claims ───────────────────────────────────────

type contextKey string

const claimsKey contextKey = "claims"

func claimsFromContext(ctx context.Context) (*Claims, bool) {
	c, ok := ctx.Value(claimsKey).(*Claims)
	return c, ok
}

// ─── HTTP Handlers ───────────────────────────────────────────────────────────

// writeJSON sends a JSON response with the given status code.
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

// writeError sends a JSON error response. We deliberately keep error messages
// vague for auth failures to avoid leaking information to attackers.
func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, map[string]string{"error": msg})
}

// RegisterRequest is the expected JSON body for POST /auth/register
type RegisterRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

// handleRegister — POST /auth/register
// Validates input, hashes the password, stores the user.
func handleRegister(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
		return
	}

	var req RegisterRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	// Basic validation — in production use a proper validator
	if len(req.Email) == 0 || len(req.Password) < 8 {
		writeError(w, http.StatusBadRequest, "email required, password must be at least 8 characters")
		return
	}

	mu.Lock()
	defer mu.Unlock()

	if _, exists := users[req.Email]; exists {
		// Return the same error as "wrong password" on login to avoid
		// leaking which emails are registered (user enumeration attack).
		writeError(w, http.StatusConflict, "registration failed")
		return
	}

	hash, err := hashPassword(req.Password)
	if err != nil {
		log.Printf("hashPassword error: %v", err) // log internally, don't expose
		writeError(w, http.StatusInternalServerError, "internal error")
		return
	}

	// Generate a simple user ID (use UUID or DB auto-increment in production)
	userID := fmt.Sprintf("user_%d", time.Now().UnixNano())

	users[req.Email] = &User{
		ID:           userID,
		Email:        req.Email,
		PasswordHash: hash,
		Role:         "user", // default role
	}

	writeJSON(w, http.StatusCreated, map[string]string{"id": userID})
}

// LoginRequest is the expected JSON body for POST /auth/login
type LoginRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

// handleLogin — POST /auth/login
// Verifies credentials and issues a JWT access token.
func handleLogin(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
		return
	}

	var req LoginRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	mu.RLock()
	u, exists := users[req.Email]
	mu.RUnlock()

	// ⚠️  Run Argon2 even when user doesn't exist to prevent timing attacks
	// that could reveal which emails are registered.
	//
	// If we returned early on "user not found", the response for non-existent
	// users would be faster (no Argon2 computation) than for existing users,
	// letting an attacker enumerate valid email addresses by measuring response times.
	dummyHash := "$argon2id$v=19$m=65536,t=3,p=4$AAAAAAAAAAAAAAAA$AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
	hashToCheck := dummyHash
	if exists {
		hashToCheck = u.PasswordHash
	}

	err := verifyPassword(req.Password, hashToCheck)

	// Only after constant-time verification do we check if user actually existed.
	if !exists || err != nil {
		// Generic error — same message for "user not found" and "wrong password"
		writeError(w, http.StatusUnauthorized, "invalid credentials")
		return
	}

	accessToken, err := issueAccessToken(u)
	if err != nil {
		log.Printf("issueAccessToken error: %v", err)
		writeError(w, http.StatusInternalServerError, "internal error")
		return
	}

	// In a full implementation, also issue a refresh token here and
	// set it as an httpOnly, Secure, SameSite=Strict cookie.
	//
	// http.SetCookie(w, &http.Cookie{
	//     Name:     "refresh_token",
	//     Value:    refreshToken,
	//     HttpOnly: true,
	//     Secure:   true,           // HTTPS only
	//     SameSite: http.SameSiteStrictMode,
	//     Path:     "/auth/refresh",
	//     MaxAge:   int((7 * 24 * time.Hour).Seconds()),
	// })

	writeJSON(w, http.StatusOK, map[string]string{
		"access_token": accessToken,
		"token_type":   "Bearer",
		"expires_in":   fmt.Sprintf("%d", int(cfg.AccessTTL.Seconds())),
	})
}

// ─── Auth Middleware ──────────────────────────────────────────────────────────

// AuthMiddleware validates the Bearer token in the Authorization header
// and injects the claims into the request context.
//
// Protected handlers retrieve claims with:
//   claims, ok := claimsFromContext(r.Context())
func AuthMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" {
			writeError(w, http.StatusUnauthorized, "missing Authorization header")
			return
		}

		// Expect "Bearer <token>"
		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 || !strings.EqualFold(parts[0], "bearer") {
			writeError(w, http.StatusUnauthorized, "malformed Authorization header")
			return
		}

		claims, err := parseAccessToken(parts[1])
		if err != nil {
			if errors.Is(err, jwt.ErrTokenExpired) {
				writeError(w, http.StatusUnauthorized, "token expired")
			} else {
				writeError(w, http.StatusUnauthorized, "invalid token")
			}
			return
		}

		// Inject claims into context — downstream handlers use claimsFromContext()
		ctx := context.WithValue(r.Context(), claimsKey, claims)
		next(w, r.WithContext(ctx))
	}
}

// handleProfile — GET /profile (protected)
// Demonstrates reading claims injected by AuthMiddleware.
func handleProfile(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
		return
	}

	claims, ok := claimsFromContext(r.Context())
	if !ok {
		// Should never happen if AuthMiddleware is correctly applied,
		// but defensive check is good practice.
		writeError(w, http.StatusInternalServerError, "no claims in context")
		return
	}

	writeJSON(w, http.StatusOK, map[string]any{
		"user_id": claims.Subject,
		"email":   claims.Email,
		"role":    claims.Role,
	})
}

// ─── Main ────────────────────────────────────────────────────────────────────

func main() {
	mux := http.NewServeMux()

	// Public endpoints
	mux.HandleFunc("/auth/register", handleRegister)
	mux.HandleFunc("/auth/login", handleLogin)

	// Protected endpoints — wrapped with AuthMiddleware
	mux.HandleFunc("/profile", AuthMiddleware(handleProfile))

	addr := ":8080"
	log.Printf("Listening on %s", addr)
	if err := http.ListenAndServe(addr, mux); err != nil {
		log.Fatal(err)
	}
}
```

**Testing with curl:**

```bash
# Register
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"hunter2hunter2"}' | jq

# Login
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"hunter2hunter2"}' | jq -r .access_token)
echo "Got token: $TOKEN"

# Access protected route
curl -s http://localhost:8080/profile \
  -H "Authorization: Bearer $TOKEN" | jq

# Try without token → 401
curl -s http://localhost:8080/profile | jq
```

---

## 11. Role-Based Authorization in Claims

Role-based access control (RBAC) can be layered on top of authentication by including a `role` or `roles` field in the JWT claims and checking it in a separate middleware or at the handler level.

```go
package main

import (
	"net/http"
)

// RequireRole is a middleware factory that wraps a handler with role enforcement.
// It must be applied AFTER AuthMiddleware (which injects the claims).
//
// Usage:
//   mux.HandleFunc("/admin/users", AuthMiddleware(RequireRole("admin", handleAdminUsers)))
func RequireRole(requiredRole string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		claims, ok := claimsFromContext(r.Context())
		if !ok {
			// AuthMiddleware should have caught this — defensive check
			writeError(w, http.StatusUnauthorized, "unauthorized")
			return
		}

		if claims.Role != requiredRole {
			// 403 Forbidden — the user is authenticated but lacks permission.
			// Do NOT return 401 here — that means "not authenticated".
			writeError(w, http.StatusForbidden, "insufficient permissions")
			return
		}

		next(w, r)
	}
}

// RequireAnyRole allows access if the user has any of the given roles.
// Useful when multiple roles can access the same resource.
//
// Note: Go requires a variadic parameter (`...`) to be the LAST parameter, so
// we can't write `func(roles ...string, next http.HandlerFunc)`. We pass the
// roles as a []string slice instead, which reads cleanly with a trailing next.
//
// Usage:
//   mux.HandleFunc("/reports", AuthMiddleware(
//       RequireAnyRole([]string{"admin", "moderator"}, handleReports)))
func RequireAnyRole(roles []string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		claims, ok := claimsFromContext(r.Context())
		if !ok {
			writeError(w, http.StatusUnauthorized, "unauthorized")
			return
		}
		for _, role := range roles {
			if claims.Role == role {
				next(w, r)
				return
			}
		}
		writeError(w, http.StatusForbidden, "insufficient permissions")
	}
}

// For more complex scenarios, include a []string of roles in the claims:
type MultiRoleClaims struct {
	jwt.RegisteredClaims
	Email string   `json:"email"`
	Roles []string `json:"roles"` // e.g. ["user", "moderator"]
}

// hasRole checks if the user has a specific role in a multi-role claim.
func hasRole(claims *MultiRoleClaims, role string) bool {
	for _, r := range claims.Roles {
		if r == role {
			return true
		}
	}
	return false
}
```

**Wiring it up:**

```go
func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/auth/register", handleRegister)
	mux.HandleFunc("/auth/login", handleLogin)

	// Any authenticated user
	mux.HandleFunc("/profile", AuthMiddleware(handleProfile))

	// Admin-only endpoint
	mux.HandleFunc("/admin/dashboard", AuthMiddleware(RequireRole("admin", handleAdminDashboard)))

	http.ListenAndServe(":8080", mux)
}

func handleAdminDashboard(w http.ResponseWriter, r *http.Request) {
	// We know the user is authenticated AND has the "admin" role here.
	writeJSON(w, http.StatusOK, map[string]string{"page": "admin dashboard"})
}
```

**Grant admin role on registration (for testing):**

```go
// In handleRegister, check if an admin seed email is being registered:
role := "user"
if req.Email == os.Getenv("ADMIN_SEED_EMAIL") {
    role = "admin"
}
users[req.Email] = &User{..., Role: role}
```

---

## 12. Security Best Practices & Gotchas

### Never Log Secrets or Tokens

```go
// ❌ WRONG — logs the JWT which grants access
log.Printf("User logged in, token: %s", accessToken)

// ❌ WRONG — logs the password
log.Printf("Login attempt for %s with password %s", email, password)

// ✅ CORRECT — log only non-sensitive context
log.Printf("Login successful for user %s (id: %s)", email, userID)
log.Printf("Token issued, expires: %s", expiresAt.Format(time.RFC3339))
```

### Secret Management

```go
// ❌ WRONG — hard-coded secret in source code
var jwtSecret = []byte("my-secret")

// ❌ WRONG — secret in source that "we'll replace later"
var jwtSecret = []byte("TODO-replace-me")

// ✅ CORRECT — loaded from environment, validated at startup
func loadSecret(envKey string, minLen int) []byte {
	val := os.Getenv(envKey)
	if len(val) < minLen {
		log.Fatalf("Environment variable %s must be at least %d characters; got %d",
			envKey, minLen, len(val))
	}
	return []byte(val)
}

var jwtAccessSecret = loadSecret("JWT_ACCESS_SECRET", 32)
```

**Secret rotation:** When you rotate the JWT signing secret, existing tokens immediately become invalid. To support graceful rotation:
1. Support two keys simultaneously for a short window (old + new).
2. Issue all new tokens with the new key.
3. After one access-token TTL (e.g., 15 minutes), drop the old key.

### Token Expiry — Choose TTLs Deliberately

| Scenario | Access TTL | Refresh TTL | Notes |
|----------|-----------|-------------|-------|
| Banking / high-security | 5 min | 1 day (with MFA on refresh) | Aggressive, good for sensitive data |
| Standard SaaS | 15 min | 7 days | Good default |
| Mobile app (user comfort) | 1 hour | 30 days | Acceptable with refresh rotation |
| Long-lived API key | N/A | N/A | Use opaque API keys in DB instead |

### Do Not Put Sensitive Data in JWT Payload

```json
// ❌ WRONG — payload is base64url encoded, not encrypted
{
  "sub": "42",
  "ssn": "123-45-6789",
  "credit_card": "4111111111111111",
  "password_hint": "first pet's name"
}

// ✅ CORRECT — only include what the client needs and can safely see
{
  "sub": "42",
  "email": "alice@example.com",
  "role": "user",
  "iat": 1750000000,
  "exp": 1750000900
}
```

### Revocation Strategies

JWTs are stateless by design — the server doesn't track them. Revocation requires one of these strategies:

| Strategy | Complexity | Scalability | Use case |
|----------|-----------|-------------|---------|
| **Short TTL** | None | Perfect | Tolerate stale access for up to 15m after logout |
| **Blocklist (Redis)** | Low | High | Store revoked `jti` values with TTL = token expiry |
| **Nonce in DB** | Medium | Medium | Store a per-user `token_version`; increment on logout; include in token |
| **Opaque tokens** | High | Requires DB per request | Full revocation control; use for high-security requirements |

```go
// Blocklist approach using a simple in-memory map (replace with Redis in production):
var (
	revokedTokens   = map[string]time.Time{} // jti → expiry
	revokedTokensMu sync.RWMutex
)

// revokeToken adds a jti to the blocklist until its natural expiry.
func revokeToken(jti string, expiry time.Time) {
	revokedTokensMu.Lock()
	revokedTokens[jti] = expiry
	revokedTokensMu.Unlock()
}

// isRevoked checks whether a jti is on the blocklist.
func isRevoked(jti string) bool {
	revokedTokensMu.RLock()
	exp, ok := revokedTokens[jti]
	revokedTokensMu.RUnlock()
	if !ok {
		return false
	}
	// Clean up expired entries lazily
	if time.Now().After(exp) {
		revokedTokensMu.Lock()
		delete(revokedTokens, jti)
		revokedTokensMu.Unlock()
		return false
	}
	return true
}

// In AuthMiddleware, add after parseAccessToken:
// if isRevoked(claims.ID) {
//     writeError(w, http.StatusUnauthorized, "token has been revoked")
//     return
// }
```

### Algorithm Confusion (Recap)

```go
// ALWAYS check the signing method in the keyFunc.
// ALWAYS pass jwt.WithValidMethods([]string{"HS256"}) (or your chosen alg).
// NEVER accept "none" algorithm in a production system.
// NEVER use a public key as an HMAC secret (RS256 → HS256 downgrade attack).
```

### HTTPS Is Non-Negotiable

Tokens sent over plain HTTP are trivially intercepted. Enforce HTTPS at the infrastructure level (reverse proxy / load balancer). In Go:

```go
// Redirect HTTP to HTTPS at the application level
http.ListenAndServe(":80", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    target := "https://" + r.Host + r.URL.Path
    if r.URL.RawQuery != "" {
        target += "?" + r.URL.RawQuery
    }
    http.Redirect(w, r, target, http.StatusMovedPermanently)
}))
```

### Checklist: Security Review Before Shipping

```
[ ] Passwords are hashed with Argon2id (not SHA, MD5, or bcrypt)
[ ] Argon2 params tune to 200-500ms on your server hardware
[ ] PHC string format used — params stored with hash
[ ] Salt generated from crypto/rand, never math/rand
[ ] Password comparison uses subtle.ConstantTimeCompare
[ ] JWT secrets loaded from environment variables, minimum 32 bytes
[ ] JWT secrets are different for access vs refresh tokens
[ ] JWT algorithm is explicitly checked in the keyFunc
[ ] jwt.WithValidMethods() whitelist is applied
[ ] Refresh tokens are stored in httpOnly cookies, not localStorage
[ ] All endpoints served over HTTPS
[ ] Access token TTL is 5-15 minutes
[ ] Login returns same error for "user not found" and "wrong password"
[ ] No sensitive data (passwords, PII) in JWT payload
[ ] JWT payload is never logged
[ ] Revocation strategy defined (blocklist, short TTL, or token version)
[ ] Input validation on all auth endpoints (email format, password length)
[ ] Rate limiting on /auth/login to slow brute force
```

### Common Gotchas

**Gotcha 1 — `math/rand` vs `crypto/rand`**

```go
// ❌ math/rand is seeded, predictable, NOT cryptographically secure
salt := make([]byte, 16)
rand.Read(salt) // This is math/rand.Read — DO NOT USE for security

// ✅ crypto/rand is the OS CSPRNG
import "crypto/rand"
salt := make([]byte, 16)
crypto_rand.Read(salt)
```

**Gotcha 2 — Comparing error types in jwt/v5**

```go
// ⚡ Version note (v5): Use errors.Is() not string comparison.
// The library wraps multiple errors under jwt.ErrTokenInvalidClaims, etc.
err := validateToken(...)

// ❌ WRONG — brittle string matching
if strings.Contains(err.Error(), "expired") { ... }

// ✅ CORRECT — structured error checking
if errors.Is(err, jwt.ErrTokenExpired) { ... }
if errors.Is(err, jwt.ErrSignatureInvalid) { ... }
```

**Gotcha 3 — Clock skew with exp/nbf**

```go
// Servers in a distributed system may have slightly different clocks.
// jwt/v5 supports a clock skew leeway:
jwt.ParseWithClaims(raw, &claims, keyFunc,
    jwt.WithLeeway(30 * time.Second), // tolerate ±30s clock drift
)
```

**Gotcha 4 — bcrypt 72-byte truncation**

```go
// bcrypt silently ignores characters beyond byte 72.
// A 100-character password and a 72-character prefix of it hash identically.
// Argon2id has no such limit.
// This is one reason to prefer Argon2id over bcrypt.
```

**Gotcha 5 — User enumeration in login**

```go
// ❌ WRONG — response time is shorter for non-existent users
user, err := db.GetUserByEmail(email)
if err != nil {                         // fast path — no Argon2
    return ErrInvalidCredentials
}
if err := verifyPassword(password, user.Hash); err != nil { // slow path — Argon2
    return ErrInvalidCredentials
}

// ✅ CORRECT — always run Argon2 regardless of whether user exists
user, err := db.GetUserByEmail(email)
hashToVerify := dummyHash
if err == nil {
    hashToVerify = user.Hash
}
verifyErr := verifyPassword(password, hashToVerify) // always slow
if err != nil || verifyErr != nil {
    return ErrInvalidCredentials
}
```

**Gotcha 6 — Storing access tokens in localStorage**

```
localStorage is accessible by any JavaScript on the page.
A single XSS vulnerability exposes every stored token.
Store refresh tokens in httpOnly cookies.
Store access tokens in memory only (a JavaScript variable in a closure or
React state) — they're short-lived anyway.
```

---

## 13. Study Path

Work through these in order. Each project forces you to apply a specific concept before you move on — you'll understand it because you broke it first.

### Phase 1 — Foundations (Week 1–2)

**Project 1 — Password Hasher CLI**
Build a command-line tool that:
- Takes a password via stdin (not argv — argv leaks to `ps`)
- Hashes it with Argon2id using the full package from Section 5
- Prints the PHC string
- Accepts a `--verify` flag: reads a stored hash and reports match/no-match
- Goal: understand PHC format and constant-time comparison

**Project 2 — JWT Inspector**
Build a CLI that:
- Decodes a JWT without verifying (base64url decode header + payload)
- Parses and verifies a JWT given a secret
- Prints each claim with its type and value
- Goal: demystify JWT structure; see that the payload is just JSON

### Phase 2 — Core Patterns (Week 3–4)

**Project 3 — Auth API**
Build the server from Section 10 but add:
- A real SQLite database (`github.com/mattn/go-sqlite3` or `modernc.org/sqlite`)
- Input validation middleware
- Rate limiting on `/auth/login` (simple token bucket or `golang.org/x/time/rate`)
- Proper error logging (structured, with `log/slog` from Go 1.21)
- Goal: production-grade foundation

**Project 4 — Refresh Token Flow**
Extend Project 3 with:
- `/auth/refresh` endpoint
- Refresh tokens in httpOnly cookies
- Token rotation (invalidate old jti, issue new one)
- `/auth/logout` that revokes the refresh token
- Goal: understand stateless vs stateful token lifecycle

### Phase 3 — Hardening (Week 5–6)

**Project 5 — RBAC Protected API**
Build a mini blog API with:
- Roles: `reader`, `author`, `admin`
- `GET /posts` — public
- `POST /posts` — requires `author` or `admin`
- `DELETE /posts/:id` — requires `admin`
- Users can only edit their own posts (check `sub` claim)
- Goal: understand the difference between authentication and authorization

**Project 6 — Microservice Token Verification**
Split into two services:
- **Auth service**: issues RS256 tokens, serves its public key at `/.well-known/jwks.json`
- **Resource service**: fetches the public key, verifies RS256 tokens without calling the auth service
- Goal: understand when to use asymmetric signing and how JWKS works

### Phase 4 — Production Concerns (Week 7–8)

**Project 7 — Secret Rotation**
Extend Project 3 to:
- Support two JWT secrets simultaneously (current + previous)
- The keyFunc tries `current` first, falls back to `previous`
- Add a `/auth/rotate-secret` admin endpoint that shifts `current` to `previous` and generates a new `current`
- Goal: understand zero-downtime secret rotation

**Project 8 — Full Stack Auth**
Connect a React frontend (or HTMX if you prefer) to your auth API:
- Store access token in memory, refresh token in httpOnly cookie
- Axios/fetch interceptor that catches 401 and calls `/auth/refresh` automatically
- Logout button that calls `/auth/logout` and clears memory
- Goal: see the complete picture from browser to server and back

### Key Reference Material (offline-friendly)

| Resource | What to study |
|----------|---------------|
| RFC 7519 | Full JWT specification — read the claims table and security considerations |
| RFC 9106 | Argon2 specification — understand why memory-hardness works |
| OWASP Authentication Cheat Sheet | Exhaustive checklist; re-read after each project |
| OWASP Password Storage Cheat Sheet | Argon2id params, upgrade strategies |
| `golang-jwt/jwt` v5 source | Read `parser.go` and `claims.go` — 500 lines, very readable |
| `golang.org/x/crypto/argon2` source | Read `argon2.go` — understand the IDKey function signature |
| Go `crypto/subtle` source | Read `constant_time.go` — understand why it works |

### Mental Models to Lock In

```
Authentication = "who are you?"      → JWT validates identity
Authorization  = "what can you do?"  → Role check enforces permissions

Hashing  = one-way, cannot reverse   → Argon2id for passwords
Encoding = two-way (base64)          → JWT payload is NOT secure storage
Encryption = two-way, key required   → Use JWE if you need encrypted payload

Session token  = opaque, server tracks state   → O(1) revocation, stateful
JWT            = self-contained, server stateless → Fast, hard to revoke early

"A JWT is a signed assertion. Anyone with the token can prove they have it.
 The signature proves nobody tampered with it. That's all."
```

---

*Guide accurate as of June 2026. `github.com/golang-jwt/jwt/v5` (v5.x) and `golang.org/x/crypto` (current). Always consult the official OWASP cheat sheets and library changelogs before shipping to production.*
