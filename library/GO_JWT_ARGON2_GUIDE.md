# Go JWT + Argon2 — Authentication — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Go developers who want to build secure, production-grade authentication *from scratch* and understand every line — password hashing with **Argon2id** and stateless sessions with **JSON Web Tokens (JWT)**. This is a learn-offline study text: read it top to bottom the first time. Every concept is explained in prose first — *what it is, the logic/why, what it's for and when to use it, how to use it, the key parameters, best practices, and the security implications* — and only then shown as heavily-commented, runnable code. Authentication is security-critical, so the "why" matters as much as the "how"; a working auth system that you do not understand is a breach waiting to happen.
>
> **Version note:** This guide targets **`github.com/golang-jwt/jwt/v5`** (v5.x) and **`golang.org/x/crypto/argon2`**, on **Go 1.23 / 1.24** (current in 2026). golang-jwt v5 changed several APIs from v4 (`StandardClaims` → `RegisteredClaims`, parser validation options, typed errors) — every code sample here is v5. Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; shell commands are shown for both PowerShell and POSIX where they differ. Always confirm exact signatures at pkg.go.dev and re-check the OWASP cheat sheets before shipping.
>
> Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. Companion guides in this library: **`GO_NET_HTTP_REST_API_GUIDE.md`** (the `net/http` server this builds on), **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** (the Gin framework, whose middleware we integrate in §13), and **`BETTERAUTH_GUIDE.md`** (a batteries-included TypeScript auth framework — useful contrast for what a full-service library does for you). For the language itself see **`GO_GUIDE.md`**.

---

## Table of Contents

**PART A — Password Security & Argon2**
1. [The Password Threat Model: Why You Hash](#1-the-password-threat-model-why-you-hash) **[B]**
2. [Salts, Pepper & Slow Hashes](#2-salts-pepper--slow-hashes) **[B]**
3. [Argon2id In Depth: Variants & Parameters](#3-argon2id-in-depth-variants--parameters) **[B/I]**
4. [Generating a Salt & Calling argon2.IDKey](#4-generating-a-salt--calling-argon2idkey) **[B/I]**
5. [The PHC String Format](#5-the-phc-string-format) **[I]**
6. [Verifying a Password (Constant-Time Compare)](#6-verifying-a-password-constant-time-compare) **[I]**
7. [A Complete, Reusable password Package](#7-a-complete-reusable-password-package) **[I]**

**PART B — JSON Web Tokens**
8. [What a JWT Is: Structure, Claims, Signed-Not-Secret](#8-what-a-jwt-is-structure-claims-signed-not-secret) **[B/I]**
9. [Signing Algorithms: HMAC vs RSA vs ECDSA](#9-signing-algorithms-hmac-vs-rsa-vs-ecdsa) **[I]**
10. [Creating & Signing a Token (golang-jwt v5)](#10-creating--signing-a-token-golang-jwt-v5) **[I]**
11. [Parsing & Validating + the alg-Confusion Attack](#11-parsing--validating--the-alg-confusion-attack) **[I/A]**
12. [Access + Refresh Tokens: Rotation & Revocation](#12-access--refresh-tokens-rotation--revocation) **[I/A]**

**PART C — Putting It Together**
13. [Auth Middleware: net/http and Gin](#13-auth-middleware-nethttp-and-gin) **[I]**
14. [Full Auth Flow with net/http](#14-full-auth-flow-with-nethttp) **[I]**
15. [RBAC: Roles, Permissions & Enforcement](#15-rbac-roles-permissions--enforcement) **[I/A]**
16. [Asymmetric Signing & Key Management/Rotation](#16-asymmetric-signing--key-managementrotation) **[A]**
17. [The Security Section: Attacks & Defenses](#17-the-security-section-attacks--defenses) **[A]**
18. [Quick Reference Tables](#18-quick-reference-tables)
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## PART A — Password Security & Argon2

---

## 1. The Password Threat Model: Why You Hash **[B]**

Before a single line of code, you must internalize **what you are defending against**. Security engineering is threat-modelling: you decide who the attacker is, what they can do, and what you are protecting — and only then do the technical choices make sense.

### What you are protecting and from whom

When a user registers, they hand you a secret — their password. You must store *something* so that you can later check whether someone presents the same secret. The naive approach is to store the password itself. **Never do this.** The threat model is concrete and assumes the worst realistic case:

- **Assume your database *will* leak.** SQL injection, a stolen backup, a misconfigured S3 bucket, a malicious or compromised insider, a leaked `.env` file in a git history — database compromise is one of the most common breach categories. Your password storage scheme must survive an attacker holding a full copy of the users table offline.
- **The attacker is patient and well-resourced.** Once they have the table, they are not rate-limited by your login endpoint. They run *offline* attacks on their own hardware — GPUs, ASIC farms, or rented cloud clusters — testing billions of guesses per second against the leaked data.
- **Users reuse passwords.** A password you leak is not just a compromise of *your* service — the same email/password unlocks the victim's bank, email, and everything else. The blast radius of a plaintext leak is the entire internet for that user.

### The defense: a one-way function

A **cryptographic hash** turns an input into a fixed-size output (the *digest*) such that: (a) you cannot feasibly run it backwards to recover the input, and (b) you cannot feasibly find two inputs with the same output. You store the digest, never the password. On login you hash the presented password and compare digests. If the database leaks, the attacker has digests, not passwords — *provided* the hash is the right kind.

### Why ordinary hashes (SHA-256, MD5) are catastrophically wrong

This is the single most common, most damaging mistake. SHA-256, SHA-3, MD5, and friends are *general-purpose* hashes designed to be **fast** — fast is exactly what you want for verifying a file download and exactly what you do *not* want for passwords:

- **Speed is the enemy.** A general-purpose hash is built to run as fast as possible. A modern GPU computes *billions* of SHA-256 hashes per second. An attacker with a leaked table of SHA-256 password hashes can test the entire set of common passwords against every user in minutes.
- **Rainbow tables.** Because plain hashes are deterministic and unsalted, attackers precompute enormous lookup tables mapping common passwords (and their permutations) to their digests. A leaked unsalted MD5/SHA hash of a common password is reversed by a *table lookup* — effectively instant.
- **Identical passwords → identical hashes.** Without per-user randomness, every user who chose `password123` has the same digest. Crack one, crack them all; and you can *see* which users share a password.

> **The rule:** Password storage is the *one* place where you deliberately choose a **slow, memory-hard** hash. For everything else (file integrity, HMAC, content addressing) you use a fast hash. Confusing the two is the root of most credential-storage breaches.

---

## 2. Salts, Pepper & Slow Hashes **[B]**

Three ideas together turn a hash into safe password storage: a **salt**, an optional **pepper**, and a deliberately **slow** function. Understand each independently.

### The salt — defeats precomputation and reveals nothing

A **salt** is a unique, random value generated per password and mixed into the hash. It is *not* secret — it is stored right next to the hash. Its job is purely to make precomputation useless and identical passwords look different:

- **Defeats rainbow tables.** A precomputed table is built for *unsalted* inputs. With a random 16-byte salt per user, the attacker would need a separate table per salt — astronomically expensive. They are forced back to attacking one user at a time.
- **Breaks the "crack one, crack all" property.** Two users with the same password now have different salts, so different hashes. The attacker cannot batch them.
- **Why it can be public.** The salt's only purpose is uniqueness, not secrecy. Even knowing the salt, the attacker still has to brute-force each password individually against the slow hash. Storing it in plaintext alongside the hash is correct and standard (the PHC format in §5 does exactly this).

> **Generate salts with a CSPRNG.** Use `crypto/rand` (the OS entropy pool), **never** `math/rand` (deterministic and predictable). This is one of the most common Go security bugs — see the gotcha in §17. 16 bytes (128 bits) of salt is the standard.

### The pepper — an optional second factor, kept out of the DB

A **pepper** is a *secret* value (the same for all users) that is combined with the password before/within hashing, but stored **separately from the database** — in an environment variable, a secrets manager, or an HSM. The logic: if *only* the database leaks (the most common case) but the application secret does not, the attacker cannot even begin cracking, because they are missing the pepper.

- **How:** the simplest correct approach is to HMAC the password with the pepper key *before* feeding it to Argon2, or to use Argon2's optional `secret` parameter (note: `golang.org/x/crypto/argon2`'s `IDKey` does not expose the secret parameter, so the HMAC-pre-step is the practical Go approach).
- **Trade-off:** a pepper protects against DB-only leaks but is useless if the app server is also compromised (the attacker then has the pepper). It also complicates rotation. It is a *defense in depth* layer, not a substitute for a strong hash. Most systems are fine without one; add it for high-value targets.

### The slow hash — pricing out offline attacks

The final and most important idea: the password hash should be **deliberately expensive** to compute. A fast hash lets an attacker test billions of guesses per second; a slow hash tuned to take ~0.25 seconds limits them to ~4 guesses per second per core. That difference is the entire game.

- **bcrypt** (2000s standard): intentionally slow, includes a salt, CPU-bound. Better than nothing, but limited to 72-byte inputs (silent truncation — see §17), with a fixed ~4 KB memory footprint that makes GPU attacks cheap.
- **scrypt**: adds *memory-hardness* — it forces the attacker to spend RAM, not just CPU, which neutralizes cheap parallel GPU/ASIC attacks. Strong, but its parameters are harder to reason about and it is more vulnerable to certain side-channel and time-memory trade-off attacks than Argon2.
- **Argon2id** (current gold standard, §3): memory-hard like scrypt, with clean independent parameters and the best resistance profile. **This is what OWASP recommends for new systems and what this guide uses.**

> **Mental model:** Salt = "make each guess specific to one user." Slow/memory-hard = "make each guess expensive." Pepper = "make guessing impossible without a secret you keep elsewhere." You always use a salt and a slow hash; pepper is optional hardening.

---

## 3. Argon2id In Depth: Variants & Parameters **[B/I]**

**Argon2** won the 2015 Password Hashing Competition (PHC) and is specified in RFC 9106. It is the modern default for password hashing because it is **memory-hard**: it forces the attacker to allocate a large block of RAM *per concurrent guess*. A GPU has thousands of cores but limited memory bandwidth and capacity; demanding 64 MiB per guess collapses the parallelism advantage that makes GPUs devastating against fast hashes. You make attacking expensive in the one resource attackers cannot cheaply scale.

### The three variants — and why **id**

| Variant | Memory access pattern | Resists | Use for |
|---------|----------------------|---------|---------|
| **Argon2d** | Data-*dependent* | GPU/ASIC cracking (max) | Cryptocurrencies, where side channels are not a concern |
| **Argon2i** | Data-*independent* | Side-channel (timing/cache) attacks | Pure side-channel-sensitive contexts |
| **Argon2id** | Hybrid: `i` first half, `d` second half | **Both** — GPU *and* side-channel | **Password hashing (recommended)** |

- **Argon2d** indexes memory based on the data being hashed, which maximises resistance to time-memory trade-off (GPU) attacks but leaks information through cache-timing side channels — bad when an attacker can observe the machine doing the hashing.
- **Argon2i** uses a data-*independent* access pattern (immune to those side channels) but is weaker against GPU attacks.
- **Argon2id** runs `i` for the first pass and `d` for the rest, getting most of both protections. **OWASP and RFC 9106 both recommend Argon2id for password storage.** In Go this is `argon2.IDKey`.

### The parameters — what each one does and how to tune it

Argon2id is tuned by three primary knobs plus two sizes. Understanding each is essential because *you* are responsible for choosing a security level.

| Parameter | Go arg | PHC field | What it does | Tuning logic |
|-----------|--------|-----------|--------------|--------------|
| **Memory** | `memory` (KiB) | `m=` | RAM allocated per hash | The key memory-hardness lever. Higher = far more expensive for attackers (linear in cost). Raise this first. |
| **Iterations / time** | `time` (passes) | `t=` | Number of passes over the memory buffer | Linear CPU cost. Raise when you cannot afford more memory but want more work. |
| **Parallelism** | `threads` | `p=` | Lanes computed in parallel | Lets *you* use multiple cores to hit a target time at higher memory. Does not increase total work; it spreads it. |
| **Salt length** | (length of salt) | embedded | Random bytes of salt | 16 bytes (128 bits) standard. |
| **Key length** | `keyLen` | output | Output digest size in bytes | 32 bytes (256 bits) is ample. |

**OWASP-recommended starting points (2025/2026):**

| Parameter | Minimum | Recommended |
|-----------|---------|-------------|
| Memory (`m`) | 19 MiB (19456 KiB) | 64 MiB (65536 KiB) |
| Iterations (`t`) | 2 | 3 |
| Parallelism (`p`) | 1 | 4 |
| Hash output length | 32 bytes | 32 bytes |
| Salt length | 16 bytes | 16 bytes |

> **Source:** OWASP Password Storage Cheat Sheet. These target ~0.25–1 second of compute on a modern server. **Always benchmark on your own production hardware** and tune so a single hash takes **200–500 ms** — long enough to make offline cracking painful, short enough that your login endpoint and registration flow stay responsive and you are not trivially DoS-able. Time it with `go test -bench` (see §19, Project 1).

### How to tune in practice

1. Pick a **target latency** (e.g., 300 ms) and a **memory budget** per concurrent login (e.g., 64 MiB). If you expect 50 simultaneous logins, that is ~3.2 GiB peak — make sure your server has it, or you have created a memory-exhaustion DoS.
2. Set `parallelism` to the number of cores you are willing to dedicate per hash (often 1–4). More parallelism lets you afford more memory within the latency budget.
3. Increase `memory` until you hit your latency target. If memory is capped by your budget, increase `iterations` instead.
4. **Re-tune over time.** Hardware gets faster; raise parameters every couple of years. Because the PHC format (§5) stores the parameters *with each hash*, you can raise them and silently upgrade old hashes on next login (the `NeedsRehash` pattern in §7).

> **Security recommendation:** Treat the parameters as a *floor that only goes up*. Lowering them later weakens every hash created at the new level. Store params with the hash (PHC) so you are never locked in.

### Module setup

```bash
# Initialize your module (replace with your own path)
go mod init github.com/yourname/authexample

# golang.org/x/crypto provides the argon2 subpackage
go get golang.org/x/crypto

# The JWT library — note the /v5 in the import path
go get github.com/golang-jwt/jwt/v5

# (Optional) Gin, used in §13
go get github.com/gin-gonic/gin
```

> **⚡ Version note:** `golang.org/x/crypto/argon2` is part of the extended standard library ("golang.org/x"). The `IDKey` signature has been stable since Go 1.13. Pull it from the module proxy — do **not** vendor a random fork of an Argon2 implementation; a subtly wrong implementation silently produces insecure hashes.

---

## 4. Generating a Salt & Calling argon2.IDKey **[B/I]**

Now the code. The core call is `argon2.IDKey`, which performs the actual key derivation. Before calling it you must produce a fresh random salt; after calling it you have raw bytes that you must encode for storage (§5).

### The `IDKey` signature, argument by argument

```
argon2.IDKey(password []byte, salt []byte, time uint32, memory uint32, threads uint8, keyLen uint32) []byte
```

- `password` — the user's password as bytes. Argon2 has **no input-length limit** (unlike bcrypt's 72-byte trap), so long passphrases are fully respected.
- `salt` — your CSPRNG-generated salt bytes.
- `time` — the **iterations** (`t=`). Passes over the memory buffer.
- `memory` — KiB of RAM (`m=`). `64 * 1024` = 64 MiB.
- `threads` — **parallelism** (`p=`).
- `keyLen` — output length in bytes (32 = 256-bit digest).

It returns the derived key (the hash). It never returns an error — invalid parameters panic, which is appropriate (a misconfigured hash must not silently produce weak output).

### Generating the salt — `crypto/rand`, always

`crypto/rand.Read` fills a byte slice from the OS cryptographically secure RNG (`/dev/urandom`, `getrandom(2)`, or `CryptGenRandom`/`BCryptGenRandom` on Windows). It returns an error only if the OS entropy source is broken — which on a healthy machine never happens, but you must still handle it (do not ignore the error).

```go
package main

import (
	"crypto/rand" // CSPRNG — NOT math/rand
	"fmt"
	"log"

	"golang.org/x/crypto/argon2"
)

// Argon2Params holds the tunable parameters for Argon2id.
// We carry them in a struct (rather than scattering magic numbers) so that
// the same values are used for hashing AND embedded into the stored PHC string,
// which lets us verify and later upgrade hashes without breaking old ones.
type Argon2Params struct {
	Memory      uint32 // KiB of memory to use  (m=)
	Iterations  uint32 // number of passes       (t=)
	Parallelism uint8  // degree of parallelism  (p=)
	SaltLength  uint32 // bytes of random salt to generate
	KeyLength   uint32 // bytes in the output hash
}

// DefaultParams are OWASP-recommended settings for 2025/2026.
// Benchmark on your own hardware and tune so Hash() takes 200–500 ms.
var DefaultParams = &Argon2Params{
	Memory:      64 * 1024, // 64 MiB expressed in KiB
	Iterations:  3,
	Parallelism: 4,
	SaltLength:  16, // 128-bit salt
	KeyLength:   32, // 256-bit digest
}

// generateSalt returns n cryptographically random bytes.
// It propagates any error from the OS RNG rather than ignoring it.
func generateSalt(n uint32) ([]byte, error) {
	salt := make([]byte, n)
	if _, err := rand.Read(salt); err != nil {
		// A failure here means the OS entropy source is unavailable — treat as fatal.
		return nil, fmt.Errorf("generateSalt: reading from crypto/rand: %w", err)
	}
	return salt, nil
}

func main() {
	password := "correct-horse-battery-staple"
	p := DefaultParams

	// Step 1 — fresh random salt for THIS password.
	salt, err := generateSalt(p.SaltLength)
	if err != nil {
		log.Fatal(err)
	}

	// Step 2 — derive the hash with Argon2id.
	// Argument order is (password, salt, time, memory, threads, keyLen).
	hash := argon2.IDKey(
		[]byte(password),
		salt,
		p.Iterations,  // time
		p.Memory,      // memory
		p.Parallelism, // threads
		p.KeyLength,   // keyLen
	)

	fmt.Printf("Salt (hex): %x\n", salt)
	fmt.Printf("Hash (hex): %x\n", hash)
	// WARNING: never store raw hex like this. The hash alone is useless without
	// the salt AND the parameters used to make it. Section 5 bundles all three
	// into one self-describing string — that is what you store.
}
```

> **Security recommendations for this step:** (1) Always `crypto/rand`, never `math/rand`. (2) Never log the password or the raw hash. (3) Do not reuse a salt across users. (4) Keep the parameter struct in one place so hashing and verification cannot drift out of sync.

---

## 5. The PHC String Format **[I]**

A raw 32-byte digest is meaningless on its own: to verify a password later you need the *exact* algorithm, version, parameters, and salt that produced it. The **PHC string format** (from the Password Hashing Competition) packs all of that into a single self-describing line you store in one database column.

### Anatomy of the string

```
$argon2id$v=19$m=65536,t=3,p=4$<base64-salt>$<base64-hash>
 └──┬───┘ └─┬─┘ └──────┬──────┘ └────┬─────┘ └────┬────┘
   algo   version    params         salt          digest
```

- **`argon2id`** — the algorithm/variant identifier. Lets you detect (and reject, or migrate) hashes made with a different scheme.
- **`v=19`** — the Argon2 spec version (0x13 = 19). `argon2.Version` is this constant. Verify it on decode so a future version mismatch is caught rather than silently mis-verified.
- **`m=…,t=…,p=…`** — the memory, iterations, parallelism used. **This is why old hashes keep working when you raise your defaults:** each hash carries the parameters it was made with.
- **`<base64-salt>$<base64-hash>`** — salt and digest, base64-encoded.

### Why `base64.RawStdEncoding`

The PHC spec uses base64 **without `=` padding**. In Go that is `base64.RawStdEncoding` (standard alphabet, no padding). Using `StdEncoding` (with padding) or `URLEncoding` (different alphabet) produces strings that other PHC-compliant tools cannot read and that your own decoder must special-case. Match the spec exactly so your hashes are portable.

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
	Memory: 64 * 1024, Iterations: 3, Parallelism: 4, SaltLength: 16, KeyLength: 32,
}

// encodeHash formats params + salt + digest into a PHC Argon2id string.
// Format: $argon2id$v=<ver>$m=<mem>,t=<iter>,p=<par>$<b64salt>$<b64hash>
func encodeHash(p *Argon2Params, salt, hash []byte) string {
	// RawStdEncoding == standard base64 alphabet, NO '=' padding (PHC requirement).
	b64Salt := base64.RawStdEncoding.EncodeToString(salt)
	b64Hash := base64.RawStdEncoding.EncodeToString(hash)

	// argon2.Version is the constant 0x13 == 19 (the current Argon2 spec version).
	return fmt.Sprintf(
		"$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version,
		p.Memory,
		p.Iterations,
		p.Parallelism,
		b64Salt,
		b64Hash,
	)
}

func main() {
	p := DefaultParams
	password := "correct-horse-battery-staple"

	salt := make([]byte, p.SaltLength)
	if _, err := rand.Read(salt); err != nil {
		log.Fatal(err)
	}

	hash := argon2.IDKey([]byte(password), salt, p.Iterations, p.Memory, p.Parallelism, p.KeyLength)

	encoded := encodeHash(p, salt, hash)
	fmt.Println(encoded)
	// e.g. $argon2id$v=19$m=65536,t=3,p=4$FNk/8Cq3...$2X3Nf9...
}
```

**What you store in the database** — a single text column holds the entire PHC string:

```sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,      -- the full PHC string ($argon2id$...)
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

> **Best practice:** store *only* the PHC string. Do not split salt/params into separate columns — that invites the two getting out of sync and offers no benefit. The self-describing format is the whole point.

---

## 6. Verifying a Password (Constant-Time Compare) **[I]**

Verification reverses §5 then re-derives and compares. The logic:

1. **Parse** the stored PHC string back into its params, salt, and stored digest.
2. **Re-run** `argon2.IDKey` on the *candidate* password using the *same* params and salt extracted from the stored string. (This is why the params travel with the hash — you literally need them to verify.)
3. **Compare** the freshly derived digest against the stored one — using a **constant-time** comparison.

### Why constant-time comparison is mandatory

A normal comparison (`==`, `bytes.Equal`) **short-circuits**: it returns `false` the instant it finds the first differing byte. That means a near-correct guess takes *measurably longer* to reject than a wildly wrong one. An attacker who can time your responses can use this **timing side channel** to recover the target value one byte at a time — a classic timing attack.

`crypto/subtle.ConstantTimeCompare` always inspects **every** byte regardless of where (or whether) a mismatch occurs, so the time taken reveals nothing about *how close* the guess was. It returns `1` for equal, `0` otherwise. (In practice the input to this compare is the *digest* of the candidate, not the password itself, but using constant-time compare here is correct discipline and protects against any digest-level timing leak.)

```go
package main

import (
	"crypto/subtle" // constant-time comparison — defeats timing attacks
	"encoding/base64"
	"errors"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

// Sentinel errors let callers distinguish "wrong password" from "corrupt hash".
var (
	ErrInvalidHash         = errors.New("argon2: encoded hash is not in the correct PHC format")
	ErrIncompatibleVersion = errors.New("argon2: incompatible version of argon2")
	ErrMismatch            = errors.New("argon2: password does not match")
)

type Argon2Params struct {
	Memory      uint32
	Iterations  uint32
	Parallelism uint8
	SaltLength  uint32
	KeyLength   uint32
}

// decodeHash parses a PHC Argon2id string back into params, salt, and digest.
// It validates the algorithm id and version BEFORE trusting the rest.
func decodeHash(encoded string) (p *Argon2Params, salt, hash []byte, err error) {
	// Splitting "$argon2id$v=19$m=..,t=..,p=..$<salt>$<hash>" on '$' yields 6 parts;
	// part[0] is the empty string before the leading '$'.
	vals := strings.Split(encoded, "$")
	if len(vals) != 6 {
		return nil, nil, nil, ErrInvalidHash
	}
	// vals[1]=algo, vals[2]="v=..", vals[3]="m=..,t=..,p=..", vals[4]=salt, vals[5]=hash
	if vals[1] != "argon2id" {
		return nil, nil, nil, ErrInvalidHash
	}

	var version int
	if _, e := fmt.Sscanf(vals[2], "v=%d", &version); e != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	if version != argon2.Version {
		return nil, nil, nil, ErrIncompatibleVersion
	}

	p = &Argon2Params{}
	if _, e := fmt.Sscanf(vals[3], "m=%d,t=%d,p=%d", &p.Memory, &p.Iterations, &p.Parallelism); e != nil {
		return nil, nil, nil, ErrInvalidHash
	}

	if salt, err = base64.RawStdEncoding.DecodeString(vals[4]); err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	p.SaltLength = uint32(len(salt))

	if hash, err = base64.RawStdEncoding.DecodeString(vals[5]); err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	p.KeyLength = uint32(len(hash))

	return p, salt, hash, nil
}

// verifyPassword reports whether password matches the stored PHC hash.
// It returns ErrMismatch for a wrong password and a different error for a
// malformed/incompatible stored hash — callers should map BOTH to a single
// generic "invalid credentials" message at the HTTP layer (see §14).
func verifyPassword(password, encoded string) error {
	p, salt, storedHash, err := decodeHash(encoded)
	if err != nil {
		return err
	}

	// Re-derive using the SAME parameters and salt taken from the stored hash.
	candidate := argon2.IDKey([]byte(password), salt, p.Iterations, p.Memory, p.Parallelism, p.KeyLength)

	// subtle.ConstantTimeCompare returns 1 iff the slices are equal, reading both
	// in full regardless of mismatches — no early exit, no timing leak.
	if subtle.ConstantTimeCompare(storedHash, candidate) != 1 {
		return ErrMismatch
	}
	return nil
}
```

The timing-attack contrast, made explicit:

```go
// WRONG — short-circuits, leaks how many leading bytes matched via timing:
//   if bytes.Equal(storedHash, candidate) { ... }
//
// RIGHT — constant time regardless of where (or whether) they differ:
//   if subtle.ConstantTimeCompare(storedHash, candidate) == 1 { ... }
//
// Attacker strategy against a timing oracle: send many guesses, measure response
// times, and use "this one took slightly longer" to learn matched-prefix length,
// recovering the secret byte by byte. Constant-time compare closes that channel.
```

> **Security recommendations:** (1) Always `subtle.ConstantTimeCompare`. (2) Return identical errors/timing for "wrong password" and "user not found" at the API boundary (§14) so you do not leak which accounts exist. (3) Never reveal whether it was the format, the version, or the password that failed — to the *client*. Log the distinction internally.

---

## 7. A Complete, Reusable `password` Package **[I]**

Here is the full, production-ready package — drop it into `internal/password/` and reuse it across projects. It adds `NeedsRehash`, which lets you transparently upgrade weak old hashes to your current parameters the next time a user logs in successfully. This is the *correct* way to raise your security level over time without forcing password resets.

```go
// Package password provides Argon2id password hashing and verification using
// the PHC string format, so the parameters live WITH each hash and can be
// upgraded later without breaking existing stored credentials.
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

// Sentinel errors. Callers compare with errors.Is.
var (
	ErrInvalidHash         = errors.New("password: encoded hash is not a valid PHC Argon2id string")
	ErrIncompatibleVersion = errors.New("password: argon2 version mismatch — re-hash required")
	ErrMismatch            = errors.New("password: password does not match hash")
)

// Params configures the Argon2id KDF. Use DefaultParams() and tune so Hash()
// takes 200–500 ms on YOUR hardware. Treat the values as a floor that only rises.
type Params struct {
	Memory      uint32 // KiB of RAM, e.g. 65536 = 64 MiB
	Iterations  uint32 // passes over the memory buffer
	Parallelism uint8  // lanes / threads
	SaltLength  uint32 // random salt bytes
	KeyLength   uint32 // output digest bytes
}

// DefaultParams returns OWASP-recommended Argon2id parameters for 2025/2026.
// Always call this rather than hard-coding numbers at call sites, so hashing
// and the rehash check stay in sync.
func DefaultParams() *Params {
	return &Params{
		Memory:      64 * 1024, // 64 MiB
		Iterations:  3,
		Parallelism: 4,
		SaltLength:  16,
		KeyLength:   32,
	}
}

// Hash derives an Argon2id hash and returns a self-describing PHC string that
// is safe to store directly in a database column.
//
//	$argon2id$v=19$m=65536,t=3,p=4$<b64salt>$<b64hash>
func Hash(plaintext string, params *Params) (string, error) {
	if params == nil {
		params = DefaultParams()
	}

	salt := make([]byte, params.SaltLength)
	if _, err := rand.Read(salt); err != nil { // crypto/rand — CSPRNG
		return "", fmt.Errorf("password.Hash: generating salt: %w", err)
	}

	key := argon2.IDKey([]byte(plaintext), salt,
		params.Iterations, params.Memory, params.Parallelism, params.KeyLength)

	return fmt.Sprintf(
		"$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version,
		params.Memory, params.Iterations, params.Parallelism,
		base64.RawStdEncoding.EncodeToString(salt),
		base64.RawStdEncoding.EncodeToString(key),
	), nil
}

// Verify reports whether plaintext matches encoded.
//   - nil                     → correct password
//   - ErrMismatch             → wrong password
//   - ErrInvalidHash / ...    → the stored hash is malformed/incompatible
//
// At the HTTP layer, collapse ALL non-nil results into one generic
// "invalid credentials" response so you never leak account existence or
// internal storage details.
func Verify(plaintext, encoded string) error {
	params, salt, storedKey, err := decode(encoded)
	if err != nil {
		return err
	}

	candidate := argon2.IDKey([]byte(plaintext), salt,
		params.Iterations, params.Memory, params.Parallelism, params.KeyLength)

	// Constant-time comparison — mandatory to defeat timing attacks.
	if subtle.ConstantTimeCompare(storedKey, candidate) != 1 {
		return ErrMismatch
	}
	return nil
}

// NeedsRehash reports whether encoded was produced with weaker parameters than p.
// Call it AFTER a successful Verify to transparently upgrade old hashes:
//
//	if err := password.Verify(pw, stored); err == nil {
//	    if password.NeedsRehash(stored, password.DefaultParams()) {
//	        if newHash, e := password.Hash(pw, password.DefaultParams()); e == nil {
//	            db.UpdatePasswordHash(userID, newHash) // silent upgrade
//	        }
//	    }
//	}
func NeedsRehash(encoded string, p *Params) bool {
	stored, _, _, err := decode(encoded)
	if err != nil {
		return true // malformed/old → definitely re-hash
	}
	return stored.Memory != p.Memory ||
		stored.Iterations != p.Iterations ||
		stored.Parallelism != p.Parallelism ||
		stored.KeyLength != p.KeyLength
}

// decode parses a PHC Argon2id string into its parts, validating algo + version.
func decode(encoded string) (p *Params, salt, key []byte, err error) {
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 {
		return nil, nil, nil, ErrInvalidHash
	}
	// parts: ["", "argon2id", "v=19", "m=..,t=..,p=..", "<b64salt>", "<b64hash>"]
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

	if salt, err = base64.RawStdEncoding.DecodeString(parts[4]); err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	p.SaltLength = uint32(len(salt))

	if key, err = base64.RawStdEncoding.DecodeString(parts[5]); err != nil {
		return nil, nil, nil, ErrInvalidHash
	}
	p.KeyLength = uint32(len(key))

	return p, salt, key, nil
}
```

Usage:

```go
package main

import (
	"errors"
	"fmt"
	"log"

	"github.com/yourname/authexample/internal/password"
)

func main() {
	plain := "hunter2hunter2"

	encoded, err := password.Hash(plain, password.DefaultParams()) // on registration
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Store this:", encoded)

	if err := password.Verify(plain, encoded); err == nil { // on login
		fmt.Println("Login OK")
	}

	if err := password.Verify("wrong", encoded); errors.Is(err, password.ErrMismatch) {
		fmt.Println("Wrong password rejected") // generic msg to client; detail logged internally
	}
}
```

> **Best practice:** keep this package free of HTTP/DB concerns — it is pure crypto. The web layer (§14) decides what to *tell the client* and what to *log*. Separation keeps the security-critical core small and auditable.

---

## PART B — JSON Web Tokens

---

## 8. What a JWT Is: Structure, Claims, Signed-Not-Secret **[B/I]**

Password hashing answers *"is this the right secret at login time?"* JWTs answer the next question: *"on every subsequent request, how does the server know who this is — without a database lookup each time?"* A JWT is a **self-contained, signed assertion of identity** that the client carries and presents with each request.

### Stateless vs stateful sessions — the core trade-off

- **Stateful sessions** (the classic approach): on login the server stores session data and gives the client an opaque random ID (a cookie). Every request, the server looks that ID up in a session store. **Pro:** instant revocation (delete the row). **Con:** every request hits the store; harder to scale horizontally.
- **Stateless JWTs:** the server signs a token containing the identity claims and gives it to the client. On each request the server *verifies the signature* — no lookup needed — and trusts the claims inside. **Pro:** no per-request DB hit; trivially scales across many servers that share only the signing key. **Con:** revocation is hard (the token is valid until it expires; §12, §17 cover mitigations).

JWTs trade easy revocation for statelessness and scale. That trade-off is the single most important thing to understand about them.

### The three parts

A JWT is three base64url-encoded JSON segments joined by dots: `header.payload.signature`.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9        ← header
.
eyJzdWIiOiI0MiIsImVtYWlsIjoiYWxpY2VAZXgu... ← payload (claims)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c ← signature
```

**Header** — declares the algorithm and type:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload (claims)** — the assertions:
```json
{ "sub": "42", "email": "alice@example.com", "role": "user", "exp": 1750000900 }
```

**Signature** — the issuer's HMAC or signature over `base64url(header) + "." + base64url(payload)`. Anyone can *read* the first two parts; only the holder of the signing key can *produce a valid signature*. That asymmetry is what makes the token trustworthy.

### THE critical misconception: signed ≠ encrypted

> **A JWT is signed, not encrypted.** The payload is base64url-*encoded*, which is reversible by anyone — it is **not** secret. Paste any JWT into a decoder (or `base64 -d` the middle segment) and read every claim. The signature guarantees *integrity and authenticity* (nobody tampered with it, and it came from someone holding the key) — it guarantees **nothing** about confidentiality.
>
> Therefore: **never put secrets in a JWT payload** — no passwords, SSNs, card numbers, API keys, or sensitive PII. Put only what the client is allowed to see and what the server needs to identify and authorize them. If you genuinely need an encrypted payload, that is **JWE** (JSON Web Encryption), a different and heavier construct — usually a sign you should rethink the design.

### Registered claims (RFC 7519)

These standard claim names have defined meanings; the JWT library validates several automatically.

| Claim | Name | Type | Purpose |
|-------|------|------|---------|
| `iss` | Issuer | string | Which service signed the token (e.g. `"auth.example.com"`) |
| `sub` | Subject | string | Who the token is about — use a **stable** ID (DB primary key), not email |
| `aud` | Audience | string / []string | Intended recipient(s); a service should reject tokens not meant for it |
| `exp` | Expiration | NumericDate | Reject after this time — **always set it** |
| `nbf` | Not Before | NumericDate | Reject before this time |
| `iat` | Issued At | NumericDate | When it was created |
| `jti` | JWT ID | string | Unique id — the hook for revocation/blocklisting (§12) |

You add your own *custom* (private) claims (`email`, `role`, etc.) alongside these. Keep the payload lean: it is sent on **every** request, so bytes cost bandwidth, and anything you put in is readable by the client.

---

## 9. Signing Algorithms: HMAC vs RSA vs ECDSA **[I]**

The `alg` in the header determines how the signature is produced and verified. The choice is architectural, not cosmetic.

### Symmetric (HMAC) — one shared secret

**HS256/384/512** use HMAC with a SHA-2 hash and a **single shared secret** (`[]byte`). The same secret both *signs* and *verifies*. 

- **Logic:** fast, simple, small tokens. But anyone who can verify can also forge, because they hold the same key.
- **Use when:** the issuer and the verifier are the **same trust domain** — a monolith, or a single API that both issues and checks its own tokens. This is the common case and the default for most apps.
- **Key requirement:** a high-entropy secret of **at least 32 random bytes** (256 bits) for HS256. A short or guessable secret is brute-forceable offline once an attacker has any valid token (they try secrets until the signature verifies).

### Asymmetric (RSA / ECDSA) — a key pair

**RS256** (RSA), **ES256** (ECDSA), **PS256** (RSA-PSS) use a **private key to sign** and a **public key to verify**.

- **Logic:** the auth server keeps the private key secret and *publishes the public key*. Any number of other services can verify tokens with the public key but **cannot forge** them — they lack the private key.
- **Use when:** **multiple independent services** verify tokens but only one issues them (microservices), or a **third party** needs to verify your tokens without being trusted to mint them. This is the standard for distributed systems and is how OIDC/JWKS works (§16).
- **ES256 vs RS256:** ECDSA keys and signatures are much smaller and verification is faster, for equivalent security — prefer **ES256** for new asymmetric designs unless an existing system mandates RSA. **PS256** is RSA with safer probabilistic padding (preferable to RS256 if you must use RSA).

### Choosing — the one-line rule

| Situation | Choose |
|-----------|--------|
| Single service issues and verifies its own tokens | **HS256** + 32-byte random secret |
| Many services verify, one issues / third parties verify | **ES256** (or RS256/PS256) — publish the public key |

| Algorithm | Type | Keys | Token size | When |
|-----------|------|------|-----------|------|
| HS256/384/512 | HMAC (symmetric) | one shared `[]byte` secret | small | monolith / single service |
| RS256 | RSA (asymmetric) | private signs, public verifies | large | legacy / RSA-mandated microservices |
| PS256 | RSA-PSS (asymmetric) | RSA pair, safer padding | large | prefer over RS256 when RSA required |
| ES256 | ECDSA (asymmetric) | EC pair | small | **preferred asymmetric** choice |

> **⚡ Version note (golang-jwt v5):** the per-algorithm signing methods are values like `jwt.SigningMethodHS256`, `jwt.SigningMethodRS256`, `jwt.SigningMethodES256`. v5 validates `exp` automatically in `ParseWithClaims` and prefers typed `RegisteredClaims` over the removed `StandardClaims`. The single most important security consequence of the algorithm field is the **alg-confusion attack** — covered in §11, and it is not optional reading.

---

## 10. Creating & Signing a Token (golang-jwt v5) **[I]**

To issue a token you: define your claims struct (embedding `jwt.RegisteredClaims`), build a `*jwt.Token` with a signing method and those claims, then `SignedString` with the key. The output is the `header.payload.signature` string you return to the client.

### Designing the claims struct

Embed `jwt.RegisteredClaims` (a value type, not pointer) to get `sub`/`exp`/`iat`/`iss`/etc. with automatic validation, then add lean custom fields. Each custom field needs a JSON tag — that string is the claim name in the payload.

```go
// Package token issues and validates JWTs for the auth service.
package token

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// CustomClaims = standard registered claims + our app-specific claims.
//
// ⚡ Version note (v5): embed jwt.RegisteredClaims (jwt.StandardClaims was
// REMOVED in v5). Keep the custom part minimal: the token rides on EVERY
// request and the payload is readable by the client — never put secrets here.
type CustomClaims struct {
	jwt.RegisteredClaims          // sub, exp, iat, iss, nbf, aud, jti
	Email                string `json:"email"`
	Role                 string `json:"role"`
}

// Config holds the secrets and lifetimes for issuing tokens.
// Load every field from the environment / a secrets manager — never hard-code.
type Config struct {
	AccessSecret  []byte        // HS256 secret for access tokens (>= 32 random bytes)
	RefreshSecret []byte        // SEPARATE secret for refresh tokens (defense in depth)
	AccessTTL     time.Duration // short, e.g. 15 * time.Minute
	RefreshTTL    time.Duration // long, e.g. 7 * 24 * time.Hour
	Issuer        string        // e.g. "auth.example.com"
	Audience      string        // who should accept these, e.g. "api.example.com"
}

// NewAccessToken builds and signs a short-lived access token for a user.
func NewAccessToken(userID, email, role string, cfg *Config) (string, error) {
	now := time.Now()

	claims := CustomClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,                                       // STABLE id (DB PK), not email
			Issuer:    cfg.Issuer,                                   // who signed it
			Audience:  jwt.ClaimStrings{cfg.Audience},               // who may accept it
			IssuedAt:  jwt.NewNumericDate(now),                      // iat
			NotBefore: jwt.NewNumericDate(now),                      // nbf — not usable before now
			ExpiresAt: jwt.NewNumericDate(now.Add(cfg.AccessTTL)),   // exp — auto-validated on parse
		},
		Email: email,
		Role:  role,
	}

	// jwt.SigningMethodHS256 is HMAC-SHA256. Pair the method with the claims...
	tok := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

	// ...then sign with the secret to get "header.payload.signature".
	// For HS256 the key is the []byte secret (>= 32 random bytes).
	signed, err := tok.SignedString(cfg.AccessSecret)
	if err != nil {
		return "", fmt.Errorf("token.NewAccessToken: signing: %w", err)
	}
	return signed, nil
}
```

Constructing the config at startup:

```go
cfg := &token.Config{
	AccessSecret:  []byte(os.Getenv("JWT_ACCESS_SECRET")),  // validate length at startup!
	RefreshSecret: []byte(os.Getenv("JWT_REFRESH_SECRET")),
	AccessTTL:     15 * time.Minute,
	RefreshTTL:    7 * 24 * time.Hour,
	Issuer:        "auth.example.com",
	Audience:      "api.example.com",
}
access, err := token.NewAccessToken("42", "alice@example.com", "admin", cfg)
```

Generating a strong secret:

```bash
# POSIX / Git Bash — 32 random bytes, hex-encoded (64 hex chars = 256 bits):
openssl rand -hex 32

# PowerShell equivalent:
# [Convert]::ToHexString((1..32 | ForEach-Object {Get-Random -Max 256}))  # quick-and-dirty
# Prefer a real CSPRNG; in Go: b:=make([]byte,32); crypto/rand.Read(b); hex.EncodeToString(b)
```

> **Security recommendations:** (1) Always set `exp` — a token without expiry never dies. (2) Set `iss` and `aud` and validate them on the verify side so a token minted for service A cannot be replayed at service B. (3) Use `sub` = stable internal ID, never email (emails change; you would break tokens or, worse, mis-identify a reused address). (4) Separate secrets for access vs refresh so leaking one does not compromise the other.

---

## 11. Parsing & Validating + the alg-Confusion Attack **[I/A]**

Parsing is where the security lives. `jwt.ParseWithClaims` checks the signature and the time-based claims (`exp`, `nbf`) for you — **but only if you supply the key function correctly**. Get the key function wrong and you open the most famous JWT vulnerability class.

### The alg-confusion / `alg:none` attack — the threat

The token *header* declares which algorithm to use. A naive verifier reads `alg` from the header and trusts it. Two devastating attacks follow:

1. **`alg: none`.** The attacker crafts a token with header `{"alg":"none"}` and *no signature*, with whatever claims they like (`"role":"admin"`). A verifier that honors the header's `none` accepts an unsigned, attacker-controlled token. Instant total compromise.
2. **RS256 → HS256 downgrade (key confusion).** A server uses RS256 and verifies with its **public** key (which is, by design, public). The attacker changes the header to `HS256`, then signs a forged token using the *public key bytes as the HMAC secret*. If the verifier blindly feeds whatever key it has into whatever algorithm the header names, the HMAC verification succeeds — the attacker forged a valid token using only public information.

**The defense is the same for both:** the *verifier* decides the algorithm, never the token. In golang-jwt you do this in two layers — assert the concrete method type inside the key function, **and** pass `jwt.WithValidMethods` as a whitelist.

```go
package token

import (
	"errors"
	"fmt"

	"github.com/golang-jwt/jwt/v5"
)

// ParseAccessToken validates a raw JWT and returns the typed claims.
// Enforced: correct signature, algorithm == HS256, exp not passed, nbf reached,
// and matching iss / aud.
func ParseAccessToken(raw string, cfg *Config) (*CustomClaims, error) {
	tok, err := jwt.ParseWithClaims(
		raw,
		&CustomClaims{},
		func(t *jwt.Token) (interface{}, error) {
			// ───── ALGORITHM-CONFUSION DEFENSE (critical) ─────
			// Assert the concrete signing-method TYPE the verifier expects.
			// Reject anything that is not HMAC BEFORE returning a key. This kills
			// both "alg:none" (not *SigningMethodHMAC) and the RS256→HS256 downgrade
			// (an RSA-method token would be rejected here, so the public key is
			// never handed to the HMAC verifier).
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}
			// Only now return the secret used to verify the signature.
			return cfg.AccessSecret, nil
		},
		// Belt-and-suspenders: a hard whitelist of acceptable alg strings.
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithIssuer(cfg.Issuer),                // verify iss
		jwt.WithAudience(cfg.Audience),            // verify aud — reject tokens for other services
		jwt.WithExpirationRequired(),              // reject tokens with no exp at all
		// jwt.WithLeeway(30*time.Second),         // optional clock-skew tolerance (§17)
	)
	if err != nil {
		// v5 returns wrapped sentinel errors — match with errors.Is, never strings.
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

	claims, ok := tok.Claims.(*CustomClaims)
	if !ok || !tok.Valid {
		return nil, errors.New("token: could not extract valid claims")
	}
	return claims, nil
}
```

What the attack looks like, and why the check stops it:

```go
// Forged "none" token the attacker submits:
//   Header:    {"alg":"none","typ":"JWT"}
//   Payload:   {"sub":"attacker","role":"superuser","exp":9999999999}
//   Signature: (empty)
//
// In the keyFunc, t.Method is the "none" method — NOT *jwt.SigningMethodHMAC —
// so we return an error before any key is produced. jwt.WithValidMethods
// (["HS256"]) rejects it independently as a second gate. Both layers, always.
```

> **⚡ Version note (v5):** the validation knobs are passed as variadic `ParserOption`s to `ParseWithClaims` (`WithValidMethods`, `WithIssuer`, `WithAudience`, `WithExpirationRequired`, `WithLeeway`). In v5, `exp` is validated automatically when present; `WithExpirationRequired()` additionally rejects tokens that *omit* `exp`. Always check errors with `errors.Is` against `jwt.ErrTokenExpired`, `jwt.ErrSignatureInvalid`, etc. — string matching on `err.Error()` is brittle and breaks across versions.

> **Security recommendations:** (1) **Never** trust the header's `alg`. (2) Assert the method type *and* whitelist with `WithValidMethods`. (3) Validate `iss` and `aud`. (4) Require `exp`. (5) Use the matching key type for the matching algorithm — HMAC secret for HMAC only, public key for RSA/ECDSA only.

---

## 12. Access + Refresh Tokens: Rotation & Revocation **[I/A]**

### Why two tokens

A single long-lived token is a liability: if stolen it works for its whole lifetime, and you cannot easily revoke it (JWTs are stateless). The **access + refresh** pattern resolves the tension between *security* (short lifetimes) and *usability* (not forcing re-login every 15 minutes):

| Token | TTL | Where the client keeps it | Sent... | Job |
|-------|-----|---------------------------|---------|-----|
| **Access** | 5–15 min | in memory (JS variable) | on every API request (`Authorization: Bearer`) | proves identity for the request |
| **Refresh** | days–weeks | **httpOnly cookie** | only to the `/auth/refresh` endpoint | exchanges for a new access token when the old one expires |

The logic: the access token is short-lived, so a stolen one is useless within minutes. The refresh token lives long but is **never exposed to JavaScript** (httpOnly) and is sent to exactly one endpoint, shrinking its attack surface. Crucially, a short access TTL means revocation is "good enough" by default — a logged-out/banned user loses access within one access-token lifetime even if you do nothing else.

### Client-side storage — the XSS/CSRF trade-off (preview of §17)

| Location | XSS exposure | CSRF exposure | Verdict |
|----------|-------------|---------------|---------|
| `localStorage` / `sessionStorage` | **HIGH** — any script reads it | low | **Avoid** for tokens |
| **httpOnly cookie** | none — JS cannot read it | moderate → mitigate with `SameSite` + CSRF token | **best for refresh tokens** |
| memory (JS closure/state) | low — gone on reload | none | **best for the short-lived access token** |

```go
package token

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// RefreshClaims is intentionally minimal — a refresh token only needs to
// identify the user and itself (jti) so it can be rotated/revoked.
type RefreshClaims struct {
	jwt.RegisteredClaims // Subject = userID, ID (jti) = unique token id for revocation
}

// NewRefreshToken mints a long-lived refresh token. The jti should be a random
// UUID that you ALSO store server-side (DB/Redis); revocation = delete that row.
func NewRefreshToken(userID, jti string, cfg *Config) (string, error) {
	now := time.Now()
	claims := RefreshClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    cfg.Issuer,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(cfg.RefreshTTL)),
			ID:        jti, // jti — the revocation handle
		},
	}
	t := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signed, err := t.SignedString(cfg.RefreshSecret) // SEPARATE secret from access
	if err != nil {
		return "", fmt.Errorf("token.NewRefreshToken: %w", err)
	}
	return signed, nil
}

// ParseRefreshToken validates a refresh token with the same alg-confusion
// defenses as access tokens, but using the refresh secret.
func ParseRefreshToken(raw string, cfg *Config) (*RefreshClaims, error) {
	t, err := jwt.ParseWithClaims(raw, &RefreshClaims{},
		func(tok *jwt.Token) (interface{}, error) {
			if _, ok := tok.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", tok.Header["alg"])
			}
			return cfg.RefreshSecret, nil // distinct secret: a leaked access secret can't forge refresh tokens
		},
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithIssuer(cfg.Issuer),
		jwt.WithExpirationRequired(),
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

### Refresh-token rotation — single-use refresh tokens

The strongest pattern is **rotation**: each refresh consumes the old refresh token and issues a brand-new one (new `jti`). The old `jti` is invalidated server-side. The logic and its security payoff:

```
On POST /auth/refresh:
 1. Parse + validate the presented refresh token (signature, exp, iss).
 2. Look up its jti in the store. If absent/already-used/revoked → REJECT.
 3. Invalidate the old jti (delete or mark used).
 4. Issue a NEW access token AND a NEW refresh token with a fresh jti.
 5. Persist the new jti; return access token in body, refresh token in httpOnly cookie.

Why this is powerful (reuse detection):
 A refresh token can be used exactly ONCE. If an attacker steals one and uses it,
 the legitimate user's next refresh presents the now-invalidated old jti → it
 fails. That failure is a detectable signal of theft: you can then revoke the
 ENTIRE token family for that user and force re-authentication.
```

### Revocation strategies — making stateless tokens cancellable

Because JWTs are self-validating, "logging out" or "banning a user" does not automatically kill outstanding tokens. Pick a strategy proportional to your needs:

| Strategy | Complexity | Scales | When to use |
|----------|-----------|--------|-------------|
| **Short access TTL** | none | perfectly | tolerate ≤15 min of stale access after logout; refresh handles the rest |
| **jti blocklist (Redis)** | low | high | store revoked jtis with TTL = token's remaining life; check on each request |
| **per-user token_version** | medium | high | store an int per user; embed it in tokens; bump it to invalidate ALL of a user's tokens at once (logout-everywhere, password change) |
| **opaque/reference tokens** | high | needs a lookup per request | full instant revocation; effectively stateful sessions |

```go
// Minimal in-memory jti blocklist (use Redis with TTL in production):
var (
	revoked   = map[string]time.Time{} // jti -> expiry
	revokedMu sync.RWMutex
)

func revoke(jti string, exp time.Time) {
	revokedMu.Lock()
	revoked[jti] = exp
	revokedMu.Unlock()
}

func isRevoked(jti string) bool {
	revokedMu.RLock()
	exp, ok := revoked[jti]
	revokedMu.RUnlock()
	if !ok {
		return false
	}
	if time.Now().After(exp) { // lazily drop expired entries
		revokedMu.Lock()
		delete(revoked, jti)
		revokedMu.Unlock()
		return false
	}
	return true
}
// In the auth middleware, after parsing: if isRevoked(claims.ID) → 401.
```

> **Security recommendation:** the pragmatic baseline for most apps is **short access TTL + rotating refresh tokens + a refresh-token store you can delete from**. Add a jti blocklist or token_version only when you need faster-than-TTL revocation of *access* tokens. Always store refresh tokens server-side in *some* form — a refresh token you cannot invalidate defeats the purpose of the pattern.

---

## PART C — Putting It Together

---

## 13. Auth Middleware: net/http and Gin **[I]**

The job of auth middleware is identical in any framework: pull the bearer token out of the `Authorization` header, validate it (§11), and on success attach the verified claims so downstream handlers can use them — on failure, short-circuit with `401`. The pattern differs only in *how each framework passes data forward*: `net/http` uses the request `context.Context`; Gin uses its `*gin.Context` key/value store.

### Extracting the bearer token

The header format is `Authorization: Bearer <token>`. Split on the first space, case-insensitively match the scheme, take the rest as the token. Reject anything malformed before you even try to parse.

### net/http middleware (context-based)

`net/http` middleware is a function that wraps a handler. We store claims in the request context under a private key type (using a custom unexported type prevents collisions with other packages' context keys — a standard Go idiom).

```go
package main

import (
	"context"
	"errors"
	"net/http"
	"strings"

	"github.com/golang-jwt/jwt/v5"
)

// Private context-key type: prevents any other package from colliding with
// or reading our key by accident. Idiomatic for context values.
type ctxKey string

const claimsKey ctxKey = "claims"

// claimsFromContext retrieves the claims that the middleware injected.
func claimsFromContext(ctx context.Context) (*CustomClaims, bool) {
	c, ok := ctx.Value(claimsKey).(*CustomClaims)
	return c, ok
}

// AuthMiddleware validates the Bearer token and injects claims into the context.
// Wrap any protected handler: mux.Handle("/profile", AuthMiddleware(handler)).
func AuthMiddleware(cfg *Config, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		raw, err := bearerToken(r)
		if err != nil {
			http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
			return
		}

		claims, err := ParseAccessToken(raw, cfg) // §11 — all the alg-confusion defenses live here
		if err != nil {
			// Keep the client message generic; the detailed reason is for logs only.
			http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
			return
		}

		// (Optional) revocation check: if isRevoked(claims.ID) { 401 }

		// Attach claims; downstream handlers read them with claimsFromContext.
		ctx := context.WithValue(r.Context(), claimsKey, claims)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// bearerToken extracts the token from "Authorization: Bearer <token>".
func bearerToken(r *http.Request) (string, error) {
	h := r.Header.Get("Authorization")
	if h == "" {
		return "", errors.New("missing Authorization header")
	}
	parts := strings.SplitN(h, " ", 2)
	if len(parts) != 2 || !strings.EqualFold(parts[0], "bearer") || parts[1] == "" {
		return "", errors.New("malformed Authorization header")
	}
	return parts[1], nil
}
```

A protected handler reads the claims:

```go
func handleProfile(w http.ResponseWriter, r *http.Request) {
	claims, ok := claimsFromContext(r.Context())
	if !ok { // should never happen if the middleware ran; defensive
		http.Error(w, `{"error":"server error"}`, http.StatusInternalServerError)
		return
	}
	writeJSON(w, http.StatusOK, map[string]any{
		"user_id": claims.Subject, "email": claims.Email, "role": claims.Role,
	})
}
```

### Gin middleware (context-based)

In Gin, middleware is a `gin.HandlerFunc`; it stores values with `c.Set` and aborts with `c.AbortWithStatusJSON`. Downstream handlers read with `c.Get`/`c.MustGet`. For the full Gin treatment (routing, binding, file upload) see **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`**; here we focus on the auth layer.

```go
package main

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
)

const ginClaimsKey = "claims" // key in gin.Context

// GinAuthMiddleware validates the bearer token and stores claims on the context.
// Apply per-route: r.GET("/profile", GinAuthMiddleware(cfg), handleProfileGin)
// or per-group:    auth := r.Group("/api"); auth.Use(GinAuthMiddleware(cfg))
func GinAuthMiddleware(cfg *Config) gin.HandlerFunc {
	return func(c *gin.Context) {
		h := c.GetHeader("Authorization")
		parts := strings.SplitN(h, " ", 2)
		if len(parts) != 2 || !strings.EqualFold(parts[0], "bearer") || parts[1] == "" {
			// AbortWithStatusJSON stops the chain AND writes the response.
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
			return
		}

		claims, err := ParseAccessToken(parts[1], cfg)
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
			return
		}

		c.Set(ginClaimsKey, claims) // make claims available to later handlers
		c.Next()                    // proceed down the chain
	}
}

// claimsFromGin pulls the typed claims back out.
func claimsFromGin(c *gin.Context) (*CustomClaims, bool) {
	v, ok := c.Get(ginClaimsKey)
	if !ok {
		return nil, false
	}
	claims, ok := v.(*CustomClaims)
	return claims, ok
}

func handleProfileGin(c *gin.Context) {
	claims, ok := claimsFromGin(c)
	if !ok {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "server error"})
		return
	}
	c.JSON(http.StatusOK, gin.H{
		"user_id": claims.Subject, "email": claims.Email, "role": claims.Role,
	})
}
```

> **Best practice:** the *validation* logic (`ParseAccessToken`) is framework-agnostic and lives in one place; only the wrapper that reads the header and stores claims differs per framework. This keeps the security-critical code in a single auditable function and makes switching frameworks (or supporting both) trivial. Never duplicate the alg-confusion checks across middlewares.

---

## 14. Full Auth Flow with net/http **[I]**

A complete, self-contained server — register, login, and a protected endpoint — using only the standard library plus the two crypto packages. It demonstrates the patterns end to end: PHC hashing, JWT issuance, the bearer middleware, and the **user-enumeration / timing defenses** that beginners almost always miss. Builds directly on **`GO_NET_HTTP_REST_API_GUIDE.md`**; in production replace the in-memory map with a real database.

```go
// main.go — a complete, runnable auth server (in-memory store).
package main

import (
	"context"
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
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
)

// ── In-memory "database" (swap for PostgreSQL/SQLite in production) ──────────
type User struct {
	ID, Email, PasswordHash, Role string
}

var (
	mu    sync.RWMutex
	users = map[string]*User{} // keyed by email
)

// ── Config (load secrets from the environment; the fallbacks are DEV-ONLY) ──
var cfg = struct {
	AccessSecret []byte
	AccessTTL    time.Duration
	Issuer       string
}{
	AccessSecret: mustSecret("JWT_ACCESS_SECRET", 32), // fail fast if too short
	AccessTTL:    15 * time.Minute,
	Issuer:       "auth.example.com",
}

// mustSecret loads a secret from the env and refuses to start if it is too weak.
// Validating secret strength at boot prevents shipping a guessable HMAC key.
func mustSecret(key string, minLen int) []byte {
	v := os.Getenv(key)
	if len(v) < minLen {
		log.Fatalf("env %s must be >= %d bytes (got %d)", key, minLen, len(v))
	}
	return []byte(v)
}

// ── Argon2id helpers (condensed from the §7 package) ────────────────────────
const (
	argMem  = 64 * 1024
	argTime = 3
	argPar  = 4
	argSalt = 16
	argKey  = 32
)

func hashPassword(pw string) (string, error) {
	salt := make([]byte, argSalt)
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}
	k := argon2.IDKey([]byte(pw), salt, argTime, argMem, argPar, argKey)
	return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version, argMem, argTime, argPar,
		base64.RawStdEncoding.EncodeToString(salt),
		base64.RawStdEncoding.EncodeToString(k)), nil
}

func verifyPassword(pw, encoded string) error {
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 || parts[1] != "argon2id" {
		return errors.New("invalid hash format")
	}
	var mem, t uint32
	var par uint8
	if _, err := fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &mem, &t, &par); err != nil {
		return errors.New("invalid hash params")
	}
	salt, err := base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return errors.New("invalid salt")
	}
	stored, err := base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return errors.New("invalid hash")
	}
	cand := argon2.IDKey([]byte(pw), salt, t, mem, par, uint32(len(stored)))
	if subtle.ConstantTimeCompare(stored, cand) != 1 { // constant-time!
		return errors.New("password mismatch")
	}
	return nil
}

// ── JWT helpers ─────────────────────────────────────────────────────────────
type CustomClaims struct {
	jwt.RegisteredClaims
	Email string `json:"email"`
	Role  string `json:"role"`
}

func issueAccessToken(u *User) (string, error) {
	now := time.Now()
	claims := CustomClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   u.ID,
			Issuer:    cfg.Issuer,
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(cfg.AccessTTL)),
		},
		Email: u.Email, Role: u.Role,
	}
	return jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(cfg.AccessSecret)
}

func parseAccessToken(raw string) (*CustomClaims, error) {
	tok, err := jwt.ParseWithClaims(raw, &CustomClaims{},
		func(t *jwt.Token) (interface{}, error) {
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok { // alg-confusion defense
				return nil, fmt.Errorf("unexpected alg: %v", t.Header["alg"])
			}
			return cfg.AccessSecret, nil
		},
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithIssuer(cfg.Issuer),
		jwt.WithExpirationRequired(),
	)
	if err != nil {
		return nil, err
	}
	c, ok := tok.Claims.(*CustomClaims)
	if !ok || !tok.Valid {
		return nil, errors.New("invalid token")
	}
	return c, nil
}

// ── context plumbing ────────────────────────────────────────────────────────
type ctxKey string

const claimsKey ctxKey = "claims"

func claimsFromContext(ctx context.Context) (*CustomClaims, bool) {
	c, ok := ctx.Value(claimsKey).(*CustomClaims)
	return c, ok
}

// ── JSON helpers ────────────────────────────────────────────────────────────
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(v)
}
func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, map[string]string{"error": msg})
}

// ── Handlers ────────────────────────────────────────────────────────────────
type credentials struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

// POST /auth/register
func handleRegister(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
		return
	}
	var req credentials
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid request body")
		return
	}
	if len(req.Email) == 0 || len(req.Password) < 8 { // validate input
		writeError(w, http.StatusBadRequest, "email required; password >= 8 chars")
		return
	}

	mu.Lock()
	defer mu.Unlock()
	if _, exists := users[req.Email]; exists {
		// Generic message — do not confirm which emails are already registered.
		writeError(w, http.StatusConflict, "registration failed")
		return
	}

	hash, err := hashPassword(req.Password)
	if err != nil {
		log.Printf("hashPassword: %v", err) // log internally, never expose
		writeError(w, http.StatusInternalServerError, "internal error")
		return
	}

	id := fmt.Sprintf("user_%d", time.Now().UnixNano()) // use UUID/DB PK in prod
	role := "user"
	if req.Email == os.Getenv("ADMIN_SEED_EMAIL") {     // bootstrap an admin for testing
		role = "admin"
	}
	users[req.Email] = &User{ID: id, Email: req.Email, PasswordHash: hash, Role: role}
	writeJSON(w, http.StatusCreated, map[string]string{"id": id})
}

// A precomputed dummy hash with the SAME params, so verifying it costs the same
// as verifying a real one. Computed once at startup.
var dummyHash, _ = hashPassword("a-fixed-string-noone-logs-in-with")

// POST /auth/login
func handleLogin(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
		return
	}
	var req credentials
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	mu.RLock()
	u, exists := users[req.Email]
	mu.RUnlock()

	// USER-ENUMERATION / TIMING DEFENSE: run Argon2 even when the user does not
	// exist, against a dummy hash with identical params, so the response time for
	// "no such user" matches "wrong password". Otherwise an attacker times the
	// endpoint to discover which emails are registered.
	toCheck := dummyHash
	if exists {
		toCheck = u.PasswordHash
	}
	verifyErr := verifyPassword(req.Password, toCheck)

	if !exists || verifyErr != nil {
		// One generic error for BOTH "no such user" and "wrong password".
		writeError(w, http.StatusUnauthorized, "invalid credentials")
		return
	}

	access, err := issueAccessToken(u)
	if err != nil {
		log.Printf("issueAccessToken: %v", err)
		writeError(w, http.StatusInternalServerError, "internal error")
		return
	}

	// In a full implementation also mint a refresh token here and set it as an
	// httpOnly, Secure, SameSite=Strict cookie scoped to /auth/refresh:
	//
	//   http.SetCookie(w, &http.Cookie{
	//       Name: "refresh_token", Value: refresh,
	//       HttpOnly: true, Secure: true, SameSite: http.SameSiteStrictMode,
	//       Path: "/auth/refresh", MaxAge: int((7 * 24 * time.Hour).Seconds()),
	//   })

	writeJSON(w, http.StatusOK, map[string]string{
		"access_token": access,
		"token_type":   "Bearer",
		"expires_in":   fmt.Sprintf("%d", int(cfg.AccessTTL.Seconds())),
	})
}

// AuthMiddleware: validate bearer token, inject claims.
func AuthMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		h := r.Header.Get("Authorization")
		parts := strings.SplitN(h, " ", 2)
		if len(parts) != 2 || !strings.EqualFold(parts[0], "bearer") || parts[1] == "" {
			writeError(w, http.StatusUnauthorized, "missing or malformed Authorization header")
			return
		}
		claims, err := parseAccessToken(parts[1])
		if err != nil {
			writeError(w, http.StatusUnauthorized, "invalid token") // generic to client
			return
		}
		ctx := context.WithValue(r.Context(), claimsKey, claims)
		next(w, r.WithContext(ctx))
	}
}

// GET /profile (protected)
func handleProfile(w http.ResponseWriter, r *http.Request) {
	claims, ok := claimsFromContext(r.Context())
	if !ok {
		writeError(w, http.StatusInternalServerError, "no claims in context")
		return
	}
	writeJSON(w, http.StatusOK, map[string]any{
		"user_id": claims.Subject, "email": claims.Email, "role": claims.Role,
	})
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/auth/register", handleRegister)
	mux.HandleFunc("/auth/login", handleLogin)
	mux.HandleFunc("/profile", AuthMiddleware(handleProfile)) // protected

	log.Println("listening on :8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		log.Fatal(err)
	}
}
```

Exercise it (POSIX shell):

```bash
export JWT_ACCESS_SECRET=$(openssl rand -hex 32)   # set BEFORE running the server
go run .

# Register
curl -s -X POST localhost:8080/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"alice@example.com","password":"hunter2hunter2"}'

# Login → capture the token
TOKEN=$(curl -s -X POST localhost:8080/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"alice@example.com","password":"hunter2hunter2"}' | sed 's/.*"access_token":"\([^"]*\)".*/\1/')

# Protected route
curl -s localhost:8080/profile -H "Authorization: Bearer $TOKEN"

# No token → 401
curl -s localhost:8080/profile
```

> **Security recommendations recap for this flow:** identical errors and timing for login failures (no enumeration); generic client messages with detailed server logs; secret validated at boot; constant-time password compare; `exp` always set and required; refresh token (when added) in an httpOnly+Secure+SameSite cookie.

---

## 15. RBAC: Roles, Permissions & Enforcement **[I/A]**

**Authentication** answers *"who are you?"* (the JWT, validated). **Authorization** answers *"what are you allowed to do?"* — and that is **RBAC** (Role-Based Access Control). The two are separate layers: a valid token gets you *in the door* (`401` if not); your role decides which *rooms* you may enter (`403` if not). Returning the wrong status code here is a common, meaningful bug: **`401 Unauthorized` = "I don't know who you are"; `403 Forbidden` = "I know who you are, and you may not."**

### Roles vs permissions

- **Roles** (`user`, `author`, `admin`) are coarse labels grouping capabilities. Simple, but coarse-grained.
- **Permissions** (`posts:create`, `posts:delete`, `users:ban`) are fine-grained capabilities. More flexible; usually you map roles → permissions and check the *permission* at the handler. This decouples "what a route needs" from "which roles happen to have it," so you can re-org roles without touching every route.

### Where the role lives

Embed the role (or roles) as a claim so enforcement needs no DB lookup — but understand the trade-off: because the token is self-contained, a role *change takes effect only when the token is refreshed*. For instant de-privileging (e.g., firing an admin) you need a revocation mechanism (§12) or a per-request DB check for the most sensitive actions. Put `role` in the access token for ordinary gating; do a live check for the truly dangerous operations.

### Role-enforcement middleware (net/http)

```go
package main

import "net/http"

// RequireRole wraps a handler and allows it only for a single role.
// Apply it AFTER AuthMiddleware, which injects the claims:
//   mux.HandleFunc("/admin", AuthMiddleware(RequireRole("admin", handleAdmin)))
func RequireRole(required string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		claims, ok := claimsFromContext(r.Context())
		if !ok {
			writeError(w, http.StatusUnauthorized, "unauthorized") // shouldn't happen post-auth
			return
		}
		if claims.Role != required {
			// 403, NOT 401 — they are authenticated but lack permission.
			writeError(w, http.StatusForbidden, "insufficient permissions")
			return
		}
		next(w, r)
	}
}

// RequireAnyRole allows access if the user has ANY of the listed roles.
// (Variadics must be last in Go, so we take a []string and keep next trailing.)
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
```

### Permission-based model (multi-role + permission map)

```go
// A claims type carrying multiple roles. For permissions, either embed them or
// derive them from roles via a server-side map (shown below).
type MultiRoleClaims struct {
	jwt.RegisteredClaims
	Email string   `json:"email"`
	Roles []string `json:"roles"` // e.g. ["author","moderator"]
}

// rolePermissions maps each role to the permissions it grants. Keeping this
// server-side (not in the token) lets you change a role's powers without
// reissuing every token.
var rolePermissions = map[string][]string{
	"reader": {"posts:read"},
	"author": {"posts:read", "posts:create", "posts:update_own"},
	"admin":  {"posts:read", "posts:create", "posts:update_own", "posts:delete", "users:ban"},
}

// hasPermission reports whether ANY of the user's roles grants perm.
func hasPermission(roles []string, perm string) bool {
	for _, role := range roles {
		for _, p := range rolePermissions[role] {
			if p == perm {
				return true
			}
		}
	}
	return false
}
```

### Ownership checks — beyond roles

Roles are not enough when users act on *their own* resources. "An author may edit a post" must become "an author may edit *their* post" — otherwise any author edits anyone's. Compare the resource owner against the `sub` claim:

```go
// handleUpdatePost: an author may edit ONLY their own post; an admin may edit any.
func handleUpdatePost(w http.ResponseWriter, r *http.Request) {
	claims, _ := claimsFromContext(r.Context())
	post := loadPost(r) // pseudo: fetch the target post

	// Ownership OR elevated role. This object-level check is what stops the
	// "horizontal privilege escalation" / IDOR class of bugs.
	if post.AuthorID != claims.Subject && claims.Role != "admin" {
		writeError(w, http.StatusForbidden, "you can only edit your own posts")
		return
	}
	// ... perform the update ...
}
```

Wiring it together:

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/auth/register", handleRegister)
	mux.HandleFunc("/auth/login", handleLogin)
	mux.HandleFunc("/profile", AuthMiddleware(handleProfile))                       // any authenticated user
	mux.HandleFunc("/admin/dashboard", AuthMiddleware(RequireRole("admin", adminDash))) // admins only
	http.ListenAndServe(":8080", mux)
}
```

> **Security recommendations:** (1) `401` for unauthenticated, `403` for authenticated-but-forbidden — never blur them. (2) **Deny by default** — a route with no auth wrapper should be a deliberate, reviewed choice; prefer applying auth at the router/group level so new routes are protected unless explicitly opened. (3) Always add **object-level** ownership checks (the IDOR defense), not just role checks. (4) Remember role claims are stale until token refresh — use revocation or live checks for instant de-privileging.

---

## 16. Asymmetric Signing & Key Management/Rotation **[A]**

When more than one service must verify tokens, symmetric HMAC stops being safe: sharing the secret with every verifier means every verifier can also *forge*. The asymmetric model fixes this — the auth server signs with a **private** key it never shares, and verifiers check with the corresponding **public** key, which is safe to distribute.

### Generating an ECDSA key pair (preferred) and RSA (legacy)

```bash
# ECDSA P-256 (ES256) — small, fast, modern. PREFERRED.
openssl ecparam -name prime256v1 -genkey -noout -out ec_private.pem
openssl ec -in ec_private.pem -pubout -out ec_public.pem

# RSA 2048+ (RS256/PS256) — larger; use when a system mandates RSA.
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out rsa_private.pem
openssl rsa -in rsa_private.pem -pubout -out rsa_public.pem
```

### Signing and verifying with ES256

```go
package token

import (
	"crypto/ecdsa"
	"crypto/x509"
	"encoding/pem"
	"fmt"
	"os"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// loadECPrivateKey parses a PEM EC private key (held ONLY by the auth server).
func loadECPrivateKey(path string) (*ecdsa.PrivateKey, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, fmt.Errorf("no PEM block in %s", path)
	}
	// golang-jwt provides parsers, but the stdlib works too:
	return x509.ParseECPrivateKey(block.Bytes)
}

// loadECPublicKey parses a PEM EC public key (distributed to verifiers).
func loadECPublicKey(path string) (*ecdsa.PublicKey, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, fmt.Errorf("no PEM block in %s", path)
	}
	pub, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	ecPub, ok := pub.(*ecdsa.PublicKey)
	if !ok {
		return nil, fmt.Errorf("not an ECDSA public key")
	}
	return ecPub, nil
}

// SignES256 signs with the PRIVATE key (auth server only).
func SignES256(claims jwt.Claims, priv *ecdsa.PrivateKey, kid string) (string, error) {
	tok := jwt.NewWithClaims(jwt.SigningMethodES256, claims)
	tok.Header["kid"] = kid // KEY ID — tells verifiers which public key to use (see rotation)
	return tok.SignedString(priv)
}

// VerifyES256 verifies with the PUBLIC key (any verifier).
func VerifyES256(raw string, pub *ecdsa.PublicKey, issuer string) (*CustomClaims, error) {
	tok, err := jwt.ParseWithClaims(raw, &CustomClaims{},
		func(t *jwt.Token) (interface{}, error) {
			// Same alg-confusion discipline as HMAC, but assert the ECDSA type.
			if _, ok := t.Method.(*jwt.SigningMethodECDSA); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}
			return pub, nil
		},
		jwt.WithValidMethods([]string{"ES256"}), // whitelist — never accept HS256 here!
		jwt.WithIssuer(issuer),
		jwt.WithExpirationRequired(),
	)
	if err != nil {
		return nil, err
	}
	c, ok := tok.Claims.(*CustomClaims)
	if !ok || !tok.Valid {
		return nil, fmt.Errorf("invalid token")
	}
	_ = time.Now
	return c, nil
}
```

> **Critical:** when a service verifies asymmetric tokens, its `WithValidMethods` whitelist must list **only** the asymmetric algorithm (`ES256`/`RS256`). If a verifier that holds a public key also accepts `HS256`, an attacker can mount the RS256→HS256 downgrade (§11) using the public key as an HMAC secret. The two-layer defense (type assertion + whitelist) is non-negotiable in the asymmetric case.

### Key rotation and the `kid` header — JWKS

Keys must be rotatable: routinely (hygiene) and urgently (on compromise). You cannot flip instantly because tokens signed with the old key are still in flight. The mechanism is the **`kid` (key ID)** header plus a *set* of public keys (a **JWKS** — JSON Web Key Set):

```
Rotation with kid + JWKS:
 1. Auth server holds the CURRENT private key and tags each token's header
    with kid = "key-2026-06".
 2. It publishes ALL currently-valid PUBLIC keys at a JWKS endpoint, e.g.
    GET /.well-known/jwks.json  →  { "keys": [ {kid:"key-2026-06", ...}, {kid:"key-2025-12", ...} ] }
 3. A verifier reads the token's kid, picks the matching public key from the set,
    and verifies. (Cache the JWKS; refresh periodically or on an unknown kid.)
 4. To rotate: generate a new key, add its public half to the JWKS, start signing
    with it. Keep the OLD public key in the set until every token signed with it
    has expired — then remove it. Zero downtime.
 5. To revoke a compromised key: remove it from the JWKS immediately; all its
    tokens fail verification at once.
```

For symmetric (HMAC) rotation, the analogue is supporting two secrets at once:

```go
// Symmetric rotation: try the current secret, then the previous one, within a
// short overlap window (~ one access-token TTL). Issue new tokens with current.
func parseWithRotation(raw string, current, previous []byte, issuer string) (*CustomClaims, error) {
	keyFunc := func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected alg: %v", t.Header["alg"])
		}
		// We cannot know which secret was used, so we return current and, on
		// failure, retry with previous (below). The library calls keyFunc once,
		// so the retry is at the caller level.
		return current, nil
	}
	tok, err := jwt.ParseWithClaims(raw, &CustomClaims{}, keyFunc,
		jwt.WithValidMethods([]string{"HS256"}), jwt.WithIssuer(issuer), jwt.WithExpirationRequired())
	if err == nil {
		if c, ok := tok.Claims.(*CustomClaims); ok && tok.Valid {
			return c, nil
		}
	}
	// Retry with the previous secret during the overlap window.
	tok, err = jwt.ParseWithClaims(raw, &CustomClaims{},
		func(t *jwt.Token) (interface{}, error) { return previous, nil },
		jwt.WithValidMethods([]string{"HS256"}), jwt.WithIssuer(issuer), jwt.WithExpirationRequired())
	if err != nil {
		return nil, err
	}
	c, ok := tok.Claims.(*CustomClaims)
	if !ok || !tok.Valid {
		return nil, fmt.Errorf("invalid token")
	}
	return c, nil
}
```

> **Security recommendations:** (1) **Never** distribute a private key — only the auth server holds it. (2) Store private keys in a secrets manager / KMS / HSM, not in the repo or a plain env var if you can avoid it; restrict file permissions on PEM files. (3) Always tag tokens with `kid` so rotation is possible. (4) Rotate on a schedule and immediately on suspected compromise. (5) For verifiers, lock `WithValidMethods` to the exact asymmetric alg. (6) Choose RSA ≥ 2048 bits or ECDSA P-256; reject weaker keys.

---

## 17. The Security Section: Attacks & Defenses **[A]**

A consolidated, rigorous reference. Authentication is adversarial; treat each item as a control you must consciously have or consciously waive.

### Timing attacks

- **Password compare:** always `crypto/subtle.ConstantTimeCompare` — never `==`/`bytes.Equal` on secret-derived values (§6).
- **User enumeration via login timing:** run Argon2 against a dummy hash even when the user does not exist, so "no such user" and "wrong password" take the same time and return the same error (§14). The same applies to registration ("email already taken" should not be distinguishable) and password-reset endpoints ("we sent an email if the account exists").

### Token storage on the client — httpOnly cookies vs localStorage

This is the decision that most affects real-world breach impact.

| Store | Read by JS? | Sent automatically? | Main risk | Use for |
|-------|-------------|---------------------|-----------|---------|
| `localStorage`/`sessionStorage` | **yes** | no | **XSS** steals every token | nothing sensitive |
| **httpOnly cookie** | **no** | yes (to matching paths) | **CSRF** | **refresh tokens** |
| in-memory JS variable | yes (same page) | no | lost on reload; minor XSS window | short-lived access tokens in SPAs |

- **XSS** (cross-site scripting): if an attacker can run JavaScript on your page, anything in `localStorage` is exfiltrated instantly. `httpOnly` cookies are invisible to JS, so XSS cannot read them. Therefore: **refresh tokens in httpOnly cookies; access tokens in memory only.** And of course, prevent XSS at the source (output encoding, a strict Content-Security-Policy).
- **CSRF** (cross-site request forgery): the flip side. Because the browser sends cookies *automatically*, a malicious site can trigger authenticated requests using the victim's cookie. Mitigations:
  - **`SameSite=Strict`** (or `Lax`) on the cookie — the browser will not send it on cross-site requests. This alone stops most CSRF.
  - **`Secure`** — only sent over HTTPS.
  - **`HttpOnly`** — not readable by JS.
  - Scope with **`Path`** (e.g., `/auth/refresh`) so the refresh cookie rides only on the one endpoint that needs it.
  - For state-changing requests, add a **CSRF token** (double-submit cookie or synchronizer token) as defense in depth, especially if you must use `SameSite=Lax`.

```go
http.SetCookie(w, &http.Cookie{
	Name:     "refresh_token",
	Value:    refresh,
	HttpOnly: true,                    // JS cannot read it → XSS-resistant
	Secure:   true,                    // HTTPS only
	SameSite: http.SameSiteStrictMode, // CSRF-resistant
	Path:     "/auth/refresh",         // narrow scope
	MaxAge:   int((7 * 24 * time.Hour).Seconds()),
})
```

### Replay attacks

A captured valid token can be re-sent. Defenses: **HTTPS everywhere** (prevents capture on the wire); **short access TTLs** (a replayed token expires quickly); **refresh-token rotation with reuse detection** (§12) so a replayed refresh token is caught and invalidates the family; `jti` blocklisting for single-use tokens (e.g., password-reset/magic-link tokens must be one-time). Validate `aud` so a token captured at one service cannot be replayed at another.

### Secret management

- **Never** hard-code secrets or commit them. Load from environment / secrets manager (Vault, AWS/GCP Secrets Manager, KMS). Validate length/strength **at startup** and refuse to boot if weak (the `mustSecret` pattern, §14).
- Use **distinct** secrets for access vs refresh tokens, and per environment (dev/staging/prod).
- Rotate regularly and on suspected compromise (§16). Keep private keys off application servers where feasible.
- **Never log** secrets, tokens, or passwords:

```go
// WRONG — leaks credentials/tokens into logs (which are often less protected):
//   log.Printf("login: %s / %s -> token %s", email, password, token)
// RIGHT — log only non-sensitive context:
log.Printf("login ok user=%s id=%s", email, userID)
```

### Token expiry — pick TTLs deliberately

| Scenario | Access TTL | Refresh TTL | Notes |
|----------|-----------|-------------|-------|
| Banking / high security | 5 min | 1 day (MFA on refresh) | aggressive |
| Standard SaaS | 15 min | 7 days | good default |
| Mobile (comfort) | 1 hour | 30 days | acceptable *with* rotation |
| Long-lived integration | — | — | use opaque DB-backed API keys instead of JWTs |

### Algorithm confusion (recap, because it is the #1 JWT CVE class)

```
ALWAYS assert the concrete signing-method type in the keyFunc.
ALWAYS pass jwt.WithValidMethods([...]) with ONLY your chosen algorithm(s).
NEVER let a verifier that holds a public key also accept HS256.
NEVER accept "alg":"none" in production.
NEVER feed the wrong key type to an algorithm (public key → HMAC = forgery).
```

### Brute force & rate limiting

The login endpoint is an *online* attack surface (offline is handled by Argon2). Add **rate limiting** (e.g. `golang.org/x/time/rate`, or a reverse-proxy limit) keyed by IP and by account; apply progressive backoff / temporary lockout after repeated failures; consider CAPTCHA after N failures; log and alert on credential-stuffing patterns. Combine with breached-password checks (reject known-compromised passwords).

### HTTPS is non-negotiable

Tokens and passwords over plain HTTP are trivially intercepted. Terminate TLS at your proxy/load balancer and redirect HTTP→HTTPS; set `Strict-Transport-Security` (HSTS).

```go
// App-level HTTP→HTTPS redirect (usually do this at the proxy instead):
http.ListenAndServe(":80", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	target := "https://" + r.Host + r.URL.RequestURI()
	http.Redirect(w, r, target, http.StatusMovedPermanently)
}))
```

### Pre-ship security checklist

```
[ ] Passwords hashed with Argon2id (never SHA/MD5; bcrypt only if legacy)
[ ] Argon2 params tuned to 200–500 ms on production hardware
[ ] PHC string format — params stored WITH each hash; NeedsRehash upgrade path
[ ] Salt from crypto/rand, 16 bytes; NEVER math/rand
[ ] Password compare uses subtle.ConstantTimeCompare
[ ] JWT secrets from env/secrets manager, >= 32 bytes, validated at startup
[ ] Distinct secrets for access vs refresh; distinct per environment
[ ] keyFunc asserts the signing-method TYPE; WithValidMethods whitelist applied
[ ] iss and aud validated; exp always set and required
[ ] Access TTL 5–15 min; refresh tokens rotated with reuse detection
[ ] Refresh token in httpOnly + Secure + SameSite cookie, narrow Path
[ ] Access token in memory only (never localStorage)
[ ] Login: identical error + timing for "no user" and "wrong password"
[ ] No secrets/tokens/passwords in logs; no sensitive data in JWT payload
[ ] Revocation strategy chosen (short TTL / jti blocklist / token_version)
[ ] Rate limiting + lockout on /auth/login; HTTPS + HSTS enforced everywhere
[ ] Object-level (ownership) authorization checks, not just role checks
```

### Common gotchas

**1 — `math/rand` vs `crypto/rand`.** `math/rand` is deterministic and predictable; using it for salts/secrets/jtis is a security bug. Always `crypto/rand`. (`math/rand/v2` is still NOT for security.)

**2 — Error comparison in jwt/v5.** Use `errors.Is(err, jwt.ErrTokenExpired)`, not `strings.Contains(err.Error(), "expired")`. v5 wraps typed sentinel errors; string matching breaks silently across versions.

**3 — Clock skew.** Distributed servers have slightly different clocks; a freshly issued token can look "not yet valid" or expire "early." Allow a small leeway:

```go
jwt.ParseWithClaims(raw, &claims, keyFunc, jwt.WithLeeway(30*time.Second))
```

**4 — bcrypt's 72-byte truncation.** bcrypt silently ignores bytes past 72, so a 100-char password and its 72-char prefix hash identically — a real vulnerability with long passphrases. Argon2id has no such limit; it is one more reason to prefer it.

**5 — User enumeration.** Returning early (no Argon2) when the user is missing makes "no such user" measurably faster. Always run the hash against a dummy and return one generic error (§14).

**6 — Trusting the token's `alg`.** The header is attacker-controlled. The verifier decides the algorithm — never read `alg` and act on it (§11).

**7 — Putting secrets in the payload.** It is base64, not encryption. Anyone with the token reads every claim. Keep PII/secrets out (§8).

---

## 18. Quick Reference Tables

### Argon2id parameters

| Param | Go arg | PHC | Minimum | Recommended |
|-------|--------|-----|---------|-------------|
| Memory | `memory` (KiB) | `m=` | 19456 | 65536 (64 MiB) |
| Iterations | `time` | `t=` | 2 | 3 |
| Parallelism | `threads` | `p=` | 1 | 4 |
| Salt length | — | — | 16 B | 16 B |
| Key length | `keyLen` | — | 32 B | 32 B |

### golang-jwt v5 essentials

| Need | API |
|------|-----|
| Standard claims base | embed `jwt.RegisteredClaims` |
| Build a token | `jwt.NewWithClaims(method, claims)` |
| HMAC method | `jwt.SigningMethodHS256` |
| ECDSA / RSA method | `jwt.SigningMethodES256` / `…RS256` |
| Sign | `tok.SignedString(key)` |
| Parse + validate | `jwt.ParseWithClaims(raw, &claims, keyFunc, opts...)` |
| Algorithm whitelist | `jwt.WithValidMethods([]string{"HS256"})` |
| Validate iss / aud | `jwt.WithIssuer(...)` / `jwt.WithAudience(...)` |
| Require exp | `jwt.WithExpirationRequired()` |
| Clock-skew leeway | `jwt.WithLeeway(d)` |
| Typed errors | `jwt.ErrTokenExpired`, `jwt.ErrSignatureInvalid`, `jwt.ErrTokenNotValidYet` |

### Status codes

| Code | Meaning | When |
|------|---------|------|
| 200 / 201 | OK / Created | success |
| 400 | Bad Request | malformed body / failed validation |
| 401 | Unauthorized | missing/invalid/expired token; bad credentials |
| 403 | Forbidden | authenticated but lacks permission (RBAC) |
| 409 | Conflict | duplicate registration (generic message) |
| 429 | Too Many Requests | rate limit hit on login |

### Mental models

```
Authentication = "who are you?"      → JWT proves identity
Authorization  = "what can you do?"  → role/permission/ownership checks

Hashing    = one-way, irreversible   → Argon2id for passwords
Encoding   = two-way (base64)        → JWT payload is NOT secret
Encryption = two-way, key required   → JWE if you truly need a secret payload

Session token = opaque, server tracks state    → easy revoke, stateful
JWT           = self-contained, signed         → scales, hard to revoke early

"A JWT is a signed assertion. Anyone holding it can present it; the signature
 proves nobody tampered with it and that the issuer minted it. That is ALL."
```

---

## 19. Study Path & Build-to-Learn Projects

Work in order. Each project forces one concept; you will understand it because you built (and broke) it.

### Phase 1 — Foundations (Week 1–2)

**Project 1 — Password Hasher CLI.** Read a password from **stdin** (not argv — argv leaks to `ps`/shell history). Hash it with the §7 package, print the PHC string, and add a `--verify` flag that checks a stored hash. Add a `go test -bench` benchmark and **tune the Argon2 params to ~300 ms on your machine**. *Goal: PHC format, constant-time compare, real parameter tuning.*

**Project 2 — JWT Inspector.** A CLI that base64url-decodes a JWT's header+payload *without* verifying (to prove the payload is readable), then verifies given a secret and prints each claim with its type. *Goal: demystify JWT structure; internalize "signed, not secret."*

### Phase 2 — Core Patterns (Week 3–4)

**Project 3 — Auth API.** The §14 server, but with a real database (SQLite via `modernc.org/sqlite`), input-validation middleware, **rate limiting** on `/auth/login` (`golang.org/x/time/rate`), and structured logging (`log/slog`). *Goal: a production-grade foundation.*

**Project 4 — Refresh Token Flow.** Add `/auth/refresh` (refresh token in an httpOnly cookie), **rotation with reuse detection**, and `/auth/logout` that revokes the refresh token. *Goal: stateless vs stateful token lifecycle.*

### Phase 3 — Hardening (Week 5–6)

**Project 5 — RBAC Blog API.** Roles `reader`/`author`/`admin`; `GET /posts` public, `POST /posts` needs author+, `DELETE /posts/:id` admin-only, and **authors edit only their own posts** (the §15 ownership check). *Goal: authentication vs authorization vs ownership.*

**Project 6 — Microservice + JWKS.** Split into an **auth service** that issues **ES256** tokens and serves its public key at `/.well-known/jwks.json`, and a **resource service** that verifies tokens using the fetched public key — never calling the auth service per request. *Goal: asymmetric signing, `kid`, JWKS.*

### Phase 4 — Production Concerns (Week 7–8)

**Project 7 — Key Rotation.** Implement zero-downtime rotation: HMAC two-secret overlap (§16) *or* ES256 `kid`-based JWKS rotation with an admin endpoint to roll keys. *Goal: rotate without logging everyone out.*

**Project 8 — Full-Stack Auth.** Wire a frontend (React or HTMX) to your API: access token in memory, refresh token in httpOnly cookie, a fetch/axios interceptor that auto-refreshes on `401`, and a working logout. As a contrast exercise, skim **`BETTERAUTH_GUIDE.md`** to see what a full-service auth framework gives you for free — and decide consciously when rolling your own (this guide) versus adopting a framework is the right call. *Goal: the complete browser-to-server picture.*

### Offline reference material

| Resource | Study |
|----------|-------|
| RFC 7519 (JWT) | claims table + Security Considerations |
| RFC 9106 (Argon2) | why memory-hardness works; parameter guidance |
| OWASP Authentication Cheat Sheet | re-read after each project |
| OWASP Password Storage Cheat Sheet | Argon2id params, upgrade strategy |
| OWASP JWT / Session Mgmt Cheat Sheets | storage, revocation, CSRF |
| `golang-jwt/jwt` v5 source | `parser.go`, `claims.go` — short and very readable |
| `golang.org/x/crypto/argon2` source | `argon2.go` — the `IDKey` signature |
| `crypto/subtle` source | `constant_time.go` — why constant-time works |
| `GO_NET_HTTP_REST_API_GUIDE.md` | the HTTP server foundations |
| `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md` | Gin middleware & routing |
| `BETTERAUTH_GUIDE.md` | a full-service auth framework, for contrast |

---

*Guide accurate as of June 2026. `github.com/golang-jwt/jwt/v5` (v5.x) and `golang.org/x/crypto` (current), Go 1.23/1.24. Authentication is security-critical: re-read the OWASP cheat sheets and the library changelogs, and run a security review, before shipping to production.*
