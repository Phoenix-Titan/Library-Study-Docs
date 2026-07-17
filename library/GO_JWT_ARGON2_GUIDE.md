# Go JWT + Argon2 — Authentication — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Go developers who want to build secure, production-grade authentication *from scratch* and understand every line — password hashing with **Argon2id** and stateless sessions with **JSON Web Tokens (JWT)**. This is a learn-offline study text: read it top to bottom the first time. Every concept is explained in prose first — *what it is, the logic/why, what it's for and when to use it, how to use it, the key parameters, best practices, and the security implications* — and only then shown as heavily-commented, runnable code. Authentication is security-critical, so the "why" matters as much as the "how"; a working auth system that you do not understand is a breach waiting to happen.
>
> **Version note:** This guide targets **`github.com/golang-jwt/jwt/v5`** (v5.x) and **`golang.org/x/crypto/argon2`**, on **Go 1.25 / 1.26** (current in 2026). golang-jwt v5 changed several APIs from v4 (`StandardClaims` → `RegisteredClaims`, parser validation options, typed errors) — every code sample here is v5. Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; shell commands are shown for both PowerShell and POSIX where they differ. Always confirm exact signatures at pkg.go.dev and re-check the OWASP cheat sheets before shipping.
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
18. [One-Time Passwords (OTP) via Email & WhatsApp](#18-one-time-passwords-otp-via-email--whatsapp) **[A]**
19. [Multi-Factor Authentication (TOTP, OTP & Recovery Codes)](#19-multi-factor-authentication-totp-otp--recovery-codes) **[A]**
20. [OAuth 2.0 / OIDC Social Login](#20-oauth-20--oidc-social-login) **[A]**
21. [Passkeys / WebAuthn (Passwordless)](#21-passkeys--webauthn-passwordless) **[A]**
22. [Encryption in Transit & at Rest](#22-encryption-in-transit--at-rest) **[A]**
23. [Rate Limiting & Abuse Prevention](#23-rate-limiting--abuse-prevention) **[A]**
24. [File & Image Upload Validation (Defeating Hidden Payloads)](#24-file--image-upload-validation-defeating-hidden-payloads) **[A]**
25. [Request Payload & Multipart-Form Validation](#25-request-payload--multipart-form-validation) **[A]**
26. [Quick Reference Tables](#26-quick-reference-tables)
27. [Study Path & Build-to-Learn Projects](#27-study-path--build-to-learn-projects)

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

> **Source:** OWASP Password Storage Cheat Sheet. These target ~0.25–1 second of compute on a modern server. **Always benchmark on your own production hardware** and tune so a single hash takes **200–500 ms** — long enough to make offline cracking painful, short enough that your login endpoint and registration flow stay responsive and you are not trivially DoS-able. Time it with `go test -bench` (see §27, Project 1).

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

### Where every piece goes — the auth-service project layout **[I]**

This guide is a *cookbook*: each section is a self-contained recipe (password hashing, JWTs, refresh rotation, RBAC, OTP, MFA, OAuth, passkeys, encryption, rate-limiting). In a real service they assemble into one package tree. Here is that layout, so you always know *which file* a given block belongs in — and **from here on, each self-contained code block is headed with a comment naming its file** (e.g. `// internal/password/password.go`), while §14 shows a runnable single-file version that inlines several of these for demonstration.

```text
auth-service/
├── cmd/
│   └── server/
│       └── main.go              # wiring (config → store → router → serve);
│                                #   §14 shows a self-contained single-file version
├── internal/
│   ├── password/
│   │   └── password.go          # §7   Argon2id hashing (Hash / Verify / NeedsRehash)
│   ├── token/
│   │   ├── jwt.go               # §10 sign · §11 parse+validate (alg-confusion guard)
│   │   └── refresh.go           # §12  access+refresh rotation & revocation
│   ├── auth/
│   │   ├── middleware.go        # §13  bearer-token middleware (net/http + Gin)
│   │   ├── handlers.go          # §14  register / login / protected handlers
│   │   └── rbac.go              # §15  roles, permissions, enforcement
│   ├── mfa/
│   │   ├── otp.go               # §18  email / WhatsApp one-time passwords
│   │   ├── totp.go              # §19  TOTP enrollment/verify + recovery codes
│   │   └── webauthn.go          # §21  passkeys / WebAuthn ceremonies
│   ├── oauth/
│   │   └── oidc.go              # §20  OAuth2 / OIDC social login (PKCE)
│   ├── crypto/
│   │   └── aesgcm.go            # §22.3  AES-256-GCM application-level encryption
│   └── ratelimit/
│       └── limiter.go           # §23  Redis token-bucket abuse prevention
├── migrations/                  # the SQL schema shown alongside each module
│   ├── 0001_users.sql
│   ├── 0002_refresh_tokens.sql  # §12
│   ├── 0003_otps.sql            # §18.2
│   ├── 0004_mfa.sql             # §19.4
│   ├── 0005_oauth_identities.sql # §20.5
│   └── 0006_webauthn.sql        # §21.3
├── .env                         # secrets: JWT_SECRET/keys, DB URL — never committed
└── go.mod
```

```go
// internal/password/password.go
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

## 18. One-Time Passwords (OTP) via Email & WhatsApp **[A]**

An **OTP (One-Time Password)** is a short-lived, single-use secret that a user must
present to prove they control a particular channel — usually an email inbox or a
phone number. You issue it for a specific *purpose*: confirming a login, verifying a
new email/phone, authorizing a sensitive ("step-up") action like a wire transfer, or
acting as a passwordless credential. The user receives the code out-of-band (email,
SMS, WhatsApp), types it back, and you check it.

**Threat model.** An OTP is a *possession* factor: it proves the requester can read a
specific destination right now. That is exactly its limit. On its own an OTP is **not
MFA** — it is a single factor. If your "login" is *just* an emailed code, then anyone
who pops the inbox (or your SMTP provider, or the mailbox via a SIM swap on a recovery
number) owns the account. OTP becomes meaningful MFA only when combined with a second,
*independent* factor (a password verified with Argon2 — see §3 and §7 — or a passkey).
The attacks you must design against:

- **Brute force / guessing** — a 6-digit code is only 1-in-1,000,000. Without attempt
  caps and rate limits, an attacker scripts a million tries. Mitigate with short TTL,
  per-code attempt caps, and per-destination + per-IP rate limiting (see §23).
- **Code theft at rest** — a leaked DB dump must not hand out valid codes. Store a
  **hash**, never plaintext.
- **Replay** — a code must work exactly once; mark it consumed atomically.
- **Enumeration** — responses must not reveal whether an account/destination exists.
- **Toll / pumping fraud** — attackers trigger floods of SMS/WhatsApp sends to numbers
  they own to farm telecom revenue (a.k.a. "SMS pumping"). Cost controls are essential.
- **Phishing relay** — an attacker tricks the victim into reading them the code. OTP
  cannot fully stop this; bind codes to a purpose and keep TTLs tight to shrink the
  window. For phishing resistance, prefer passkeys.

Banking-grade OTP therefore means: **hashed at rest, single-use, short TTL, attempt
caps, rate limits, audit logging (never the code itself), and anti-enumeration** — all
of which we build below.

### 18.1 Generating the code securely **[A]**

The cardinal rule: generate OTPs with **`crypto/rand`**, never `math/rand`.
`math/rand` is a deterministic PRNG seeded from predictable state; an attacker who
learns or guesses the seed can predict every future code. `crypto/rand` reads from the
operating system CSPRNG and is unpredictable.

Use 6–8 numeric digits. Six digits (10^6 ≈ 20 bits) is the industry norm for emailed
codes; for high-value step-up actions consider 7–8 digits to widen the keyspace. The
subtle trap is **modulo bias**: doing `n % 1000000` over raw random bytes makes lower
values slightly more likely because 2^k is not a clean multiple of 10^6. We avoid bias
with rejection sampling against the largest multiple of the range that fits the draw.

```go
package otp

import (
	"crypto/rand"
	"encoding/binary"
	"fmt"
)

// digitsToRange maps a digit count to its exclusive upper bound, e.g. 6 -> 1_000_000.
func digitsToRange(digits int) uint64 {
	n := uint64(1)
	for i := 0; i < digits; i++ {
		n *= 10
	}
	return n
}

// GenerateNumeric returns a cryptographically random, zero-padded numeric OTP with
// uniform distribution (no modulo bias). digits should be 6..8.
func GenerateNumeric(digits int) (string, error) {
	if digits < 6 || digits > 8 {
		return "", fmt.Errorf("otp: digits must be 6..8, got %d", digits)
	}
	upper := digitsToRange(digits) // exclusive bound, e.g. 1_000_000

	// Largest multiple of `upper` that fits in uint64; anything >= limit is rejected
	// so that every value in [0, upper) is equally likely.
	limit := (^uint64(0) / upper) * upper

	var buf [8]byte
	for {
		if _, err := rand.Read(buf[:]); err != nil {
			return "", fmt.Errorf("otp: read entropy: %w", err)
		}
		v := binary.BigEndian.Uint64(buf[:])
		if v >= limit {
			continue // reject to keep the distribution uniform
		}
		code := v % upper
		return fmt.Sprintf("%0*d", digits, code), nil
	}
}
```

The returned string is what you send to the user — and the only place plaintext ever
exists. From here on it is hashed.

### 18.2 Storing OTPs safely: schema + hashing **[A]**

Treat an OTP like a password: **never store it in plaintext.** A database leak (see the
backup/replica concerns in `POSTGRESQL_GUIDE.md`) must not yield live codes. Because
OTPs are short, single-use, and already high-entropy, a fast cryptographic hash
(**SHA-256**) is acceptable and keeps verification cheap under load. If you want
defense-in-depth identical to your password path, hash with **Argon2id** exactly as in
§3/§7 — at OTP volumes the extra cost is usually fine, and it removes any "fast hash"
debate in audits. Below we show SHA-256 with a server-side **pepper** (a secret key
from config) so a stolen DB alone cannot even offline-test guesses.

Each row binds the code to a single **purpose** and **destination**, carries a short
`expires_at`, an `attempts` counter, and a `consumed_at` marker for single-use.

```sql
-- POSTGRESQL_GUIDE.md covers extensions, roles, and least-privilege grants.
CREATE TABLE otp_codes (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id      BIGINT          REFERENCES users (id) ON DELETE CASCADE, -- nullable: passwordless/verify-email may precede an account
    destination  TEXT        NOT NULL,                       -- normalized email or E.164 phone
    purpose      TEXT        NOT NULL CHECK (purpose IN ('login','verify_email','verify_phone','step_up')),
    code_hash    BYTEA       NOT NULL,                        -- HMAC-SHA256(code, pepper) or Argon2id; NEVER plaintext
    attempts     INT         NOT NULL DEFAULT 0,
    max_attempts INT         NOT NULL DEFAULT 5,
    expires_at   TIMESTAMPTZ NOT NULL,                        -- short, ~5 minutes
    consumed_at  TIMESTAMPTZ,                                 -- non-NULL once used (single-use)
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_ip   INET
);

-- Fast lookup of the latest live code for a (destination, purpose).
CREATE INDEX idx_otp_lookup ON otp_codes (destination, purpose, expires_at DESC)
    WHERE consumed_at IS NULL;

-- Background sweeper / TTL: delete expired rows so the table stays small.
CREATE INDEX idx_otp_expiry ON otp_codes (expires_at);
```

> A short, indexed table also lives comfortably in Redis if you prefer TTL-native
> storage — see `REDIS_GUIDE.md` for `SET key val EX 300` semantics. Postgres is shown
> here because banking systems usually want the durable audit trail in the RDBMS.

Hashing helper and insert. Note `invalidate old OTPs on resend`: before issuing a new
code for the same `(destination, purpose)`, consume any outstanding ones so only one
code is ever live.

```go
package otp

import (
	"context"
	"crypto/hmac"
	"crypto/sha256"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

type Store struct {
	db     *pgxpool.Pool
	pepper []byte // loaded from config/secret manager, e.g. os.Getenv("OTP_PEPPER")
}

// hash binds the code to a server-side secret so a DB leak alone can't offline-test it.
func (s *Store) hash(code string) []byte {
	mac := hmac.New(sha256.New, s.pepper)
	mac.Write([]byte(code))
	return mac.Sum(nil)
}

const otpTTL = 5 * time.Minute

// Issue invalidates any live codes for (destination, purpose), then inserts a fresh one.
func (s *Store) Issue(ctx context.Context, userID *int64, destination, purpose, code, ip string) error {
	tx, err := s.db.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx) //nolint:errcheck

	// One live code per destination+purpose: burn the rest on resend.
	if _, err := tx.Exec(ctx,
		`UPDATE otp_codes SET consumed_at = now()
		   WHERE destination = $1 AND purpose = $2 AND consumed_at IS NULL`,
		destination, purpose); err != nil {
		return err
	}

	if _, err := tx.Exec(ctx,
		`INSERT INTO otp_codes
		   (user_id, destination, purpose, code_hash, expires_at, created_ip)
		 VALUES ($1, $2, $3, $4, $5, $6)`,
		userID, destination, purpose, s.hash(code), time.Now().Add(otpTTL), ip,
	); err != nil {
		return err
	}
	return tx.Commit(ctx)
}
```

### 18.3 Verifying: constant-time, atomic, race-free **[A]**

Verification must satisfy four properties at once: (a) the comparison is **constant
time** so timing can't leak how many leading bytes matched; (b) the code is not expired
and not already consumed; (c) `attempts < max_attempts`; and (d) success **marks the
row consumed atomically** so two concurrent requests can never both win (replay/race).

The clean way to get atomicity is a single `UPDATE ... WHERE ... RETURNING` guarded by
all the conditions, so the database — not application logic — enforces single-use under
concurrency. We fetch the candidate row, do the constant-time compare in Go (we can't
hash inside SQL without the pepper), then conditionally consume in one statement.

```go
package otp

import (
	"context"
	"crypto/subtle"
	"errors"

	"github.com/jackc/pgx/v5"
)

var (
	ErrInvalid = errors.New("otp: invalid or expired code") // generic on purpose
	ErrLocked  = errors.New("otp: too many attempts")
)

func (s *Store) Verify(ctx context.Context, destination, purpose, code string) (userID *int64, err error) {
	want := s.hash(code)

	tx, err := s.db.Begin(ctx)
	if err != nil {
		return nil, err
	}
	defer tx.Rollback(ctx) //nolint:errcheck

	// Lock the newest live row so the attempts bump and consume are serialized.
	var (
		id          int64
		stored      []byte
		attempts    int
		maxAttempts int
		uid         *int64
	)
	err = tx.QueryRow(ctx,
		`SELECT id, code_hash, attempts, max_attempts, user_id
		   FROM otp_codes
		  WHERE destination = $1 AND purpose = $2
		    AND consumed_at IS NULL AND expires_at > now()
		  ORDER BY created_at DESC
		  LIMIT 1
		  FOR UPDATE`,
		destination, purpose).Scan(&id, &stored, &attempts, &maxAttempts, &uid)
	if errors.Is(err, pgx.ErrNoRows) {
		return nil, ErrInvalid // no live code -> generic failure (anti-enumeration)
	}
	if err != nil {
		return nil, err
	}

	if attempts >= maxAttempts {
		// Burn it so it can never be guessed again.
		_, _ = tx.Exec(ctx, `UPDATE otp_codes SET consumed_at = now() WHERE id = $1`, id)
		_ = tx.Commit(ctx)
		return nil, ErrLocked
	}

	// Constant-time comparison: 1 if equal, 0 otherwise; no early exit, no timing leak.
	if subtle.ConstantTimeCompare(stored, want) != 1 {
		_, _ = tx.Exec(ctx, `UPDATE otp_codes SET attempts = attempts + 1 WHERE id = $1`, id)
		_ = tx.Commit(ctx)
		return nil, ErrInvalid
	}

	// Atomic single-use: consume only if STILL unconsumed; RETURNING proves we won.
	var consumedID int64
	err = tx.QueryRow(ctx,
		`UPDATE otp_codes SET consumed_at = now()
		   WHERE id = $1 AND consumed_at IS NULL
		 RETURNING id`, id).Scan(&consumedID)
	if errors.Is(err, pgx.ErrNoRows) {
		return nil, ErrInvalid // lost the race -> already consumed
	}
	if err != nil {
		return nil, err
	}
	if err := tx.Commit(ctx); err != nil {
		return nil, err
	}
	return uid, nil
}
```

On success, the caller proceeds to mint tokens — typically the same access/refresh pair
and rotation scheme described in §12. Treat a verified `step_up` OTP as authorization
for a single sensitive action, not a new long-lived session.

### 18.4 Sending via email **[A]**

Keep transport behind a provider-agnostic interface so you can swap SMTP for a
transactional API (SES, Postmark, SendGrid) without touching the OTP core. The two
non-negotiables: the email path must **never log the code**, and the message body
should state the purpose and TTL ("expires in 5 minutes") so users recognize phishing
that asks for codes out of context.

```go
package otp

import (
	"context"
	"fmt"
	"net/smtp"
)

// Sender abstracts the delivery channel (email today, more tomorrow).
type Sender interface {
	Send(ctx context.Context, to, code, purpose string) error
}

// SMTPSender is a minimal net/smtp implementation. In production prefer STARTTLS/TLS
// and a pooled provider client; this shows the shape.
type SMTPSender struct {
	Host string // e.g. "smtp.example.com"
	Port string // e.g. "587"
	Auth smtp.Auth
	From string
}

func (m *SMTPSender) Send(ctx context.Context, to, code, purpose string) error {
	subject := "Your verification code"
	body := fmt.Sprintf(
		"Your %s code is %s. It expires in 5 minutes. If you didn't request it, ignore this email.",
		purpose, code)
	msg := []byte("To: " + to + "\r\n" +
		"From: " + m.From + "\r\n" +
		"Subject: " + subject + "\r\n\r\n" + body + "\r\n")

	// NOTE: do NOT log `code` or `body`. Log only {to: masked, purpose, result}.
	addr := m.Host + ":" + m.Port
	return smtp.SendMail(addr, m.Auth, m.From, []string{to}, msg)
}
```

A provider HTTP API impl looks the same from the outside — implement `Sender`, post
JSON to the vendor, return the error. The OTP issuance code only sees `Sender`.

### 18.5 Sending via WhatsApp (Meta Cloud API) **[A]**

WhatsApp delivery uses the **Meta WhatsApp Business Cloud API**. You cannot send
arbitrary free-form text to start a conversation — you must use a **pre-approved
template message**. Meta provides a dedicated **authentication** template category
specifically for OTPs; you submit a template (e.g. named `otp_login`) for review, and
once approved you send it with the code passed as a runtime **parameter**. Sends go to
`POST https://graph.facebook.com/v21.0/{phone_number_id}/messages` with a bearer access
token. (For uploading any media or general Graph request patterns, the HTTP client
style in `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md` carries over.)

The request body for an authentication template with one body parameter (the code) and
the matching URL-button copy-code parameter:

```http
POST /v21.0/{phone_number_id}/messages HTTP/1.1
Host: graph.facebook.com
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json
```

```json
{
  "messaging_product": "whatsapp",
  "to": "15551234567",
  "type": "template",
  "template": {
    "name": "otp_login",
    "language": { "code": "en_US" },
    "components": [
      {
        "type": "body",
        "parameters": [{ "type": "text", "text": "482913" }]
      },
      {
        "type": "button",
        "sub_type": "url",
        "index": "0",
        "parameters": [{ "type": "text", "text": "482913" }]
      }
    ]
  }
}
```

The Go client. Config supplies the phone-number ID and access token (from a secret
manager, never hard-coded); the destination must already be validated E.164.

```go
package otp

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"time"
)

type WhatsAppSender struct {
	PhoneNumberID string
	AccessToken   string // bearer token from config/secret manager
	TemplateName  string // e.g. "otp_login"
	LangCode      string // e.g. "en_US"
	HTTP          *http.Client
}

func (w *WhatsAppSender) Send(ctx context.Context, to, code, _ string) error {
	body := map[string]any{
		"messaging_product": "whatsapp",
		"to":                to, // E.164 without leading '+', e.g. "15551234567"
		"type":              "template",
		"template": map[string]any{
			"name":     w.TemplateName,
			"language": map[string]string{"code": w.LangCode},
			"components": []any{
				map[string]any{
					"type":       "body",
					"parameters": []any{map[string]string{"type": "text", "text": code}},
				},
				map[string]any{
					"type":     "button",
					"sub_type": "url",
					"index":    "0",
					"parameters": []any{map[string]string{"type": "text", "text": code}},
				},
			},
		},
	}
	raw, err := json.Marshal(body)
	if err != nil {
		return err
	}

	url := fmt.Sprintf("https://graph.facebook.com/v21.0/%s/messages", w.PhoneNumberID)
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(raw))
	if err != nil {
		return err
	}
	req.Header.Set("Authorization", "Bearer "+w.AccessToken)
	req.Header.Set("Content-Type", "application/json")

	client := w.HTTP
	if client == nil {
		client = &http.Client{Timeout: 10 * time.Second}
	}
	resp, err := client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 300 {
		b, _ := io.ReadAll(io.LimitReader(resp.Body, 4<<10))
		// Log the Graph error, but never the `code`.
		return fmt.Errorf("whatsapp send failed: %s: %s", resp.Status, b)
	}
	return nil
}
```

### 18.6 Verification flow & abuse prevention **[A]**

A correct crypto core is undone by a sloppy flow. Wire these controls around it:

- **Attempt caps then invalidate.** After `max_attempts` wrong tries, burn the code
  (done in §18.3). The user must request a fresh one.
- **Rate limiting + resend cooldown.** Apply per-destination *and* per-IP limits on
  both *issuance* and *verification*, plus a resend cooldown (e.g. 30–60 s) so a single
  destination can't be spammed and an attacker can't farm fresh codes. Use the token
  bucket / sliding window from §23, backed by Redis (`REDIS_GUIDE.md`).
- **Cost controls / anti-pumping.** WhatsApp and SMS cost money per send; cap daily
  sends per destination and per account, watch for bursts to unusual prefixes, and
  alert/lock on anomalies (SMS-pumping fraud).
- **Anti-enumeration.** Always return a generic response ("If that destination exists,
  we sent a code") and the same generic `ErrInvalid` for wrong/expired/unknown — never
  reveal whether the account or destination exists, and keep response timing uniform.
- **Single-use & short TTL.** Enforced in the schema and the atomic consume.
- **Constant-time compare.** Use `crypto/subtle` (§18.3), never `==` on the hash.
- **Audit logging — never the code.** Log issuance and verification events with
  `{event, user_id, destination(masked), purpose, ip, result, ts}`. The plaintext code
  must never reach logs, traces, or error messages.

```go
// maskDest hides most of a destination for logs: a***@x.com, +1******4567.
func maskDest(d string) string { /* ... */ return d }

func (s *Store) auditIssue(ctx context.Context, userID *int64, dest, purpose, ip string) {
	// INSERT into an append-only audit table; NO code field exists by design.
	_, _ = s.db.Exec(ctx,
		`INSERT INTO otp_audit (event, user_id, destination, purpose, ip)
		 VALUES ('issue', $1, $2, $3, $4)`,
		userID, maskDest(dest), purpose, ip)
}
```

### 18.7 Best practices **[A]**

- **Short TTL** (~5 min) — shrinks the brute-force and phishing-relay window.
- **Hashed at rest** — SHA-256 + pepper minimum; Argon2id (§3/§7) for parity with
  passwords. Never plaintext.
- **Single-use** — atomic consume via `UPDATE ... RETURNING` (§18.3).
- **Bind to purpose + destination** — a `verify_email` code can't authorize a wire.
- **Invalidate old OTPs on resend** — only one live code per (destination, purpose).
- **Validate phone numbers as E.164** before sending (e.g. with `nyaruka/phonenumbers`)
  to avoid wasted/fraudulent sends.
- **Cost controls** — per-destination/per-account send caps + anomaly alerts to defeat
  SMS/WhatsApp pumping.
- **OTP is one factor** — combine with a password (§3/§7) or passkey for real MFA; for
  phishing resistance, prefer passkeys outright.

> **Banking-grade OTP checklist:**
> - [ ] Codes generated with `crypto/rand`, uniform (no modulo bias), 6–8 digits.
> - [ ] Stored **hashed** (SHA-256+pepper or Argon2id), never plaintext.
> - [ ] Short TTL (~5 min); old codes invalidated on resend (one live code per purpose).
> - [ ] Bound to a single `purpose` + `destination`; single-use via atomic `UPDATE … RETURNING`.
> - [ ] Constant-time compare (`crypto/subtle`); per-code attempt cap then burn.
> - [ ] Per-destination + per-IP rate limits and resend cooldown (§23, Redis).
> - [ ] Cost/anti-pumping caps; alert + lock on anomalies.
> - [ ] Anti-enumeration: generic responses, uniform timing.
> - [ ] Audit issuance & verification — **never** the code, in logs or traces.
> - [ ] OTP treated as one factor; paired with password/passkey for MFA; success mints/rotates tokens per §12.

---

## 19. Multi-Factor Authentication (TOTP, OTP & Recovery Codes) **[A]**

Multi-factor authentication (MFA) is the single highest-leverage control you can add to a login flow: even a fully compromised password becomes useless to an attacker who lacks the second factor. For banking-grade systems it is not optional. This section builds a complete MFA subsystem in Go 1.25/1.26: TOTP enrollment and verification, encrypted secrets at rest, hashed single-use recovery codes, a safe login state machine, step-up authentication for sensitive operations, and a secure disable flow with audit logging and rate limiting.

### 19.1 Concepts: factors, AAL, and why factor choice matters

Authentication factors fall into three categories:

- **Something you know** — a password or PIN (covered in §7).
- **Something you have** — a phone running an authenticator app (TOTP), a hardware security key, or a device receiving an OTP (§18).
- **Something you are** — a biometric (fingerprint, face), typically mediated by the device, not your server.

**2FA** is exactly two factors from two *different* categories; **MFA** is two or more. Requiring a password plus an SMS code your own server generated is still "know + have." Requiring a password plus a *second password* is not MFA at all — both are "know."

**Step-up MFA** means you do not demand the second factor on every request; you demand it again, on top of an existing session, immediately before a high-risk action (adding a payee, changing the registered email, raising a transfer limit). This balances friction against risk.

NIST SP 800-63B defines **Authenticator Assurance Levels**:

| AAL  | Requirement (high level)                                  | Typical factor                          |
|------|----------------------------------------------------------|-----------------------------------------|
| AAL1 | Single factor                                            | Password only                           |
| AAL2 | Two distinct factors; replay-resistant                   | Password + TOTP / push / OTP            |
| AAL3 | Hardware-based, phishing-resistant, verifier-impersonation resistant | Password + FIDO2/WebAuthn security key |

A banking app should target **AAL2 minimum** for login and effectively AAL3 (phishing-resistant passkeys/security keys) for the most sensitive operations. **Prefer TOTP and passkeys over SMS:** SMS is vulnerable to SIM-swap, SS7 interception, and carrier social engineering, and NIST has deprecated it as a primary out-of-band channel. Use SMS/WhatsApp/email OTP (§18) only as a fallback factor, never as the strongest one you offer.

### 19.2 TOTP enrollment with `pquerna/otp`

TOTP (RFC 6238) derives a 6-digit code from a shared secret and the current 30-second time-step using HMAC. The user's authenticator app (Google Authenticator, Authy, 1Password) and your server both hold the same secret, so both compute the same code without any network round-trip. We use `github.com/pquerna/otp`.

```bash
go get github.com/pquerna/otp@latest
go get github.com/skip2/go-qrcode@latest
```

Enrollment must be **confirmed**: generate a secret, show the user the `otpauth://` URI as a QR code, and require them to type back a valid code *before* you mark MFA as enabled. This proves the secret was successfully imported and prevents a user from locking themselves out.

```go
package mfa

import (
	"bytes"
	"fmt"

	"github.com/pquerna/otp"
	"github.com/pquerna/otp/totp"
	"github.com/skip2/go-qrcode"
)

// Enrollment holds the data returned to the client to begin TOTP setup.
type Enrollment struct {
	Secret  string // base32 secret — store ENCRYPTED, never log
	URI     string // otpauth:// URI
	QRPNG   []byte // PNG image of the URI for display
}

// GenerateEnrollment creates a new (unconfirmed) TOTP secret for a user.
func GenerateEnrollment(issuer, accountEmail string) (*Enrollment, error) {
	key, err := totp.Generate(totp.GenerateOpts{
		Issuer:      issuer,        // shows as the app name, e.g. "Acme Bank"
		AccountName: accountEmail,  // shows as the account label
		Period:      30,            // 30s time-step (RFC 6238 default)
		Digits:      otp.DigitsSix, // 6 digits
		Algorithm:   otp.AlgorithmSHA1, // SHA1 = max authenticator-app compatibility
	})
	if err != nil {
		return nil, fmt.Errorf("generate totp: %w", err)
	}

	var buf bytes.Buffer
	png, err := qrcode.Encode(key.URL(), qrcode.Medium, 256)
	if err != nil {
		return nil, fmt.Errorf("encode qr: %w", err)
	}
	buf.Write(png)

	return &Enrollment{
		Secret: key.Secret(),
		URI:    key.URL(),
		QRPNG:  buf.Bytes(),
	}, nil
}

// ConfirmEnrollment verifies the first user-supplied code against the new
// secret. Only on success do you persist (encrypted) and set confirmed_at.
func ConfirmEnrollment(secret, code string) bool {
	return totp.Validate(code, secret)
}
```

The client renders `QRPNG` (typically as a data URI: `data:image/png;base64,...`) and also shows `Secret` as text for manual entry. Treat the secret like a credential: never write it to logs, never return it again after enrollment.

### 19.3 Encrypting the TOTP secret at rest

The TOTP secret is a long-lived shared secret; if your database is dumped, plaintext secrets let an attacker mint valid codes forever. **Encrypt it with AES-256-GCM** using a data key derived from your KMS (see §22 for the full envelope-encryption / data-key story). Below is the symmetric layer; in production the 32-byte key comes from KMS, not a literal.

```go
package mfa

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"fmt"
	"io"
)

// Encrypt seals plaintext with AES-256-GCM. `key` is a 32-byte data key
// obtained via your KMS (see §22). Output = nonce || ciphertext || tag.
func Encrypt(key, plaintext []byte) ([]byte, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, fmt.Errorf("new cipher: %w", err)
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, fmt.Errorf("new gcm: %w", err)
	}
	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return nil, fmt.Errorf("nonce: %w", err)
	}
	// Seal appends ciphertext+tag to nonce, so the nonce is the prefix.
	return gcm.Seal(nonce, nonce, plaintext, nil), nil
}

// Decrypt reverses Encrypt.
func Decrypt(key, blob []byte) ([]byte, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, fmt.Errorf("new cipher: %w", err)
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, fmt.Errorf("new gcm: %w", err)
	}
	ns := gcm.NonceSize()
	if len(blob) < ns {
		return nil, fmt.Errorf("ciphertext too short")
	}
	nonce, ct := blob[:ns], blob[ns:]
	pt, err := gcm.Open(nil, nonce, ct, nil)
	if err != nil {
		return nil, fmt.Errorf("decrypt/auth failed: %w", err) // tampering or wrong key
	}
	return pt, nil
}
```

Store the result of `Encrypt([]byte(secret))` in the `secret_enc` column. At verification time, `Decrypt` it into memory, validate, and let it go out of scope — never cache decrypted secrets longer than the request.

### 19.4 Database schema

Two tables: one for confirmed MFA methods per user, one for recovery codes. Note the indexes and constraints — they enforce one confirmed method per `(user_id, type)` and make lookups cheap.

```sql
-- Confirmed/enrolled MFA methods. See POSTGRESQL_GUIDE.md for migration tooling.
CREATE TABLE user_mfa (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type          TEXT        NOT NULL CHECK (type IN ('totp','email_otp','wa_otp')),
    secret_enc    BYTEA,                       -- AES-256-GCM blob (TOTP only); NULL for OTP channels
    last_step     BIGINT,                      -- last accepted TOTP time-step (replay guard)
    confirmed_at  TIMESTAMPTZ,                 -- NULL until first code verified
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, type)
);
CREATE INDEX idx_user_mfa_user ON user_mfa (user_id) WHERE confirmed_at IS NOT NULL;

-- Single-use recovery/backup codes. Store HASHES ONLY.
CREATE TABLE mfa_recovery_codes (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash  BYTEA       NOT NULL,           -- SHA-256 or Argon2id of the code
    used_at    TIMESTAMPTZ,                    -- NULL = unused
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recovery_user_unused ON mfa_recovery_codes (user_id) WHERE used_at IS NULL;
```

### 19.5 Verifying TOTP at login (with replay prevention)

After a successful password check (§7), require a valid TOTP. Allow ±1 period of clock skew so a user typing a code near a boundary isn't rejected, but **prevent replay**: once a time-step is accepted, store it (`last_step`) and reject any code from that step or earlier. Without this, an attacker who shoulder-surfs one code can replay it within the 30–90s validity window.

```go
package mfa

import (
	"time"

	"github.com/pquerna/otp"
	"github.com/pquerna/otp/totp"
)

// VerifyTOTP validates a code with ±1 period skew and rejects replays.
// lastStep is the previously accepted time-step (0 if none).
// It returns (ok, newStep): persist newStep when ok is true.
func VerifyTOTP(secret, code string, lastStep int64) (bool, int64) {
	now := time.Now()
	valid, err := totp.ValidateCustom(code, secret, now, totp.ValidateOpts{
		Period:    30,
		Skew:      1, // accept previous, current, next step
		Digits:    otp.DigitsSix,
		Algorithm: otp.AlgorithmSHA1,
	})
	if err != nil || !valid {
		return false, lastStep
	}
	// Determine the time-step that matched so we can block reuse.
	step := now.Unix() / 30
	if step <= lastStep {
		return false, lastStep // already used this (or an older) step — replay
	}
	return true, step
}
```

In the handler, wrap this in a transaction: decrypt the secret, call `VerifyTOTP`, and on success `UPDATE user_mfa SET last_step = $1`. Rate-limit attempts (§23) and audit every outcome (§19.9).

### 19.6 Recovery (backup) codes

If a user loses their phone, recovery codes are their way back in. Generate N high-entropy single-use codes (10 is typical), **show them exactly once**, and store only **hashes**. Verify with a constant-time compare, mark the matched code used, and let the user regenerate — which invalidates all old codes.

```go
package mfa

import (
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
	"encoding/base32"
	"fmt"
	"strings"
)

const recoveryCodeCount = 10

// GenerateRecoveryCodes returns plaintext codes (show once) and their hashes.
func GenerateRecoveryCodes() (plain []string, hashes [][]byte, err error) {
	for i := 0; i < recoveryCodeCount; i++ {
		raw := make([]byte, 10) // 80 bits of entropy
		if _, err = rand.Read(raw); err != nil {
			return nil, nil, fmt.Errorf("rand: %w", err)
		}
		// e.g. "abcde-fghij" — base32, lowercase, grouped for readability.
		enc := strings.ToLower(base32.StdEncoding.WithPadding(base32.NoPadding).EncodeToString(raw))
		code := enc[:5] + "-" + enc[5:10]
		plain = append(plain, code)
		h := sha256.Sum256([]byte(code))
		hashes = append(hashes, h[:])
	}
	return plain, hashes, nil
}

// VerifyRecoveryCode constant-time compares `input` against stored hashes.
// Return the matching row id so the caller can mark it used; -1 if none.
func VerifyRecoveryCode(input string, rows []RecoveryRow) int {
	in := sha256.Sum256([]byte(strings.TrimSpace(strings.ToLower(input))))
	match := -1
	// Walk ALL rows (no early return) to keep timing uniform.
	for _, r := range rows {
		if r.UsedAt != nil {
			continue
		}
		if subtle.ConstantTimeCompare(in[:], r.CodeHash) == 1 {
			match = r.ID
		}
	}
	return match
}

type RecoveryRow struct {
	ID       int
	CodeHash []byte
	UsedAt   *string
}
```

> SHA-256 is acceptable here because recovery codes carry 80 bits of crypto/rand entropy — they are not low-entropy human passwords, so the slow KDF that §7 requires for passwords is unnecessary. If you prefer defense-in-depth, hash them with Argon2id instead. On regenerate, run `UPDATE mfa_recovery_codes SET used_at = now() WHERE user_id = $1 AND used_at IS NULL` before inserting the new batch.

### 19.7 The login state machine

Never grant a full session after only a password when MFA is enabled. Instead: password success → mint a short-lived, narrowly-scoped **"MFA-pending" token** → user submits TOTP / OTP (§18) / recovery code → on success mint the real access + refresh tokens (§12, login flow in §14). The pending token must *not* authorize anything except completing MFA.

```go
// Pending token: a separate JWT "audience"/scope so it can't be used as a
// session. Short TTL (e.g. 5 min). See §12 for signing helpers.
type MFAClaims struct {
	UserID int    `json:"uid"`
	Scope  string `json:"scope"` // always "mfa_pending"
	jwt.RegisteredClaims
}

// POST /login — step 1: password
func LoginHandler(w http.ResponseWriter, r *http.Request) {
	creds := parse(r)
	user, ok := authenticatePassword(creds) // §7 verify
	if !ok {
		rateLimitAndAudit(r, "login_fail") // §23, §19.9
		http.Error(w, "invalid credentials", http.StatusUnauthorized)
		return
	}
	if !user.MFAEnabled {
		issueSession(w, user) // §12: access + refresh
		return
	}
	// MFA required → issue a pending token ONLY.
	pending := signMFAPending(user.ID, 5*time.Minute) // scope="mfa_pending"
	writeJSON(w, http.StatusOK, map[string]any{
		"mfa_required": true,
		"mfa_token":    pending,
		"methods":      user.MFAMethods(), // e.g. ["totp","recovery"]
	})
}

// POST /login/mfa — step 2: second factor
func MFAVerifyHandler(w http.ResponseWriter, r *http.Request) {
	claims, ok := verifyMFAPending(r) // rejects normal access tokens
	if !ok || claims.Scope != "mfa_pending" {
		http.Error(w, "invalid mfa session", http.StatusUnauthorized)
		return
	}
	if rateLimited(r, claims.UserID) { // §23
		http.Error(w, "too many attempts", http.StatusTooManyRequests)
		return
	}
	body := parse(r) // { method: "totp"|"recovery"|"otp", code: "..." }
	if !verifySecondFactor(claims.UserID, body) {
		audit(claims.UserID, "mfa_fail", body.Method)
		bumpFailures(claims.UserID) // lock after N (§19.9)
		http.Error(w, "invalid code", http.StatusUnauthorized)
		return
	}
	audit(claims.UserID, "mfa_success", body.Method)
	issueSession(w, lookupUser(claims.UserID)) // now mint real tokens
}
```

The pending token is the *only* safe way to carry state between the two steps. Do not store "password verified" in a cookie/session, and do not accept the pending token at any normal API route — gate it strictly on `scope == "mfa_pending"`.

### 19.8 Alternative factors and step-up authentication

**Fallback factor.** Reuse the email/WhatsApp OTP delivery from §18 as an alternative second factor. In `verifySecondFactor`, branch on `body.Method`: `"totp"` → §19.5, `"recovery"` → §19.6, `"otp"` → send/verify via the §18 channel (codes in Redis with a TTL — see `REDIS_GUIDE.md`). Keep OTP as a *fallback*, not the default, given its weaker assurance.

**Step-up auth.** A valid session is not enough for high-risk operations. Before adding a payee, changing the registered email, or raising a transfer limit, re-prompt for the second factor and issue a short re-auth token bound to that specific action.

```go
// Require fresh MFA, then mint a 2-minute token scoped to one action.
func StepUpHandler(w http.ResponseWriter, r *http.Request) {
	sess := mustSession(r) // existing access token (§12)
	body := parse(r)       // { method, code, action: "add_payee" }
	if !verifySecondFactor(sess.UserID, body) {
		audit(sess.UserID, "stepup_fail", body.Action)
		http.Error(w, "re-auth failed", http.StatusUnauthorized)
		return
	}
	tok := signStepUp(sess.UserID, body.Action, 2*time.Minute) // scope="stepup:add_payee"
	writeJSON(w, http.StatusOK, map[string]string{"stepup_token": tok})
}

// The sensitive endpoint requires BOTH a session AND a matching step-up token.
func AddPayeeHandler(w http.ResponseWriter, r *http.Request) {
	sess := mustSession(r)
	if !validStepUp(r, sess.UserID, "add_payee") { // scope + freshness check
		http.Error(w, "step-up required", http.StatusForbidden)
		return
	}
	// ... perform the sensitive action ...
}
```

Bind the step-up token to the exact action (and ideally the resource) so it cannot be replayed against a different operation.

### 19.9 Managing MFA securely: disable, audit, rate-limit, trusted devices

**Secure disable.** Disabling MFA is itself a high-risk action: require the *current* second factor (or password + a fresh email confirmation link) before removing it. Never let a plain session toggle MFA off — that would let a stolen access token strip protection.

```go
func DisableMFAHandler(w http.ResponseWriter, r *http.Request) {
	sess := mustSession(r)
	body := parse(r)
	// Require current MFA (step-up) before disabling.
	if !verifySecondFactor(sess.UserID, body) {
		audit(sess.UserID, "mfa_disable_denied", "")
		http.Error(w, "verify current factor first", http.StatusForbidden)
		return
	}
	deleteMFA(sess.UserID) // remove user_mfa rows + invalidate recovery codes
	audit(sess.UserID, "mfa_disabled", "")
	notifyUserByEmail(sess.UserID, "MFA was disabled on your account") // §18
}
```

**Audit logging.** Log every MFA lifecycle event — enable, disable, each verify success/failure, recovery-code use, step-up, trusted-device grant/revoke — with user id, timestamp, IP, and user agent. These records are essential for fraud investigation and regulatory compliance; store them append-only.

**Rate limiting and lockout.** TOTP has only ~1M possible codes; brute force is feasible without limits. Cap MFA verification attempts per user and per IP (§23, sliding window in Redis — `REDIS_GUIDE.md`) and lock the account (or force a cool-down + email alert) after a threshold of consecutive failures. Apply the same to recovery-code attempts.

**Trusted device ("remember this device").** To reduce friction you may skip MFA on a previously verified device for a bounded period. Issue a signed, expiring, **revocable** device token (e.g. 30 days) stored server-side so it can be invalidated, and bind it to a device fingerprint. Tradeoffs to weigh: it lowers assurance on that device, so never honor a trusted-device token for *step-up* on sensitive actions — only for the initial login factor — and always offer the user a "sign out / forget all devices" control that revokes every token at once.

> **Banking-grade MFA checklist:**
> - TOTP secrets stored **AES-256-GCM encrypted at rest** with a KMS-backed data key (§22) — never plaintext, never logged.
> - Enrollment **confirmed** by validating a first code before enabling.
> - TOTP verified with ±1 skew but **replay-protected** via `last_step`.
> - Recovery codes are **high-entropy, single-use, hash-only**, constant-time compared, and regenerable (old ones invalidated).
> - Login uses a **short-lived MFA-pending token**, never a half-authorized session; full tokens minted only after the second factor (§12/§14).
> - SMS/email/WhatsApp OTP (§18) used only as a **fallback**, never the strongest factor; prefer TOTP/passkeys (AAL2+).
> - **Step-up auth** with action-bound tokens before high-risk operations.
> - Disabling MFA requires **current factor (or password + email confirm)**; user is notified.
> - **Audit-logged** lifecycle events; **rate-limited** with lockout (§23, Redis).
> - Trusted-device tokens are **signed, expiring, and server-side revocable** — and never honored for step-up.

---

## 20. OAuth 2.0 / OIDC Social Login **[A]**

"Sign in with Google / GitHub / Microsoft" looks like a checkbox feature, but for a banking-grade system it is one of the most security-sensitive flows you will ever ship: a single mistake (trusting an unverified email, skipping `state`, accepting an unsigned ID token) becomes a full account-takeover primitive. This section builds the flow end-to-end in Go 1.25/1.26 with no shortcuts. It assumes you already have your own session layer (§12) and at-rest encryption (§22); social login *feeds* those systems rather than replacing them. If you would rather not own this code at all, `BETTERAUTH_GUIDE.md` covers a managed alternative — but read this section first so you understand what that library is doing on your behalf.

### 20.1 OAuth2 vs OIDC, and why only one flow survives in 2026

The two specifications answer different questions. **OAuth 2.0 is about authorization** — "is this client allowed to access *that resource* on the user's behalf?" Its currency is the **access token**, an opaque or JWT credential you present to a resource server (e.g. the Google Calendar API). OAuth2 deliberately says *nothing* about who the user is; treating an access token as proof of identity is a classic vulnerability.

**OpenID Connect (OIDC) is an identity layer on top of OAuth2** — it answers "*who* is this user?" Its currency is the **ID token**, always a signed JWT with standardized claims (`iss`, `sub`, `aud`, `exp`, `iat`, `nonce`, and optionally `email`, `email_verified`, `name`). For login you care primarily about the ID token; request an access token only if you genuinely need to call provider APIs afterwards.

The roles:

| Role | Who it is here |
|---|---|
| **Client (Relying Party)** | Your Go backend |
| **Authorization Server / OP** | Google, GitHub, Microsoft Entra, Okta, etc. |
| **Resource Server** | The provider's API (only if you store access tokens) |
| **Resource Owner** | The human logging in |

**The only flow to use is the Authorization Code flow with PKCE.** The implicit flow (`response_type=token`) is dead — removed in OAuth 2.1, it leaked tokens through the URL fragment and browser history. PKCE (Proof Key for Code Exchange, RFC 7636) binds the authorization code to a secret your client generated, so an intercepted code is useless to an attacker. In 2026 you use PKCE for *every* client type, confidential web apps included — it costs nothing and closes the code-injection class of attacks. ID-token validation mechanics (signature, `iss`, `aud`, `exp`) are the same JWT rules covered in §11; here we add the OAuth-specific bindings (`state`, `nonce`, PKCE).

### 20.2 Setup: x/oauth2 + go-oidc, configured from secrets

Use `golang.org/x/oauth2` for the OAuth2 dance and `github.com/coreos/go-oidc/v3/oidc` for OIDC discovery and ID-token verification. The OIDC provider object fetches the discovery document (`/.well-known/openid-configuration`) once and exposes the JWKS endpoint used to verify token signatures — never hardcode keys.

Configuration comes from your secret manager / environment, never source control. Minimize scopes: ask only for `openid email profile`, and add API scopes only when a feature needs them.

```go
package socialauth

import (
	"context"
	"fmt"
	"os"

	"github.com/coreos/go-oidc/v3/oidc"
	"golang.org/x/oauth2"
)

// Provider bundles everything needed for one OP (e.g. Google).
type Provider struct {
	Name     string
	OAuth2   *oauth2.Config
	Verifier *oidc.IDTokenVerifier
}

func NewGoogleProvider(ctx context.Context) (*Provider, error) {
	clientID := os.Getenv("GOOGLE_CLIENT_ID")
	clientSecret := os.Getenv("GOOGLE_CLIENT_SECRET")
	redirectURL := os.Getenv("GOOGLE_REDIRECT_URL") // exact, registered URL

	if clientID == "" || clientSecret == "" || redirectURL == "" {
		return nil, fmt.Errorf("google oauth env not configured")
	}

	// Discovery: fetches issuer metadata + JWKS endpoint over TLS.
	// TLS hardening for this outbound call lives in NETWORKING_GUIDE.md.
	oidcProvider, err := oidc.NewProvider(ctx, "https://accounts.google.com")
	if err != nil {
		return nil, fmt.Errorf("oidc discovery: %w", err)
	}

	return &Provider{
		Name: "google",
		OAuth2: &oauth2.Config{
			ClientID:     clientID,
			ClientSecret: clientSecret,
			RedirectURL:  redirectURL,
			Endpoint:     oidcProvider.Endpoint(),
			// Minimal scopes. "openid" is required for an ID token.
			Scopes: []string{oidc.ScopeOpenID, "email", "profile"},
		},
		// Verifier checks signatures against the provider JWKS and
		// enforces aud == our client ID.
		Verifier: oidcProvider.Verifier(&oidc.Config{ClientID: clientID}),
	}, nil
}
```

### 20.3 The login (redirect) step: state, PKCE, nonce

Before redirecting the user to the provider, generate three independent secrets and bind them to the *current browser session* with a short TTL (5–10 minutes):

- **`state`** — random, opaque CSRF defence. The provider echoes it back; on the callback you reject any mismatch. This stops an attacker from feeding their own authorization code to a victim's session.
- **PKCE `code_verifier`** — high-entropy secret kept server-side; you send only its SHA-256 **`code_challenge`** to the provider, and reveal the verifier only at token exchange.
- **`nonce`** — random value embedded in the ID token by the provider; you verify it on return to prevent ID-token replay.

Store these server-side keyed by session (your session store from §12), not in a cookie the client can tamper with. `x/oauth2` provides PKCE helpers; generate `state`/`nonce` yourself.

```go
package socialauth

import (
	"crypto/rand"
	"encoding/base64"
	"net/http"
	"time"

	"golang.org/x/oauth2"
)

func randomToken(nBytes int) string {
	b := make([]byte, nBytes)
	_, _ = rand.Read(b) // crypto/rand never fails on supported platforms
	return base64.RawURLEncoding.EncodeToString(b)
}

// LoginState is persisted server-side, bound to the session, short TTL.
type LoginState struct {
	State        string
	Nonce        string
	CodeVerifier string
	ReturnTo     string // validated app-local path, never an absolute URL
	ExpiresAt    time.Time
}

func (p *Provider) Login(store SessionStore) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		verifier := oauth2.GenerateVerifier() // RFC 7636 code_verifier
		ls := LoginState{
			State:        randomToken(32),
			Nonce:        randomToken(32),
			CodeVerifier: verifier,
			ReturnTo:     safeReturnTo(r.URL.Query().Get("return_to")),
			ExpiresAt:    time.Now().Add(10 * time.Minute),
		}
		// Bind to the session id; short TTL avoids stale/replayed flows.
		if err := store.SaveLoginState(r.Context(), sessionID(r), ls); err != nil {
			http.Error(w, "server error", http.StatusInternalServerError)
			return
		}

		authURL := p.OAuth2.AuthCodeURL(
			ls.State,
			oidc.Nonce(ls.Nonce),                       // bind nonce into request
			oauth2.S256ChallengeOption(verifier),       // PKCE challenge (S256)
			oauth2.AccessTypeOffline,                   // ask for a refresh token
			oauth2.SetAuthURLParam("prompt", "consent"),
		)
		http.Redirect(w, r, authURL, http.StatusFound)
	}
}

// safeReturnTo prevents open redirects: only allow app-local absolute paths.
func safeReturnTo(v string) string {
	if len(v) == 0 || v[0] != '/' || (len(v) > 1 && v[1] == '/') {
		return "/" // reject "//evil.com", "https://...", empty
	}
	return v
}
```

### 20.4 The callback step: validate everything, trust nothing

The callback is where banking-grade rigor pays off. In order:

1. **Validate `state`** against the server-side value, then delete it (one-time use). A mismatch or missing state is an immediate hard failure.
2. **Exchange the code** with `oauth2.Exchange`, supplying the PKCE **verifier**. Without the verifier the exchange fails — that is PKCE doing its job.
3. **Verify the ID token** with go-oidc: this checks the **signature** against the provider's JWKS, plus `iss`, `aud` (must equal your client ID), and `exp`. These are the same JWT checks formalized in §11.
4. **Verify the `nonce`** in the verified claims equals the one you stored. go-oidc does not auto-check nonce; you must compare it explicitly.
5. **Extract claims and require `email_verified`.** Never trust `email` unless `email_verified == true`; an unverified email is attacker-controllable and is the root of account-takeover-via-email-collision.

```go
func (p *Provider) Callback(store SessionStore, users UserStore) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		ls, err := store.PopLoginState(ctx, sessionID(r)) // load + delete (one-time)
		if err != nil || time.Now().After(ls.ExpiresAt) {
			http.Error(w, "invalid login state", http.StatusBadRequest)
			return
		}

		// 1. state (CSRF). Constant-time compare to avoid timing leaks.
		if !ctEqual(r.URL.Query().Get("state"), ls.State) {
			http.Error(w, "state mismatch", http.StatusBadRequest)
			return
		}
		if errParam := r.URL.Query().Get("error"); errParam != "" {
			http.Error(w, "provider error", http.StatusUnauthorized)
			return
		}

		// 2. Exchange code, proving possession of the PKCE verifier.
		tok, err := p.OAuth2.Exchange(ctx, r.URL.Query().Get("code"),
			oauth2.VerifierOption(ls.CodeVerifier))
		if err != nil {
			http.Error(w, "token exchange failed", http.StatusUnauthorized)
			return
		}

		// 3. Verify ID token (signature via JWKS, iss, aud, exp).
		rawID, ok := tok.Extra("id_token").(string)
		if !ok {
			http.Error(w, "no id_token", http.StatusUnauthorized)
			return
		}
		idToken, err := p.Verifier.Verify(ctx, rawID)
		if err != nil {
			http.Error(w, "id_token verification failed", http.StatusUnauthorized)
			return
		}

		// 4. nonce replay protection (go-oidc does NOT do this for you).
		if !ctEqual(idToken.Nonce, ls.Nonce) {
			http.Error(w, "nonce mismatch", http.StatusUnauthorized)
			return
		}

		// 5. Extract claims; require verified email.
		var claims struct {
			Sub           string `json:"sub"`
			Email         string `json:"email"`
			EmailVerified bool   `json:"email_verified"`
			Name          string `json:"name"`
		}
		if err := idToken.Claims(&claims); err != nil {
			http.Error(w, "bad claims", http.StatusUnauthorized)
			return
		}
		if claims.Sub == "" || !claims.EmailVerified {
			http.Error(w, "email not verified by provider", http.StatusForbidden)
			return
		}

		userID, err := users.FindOrCreateFromOAuth(ctx, OAuthIdentity{
			Provider:       p.Name,
			ProviderUserID: claims.Sub, // stable per-provider id, NOT email
			Email:          claims.Email,
			AccessToken:    tok.AccessToken,
			RefreshToken:   tok.RefreshToken,
			Expiry:         tok.Expiry,
		})
		if err != nil {
			http.Error(w, "account error", http.StatusInternalServerError)
			return
		}

		// 6. Mint YOUR OWN session — see §20.6 / §12.
		issueAppSession(w, r, userID)
		http.Redirect(w, r, ls.ReturnTo, http.StatusFound)
	}
}
```

### 20.5 Mapping provider identity to a local user (schema + storage)

The durable link between a provider account and your user lives in an `oauth_accounts` table. The identity key is **`(provider, provider_user_id)`**, where `provider_user_id` is the OIDC `sub` — a stable, provider-issued identifier. Never key on email; emails change and can be reassigned. Provider tokens are sensitive and are stored **encrypted at rest** (AES-GCM, see §22) — never plaintext. PostgreSQL specifics (constraints, `bytea`, indexing) are in `POSTGRESQL_GUIDE.md`.

```sql
CREATE TABLE oauth_accounts (
	id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	user_id           BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
	provider          TEXT   NOT NULL,                 -- 'google', 'github', ...
	provider_user_id  TEXT   NOT NULL,                 -- OIDC 'sub'
	email             TEXT   NOT NULL,
	access_token_enc  BYTEA,                            -- AES-GCM ciphertext (§22)
	refresh_token_enc BYTEA,                            -- AES-GCM ciphertext (§22)
	token_expiry      TIMESTAMPTZ,
	created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
	updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
	-- One provider account maps to exactly one row.
	CONSTRAINT uq_provider_identity UNIQUE (provider, provider_user_id)
);

CREATE INDEX idx_oauth_accounts_user ON oauth_accounts (user_id);
```

Find-or-create logic, plus **careful account linking**. The rule: an existing provider identity logs the user straight in. A *new* identity whose verified email already belongs to a user may be auto-linked **only because the email was verified by the provider** (`email_verified == true`, enforced in §20.4); otherwise you must require the user to be authenticated before linking. This is the single most important defence against account-takeover-via-email-collision.

```go
func (s *PgUserStore) FindOrCreateFromOAuth(ctx context.Context, id OAuthIdentity) (int64, error) {
	tx, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()

	// Encrypt provider tokens before they ever touch the DB (§22).
	accEnc, _ := s.crypto.Encrypt([]byte(id.AccessToken))
	refEnc, _ := s.crypto.Encrypt([]byte(id.RefreshToken))

	// 1. Existing provider identity? Update tokens, return its user.
	var userID int64
	err = tx.QueryRowContext(ctx,
		`SELECT user_id FROM oauth_accounts WHERE provider=$1 AND provider_user_id=$2`,
		id.Provider, id.ProviderUserID).Scan(&userID)
	if err == nil {
		_, err = tx.ExecContext(ctx,
			`UPDATE oauth_accounts
			    SET access_token_enc=$1, refresh_token_enc=$2,
			        token_expiry=$3, email=$4, updated_at=now()
			  WHERE provider=$5 AND provider_user_id=$6`,
			accEnc, refEnc, id.Expiry, id.Email, id.Provider, id.ProviderUserID)
		if err != nil {
			return 0, err
		}
		return userID, tx.Commit()
	} else if err != sql.ErrNoRows {
		return 0, err
	}

	// 2. New identity. Auto-link ONLY to a user with the same VERIFIED email.
	//    (email_verified was already enforced in the callback.)
	err = tx.QueryRowContext(ctx,
		`SELECT id FROM users WHERE lower(email)=lower($1) AND email_verified=true`,
		id.Email).Scan(&userID)
	if err == sql.ErrNoRows {
		// 3. No match: create a brand-new user.
		err = tx.QueryRowContext(ctx,
			`INSERT INTO users (email, email_verified) VALUES ($1, true) RETURNING id`,
			id.Email).Scan(&userID)
	}
	if err != nil {
		return 0, err
	}

	_, err = tx.ExecContext(ctx,
		`INSERT INTO oauth_accounts
		   (user_id, provider, provider_user_id, email,
		    access_token_enc, refresh_token_enc, token_expiry)
		 VALUES ($1,$2,$3,$4,$5,$6,$7)`,
		userID, id.Provider, id.ProviderUserID, id.Email, accEnc, refEnc, id.Expiry)
	if err != nil {
		return 0, err
	}
	return userID, tx.Commit()
}
```

For linking a provider to an *already logged-in* user (the "connect your Google account" button in settings), skip the email-matching branch entirely: take `user_id` from the current authenticated session and insert directly. Linking is an authenticated action; never infer it from an email alone.

### 20.6 After login: mint your own session, don't leak provider tokens

Once the ID token is verified, the provider's job is done. **Do not hand the provider's access/refresh tokens to the browser.** Instead issue *your own* short-lived access token and rotating refresh token from §12 — that gives you uniform session management, revocation, and refresh rotation across every login method (password, passkey, social). The provider tokens stay encrypted server-side and are used only when you actually need to call the provider's API; refresh them server-side with `oauth2.TokenSource`, then re-encrypt and persist the new values. This keeps high-value provider credentials off the client entirely and under your revocation control.

### 20.7 Security pitfalls (banking-grade)

- **Always validate `state`, `nonce`, and PKCE.** Missing any one re-opens CSRF, ID-token replay, or code-injection respectively. Treat any absence as a hard failure, not a warning.
- **Fully verify the ID token:** signature (JWKS), `iss`, `aud == your client ID`, `exp`, and a small `iat` skew. go-oidc does signature/iss/aud/exp; *you* must check `nonce`. See §11.
- **Pin the redirect URI to an exact match** on both your side and the provider console. No wildcards, no path-prefix matching — open redirect on the callback turns into token theft.
- **Minimize scopes.** Request `openid email profile` and nothing more unless a feature requires it; every extra scope widens blast radius if tokens leak.
- **Account-takeover-via-email-collision:** auto-link a new provider identity to an existing user *only* on a provider-verified matching email. For anything else, require the user to be authenticated and link explicitly. This is the cardinal rule of social login.
- **Encrypt stored provider tokens** with AES-GCM (§22). A DB leak must not yield usable provider credentials.
- **Handle provider account changes/revocation:** treat the OIDC `sub` as the identity (emails change); detect refresh-token revocation and force re-authentication.
- **CSRF on the callback** is handled by `state`; still set `SameSite=Lax` (or `Strict`) cookies on your own session and serve everything over TLS (`NETWORKING_GUIDE.md`).
- **Open-redirect avoidance:** never redirect to a client-supplied absolute URL after login; whitelist app-local paths only (`safeReturnTo` above).

> **Banking-grade OAuth/OIDC checklist:** Authorization Code + PKCE only (implicit flow banned). Generate and verify `state` (CSRF), `nonce` (replay), and PKCE `code_verifier`/`code_challenge`, bound server-side to the session with a short TTL. Fully verify the ID token: JWKS signature, `iss`, `aud == client ID`, `exp`, plus explicit `nonce` check (§11). Exact redirect-URI match; minimized scopes (`openid email profile`). Key local identity on `(provider, sub)`, never email; auto-link only on a provider-verified matching email, otherwise require authenticated explicit linking. Store provider tokens encrypted at rest with AES-GCM (§22) — never plaintext, never to the browser. Mint your own rotating session (§12) after login; refresh provider tokens server-side. Serve over TLS (`NETWORKING_GUIDE.md`); reject client-supplied absolute redirect targets. Prefer the managed path in `BETTERAUTH_GUIDE.md` if you cannot own all of the above.

---

## 21. Passkeys / WebAuthn (Passwordless) **[A]**

Passwords are the single biggest liability in any auth system: they are reused, phished, leaked, and brute-forced. **Passkeys** remove the password entirely. A passkey is a **public-key credential** managed by an *authenticator* (a phone's secure enclave, a laptop's TPM, or a roaming security key like a YubiKey). When a user registers, the authenticator generates a key pair: the **private key never leaves the device**, and only the **public key** is sent to your server. To log in, the authenticator signs a server-issued challenge with that private key; your server verifies the signature with the stored public key.

The security wins are structural, not incidental:

- **Nothing secret lives in your database.** A public key is, by definition, public. A breach of `webauthn_credentials` leaks nothing an attacker can authenticate with — contrast this with password hashes, which are crackable offline.
- **Phishing-resistant by design.** Each credential is bound to a specific **Relying Party ID (RPID)** — your domain. The browser refuses to release an assertion to `evil-bank.com` for a credential registered to `bank.com`. This origin binding is enforced by the browser/OS, not by user vigilance, which is why passkeys defeat the phishing attacks that beat OTP-based MFA (§19).
- **No shared secret over the wire**, so replay, credential stuffing, and database-leak-replay all stop working.

**Authenticator types you will encounter:**

- **Platform authenticators** — built into the device (Face ID, Touch ID, Windows Hello, Android biometrics). Convenient, tied to the device's secure hardware.
- **Roaming authenticators** — external FIDO2 keys (YubiKey, Titan) over USB/NFC/Bluetooth. Portable across devices; ideal for high-assurance and account recovery.
- **Synced passkeys** — backed up to a cloud keychain (iCloud Keychain, Google Password Manager, 1Password) and available on all the user's devices. Great UX. The `backup_eligible`/`backup_state` flags tell you a credential *can be* or *is* synced.
- **Device-bound passkeys** — never leave the originating hardware. Higher assurance, worse recovery story. Security keys are device-bound.

For a bank you will typically offer passkeys as **primary passwordless login** *and* honor them as a strong **second factor** alongside the MFA policy from §19. The two are not mutually exclusive: a passkey with **user verification** already proves *possession* (the device) and *inherence/knowledge* (biometric or PIN) in one gesture.

> **Terminology bridge:** WebAuthn is the W3C browser API; FIDO2 is the umbrella spec (WebAuthn + CTAP, the authenticator protocol); "passkey" is the FIDO Alliance's user-facing brand for a discoverable WebAuthn credential. They are the same technology at different layers.

### 21.1 The Two Ceremonies

Every WebAuthn interaction is a **challenge → response** exchange. There are exactly two:

1. **Registration (attestation)** — the user creates a new credential. The server issues a random challenge; the authenticator generates a key pair and returns the public key plus an *attestation* (optional proof of what made the key). Your server stores the public key.
2. **Authentication (assertion)** — the user proves they hold a registered private key. The server issues a fresh challenge; the authenticator signs it; the server verifies the signature against the stored public key.

The pieces that make this secure:

- **Challenge** — a cryptographically random, single-use, short-lived nonce. It is the anti-replay mechanism: a signature over an old challenge is worthless. Store it server-side, bind it to the session, delete it on use.
- **RPID** — the domain the credential belongs to (`bank.com`). Must match on both ceremonies.
- **Origin** — the full origin the request came from (`https://app.bank.com`). The library checks the authenticator-reported origin against your allow-list.
- **User Verification (UV)** — did the authenticator confirm *this human* (biometric/PIN) rather than mere presence (a tap)? For banking you **require** UV.

### 21.2 Library & Configuration

Use **`github.com/go-webauthn/webauthn`** — the actively maintained Go implementation in 2026 (the successor to the archived Duo library). It handles CBOR/COSE parsing, attestation, challenge verification, and sign-count checks; you provide storage and a `User`.

```go
import (
	"github.com/go-webauthn/webauthn/webauthn"
)

func newWebAuthn() (*webauthn.WebAuthn, error) {
	return webauthn.New(&webauthn.Config{
		RPDisplayName: "Acme Bank",        // shown in the OS prompt
		RPID:          "bank.com",          // the registrable domain — NOT the full URL, NO scheme/port
		RPOrigins: []string{                 // exact origins allowed to use these credentials
			"https://app.bank.com",
		},
		// Banking-grade defaults: demand verified users.
		AuthenticatorSelection: protocol.AuthenticatorSelection{
			ResidentKey:      protocol.ResidentKeyRequirementRequired,    // discoverable passkeys
			UserVerification: protocol.VerificationRequired,              // require biometric/PIN
		},
		Timeouts: webauthn.TimeoutsConfig{
			Login:        webauthn.TimeoutConfig{Enforce: true, Timeout: 60 * time.Second},
			Registration: webauthn.TimeoutConfig{Enforce: true, Timeout: 60 * time.Second},
		},
	})
}
```

The library needs your user type to satisfy `webauthn.User`. Note that `WebAuthnID()` must be **stable and not contain PII** (use an opaque internal ID, never an email):

```go
type AuthUser struct {
	ID          []byte // opaque, stable, max 64 bytes — e.g. a random UUID's bytes
	Name        string // login handle
	DisplayName string
	Creds       []webauthn.Credential
}

func (u *AuthUser) WebAuthnID() []byte                         { return u.ID }
func (u *AuthUser) WebAuthnName() string                       { return u.Name }
func (u *AuthUser) WebAuthnDisplayName() string                { return u.DisplayName }
func (u *AuthUser) WebAuthnCredentials() []webauthn.Credential { return u.Creds }
func (u *AuthUser) WebAuthnIcon() string                       { return "" } // deprecated; return ""
```

### 21.3 Database Schema

Store one row per credential — a user may (and should) have several. The public key is **not secret**, but its **integrity matters**: if an attacker swaps in their own public key, they own the account. Protect the table with normal access controls and audit writes. See `POSTGRESQL_GUIDE.md` for row-level security and constraint patterns.

```sql
CREATE TABLE webauthn_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    credential_id   BYTEA NOT NULL UNIQUE,          -- the WebAuthn credential ID
    public_key      BYTEA NOT NULL,                 -- COSE-encoded public key
    aaguid          BYTEA,                          -- identifies authenticator model
    sign_count      BIGINT NOT NULL DEFAULT 0,      -- monotonic clone-detection counter
    transports      TEXT[] NOT NULL DEFAULT '{}',   -- usb, nfc, ble, internal, hybrid
    backup_eligible BOOLEAN NOT NULL DEFAULT FALSE, -- credential CAN be synced
    backup_state    BOOLEAN NOT NULL DEFAULT FALSE, -- credential IS currently synced
    name            TEXT,                           -- user-friendly label for management UI
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_used_at    TIMESTAMPTZ
);

CREATE INDEX idx_webauthn_creds_user ON webauthn_credentials(user_id);
-- credential_id is the lookup key during discoverable login:
CREATE UNIQUE INDEX idx_webauthn_creds_credid ON webauthn_credentials(credential_id);
```

### 21.4 Challenge / Session State

`BeginRegistration` and `BeginLogin` return a `*webauthn.SessionData` containing the challenge and expected parameters. This must be stored **server-side**, **single-use**, **short-lived**, and **bound to the current browser session** — never round-tripped to the client where it could be tampered with. A short-lived signed cookie holding only an opaque key into a server-side store (Redis) is the standard pattern; align it with the session model in §12.

```go
// Store keyed by session ID; expire in ~5 min; delete on Finish*.
func (s *Store) PutChallenge(ctx context.Context, sid string, d *webauthn.SessionData) error {
	b, _ := json.Marshal(d)
	return s.rdb.Set(ctx, "wa:chal:"+sid, b, 5*time.Minute).Err()
}

func (s *Store) PopChallenge(ctx context.Context, sid string) (*webauthn.SessionData, error) {
	key := "wa:chal:" + sid
	b, err := s.rdb.GetDel(ctx, key).Bytes() // GETDEL = atomic single-use
	if err != nil {
		return nil, errors.New("challenge expired or already used")
	}
	var d webauthn.SessionData
	return &d, json.Unmarshal(b, &d)
}
```

Using `GETDEL` guarantees a challenge cannot be replayed even under concurrent requests.

### 21.5 Registration Flow

**Step 1 — Begin (Go).** Authenticate the user normally first (they must already be logged in to *add* a passkey, or be in a verified sign-up flow). Generate options and stash the session data.

```go
func (h *Handler) BeginRegister(w http.ResponseWriter, r *http.Request) {
	user := h.currentUser(r) // already authenticated
	options, sessionData, err := h.wa.BeginRegistration(
		user,
		webauthn.WithExclusions(user.ExcludedCredentials()), // prevent duplicate registration
	)
	if err != nil { http.Error(w, "begin failed", 500); return }

	if err := h.store.PutChallenge(r.Context(), h.sessionID(r), sessionData); err != nil {
		http.Error(w, "state error", 500); return
	}
	writeJSON(w, options) // contains the challenge for the browser
}
```

**Step 2 — Browser.** Pass the options to `navigator.credentials.create()`. The OS prompts for biometric/PIN, the authenticator mints the key pair, and the result is posted back.

```js
// options came from /register/begin (base64url fields decoded by a helper or @github/webauthn-json)
const credential = await navigator.credentials.create({ publicKey: options });
await fetch("/register/finish", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(credential), // serialize the PublicKeyCredential
});
```

**Step 3 — Finish (Go).** Validate against the stored challenge, then persist the credential.

```go
func (h *Handler) FinishRegister(w http.ResponseWriter, r *http.Request) {
	user := h.currentUser(r)
	sessionData, err := h.store.PopChallenge(r.Context(), h.sessionID(r))
	if err != nil { http.Error(w, "no challenge", 400); return }

	cred, err := h.wa.FinishRegistration(user, *sessionData, r)
	if err != nil { http.Error(w, "verification failed", 400); return }

	// Banking-grade: reject if the authenticator did not perform user verification.
	if !cred.Flags.UserVerified {
		http.Error(w, "user verification required", 400); return
	}

	if err := h.store.SaveCredential(r.Context(), user.ID, cred); err != nil {
		http.Error(w, "persist failed", 500); return
	}
	w.WriteHeader(http.StatusCreated)
}
```

```go
func (s *Store) SaveCredential(ctx context.Context, userID []byte, c *webauthn.Credential) error {
	_, err := s.db.ExecContext(ctx, `
		INSERT INTO webauthn_credentials
		  (user_id, credential_id, public_key, aaguid, sign_count,
		   transports, backup_eligible, backup_state)
		VALUES ($1,$2,$3,$4,$5,$6,$7,$8)`,
		userID, c.ID, c.PublicKey, c.Authenticator.AAGUID,
		c.Authenticator.SignCount, pq.Array(transportsToStrings(c.Transport)),
		c.Flags.BackupEligible, c.Flags.BackupState,
	)
	return err
}
```

### 21.6 Authentication Flow

For passwordless UX prefer **discoverable login** (usernameless): the user is never asked to type an identifier — the authenticator offers the right passkey for the RPID and the browser autofills it. The credential ID in the assertion tells you *which* user.

**Step 1 — Begin (Go).**

```go
func (h *Handler) BeginLogin(w http.ResponseWriter, r *http.Request) {
	options, sessionData, err := h.wa.BeginDiscoverableLogin()
	if err != nil { http.Error(w, "begin failed", 500); return }

	if err := h.store.PutChallenge(r.Context(), h.sessionID(r), sessionData); err != nil {
		http.Error(w, "state error", 500); return
	}
	writeJSON(w, options)
}
```

**Step 2 — Browser.**

```js
const assertion = await navigator.credentials.get({
  publicKey: options,
  mediation: "conditional", // enables passkey autofill in the username field
});
await fetch("/login/finish", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(assertion),
});
```

**Step 3 — Finish (Go).** The `FinishDiscoverableLogin` handler takes a lookup callback that resolves the user from the credential's user-handle/ID. After verification, **update the sign count and detect clones**, then mint your own session (§12).

```go
func (h *Handler) FinishLogin(w http.ResponseWriter, r *http.Request) {
	sessionData, err := h.store.PopChallenge(r.Context(), h.sessionID(r))
	if err != nil { http.Error(w, "no challenge", 400); return }

	var loggedInUser *AuthUser
	handler := func(rawID, userHandle []byte) (webauthn.User, error) {
		u, err := h.store.LoadUserByHandle(r.Context(), userHandle)
		loggedInUser = u
		return u, err
	}

	cred, err := h.wa.FinishDiscoverableLogin(handler, *sessionData, r)
	if err != nil { http.Error(w, "verification failed", 401); return }

	// Banking-grade: require UV on every high-assurance login.
	if !cred.Flags.UserVerified {
		http.Error(w, "user verification required", 401); return
	}

	// Clone / replay detection via the signature counter (see 21.7).
	if cred.Authenticator.CloneWarning {
		h.audit.Flag(r.Context(), loggedInUser.ID, "webauthn_clone_warning")
		http.Error(w, "credential anomaly — login blocked", 401); return
	}

	if err := h.store.UpdateSignCount(r.Context(), cred.ID,
		cred.Authenticator.SignCount, cred.Flags.BackupState); err != nil {
		http.Error(w, "state update failed", 500); return
	}

	// Success: issue your application session / JWT (§12).
	h.sessions.Issue(w, r, loggedInUser.ID)
	w.WriteHeader(http.StatusOK)
}
```

### 21.7 Sign-Count Clone Detection

Each assertion carries a **signature counter** that most authenticators increment on every use. The rule: the new count must be **strictly greater** than the stored count (some synced/cloud passkeys legitimately report `0` always — treat a constant `0` as "counter not supported," not as an attack). If the library reports `CloneWarning` — i.e. the presented count is **less than or equal to** the stored count while the counter is in use — it means two copies of the "same" private key exist: the credential has been **cloned**. For a bank, do not silently continue: block the login, alert the user, force re-enrollment of that credential, and review recent activity.

```go
func (s *Store) UpdateSignCount(ctx context.Context, credID []byte, newCount uint32, backupState bool) error {
	_, err := s.db.ExecContext(ctx, `
		UPDATE webauthn_credentials
		   SET sign_count = $1, backup_state = $2, last_used_at = now()
		 WHERE credential_id = $3`,
		newCount, backupState, credID)
	return err
}
```

The `go-webauthn` library performs the comparison for you using the `SignCount` you loaded into the `webauthn.Credential`; your job is to **load the stored count before verifying** and **persist the new one after**, and to honor `CloneWarning`.

### 21.8 Multi-Credential Management & Recovery

A real deployment needs a management surface and a recovery plan — this is where banking-grade systems live or die.

- **Multiple passkeys per user.** Encourage at least two: a platform passkey (phone/laptop) and a roaming security key kept safe. List them with `name`, `created_at`, `last_used_at`, and authenticator model (from `aaguid`).
- **Naming and deletion.** Let users label and remove credentials. **Never let a user delete their last authentication factor** without a verified fallback in place — otherwise they self-lock out of a bank account. Enforce this server-side, not just in the UI.

```go
func (s *Store) DeleteCredential(ctx context.Context, userID, credID []byte) error {
	var remaining int
	if err := s.db.QueryRowContext(ctx,
		`SELECT count(*) FROM webauthn_credentials WHERE user_id=$1`, userID).
		Scan(&remaining); err != nil {
		return err
	}
	if remaining <= 1 && !s.userHasOtherFactor(ctx, userID) {
		return errors.New("cannot remove the last authentication factor")
	}
	_, err := s.db.ExecContext(ctx,
		`DELETE FROM webauthn_credentials WHERE user_id=$1 AND credential_id=$2`, userID, credID)
	return err
}
```

- **Account recovery is the hard part.** A device can be lost. Provide a deliberate, high-friction path: one-time **recovery codes** generated at enrollment (stored hashed, encrypted at rest — see §22), step-up via an *independent* MFA factor (§19), and for high-value events, manual/identity-proofing review. Recovery is the most attacked surface in any passwordless system; make it as strong as the passkey itself, not a soft underbelly.
- **Policy combination.** Passkeys do not replace your MFA policy (§19) — they satisfy it. Decide, per risk tier, whether a UV passkey alone is sufficient or whether high-value transactions require a *second, distinct* factor.
- **Transport & origin requirements.** Passkeys only work over **TLS** on the **exact registered origin/RPID** (`localhost` is the sole HTTP exception, for dev). Get your domain, certificate, and origin allow-list right — see `NETWORKING_GUIDE.md` for TLS and origin handling. A misconfigured RPID silently breaks phishing resistance.

For a higher-level, batteries-included path that wraps much of this (challenge storage, schema, management endpoints), `BETTERAUTH_GUIDE.md` covers an integrated passkey plugin; the raw `go-webauthn` flow above is what you want when you need full control over the banking-grade checks.

> **Banking-grade passkey checklist:**
> - **Require user verification (UV)** on both registration and every high-assurance login; reject `UserVerified == false`.
> - **Strict origin + RPID** allow-listing; serve only over TLS on the exact registered domain — phishing resistance depends entirely on this.
> - **Server-side, single-use, short-lived challenges** bound to the session (atomic `GETDEL`); never trust client-held state.
> - **Sign-count clone detection**: load before verify, persist after, and **block on `CloneWarning`** (cloned authenticator); tolerate constant-`0` counters.
> - **Store only public keys + metadata** — nothing in the table is a usable secret, but protect its integrity and audit all writes.
> - **Multiple credentials per user**, with a management UI (name / last-used / delete); **never allow deletion of the last factor** without a verified fallback.
> - **Robust recovery**: hashed+encrypted recovery codes (§22), independent step-up MFA (§19), identity proofing for high-value recovery.
> - **Issue your own session/JWT** (§12) after a successful assertion; passkeys authenticate, your session layer authorizes.
> - **Combine with the existing MFA policy** (§19) per risk tier rather than treating passkeys as a total replacement.

---

## 22. Encryption in Transit & at Rest **[A]**

Encryption is the last line of defence — the assumption that something *will* be breached. A banking platform never trusts the network and never trusts the disk: it assumes a hostile actor can read packets on the wire and, separately, can walk away with a stolen drive, a leaked backup, or a snapshot of the database. These are two distinct threat domains and each needs its own controls:

- **In transit** — data moving over a network (browser → API, API → database, service → service). The threat is interception, tampering, and impersonation (man-in-the-middle). The control is **TLS** (and **mTLS** for internal traffic).
- **At rest** — data sitting on a disk, in a backup, in a snapshot, or in object storage. The threat is theft of the storage medium or unauthorised access to a dump. The control is **encryption at rest** at one or more layers: full-disk, transparent database encryption (TDE), and — for the crown jewels — **application-level field encryption**.

Defence in depth means you do not pick *one*. You layer them: TLS 1.3 on every hop, full-disk encryption under the database, *and* AES-256-GCM on the most sensitive columns so that even a DBA with `SELECT *` or someone holding a stolen backup file cannot read a customer's MFA secret or PAN. This section shows how to do each, banking-grade, in Go 1.25/1.26.

For the protocol-level mechanics of TLS (the 1.3 handshake, certificate chains, mTLS), see `NETWORKING_GUIDE.md`. For terminating TLS at the edge, see `NGINX_GUIDE.md`. Key management ties back to §16; the secrets we encrypt below (MFA seeds, OAuth tokens) come from §19/§20; database-side controls are in `POSTGRESQL_GUIDE.md` and `DATABASE_SERVER_ADMIN_GUIDE.md`.

### 22.1 In Transit — TLS Done Right

TLS protects confidentiality (eavesdroppers see ciphertext), integrity (tampering is detected), and authenticity (the client verifies it is really talking to your server). For banking-grade transport you want, at minimum:

- **TLS 1.3 as the target, TLS 1.2 as the hard floor.** Never accept TLS 1.0/1.1 or SSLv3 — they are broken and fail PCI-DSS. TLS 1.3 removes legacy ciphers, makes forward secrecy mandatory, and is faster (1-RTT handshake).
- **Strong cipher suites only** — AEAD ciphers (AES-GCM, ChaCha20-Poly1305), ECDHE key exchange for forward secrecy. No RC4, 3DES, CBC-mode legacy suites.
- **HSTS** (`Strict-Transport-Security`) with a long `max-age`, `includeSubDomains`, and `preload` so browsers refuse to ever connect over plain HTTP — defeats SSL-stripping.
- **Redirect HTTP → HTTPS** so no sensitive request is ever served on port 80.
- **Certificate automation** via ACME / Let's Encrypt (or an internal CA for service-to-service), with **OCSP stapling** so clients can check revocation without a round trip to the CA.

Here is a hardened `tls.Config` and `http.Server` in Go. Note that for TLS 1.3 the `CipherSuites` field is ignored (Go picks the secure 1.3 suites), so the cipher list below only constrains the TLS 1.2 fallback.

```go
package transport

import (
	"crypto/tls"
	"net/http"
	"time"
)

// HardenedTLSConfig returns a banking-grade server TLS configuration.
func HardenedTLSConfig() *tls.Config {
	return &tls.Config{
		// TLS 1.2 is the floor; prefer 1.3 by setting MinVersion to 1.2
		// and letting the client negotiate up. For internal-only services
		// you can set MinVersion: tls.VersionTLS13 to forbid 1.2 entirely.
		MinVersion: tls.VersionTLS12,
		MaxVersion: tls.VersionTLS13,

		// Only matters for the TLS 1.2 fallback (1.3 suites are fixed/secure).
		CipherSuites: []uint16{
			tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
			tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
			tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
			tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,
			tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
			tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
		},

		// Prefer ECDSA/Ed25519 curves with forward secrecy.
		CurvePreferences: []tls.CurveID{
			tls.X25519,
			tls.CurveP256,
		},
	}
}

func NewSecureServer(addr string, h http.Handler) *http.Server {
	return &http.Server{
		Addr:              addr,
		Handler:           h,
		TLSConfig:         HardenedTLSConfig(),
		ReadHeaderTimeout: 10 * time.Second, // slowloris mitigation
		ReadTimeout:       30 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       120 * time.Second,
	}
}
```

HSTS and the HTTP→HTTPS redirect are applied as middleware and a tiny redirector on port 80:

```go
// HSTS middleware: tell browsers to only ever use HTTPS for two years,
// across all subdomains, and submit to the preload list.
func HSTS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Strict-Transport-Security",
			"max-age=63072000; includeSubDomains; preload")
		next.ServeHTTP(w, r)
	})
}

// Redirect all plain-HTTP traffic to HTTPS (run on :80).
func RedirectToHTTPS() http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		target := "https://" + r.Host + r.URL.RequestURI()
		http.Redirect(w, r, target, http.StatusMovedPermanently)
	})
}
```

**Internal service-to-service: mTLS (zero trust).** Public-facing TLS authenticates the *server* to the client. Inside the perimeter — between your API, payment service, ledger, and auth service — you want **mutual TLS**, where each side presents a certificate and verifies the other against an internal CA. This is the cryptographic backbone of zero trust: no service trusts another just because it is "inside the network." See `NETWORKING_GUIDE.md` for the handshake detail.

```go
// mTLS server: require and verify client certificates against an internal CA.
func MutualTLSConfig(caPool *x509.CertPool, serverCert tls.Certificate) *tls.Config {
	cfg := HardenedTLSConfig()
	cfg.MinVersion = tls.VersionTLS13            // internal: enforce 1.3
	cfg.Certificates = []tls.Certificate{serverCert}
	cfg.ClientCAs = caPool                        // who we trust as clients
	cfg.ClientAuth = tls.RequireAndVerifyClientCert
	return cfg
}
```

**Terminate at the edge or go end-to-end?** Two valid patterns:

- **TLS termination at Nginx** (or a load balancer): the edge decrypts, then forwards to Go over a trusted internal network — ideally re-encrypted with mTLS. Simplest to operate; see `NGINX_GUIDE.md`.
- **End-to-end TLS**: Go terminates TLS itself (the `tls.Config` above). Required when the network between edge and app cannot be trusted.

A hardened Nginx TLS server block:

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name api.bank.example;

    ssl_certificate     /etc/letsencrypt/live/api.bank.example/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.bank.example/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305;

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;        # tickets weaken forward secrecy

    ssl_stapling on;                # OCSP stapling
    ssl_stapling_verify on;
    resolver 1.1.1.1 valid=300s;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    location / {
        proxy_pass https://go_backend;   # re-encrypt to the app (mTLS)
    }
}

# Redirect plain HTTP to HTTPS.
server {
    listen 80;
    server_name api.bank.example;
    return 301 https://$host$request_uri;
}
```

**Encrypt the data-path hops too.** TLS is not only for the browser. The database connection must use `sslmode=verify-full` so the driver verifies both the certificate chain *and* the hostname (preventing MITM and rogue-DB attacks). Redis should be reached over `rediss://` (TLS). Plaintext DB/cache traffic on a "private" network is a classic banking-grade failure.

```go
// PostgreSQL: verify-full is the only acceptable mode for sensitive data.
dsn := "postgres://app@db.internal:5432/bank" +
	"?sslmode=verify-full&sslrootcert=/etc/ssl/internal-ca.pem"

// Redis over TLS.
redisURL := "rediss://cache.internal:6380/0"
```

### 22.2 At Rest — What & How

There are three layers of at-rest protection, each defeating a different attacker:

| Layer | Protects against | Notes |
|-------|------------------|-------|
| **Full-disk / volume encryption** (LUKS, cloud KMS-backed EBS/PD encryption) | Physical theft of the drive/host | Transparent to the app; the data is plaintext once the volume is mounted. Does **not** protect against a logged-in attacker, a leaked SQL dump, or a malicious DBA. |
| **Transparent Database Encryption (TDE)** | Theft of the raw data files / cold backups | The DB engine encrypts data files; an authenticated `SELECT` still returns plaintext. See `POSTGRESQL_GUIDE.md` / `DATABASE_SERVER_ADMIN_GUIDE.md`. |
| **Application-level (column/field) encryption** | A DBA, a leaked backup, a compromised DB, an over-broad query | The app encrypts before insert and decrypts after read; the database only ever stores ciphertext. The **only** layer that protects against someone who can read the database. |

Full-disk and TDE are necessary baselines, but they share one weakness: once the system is running and authenticated, the data is readable. For the **crown jewels** — PII, OAuth/refresh tokens, MFA secrets (§19/§20), card PANs — that is not enough. You need **application-level encryption** so that a stolen backup or a DBA with full table access sees only opaque ciphertext.

### 22.3 Application-Level Encryption in Go — AES-256-GCM

Use **AES-256-GCM**: AES with a 256-bit key in Galois/Counter Mode, an **AEAD** (Authenticated Encryption with Associated Data) cipher. AEAD gives you confidentiality *and* integrity in one operation — any tampering with the ciphertext fails decryption, so you never decrypt attacker-modified data.

Three non-negotiable rules:

1. **96-bit (12-byte) random nonce per message**, generated from a CSPRNG (`crypto/rand`).
2. **Never reuse a nonce with the same key.** Nonce reuse in GCM is catastrophic — it leaks the authentication key and lets an attacker forge ciphertexts. With random 96-bit nonces, rotate the key well before ~2³² messages.
3. **Store the nonce alongside the ciphertext** (it is not secret) — the standard pattern is to prepend it.

```go
package crypto

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"errors"
	"io"
)

// Encryptor performs authenticated encryption with a single AES-256 key.
// In production the key comes from a KMS/HSM (see §22.5), never from code.
type Encryptor struct {
	aead cipher.AEAD
}

func NewEncryptor(key []byte) (*Encryptor, error) {
	if len(key) != 32 { // AES-256 requires a 32-byte key
		return nil, errors.New("crypto: key must be 32 bytes (AES-256)")
	}
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	aead, err := cipher.NewGCM(block) // 96-bit nonce, 128-bit tag
	if err != nil {
		return nil, err
	}
	return &Encryptor{aead: aead}, nil
}

// Seal encrypts plaintext and returns nonce||ciphertext||tag.
// aad (associated data, e.g. the user ID or column name) is authenticated
// but not encrypted — it binds the ciphertext to its context.
func (e *Encryptor) Seal(plaintext, aad []byte) ([]byte, error) {
	nonce := make([]byte, e.aead.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return nil, err
	}
	// Seal appends ciphertext+tag to nonce, so the result is nonce-prefixed.
	return e.aead.Seal(nonce, nonce, plaintext, aad), nil
}

// Open reverses Seal. Decryption fails if the data was tampered with
// or the aad does not match.
func (e *Encryptor) Open(blob, aad []byte) ([]byte, error) {
	ns := e.aead.NonceSize()
	if len(blob) < ns {
		return nil, errors.New("crypto: ciphertext too short")
	}
	nonce, ct := blob[:ns], blob[ns:]
	return e.aead.Open(nil, nonce, ct, aad)
}
```

Encrypting a field before persisting it, binding it to the owning user via AAD:

```go
func StoreMFASecret(ctx context.Context, db *sql.DB, enc *Encryptor, userID string, secret []byte) error {
	ct, err := enc.Seal(secret, []byte("mfa_secret|"+userID)) // AAD binds to user+field
	if err != nil {
		return err
	}
	_, err = db.ExecContext(ctx,
		`UPDATE users SET mfa_secret = $1 WHERE id = $2`, ct, userID)
	return err
}
```

Store ciphertext as raw bytes — `BYTEA` in PostgreSQL (never `TEXT`/base64, which wastes space and invites accidental logging):

```sql
ALTER TABLE users
    ADD COLUMN mfa_secret   BYTEA,   -- AES-256-GCM ciphertext (nonce-prefixed)
    ADD COLUMN oauth_token  BYTEA,
    ADD COLUMN key_version  SMALLINT NOT NULL DEFAULT 1; -- for key rotation
```

**Envelope encryption — the banking-grade pattern.** Encrypting every record directly with one master key creates a single point of failure and makes rotation painful. Instead, use a two-tier hierarchy:

- A **Data Encryption Key (DEK)** — a fresh AES-256 key, often per-record or per-tenant — encrypts the data.
- A **Key Encryption Key (KEK)** — lives in a **KMS/HSM** and never leaves it — encrypts (wraps) the DEK.

You store the *wrapped* DEK next to the ciphertext. To read a record: ask the KMS to unwrap the DEK (an audited, access-controlled call), then use the DEK locally to decrypt. The plaintext KEK never touches your application; rotating the KEK only means re-wrapping DEKs, not re-encrypting all data. This is exactly the structure §16 describes for key management.

**Key rotation.** Tag every ciphertext with a `key_version` (column above). On rotation, new writes use the new key; a background job lazily re-encrypts old rows (decrypt with vN, re-encrypt with vN+1). Never delete an old key until every row referencing it has been migrated.

### 22.4 Hashing vs Encryption vs Tokenization

Choosing the wrong primitive is a common, serious mistake — encrypting a password (reversible!) instead of hashing it, or hashing a token you later need to present. The rule is about reversibility:

| Need | Primitive | Example | Reference |
|------|-----------|---------|-----------|
| Verify a value you never need back | **One-way hash** (memory-hard) | Passwords → **Argon2id** | §3 |
| Store a secret you must recover later | **Reversible encryption** | OAuth tokens, MFA seeds → **AES-256-GCM** | §22.3, §19/§20 |
| Remove sensitive data from your systems entirely | **Tokenization** | Card PANs → token from a PCI vault | §22.5 |

- **Passwords are hashed, never encrypted** — you only ever need to *check* them. Use Argon2id (§3). Encryption would let an attacker with the key recover every password.
- **OAuth/refresh tokens and MFA secrets are encrypted** — the system must use the original value again, so it must be reversible (§22.3).
- **Card numbers are tokenized** — replaced with a meaningless token, with the real PAN held in a PCI-DSS-compliant vault you never touch. The best way to secure a PAN is to not store it at all.

### 22.5 Key Management (Banking-Grade)

Encryption is only as strong as its key management. Banking-grade rules:

- **Keys live in a KMS / HSM / secrets manager** (AWS KMS, GCP KMS, HashiCorp Vault, an HSM). **Never** in source code, never in a committed `.env`, never in a container image. A key checked into git is a key already compromised. See §16.
- **Least-privilege access.** Only the services that need to decrypt a given key class can call the KMS for it, enforced by IAM/Vault policies and audited.
- **DEK/KEK hierarchy** (envelope encryption, §22.3): the KEK never leaves the KMS; DEKs are wrapped at rest and only unwrapped in memory for the duration of an operation.
- **Rotation** on a schedule (and immediately on suspected compromise), with versioned keys so old data stays readable until migrated.
- **Separation of duties.** The person who can deploy code should not also be able to export production keys. Key operations are logged for audit.
- **Derive sub-keys with HKDF** rather than reusing one key for multiple purposes — e.g. one root key, distinct sub-keys for "tokens" vs "PII," cryptographically separated.

```go
import (
	"crypto/sha256"
	"io"

	"golang.org/x/crypto/hkdf"
)

// DeriveSubKey produces a purpose-bound 32-byte key from a root key.
// Different `info` values yield cryptographically independent keys.
func DeriveSubKey(rootKey, salt []byte, info string) ([]byte, error) {
	r := hkdf.New(sha256.New, rootKey, salt, []byte(info))
	sub := make([]byte, 32)
	if _, err := io.ReadFull(r, sub); err != nil {
		return nil, err
	}
	return sub, nil
}
```

```bash
# Keys are pulled from the secrets manager at startup, never baked in.
# Example: fetch a wrapped DEK / KEK reference from Vault.
export KEK_ARN="$(vault read -field=kek_arn secret/bank/crypto)"
# The application then calls the KMS to wrap/unwrap DEKs at runtime.
```

### 22.6 PII, Backups & Compliance

Encryption sits inside a broader data-protection discipline:

- **Encrypt PII at rest**, and field-encrypt the crown jewels (names+DOB+government IDs, account numbers, tokens, MFA secrets) so a database leak does not become a customer-data breach.
- **Minimise what you store.** The safest data is the data you never collected. Don't store a PAN if a token will do; don't retain PII past its purpose (GDPR data minimisation and storage limitation).
- **Encrypt backups and snapshots** with their own KMS-managed keys, and test restores. An encrypted production DB with plaintext nightly dumps in object storage is a breach waiting to happen.
- **Crypto-shredding for secure deletion.** When data must be erased (GDPR right to erasure), destroy the per-record/per-tenant DEK — the ciphertext becomes permanently unrecoverable without rewriting every backup. This is the practical way to "delete" data from immutable backups.
- **Logging hygiene.** Never log secrets, keys, tokens, full PANs, or PII. Redact at the logging boundary; treat logs as a data store that also needs access control and retention limits.
- **Compliance pointers (high-level):** **PCI-DSS** mandates strong crypto in transit, encryption/tokenization of PANs at rest, and strict key management with rotation and separation of duties. **GDPR** treats encryption as a recommended safeguard and can reduce breach-notification obligations when stolen data is properly encrypted. Map each requirement to the controls above — TLS 1.3, field-level AES-256-GCM, envelope encryption with KMS, audited rotation.

> **Banking-grade encryption checklist:**
> - **In transit:** TLS 1.3 target / 1.2 floor, AEAD ciphers only, no TLS 1.0/1.1; HSTS with `preload`; HTTP→HTTPS redirect; ACME-automated certs + OCSP stapling.
> - **Internal:** mTLS for every service-to-service hop (zero trust); DB over `sslmode=verify-full`; Redis over `rediss://`.
> - **At rest:** full-disk/volume encryption + TDE as the baseline; **application-level AES-256-GCM** for crown-jewel fields (PII, tokens, MFA secrets, PANs).
> - **AEAD discipline:** AES-256-GCM, fresh 96-bit CSPRNG nonce per message, never reuse a nonce with a key, AAD to bind context, ciphertext stored as `BYTEA`.
> - **Keys:** envelope encryption (DEK wrapped by KMS/HSM KEK); no keys in code/env/images; least privilege; HKDF-derived purpose-bound sub-keys; versioned keys with scheduled rotation; separation of duties + audit (§16).
> - **Right primitive:** hash passwords (Argon2id, §3), encrypt reversible secrets (§19/§20), tokenize PANs.
> - **Data protection:** minimise stored PII; encrypted, tested backups; crypto-shredding for deletion; never log secrets or PII; map controls to PCI-DSS / GDPR.

---

## 23. Rate Limiting & Abuse Prevention **[A]**

Authentication is only as strong as the number of attempts an attacker is allowed to make. A password that would take a century to brute-force offline falls in minutes if your `/auth/login` endpoint cheerfully accepts ten thousand guesses per second. Rate limiting is therefore not a "nice to have" performance knob — for a banking system it is a primary security control, sitting right next to the JWT validation of §17 and the OTP/MFA flows of §18 and §19.

This section builds rate limiting from first principles: the algorithms, an in-process limiter for a single instance, the **distributed Redis limiter** you actually need behind a load balancer, **tiered limits** that clamp down hard on auth endpoints, and the **progressive defences** (lockout, CAPTCHA, velocity checks, alerting) that turn a dumb counter into an abuse-prevention system. We finish at the edge with Nginx.

### 23.1 Why rate limit, and where it belongs

A limiter defends against several distinct threats at once:

- **Brute force & credential stuffing** — automated guessing of passwords or replaying breached username/password pairs against `/auth/login`. See the login-abuse discussion in §14.
- **OTP / SMS pumping fraud** — an attacker triggers thousands of "send me an SMS code" requests. Each one costs you real money at your SMS provider, and "SMS pumping" (a.k.a. toll fraud) can run a bill into five figures overnight. Tie this back to the OTP issuance flow in §18.
- **MFA verify hammering** — guessing 6-digit TOTP/OTP codes. A million codes is only 10^6 guesses; without a limit, a few minutes of requests has a real chance of hitting one. See §19.
- **Scraping** of account data, statements, or pricing.
- **DoS / runaway cost** — even non-malicious clients with a retry bug can melt a backend or spike your cloud bill.

The single most important principle is **defence in depth**: rate limiting belongs at *every* layer, not one.

1. **Edge / CDN / WAF** — cheapest place to drop a flood; absorbs volumetric attacks before they touch your origin (see §23.7 and `NGINX_GUIDE.md`).
2. **Reverse proxy** (Nginx) — coarse per-IP and per-connection limits.
3. **In-application** — the only layer that understands *identity* (this user, this API key, this account) and *semantics* (this is a login, that is a balance read). This is where banking-grade, per-account limits live.

No single layer is sufficient. The edge does not know which account a request targets; the app cannot cheaply shed a 50 Gbps flood. Layer them.

### 23.2 The algorithms, explained

Choosing a limiter algorithm is a trade-off between accuracy, memory, and burst behaviour.

**Fixed window.** Count requests in discrete buckets (e.g. "requests in the 12:00:00–12:00:59 minute"). Simple and cheap, but it has a notorious **burst flaw at the boundary**: a client can send the full quota at 12:00:59 and the full quota again at 12:01:00 — double the intended rate across a two-second span.

**Sliding window.** Fixes the boundary flaw by weighting the previous window. The *sliding-window-log* keeps exact timestamps (accurate, but memory grows with traffic); the *sliding-window-counter* approximates by blending the current and previous fixed windows — cheap and accurate enough for almost everything. This is the workhorse for auth endpoints.

**Token bucket.** A bucket holds up to `burst` tokens and refills at `rate` tokens/second. Each request takes one token; empty bucket means reject. It allows a controlled **burst** (good UX for legitimate bursts) while bounding the long-run average. This is what `golang.org/x/time/rate` implements and what most APIs should use for normal traffic.

**Leaky bucket.** Requests enter a queue that drains at a fixed rate; overflow is dropped. It *smooths* output to a constant rate (no bursts out). Useful in front of a downstream that must not be spiked (e.g. a payment rail). It is essentially token bucket configured for smoothness over burstiness.

**What to key on.** Limit by **IP**, **authenticated user / account**, **API key**, and **endpoint** — and crucially by *combinations*. For login, the right key is **per-account AND per-IP simultaneously** (see §23.6 for why per-account alone is a self-DoS). Quick guide:

| Scenario | Algorithm | Key |
| --- | --- | --- |
| Normal API traffic | Token bucket | API key or user |
| Login / OTP / MFA verify | Sliding window | account + IP (both) |
| Protect a fragile downstream | Leaky bucket | endpoint |
| Cheap coarse edge filter | Fixed/sliding window | IP (/64 for IPv6) |
| Expensive endpoint (reports) | Token bucket, cost-weighted | user/API key |

### 23.3 In-process limiter (single instance)

For a single binary with no load balancer, `golang.org/x/time/rate` is the standard token-bucket limiter. `rate.NewLimiter(r, b)` allows `b` burst and refills at `r` tokens/sec. The subtlety is that you need a limiter *per key* (per IP or per user), so you maintain a map and **evict** idle entries or the map becomes an unbounded memory leak (and a DoS vector itself).

```go
package ratelimit

import (
	"net"
	"net/http"
	"sync"
	"time"

	"golang.org/x/time/rate"
)

type client struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

// PerKey is an in-process, per-key token-bucket limiter with eviction.
type PerKey struct {
	mu      sync.Mutex
	clients map[string]*client
	r       rate.Limit
	b       int
	ttl     time.Duration
}

func NewPerKey(r rate.Limit, b int, ttl time.Duration) *PerKey {
	p := &PerKey{clients: map[string]*client{}, r: r, b: b, ttl: ttl}
	go p.gc()
	return p
}

func (p *PerKey) get(key string) *rate.Limiter {
	p.mu.Lock()
	defer p.mu.Unlock()
	c, ok := p.clients[key]
	if !ok {
		c = &client{limiter: rate.NewLimiter(p.r, p.b)}
		p.clients[key] = c
	}
	c.lastSeen = time.Now()
	return c.limiter
}

// gc evicts idle limiters so the map cannot grow without bound.
func (p *PerKey) gc() {
	t := time.NewTicker(time.Minute)
	for range t.C {
		p.mu.Lock()
		for k, c := range p.clients {
			if time.Since(c.lastSeen) > p.ttl {
				delete(p.clients, k)
			}
		}
		p.mu.Unlock()
	}
}

func (p *PerKey) Allow(key string) bool { return p.get(key).Allow() }
```

A `net/http` middleware. Note the **`429 Too Many Requests`** status, the **`Retry-After`** header (seconds the client should wait), and the standard-track **`RateLimit-*`** headers (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) that let well-behaved clients self-throttle:

```go
func Middleware(p *PerKey, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		key := ClientIP(r) // see §23.7 for trusted-proxy IP extraction
		if !p.Allow(key) {
			w.Header().Set("Retry-After", "1")
			w.Header().Set("RateLimit-Limit", "10")
			w.Header().Set("RateLimit-Remaining", "0")
			w.Header().Set("RateLimit-Reset", "1")
			http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func ClientIP(r *http.Request) string {
	host, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr
	}
	return host
}
```

The equivalent **Gin** middleware (matching the patterns in `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`):

```go
func GinLimit(p *PerKey) gin.HandlerFunc {
	return func(c *gin.Context) {
		if !p.Allow(ClientIP(c.Request)) {
			c.Header("Retry-After", "1")
			c.Header("RateLimit-Remaining", "0")
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate_limited",
			})
			return
		}
		c.Next()
	}
}
```

**The hard limitation:** this only works for *one* process. Two instances behind a load balancer keep two independent maps, so the effective limit doubles per instance and is non-deterministic (which instance you hit depends on the LB). For anything serious, you must share state — that means Redis.

### 23.4 Distributed rate limiting with Redis (the real requirement)

Behind a load balancer the counter must be **shared and atomic**. The canonical solution is a **Lua script run with `EVAL`** on Redis: Redis executes the whole script atomically on a single thread, so the read-modify-write of the bucket cannot race across instances. Doing the same logic with separate `GET`/`SET` calls from Go would have a check-then-act race and over-admit under concurrency.

Here is a **token-bucket Lua script**. It stores two fields per key — current token count and last-refill timestamp — refills lazily based on elapsed time, and sets a TTL so abandoned keys self-clean. It returns whether the request is allowed plus the seconds-to-retry. (See `REDIS_GUIDE.md` for `EVAL`/`EVALSHA` and scripting caveats.)

```lua
-- KEYS[1] = bucket key
-- ARGV[1] = rate (tokens per second)
-- ARGV[2] = burst (bucket capacity)
-- ARGV[3] = now (unix seconds, fractional)
-- ARGV[4] = requested tokens (usually 1; >1 for cost-based limits)
local rate     = tonumber(ARGV[1])
local burst    = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local needed   = tonumber(ARGV[4])

local data = redis.call("HMGET", KEYS[1], "tokens", "ts")
local tokens = tonumber(data[1])
local ts     = tonumber(data[2])
if tokens == nil then
  tokens = burst
  ts = now
end

-- Refill based on elapsed time, capped at burst.
local delta = math.max(0, now - ts)
tokens = math.min(burst, tokens + delta * rate)

local allowed = 0
local retry = 0
if tokens >= needed then
  tokens = tokens - needed
  allowed = 1
else
  retry = (needed - tokens) / rate
end

redis.call("HSET", KEYS[1], "tokens", tokens, "ts", now)
-- TTL: time to fully refill from empty, so idle keys expire.
redis.call("EXPIRE", KEYS[1], math.ceil(burst / rate) + 1)

return {allowed, tostring(retry), tostring(tokens)}
```

The Go wrapper using **go-redis**. Load the script once (`redis.NewScript` caches the SHA and uses `EVALSHA` automatically):

```go
package ratelimit

import (
	"context"
	"strconv"
	"time"

	"github.com/redis/go-redis/v9"
)

var tokenBucket = redis.NewScript(`...the Lua above...`)

type Redis struct {
	rdb   *redis.Client
	rate  float64
	burst int
}

func NewRedis(rdb *redis.Client, rate float64, burst int) *Redis {
	return &Redis{rdb: rdb, rate: rate, burst: burst}
}

type Result struct {
	Allowed   bool
	RetryFor  time.Duration
	Remaining float64
}

func (r *Redis) Allow(ctx context.Context, key string, cost int) (Result, error) {
	now := float64(time.Now().UnixNano()) / 1e9
	res, err := tokenBucket.Run(ctx, r.rdb, []string{key},
		r.rate, r.burst, now, cost).Slice()
	if err != nil {
		// Fail-OPEN or fail-CLOSED is a policy decision. For auth
		// endpoints in banking, fail CLOSED (deny) so a Redis outage
		// cannot become an open brute-force window. For read-only
		// public endpoints you may fail open to preserve availability.
		return Result{}, err
	}
	allowed := res[0].(int64) == 1
	retry, _ := strconv.ParseFloat(res[1].(string), 64)
	rem, _ := strconv.ParseFloat(res[2].(string), 64)
	return Result{
		Allowed:   allowed,
		RetryFor:  time.Duration(retry * float64(time.Second)),
		Remaining: rem,
	}, nil
}
```

Two banking-grade notes baked in above: the **fail-closed policy** on auth endpoints (a Redis outage must not silently disable login throttling), and **cost-weighting** via the `cost` argument (see §23.5). Use a Redis deployment with persistence/HA so the limiter state itself is resilient — consult `REDIS_GUIDE.md`.

### 23.5 Tiered & targeted limits

A flat global limit is wrong: it is either too loose for `/auth/login` or too tight for normal browsing. Apply **tiers per endpoint class**, and for auth endpoints apply **two keys at once** (per account AND per IP). Indicative numbers:

| Endpoint | Limit | Key(s) | Notes |
| --- | --- | --- | --- |
| `/auth/login` | 5 / 5 min | account **and** IP | both must pass; see §14 |
| `/auth/otp` (send) | 3 / 10 min, 10 / day | account **and** IP | SMS-pumping guard, §18 |
| `/auth/mfa/verify` | 5 / 5 min | account **and** IP | code-guessing guard, §19 |
| `/auth/refresh` | 30 / hour | refresh-token / user | detect token replay |
| `/auth/password-reset` | 3 / hour | email **and** IP | enumeration + abuse |
| Normal read API | 600 / min | user or API key | generous |
| Expensive (PDF statement, report) | 10 / min, **cost ≥ 5** | user / API key | cost-based |

Cost-based limiting reuses the `cost` parameter from §23.4: a statement-export request that pins CPU and I/O should draw 5–10 tokens, not 1, so a handful of them exhausts the bucket even though plain reads do not. Per-API-key quotas (e.g. partner integrations) are the same mechanism keyed on the API key with a larger burst and a daily ceiling enforced by a second, longer-window bucket.

Wiring tiered limits is just choosing the key and limiter per route. For login you check both buckets and reject if *either* fails:

```go
func LoginLimit(rl *Redis) gin.HandlerFunc {
	return func(c *gin.Context) {
		ip := ClientIP(c.Request)
		account := c.PostForm("username") // normalise/lower-case first
		ctx := c.Request.Context()

		byIP, _ := rl.Allow(ctx, "login:ip:"+ip, 1)
		byAcct, _ := rl.Allow(ctx, "login:acct:"+account, 1)
		if !byIP.Allowed || !byAcct.Allowed {
			retry := byIP.RetryFor
			if byAcct.RetryFor > retry {
				retry = byAcct.RetryFor
			}
			secs := int(retry.Seconds()) + 1
			c.Header("Retry-After", strconv.Itoa(secs))
			c.AbortWithStatusJSON(http.StatusTooManyRequests,
				gin.H{"error": "too_many_attempts"})
			return
		}
		c.Next()
	}
}
```

### 23.6 Progressive defences

A raw counter rejects with `429` and forgets. Banking-grade abuse prevention *escalates* as suspicion rises:

- **Exponential backoff** — after each failed login, require a growing delay (1s, 2s, 4s, 8s…) before the next attempt is honoured for that account+IP pair. Cheap, and it crushes automated guessing while barely inconveniencing a human who mistyped once.
- **Temporary lockout with auto-unlock** — after N failures, lock for a short window (e.g. 15 min) that clears itself. Never require a support call for a routine lockout; that just moves the DoS to your call centre.
- **CAPTCHA after N failures** — present a challenge once the failure count crosses a threshold. Invisible to normal users, expensive for bots.
- **Velocity / anomaly checks** — flag impossible travel (login from two countries in 5 minutes), sudden spikes of distinct usernames from one IP (classic credential stuffing), or one account hit from hundreds of IPs (distributed attack). Feed these into the alerting pipeline below.
- **Deny-lists / allow-lists** — block known-bad IP ranges and Tor exit nodes for auth; allow-list trusted partner ranges to exempt them from tight limits.
- **Alerting** — emit a metric/event whenever an attack signature fires (e.g. >100 `429`s/min on `/auth/login`, or a single account locked repeatedly). Page on it. A limiter that silently absorbs an attack tells you nothing; you want to *know* you are under credential-stuffing pressure.

**The lockout-as-DoS trap.** If you lock an account purely on failed attempts, an attacker who knows a victim's username can lock them out at will simply by submitting bad passwords — a denial-of-service against *legitimate users*. Avoid it by **keying lockout on account AND IP**, not account alone:

```go
// Lock the (account, ip) pair, not the whole account, so an attacker on
// one IP cannot lock a victim who logs in from elsewhere. Track a global
// per-account counter too, but escalate it to CAPTCHA/step-up rather than
// a hard lock, so the legitimate user is never fully denied.
func recordFailure(ctx context.Context, rdb *redis.Client, account, ip string) (locked bool) {
	key := "fail:" + account + ":" + ip
	n, _ := rdb.Incr(ctx, key).Result()
	if n == 1 {
		rdb.Expire(ctx, key, 15*time.Minute)
	}
	return n >= 5 // lock only this account+IP pair, auto-expires in 15m
}
```

The legitimate user, logging in from their normal IP, is unaffected by an attacker hammering from a different IP. A genuinely distributed attack against one account is caught by the *per-account velocity* signal and escalated to CAPTCHA / step-up MFA (§19) rather than a hard lock.

### 23.7 At the edge: Nginx, proxies, and client-IP correctness

The first and cheapest layer is the reverse proxy. Nginx `limit_req` (request rate, leaky-bucket) and `limit_conn` (concurrent connections) shed floods before they reach Go. See `NGINX_GUIDE.md` for the full directive reference.

```nginx
# Define zones in http{}: 10m of shared memory ~ 160k IPs.
limit_req_zone  $binary_remote_addr zone=login:10m rate=5r/m;
limit_req_zone  $binary_remote_addr zone=api:10m   rate=20r/s;
limit_conn_zone $binary_remote_addr zone=conn:10m;

server {
    location /auth/login {
        limit_req  zone=login burst=3 nodelay;
        limit_conn conn 10;
        limit_req_status 429;          # default is 503; 429 is correct
        proxy_pass http://go_backend;
    }

    location /api/ {
        limit_req  zone=api burst=40 nodelay;
        proxy_pass http://go_backend;
    }
}
```

Beyond Nginx, your **CDN/WAF** (Cloudflare, Fastly, AWS WAF) should carry volumetric and bot-signature rate limits as the outermost layer. The principle is the same as §23.1: layer edge + app. The edge stops the flood; the app enforces *identity-aware* limits the edge cannot see.

**Client-IP pitfalls — get these wrong and your IP limits are worthless or weaponisable:**

- **Never trust `X-Forwarded-For` blindly.** Anyone can send `X-Forwarded-For: 1.2.3.4`. If your limiter keys on it naively, an attacker rotates the header on every request and *bypasses per-IP limits entirely* — or spoofs a victim's IP to get them blocked. Only honour `X-Forwarded-For`/`X-Real-IP` when the connection came from a **trusted proxy**, and take the right-most untrusted hop, not the left-most. Configure Gin's `SetTrustedProxies` (or your framework's equivalent) — do not leave it defaulted to "trust everyone".
- **IPv6: limit by /64, not the single address.** A single user is routinely handed a /64 (or larger) prefix and can cycle through 2^64 addresses for free. Per-/128 limiting is trivially defeated; key IPv6 buckets on the /64 prefix.
- **Shared NAT / CGNAT.** A whole office, a mobile carrier, or a university can sit behind one IPv4. Per-IP limits that are too strict will lock out thousands of innocent users sharing that NAT. This is exactly why auth limits must combine **per-account** with per-IP (§23.5/§23.6) rather than relying on IP alone, and why you keep a sane allow-list for known large NATs.

A small trusted-proxy-aware extractor:

```go
// trusted is the set of proxy CIDRs you control (LB, Nginx). Only then
// is X-Forwarded-For meaningful; otherwise use the socket peer.
func RealClientIP(r *http.Request, trusted []*net.IPNet) string {
	peer, _, _ := net.SplitHostPort(r.RemoteAddr)
	if !ipIn(peer, trusted) {
		return peer // direct connection: ignore any forwarded header
	}
	xff := r.Header.Get("X-Forwarded-For")
	parts := strings.Split(xff, ",")
	// Walk right-to-left; first address NOT in our trusted set is the client.
	for i := len(parts) - 1; i >= 0; i-- {
		ip := strings.TrimSpace(parts[i])
		if !ipIn(ip, trusted) {
			return ip
		}
	}
	return peer
}
```

> **Banking-grade rate-limiting checklist:**
> - Limit at **every layer** — CDN/WAF, Nginx, and in-app — defence in depth (§23.1).
> - Use **distributed Redis limits** with an atomic `EVAL` Lua script behind any load balancer; in-process limiters do not share state (§23.3/§23.4).
> - **Fail closed** on auth endpoints if Redis is unavailable; never let an outage open a brute-force window (§23.4).
> - Apply **tiered, targeted limits**: strict on `/auth/login`, `/auth/otp`, `/auth/mfa/verify`, `/auth/refresh`, password reset; generous on normal API (§23.5).
> - Key auth limits on **account AND IP together**, never account alone — avoids lockout-as-DoS (§23.5/§23.6).
> - **Escalate progressively**: backoff → CAPTCHA → step-up MFA → auto-expiring lockout; never permanent or support-only locks (§23.6).
> - Add **velocity/anomaly checks** (impossible travel, many usernames per IP, many IPs per account) and **alert** on attack signatures (§23.6).
> - Use **cost-based** limiting for expensive endpoints; per-API-key daily quotas for partners (§23.5).
> - Return **`429`** with **`Retry-After`** and **`RateLimit-*`** headers so good clients self-throttle (§23.3).
> - Handle **client IP correctly**: trust `X-Forwarded-For` only from configured proxies, limit IPv6 by **/64**, account for **shared NAT** (§23.7).

---

## 24. File & Image Upload Validation (Defeating Hidden Payloads) **[A]**

File uploads are one of the most dangerous features you can expose. Every byte that arrives is **attacker-controlled**: the filename, the declared content type, the extension, and the content itself are all chosen by whoever is on the other end of the connection. A naive "accept a profile picture" endpoint is, in practice, an arbitrary-bytes-to-disk primitive that an attacker will try to weaponise. For a banking-grade service the rule is blunt: treat an upload as **hostile input that happens to be large**, and do not let a single thing the client *says* about the file influence a security decision.

This section shows how to validate uploads by content (not by what the client claims), how to neutralise hidden payloads by re-encoding, how to bound resource use, and how to store and serve files so that even a malicious file that slips through cannot be executed by a victim's browser. Multipart parsing mechanics (streaming readers, part limits) are covered in detail in §25; for a framework-level walkthrough see the file-upload section of `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`. Storage backends and edge-serving hardening live in `NGINX_GUIDE.md`, and container-level isolation (read-only mounts, non-root, no-exec volumes) in `DOCKER_GUIDE.md`.

### 24.1 The Threat Model: What an Upload Can Do to You

Before writing any validation code, internalise the catalogue of attacks. Each one maps to a specific defence later in this section.

- **Content-type spoofing** — a file named `avatar.jpg`, sent with `Content-Type: image/jpeg`, that is actually HTML, SVG, or a PHP/JSP script. If you trust the extension or the header and serve it back, the browser (or a misconfigured app server) executes it.
- **Polyglots** — a single file that is *simultaneously valid* as two formats. The classic **GIFAR** is a valid GIF and a valid JAR; image+JS polyglots are valid images that also parse as JavaScript when included via `<script src>`. Magic-byte checks alone pass these because the leading bytes really are a valid image header.
- **Malware** — the uploaded blob is a virus/trojan you are now hosting and distributing on your trusted domain.
- **SVG / HTML with embedded `<script>`** — SVG is XML and can carry `<script>`, `onload=`, and external entity references. Served inline from your origin it is **stored XSS** with full access to your cookies and DOM.
- **Decompression / zip bombs** — a 1 KB upload that expands to gigabytes (nested zips, or a tiny image that decodes to a 50,000×50,000 pixel buffer — a "pixel flood"). This is a memory-exhaustion DoS.
- **EXIF / metadata leakage & payloads** — images carry EXIF/XMP/IPTC blocks holding GPS coordinates, device IDs, and arbitrary attacker-supplied strings (which have historically smuggled scripts into downstream parsers).
- **Path traversal via filename** — a filename like `../../etc/cron.d/evil` or `..\\..\\web\\shell.aspx` that escapes your upload directory when you naively join it onto a base path.
- **SSRF / XXE via processed formats** — formats that reference external resources (SVG `<image href>`, XML entities, PDF/Office). If your server-side processor fetches or expands them, it can be coerced into making internal requests or reading local files.
- **Oversized files (DoS)** — a multi-gigabyte body that exhausts disk, memory, or bandwidth before you ever inspect it.

### 24.2 Never Trust the Client

Two pieces of metadata arrive with every multipart upload, and **both are attacker-controlled**:

1. The `Content-Type` header on the part (e.g. `Content-Type: image/png`). The client sets this freely.
2. The **filename**, including its extension (`.png`, `.jpg`). Also freely chosen.

Neither may be used for a security decision. The extension may be used *cosmetically* (to suggest a download name) only after you have independently determined the real type. The header is useful only as a hint to fail fast. Concretely, this means: do not branch storage paths, do not pick a handler, and do not decide "this is safe" based on the declared type or extension. Determine the type yourself, from the bytes.

### 24.3 Validate by Content (Magic Bytes + Real Decode)

The first layer is to read the leading bytes and detect the **actual** type. Go's standard library gives you `http.DetectContentType`, which sniffs the first 512 bytes using the same algorithm browsers use. For sharper detection (and a much larger signature database, including distinguishing `image/svg+xml` from generic XML), use `github.com/gabriel-vasile/mimetype`. Always **allow-list** the exact set of types you accept — never deny-list, because a deny-list is a promise to enumerate every dangerous format, which is impossible.

```go
package upload

import (
	"bytes"
	"errors"
	"io"
	"net/http"

	"github.com/gabriel-vasile/mimetype"
)

// allowedTypes is an ALLOW-LIST. Anything not here is rejected.
var allowedTypes = map[string]bool{
	"image/jpeg": true,
	"image/png":  true,
	"image/webp": true,
	// NOTE: image/svg+xml is deliberately NOT here. See 24.5.
}

// sniffType reads only the header bytes (never the whole file) and returns
// the detected MIME type. It uses both the stdlib sniffer and mimetype so
// that a disagreement can be treated as suspicious.
func sniffType(r io.Reader) (string, []byte, error) {
	header := make([]byte, 512)
	n, err := io.ReadFull(r, header)
	if err != nil && !errors.Is(err, io.ErrUnexpectedEOF) && !errors.Is(err, io.EOF) {
		return "", nil, err
	}
	header = header[:n]

	stdType := http.DetectContentType(header)         // browser-style sniff
	mt := mimetype.Detect(header)                     // richer signature DB
	if !allowedTypes[mt.String()] {
		return "", nil, errors.New("disallowed content type: " + mt.String())
	}
	// Defence-in-depth: if the two sniffers disagree on the family, reject.
	if stdType != "application/octet-stream" &&
		!bytes.HasPrefix([]byte(mt.String()), []byte(stdType[:5])) {
		return "", nil, errors.New("ambiguous content type (possible polyglot)")
	}
	return mt.String(), header, nil
}
```

Magic-byte detection is necessary but **not sufficient**: an attacker can glue a valid PNG header onto arbitrary junk, or build a polyglot whose header sniffs clean. The decisive check is to **actually parse the bytes as the claimed type**. For images, decode them. A real decode (or at least `image.DecodeConfig` to read dimensions) forces the full structure to be valid and is what defeats "header-plus-garbage" payloads.

```go
import (
	"image"
	_ "image/jpeg" // register decoders for image.Decode / DecodeConfig
	_ "image/png"
	_ "golang.org/x/image/webp"
)

// verifyDecodable confirms the bytes really form a valid image of an
// allowed format, and enforces a dimension cap BEFORE any full decode
// (see 24.4 — this is the pixel-flood / decompression-bomb guard).
const maxPixels = 40_000_000 // ~40 MP hard cap

func verifyDecodable(r io.ReadSeeker) error {
	if _, err := r.Seek(0, io.SeekStart); err != nil {
		return err
	}
	cfg, _, err := image.DecodeConfig(r) // cheap: reads only the header
	if err != nil {
		return errors.New("not a decodable image")
	}
	if cfg.Width <= 0 || cfg.Height <= 0 ||
		int64(cfg.Width)*int64(cfg.Height) > maxPixels {
		return errors.New("image dimensions exceed limit")
	}
	return nil
}
```

### 24.4 Defeat Hidden Payloads by Re-encoding

The single strongest defence for images is **server-side re-encoding**. You decode the uploaded image into an in-memory pixel buffer and then re-encode it to a clean file using `image/jpeg` or `image/png`. The output contains *only* pixels you produced — every byte the attacker appended (trailing polyglot payloads, ZIP/JAR central directories, embedded scripts) and almost all metadata are discarded, because they were never part of the decoded pixel data. This converts "validate the attacker's bytes" into "emit my own bytes," which is a far stronger position.

```go
import (
	"image/jpeg"
)

// reencodeJPEG decodes whatever valid image arrived and writes a fresh,
// clean JPEG. The dst file contains no trailing data, no foreign chunks,
// and no original EXIF — only re-encoded pixels.
func reencodeJPEG(src io.ReadSeeker, dst io.Writer) error {
	if _, err := src.Seek(0, io.SeekStart); err != nil {
		return err
	}
	img, _, err := image.Decode(src) // full decode into pixel buffer
	if err != nil {
		return errors.New("decode failed during re-encode")
	}
	// Quality 85 is a sane default; the key point is we OWN every output byte.
	return jpeg.Encode(dst, img, &jpeg.Options{Quality: 85})
}
```

Re-encoding also implicitly strips **EXIF/XMP** metadata, removing GPS coordinates and any attacker-smuggled strings; if you must preserve orientation, read the EXIF Orientation tag, rotate the pixel buffer accordingly, and then discard the metadata block. Cap dimensions (as in `verifyDecodable`) *before* the full `image.Decode`, because a tiny compressed file can declare an enormous canvas — decoding it allocates `width × height × 4` bytes and is a classic pixel-flood DoS.

**SVG is a special case** and is why it is absent from the allow-list above. SVG is executable markup, not pixels: it can contain `<script>`, event handlers (`onload`, `onclick`), `<foreignObject>` HTML, and external/`href` references that drive XSS, SSRF, and XXE. There is no "decode to pixels and re-encode" that preserves it as SVG while making it safe. For banking-grade systems the recommendation is **disallow user SVG entirely**, or, if a product requirement forces it, rasterise it to PNG in a sandboxed/isolated worker, or sanitize it (strip all script elements, all `on*` attributes, and all external references) with a vetted sanitizer. In all cases, **never serve user-supplied SVG inline from your own origin** — see 24.6.

### 24.5 Size, Dimension, and Archive Limits

Resource limits must apply *before* and *during* reading, never only after. Always wrap the request body in `http.MaxBytesReader` so an oversized upload is rejected mid-stream rather than buffered to completion. Pair it with `Request.ParseMultipartForm` or a streaming `MultipartReader` (§25) so you control the in-memory threshold.

```go
const maxUploadBytes = 8 << 20 // 8 MiB hard cap

func handler(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, maxUploadBytes)
	// ParseMultipartForm buffers up to the given size in memory, rest to disk.
	if err := r.ParseMultipartForm(maxUploadBytes); err != nil {
		http.Error(w, "file too large", http.StatusRequestEntityTooLarge)
		return
	}
	// ... fetch the file part and run the validator from 24.7 ...
}
```

If you accept **archives** (zip, gzip, tar), you must guard against decompression bombs explicitly, because the standard `archive/zip` and `compress/*` readers will happily expand whatever they are told to. Enforce three independent limits: a cap on **total decompressed bytes**, a cap on the **number of entries**, and a cap on the **compression ratio** (a single entry that expands more than, say, 100× is almost certainly hostile). Always copy through an `io.LimitReader` rather than trusting the header's declared uncompressed size.

```go
import "archive/zip"

const (
	maxTotalDecompressed = 100 << 20 // 100 MiB across all entries
	maxEntries           = 1000
	maxRatio             = 100
)

func safeUnzip(zr *zip.Reader, write func(name string, r io.Reader) error) error {
	if len(zr.File) > maxEntries {
		return errors.New("too many archive entries")
	}
	var total int64
	for _, f := range zr.File {
		// Ratio guard against bombs that lie about their size.
		if f.UncompressedSize64 > 0 && f.CompressedSize64 > 0 &&
			f.UncompressedSize64/f.CompressedSize64 > maxRatio {
			return errors.New("suspicious compression ratio")
		}
		rc, err := f.Open()
		if err != nil {
			return err
		}
		remaining := maxTotalDecompressed - total
		limited := io.LimitReader(rc, remaining+1)
		// ... validate f.Name for traversal (24.6) before writing ...
		buf := new(bytes.Buffer)
		n, err := io.Copy(buf, limited)
		rc.Close()
		if err != nil {
			return err
		}
		total += n
		if total > maxTotalDecompressed {
			return errors.New("decompressed size limit exceeded (zip bomb)")
		}
		if err := write(f.Name, buf); err != nil {
			return err
		}
	}
	return nil
}
```

### 24.6 Safe Storage and Serving

A file that passes validation can still cause harm if stored or served carelessly. Apply every one of the following.

**Generate a random server-side filename.** Never persist under the user's filename — it carries path-traversal sequences (`../`, `..\\`), null bytes, reserved Windows names (`CON`, `NUL`), and Unicode tricks. Generate a fresh random identifier and derive the extension from the *detected* type, not the upload.

```go
import (
	"crypto/rand"
	"encoding/hex"
	"path/filepath"
)

// safeName returns a random, traversal-proof filename. The extension comes
// from the SERVER-detected type, never from the client.
func safeName(detectedExt string) (string, error) {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	// filepath.Base + Clean strips any path components defensively, though
	// the random hex contains none by construction.
	name := hex.EncodeToString(b) + detectedExt
	return filepath.Base(filepath.Clean(name)), nil
}
```

**Store outside the web root** (or, better, in object storage such as S3/GCS) so the file can never be reached and executed as a script by the application server. The serving directory should be mounted **`noexec`** and the files written **without executable permissions** (`0o600`/`0o640`) — see `DOCKER_GUIDE.md` for `noexec`/read-only volume setup.

**Serve from a separate, cookieless domain.** User content must come from an origin like `usercontent.example-cdn.com` that holds no session cookies, so a file that somehow executes cannot steal credentials or ride the user's session (this is exactly the model Google uses with `googleusercontent.com`). On every response set:

- `Content-Disposition: attachment; filename="..."` so browsers download rather than render risky types inline.
- A correct, restrictive `Content-Type` matching the detected type (never `text/html`, never the client's value).
- `X-Content-Type-Options: nosniff` so the browser will not override your type by sniffing — this is what stops a polyglot from being treated as HTML/JS.

```go
func serveUserFile(w http.ResponseWriter, detectedType, downloadName string, f io.Reader) {
	w.Header().Set("Content-Type", detectedType)
	w.Header().Set("X-Content-Type-Options", "nosniff")
	w.Header().Set("Content-Disposition",
		`attachment; filename="`+downloadName+`"`)
	w.Header().Set("Content-Security-Policy", "default-src 'none'; sandbox")
	io.Copy(w, f)
}
```

Edge configuration (forcing these headers, isolating the content domain, and stripping `Set-Cookie`) is best done at the proxy — see `NGINX_GUIDE.md`. For banking-grade compliance, also run an **AV scan** (e.g. ClamAV via `clamd`) on the stored bytes before the file is made downloadable, and quarantine on a positive.

```bash
# Banking-grade: scan stored uploads out-of-band before exposing them.
clamdscan --fdpass --no-summary /srv/quarantine/uploads/
```

### 24.7 A Reusable Validator That Ties It Together

The following validator composes every layer in the correct order: cap size → sniff the real type → enforce the allow-list → confirm it truly decodes (with a dimension cap) → re-encode to strip payloads → assign a random name → hand back clean bytes for storage. Each step is a gate; failing any one rejects the upload.

```go
type Result struct {
	StoredName  string // random server-side name (24.6)
	ContentType string // server-detected type (24.3)
	CleanBytes  []byte // re-encoded, payload-free bytes (24.4)
}

// ValidateImageUpload runs the full banking-grade pipeline on one image part.
// `src` is the raw uploaded reader; the caller has already wrapped the request
// body in http.MaxBytesReader (24.5).
func ValidateImageUpload(src io.Reader) (*Result, error) {
	// 1. Buffer under a hard cap so all later steps can seek.
	raw, err := io.ReadAll(io.LimitReader(src, maxUploadBytes+1))
	if err != nil {
		return nil, err
	}
	if int64(len(raw)) > maxUploadBytes {
		return nil, errors.New("file exceeds size limit")
	}
	rs := bytes.NewReader(raw)

	// 2. Sniff the REAL type and enforce the allow-list (24.3).
	ctype, _, err := sniffType(rs)
	if err != nil {
		return nil, err // disallowed or ambiguous (possible polyglot)
	}

	// 3. Confirm it actually decodes + enforce dimension cap (24.3 / 24.4).
	if err := verifyDecodable(rs); err != nil {
		return nil, err
	}

	// 4. Re-encode to discard trailing data, polyglot tricks, and metadata.
	var clean bytes.Buffer
	if err := reencodeJPEG(rs, &clean); err != nil {
		return nil, err
	}

	// 5. Assign a random, traversal-proof name with a server-chosen ext.
	name, err := safeName(".jpg") // we re-encoded to JPEG, so the ext is ours
	if err != nil {
		return nil, err
	}

	return &Result{
		StoredName:  name,
		ContentType: "image/jpeg", // matches what we actually wrote
		CleanBytes:  clean.Bytes(),
	}, nil
}
```

Note what this design refuses to do: it never reads the client `Content-Type`, never trusts the client filename, never branches on extension, and never writes the attacker's original bytes to disk. The stored artifact is a file the *server* produced, named with server-side randomness, served as an attachment from a cookieless origin with `nosniff`. That is the banking-grade posture.

> **Banking-grade upload checklist:**
> - **Ignore** the client `Content-Type` header and the file extension for all security decisions.
> - **Allow-list** the real type detected from bytes (`http.DetectContentType` + `mimetype`); never deny-list.
> - **Actually decode/parse** the file as its claimed type — magic bytes alone do not defeat polyglots.
> - **Re-encode** images server-side to strip trailing payloads, polyglot data, and EXIF/metadata.
> - **Disallow or sanitize SVG**; never serve user SVG inline from your origin.
> - Enforce **size, dimension (pixel-flood), and zip-bomb** limits (total size + entry count + ratio).
> - Use a **random server-side filename**; sanitize against path traversal; derive extension from detected type.
> - Store **outside the web root / in object storage**, on a `noexec` mount, with non-executable perms.
> - Serve from a **separate cookieless domain** with `Content-Disposition: attachment`, a correct restrictive `Content-Type`, and `X-Content-Type-Options: nosniff`.
> - For compliance: **AV-scan** (ClamAV) and quarantine before exposing downloads.

---

## 25. Request Payload & Multipart-Form Validation **[A]**

The single most reliable way to get breached is to trust the wire. Every byte that arrives over HTTP — JSON bodies, query strings, headers, form fields, uploaded files, filenames, content types — originates from a party you do not control and cannot trust. The governing principle of this section is blunt: **all input is hostile, validate at the boundary, and allow-list rather than deny-list.** A deny-list enumerates the bad things you currently know about; an allow-list enumerates the small set of good things you actually accept, and rejects everything else by default. Deny-lists rot the moment an attacker finds a variation you forgot. Allow-lists fail closed.

Validation has two distinct halves that are easy to conflate. **Syntactic validation** asks: is this the right *shape*? Correct type, plausible length, in-range number, well-formed email, a UUID that parses, an enum value drawn from the permitted set. **Semantic validation** asks: does this make *business* sense? Is the `from_account` actually owned by the caller, is the transfer amount within today's remaining limit, does the referenced beneficiary exist and is not frozen. Syntactic checks can run statelessly at the edge; semantic checks usually need a database round-trip and belong in the service layer.

A third concern is constantly mistaken for validation and must be kept separate: **authorization**. A request can be perfectly valid — well-formed, in-range, business-coherent — and still be one the caller has no right to make. "Move $500 from account A to account B" is a valid request; whether *this* user may do it is an authorization question (see the security section §17). Never let a passed validation step imply permission. Validate, then authorize, then act.

This section is the input-shaping partner to file *content* validation in §24 (magic-byte sniffing, malware scanning, image re-encoding) — here we harden the *transport and structure* around those payloads. For the Gin-specific binding and validation ergonomics, see `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`; the examples below use `net/http` directly so the mechanics are explicit.

### 25.1 JSON Body Hardening

Decoding untrusted JSON with the defaults is a quiet liability. A naive `json.NewDecoder(r.Body).Decode(&v)` will happily attempt to buffer a multi-gigabyte body (memory-exhaustion DoS), silently ignore fields you never declared (the door to mass-assignment), accept a second JSON document glued onto the end, and bind attacker-controlled keys straight onto whatever struct you point at. We close each of these holes deliberately.

First, **cap the body** with `http.MaxBytesReader`. It wraps the request body so that reading past the limit returns an error *and* signals the server to stop, rather than letting the client stream forever. Pick a limit per endpoint — a login payload needs maybe 1 KiB, a bulk import endpoint more. Second, enable `DisallowUnknownFields()` so any key not present in your target struct is a hard error; this is your primary mass-assignment / over-posting defence at the JSON layer. Third, after a successful decode, call `Decoder.More()` (or a trailing decode) to reject trailing garbage — request smuggling and parser-differential tricks often hide a second token. Fourth, **bind to a DTO, never to your domain or DB model.** A DTO exposes exactly the fields a client is allowed to set; mapping it explicitly to the domain object means a field like `IsAdmin` or `Balance` simply has nowhere to land even if an attacker sends it.

```go
package payment

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"strings"
)

// TransferRequestDTO is the ONLY surface a client may populate. It deliberately
// omits server-owned fields (e.g. status, fee, ledger IDs). Map it to the domain
// model by hand so over-posted fields have nowhere to go.
type TransferRequestDTO struct {
	FromAccount string `json:"from_account"`
	ToAccount   string `json:"to_account"`
	AmountMinor int64  `json:"amount_minor"` // money in minor units (cents) — never floats
	Currency    string `json:"currency"`
	Reference   string `json:"reference"`
}

const maxTransferBody = 4 << 10 // 4 KiB is plenty for this endpoint

// decodeJSON enforces: hard size cap, no unknown fields, single document only.
func decodeJSON[T any](w http.ResponseWriter, r *http.Request, dst *T, maxBytes int64) error {
	r.Body = http.MaxBytesReader(w, r.Body, maxBytes)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // mass-assignment defence

	if err := dec.Decode(dst); err != nil {
		var maxErr *http.MaxBytesError
		switch {
		case errors.As(err, &maxErr):
			return fmt.Errorf("%w: body exceeds %d bytes", errBodyTooLarge, maxBytes)
		case strings.HasPrefix(err.Error(), "json: unknown field "):
			return fmt.Errorf("%w: %s", errUnknownField, strings.TrimPrefix(err.Error(), "json: unknown field "))
		case errors.Is(err, io.EOF):
			return fmt.Errorf("%w: empty body", errMalformedJSON)
		default:
			return fmt.Errorf("%w: %v", errMalformedJSON, err)
		}
	}

	// Reject trailing garbage: a single, complete document is all we accept.
	if dec.More() {
		return fmt.Errorf("%w: trailing data after JSON document", errMalformedJSON)
	}
	return nil
}
```

JSON depth is the other DoS vector: a payload like `[[[[[…]]]]]` nested tens of thousands deep can exhaust the stack during decode. Go's standard library has hardened its parser over recent releases, but for hostile input you should still bound complexity yourself — either pre-scan the byte stream counting bracket depth before decoding, or (in Go 1.25+/1.26) use the `encoding/json/v2` experimental decoder, which exposes explicit limits. A simple guard:

```go
// maxJSONDepth rejects pathologically nested JSON before it reaches the decoder.
func maxJSONDepth(body []byte, limit int) error {
	depth := 0
	inString := false
	escaped := false
	for _, b := range body {
		if inString {
			switch {
			case escaped:
				escaped = false
			case b == '\\':
				escaped = true
			case b == '"':
				inString = false
			}
			continue
		}
		switch b {
		case '"':
			inString = true
		case '{', '[':
			depth++
			if depth > limit {
				return fmt.Errorf("%w: nesting deeper than %d", errMalformedJSON, limit)
			}
		case '}', ']':
			depth--
		}
	}
	return nil
}
```

### 25.2 Struct Validation with go-playground/validator

Decoding gives you a well-formed-but-unchecked DTO. Turning that into a *valid* DTO is the job of `github.com/go-playground/validator/v10` — the de facto standard, and the engine Gin's binding uses under the hood. You annotate fields with tags and call `validate.Struct`; it returns a `validator.ValidationErrors` slice you can translate into precise, field-level feedback.

```go
import "github.com/go-playground/validator/v10"

// One validator per process; it is safe for concurrent use and caches struct metadata.
var validate = validator.New(validator.WithRequiredStructEnabled())

type TransferRequestDTO struct {
	FromAccount string `json:"from_account" validate:"required,uuid4"`
	ToAccount   string `json:"to_account"   validate:"required,uuid4,nefield=FromAccount"`
	AmountMinor int64  `json:"amount_minor" validate:"required,gt=0,lte=100000000"` // ≤ 1,000,000.00
	Currency    string `json:"currency"     validate:"required,iso4217"`
	Reference   string `json:"reference"    validate:"max=140,printascii"`
	Beneficiary struct {
		Email string `json:"email" validate:"required,email"`
		Phone string `json:"phone" validate:"omitempty,e164"`
	} `json:"beneficiary" validate:"required"`
	Tags []string `json:"tags" validate:"max=10,dive,alphanum,max=24"` // dive validates each element
}
```

The tags carry real meaning: `required` rejects zero values, `uuid4`/`iso4217`/`e164`/`email` enforce formats, `gt`/`lte` bound numbers (note the explicit upper bound — never leave a money field unbounded), `oneof=a b c` constrains to an enum, `nefield` cross-checks two fields, and `dive` descends into a slice/map to apply the trailing rules to each element. This is allow-list validation expressed declaratively.

When validation fails, **surface field-level errors without leaking internals.** Map each `FieldError` to a stable, machine-readable code and a safe message; return HTTP 422 (Unprocessable Entity) for semantic/validation failures and 400 for malformed syntax. Never serialize the raw `validator` error string back to the client — it can echo struct field names and internal tags.

```go
type FieldProblem struct {
	Field string `json:"field"`
	Code  string `json:"code"`   // stable, e.g. "required", "out_of_range"
	Hint  string `json:"hint"`   // safe, human-readable
}

func toFieldProblems(err error) []FieldProblem {
	var ve validator.ValidationErrors
	if !errors.As(err, &ve) {
		return nil
	}
	out := make([]FieldProblem, 0, len(ve))
	for _, fe := range ve {
		out = append(out, FieldProblem{
			Field: jsonFieldName(fe), // map struct field -> json tag, not the Go name
			Code:  mapTagToCode(fe.Tag()),
			Hint:  safeHint(fe.Tag(), fe.Param()),
		})
	}
	return out
}
```

### 25.3 Custom Validators & Semantic Rules

Tags cover the common cases; domain rules need custom validators. Register a function once and reference it by tag. Below, an IBAN check (syntactic — checksum and format) shows the pattern; the truly *semantic* checks (does this account exist, is it frozen, is it the caller's) cannot live here because they need state — run those in the service layer after the stateless tag pass succeeds.

```go
func init() {
	_ = validate.RegisterValidation("iban", func(fl validator.FieldLevel) bool {
		return validIBAN(fl.Field().String()) // mod-97 checksum + country length table
	})
}

// In the handler, after decode + tag validation:
//   if err := svc.AssertAccountOwnedBy(ctx, dto.FromAccount, caller.ID); err != nil { ... }
//   if err := svc.AssertWithinDailyLimit(ctx, caller.ID, dto.AmountMinor); err != nil { ... }
// These are SEMANTIC checks — distinct from validation, and distinct again from
// authorization (does this user have the 'transfer' permission at all? see §17).
```

### 25.4 String & Format Hardening

Strings are where most subtle attacks hide, so treat them with suspicion specific to text. Put a **length bound on every string** — both a minimum where it matters and a hard maximum always — because unbounded text is a memory and storage DoS and a downstream-buffer hazard. Beyond length:

- **Reject control characters and NUL bytes.** A `\x00` in a filename or path can truncate strings in C-backed libraries; control chars corrupt logs and enable log injection. Allow only the categories you expect (e.g. letters, digits, spaces, a known punctuation set).
- **Normalize Unicode to NFC** with `golang.org/x/text/unicode/norm` before comparing, storing, or displaying. Without normalization, two visually identical strings can differ byte-wise, breaking uniqueness checks and allowing duplicate-account or filter-bypass tricks.
- **Defend against homoglyphs/confusables** where identity matters (usernames, domains, beneficiary names) — restrict to a single script or run a confusables skeleton check; do not silently accept Cyrillic "а" masquerading as Latin "a".
- **Trim and canonicalize** consistently (collapse whitespace, lowercase emails' domain part) so equality is predictable.
- **Validate emails, phones, and URLs strictly** — `email`, `e164` tags for the first two; for URLs you intend to *fetch* (webhooks, avatar-by-URL, callback registration) you must additionally prevent **SSRF**.
- **Bound every number explicitly** (`gt`/`gte`/`lte`) to avoid integer overflow and absurd values; represent money in integer minor units, never floats.

SSRF deserves its own guard. An attacker who can make your server fetch a URL will point it at `http://169.254.169.254/` (cloud metadata), `http://localhost:6379/` (internal Redis), or a private 10.x host. Allow-list the scheme and host, resolve the name, and reject any address that lands in a private, loopback, link-local, or unique-local range — and re-check after redirects.

```go
import (
	"fmt"
	"net"
	"net/url"
)

var allowedFetchHosts = map[string]bool{"hooks.partner.example": true}

func validateFetchURL(raw string) (*url.URL, error) {
	u, err := url.Parse(raw)
	if err != nil || u.Scheme != "https" {
		return nil, fmt.Errorf("%w: only https URLs allowed", errInvalidURL)
	}
	if !allowedFetchHosts[u.Hostname()] { // allow-list, not deny-list
		return nil, fmt.Errorf("%w: host not permitted", errInvalidURL)
	}
	ips, err := net.LookupIP(u.Hostname())
	if err != nil {
		return nil, fmt.Errorf("%w: cannot resolve host", errInvalidURL)
	}
	for _, ip := range ips {
		if ip.IsLoopback() || ip.IsPrivate() || ip.IsLinkLocalUnicast() || ip.IsUnspecified() {
			return nil, fmt.Errorf("%w: resolves to non-routable address", errInvalidURL)
		}
	}
	return u, nil // NOTE: re-validate the final IP at dial time to defeat DNS-rebinding & redirects
}
```

### 25.5 Injection Defences Are Downstream — and Still Required

Validation narrows the input space; it does not neutralize the *interpreters* that input eventually reaches. A value can pass every tag and still be a SQL-injection payload, an XSS string, or a shell metacharacter sequence — because validity is about shape, not about what a downstream parser does with it. Validation reduces the attack surface but never replaces context-specific output handling. Defence in depth means:

- **SQL:** always use **parameterised queries / prepared statements** — never string-concatenate user input into SQL, no matter how well-validated. See the parameterised-query and injection-defence material in `POSTGRESQL_GUIDE.md`.
- **HTML/template output:** rely on `html/template` (which auto-escapes by context) and never inject raw user strings into markup.
- **OS commands:** avoid the shell; use `exec.Command` with explicit argument slices so no field is ever interpreted as a metacharacter; allow-list the binary and its flags.

Treat validation as one layer in a stack, not the wall.

### 25.6 Hardened Multipart / form-data Handling

Multipart uploads are the highest-risk request shape: large bodies, many parts, attacker-named files, and a parser that will buffer to memory or disk. This is the transport-side partner to the content checks in §24. The order of operations matters enormously.

**Cap the whole request first, before parsing.** Wrap the body in `http.MaxBytesReader` so the total request — across all parts — cannot exceed a hard ceiling. Then choose your parser. `r.ParseMultipartForm(maxMemory)` buffers up to `maxMemory` in RAM and spills the rest to temp files; it is convenient for small, bounded uploads. For large or unknown uploads, prefer the streaming `r.MultipartReader()`, which hands you one part at a time so you can enforce per-part limits and abort early without materializing the whole body. Either way: **limit the number of parts**, enforce a **per-field and per-file size cap**, validate each **field name against an allow-list**, treat every **filename as hostile** (never use it as a path; defer the real content checks — magic bytes, scanning, re-encoding — to §24), and **clean up temp files** with `form.RemoveAll()` in a `defer`. Finally, set server-level `MaxHeaderBytes` and read timeouts so a slow or oversized-header client cannot tie up a connection.

```go
const (
	maxUploadTotal  = 25 << 20 // 25 MiB hard cap across the whole request
	maxParts        = 8
	maxFilePerPart  = 10 << 20 // 10 MiB per file
	maxFieldBytes   = 4 << 10  // non-file fields are tiny
)

var allowedFields = map[string]bool{"document": true, "category": true, "note": true}

func uploadHandler(w http.ResponseWriter, r *http.Request) {
	// 1. Hard total cap BEFORE any parsing.
	r.Body = http.MaxBytesReader(w, r.Body, maxUploadTotal)

	// 2. Stream part-by-part; never buffer the whole body.
	mr, err := r.MultipartReader()
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid_multipart", "malformed upload")
		return
	}

	parts := 0
	for {
		p, err := mr.NextPart()
		if err == io.EOF {
			break
		}
		if err != nil {
			writeError(w, http.StatusBadRequest, "invalid_multipart", "malformed part")
			return
		}
		parts++
		if parts > maxParts {
			writeError(w, http.StatusRequestEntityTooLarge, "too_many_parts", "too many parts")
			return
		}

		// 3. Allow-list the field name.
		if !allowedFields[p.FormName()] {
			writeError(w, http.StatusBadRequest, "unexpected_field", "unexpected field")
			return
		}

		if p.FileName() == "" {
			// Plain field: bound it tightly.
			buf := make([]byte, maxFieldBytes+1)
			n, _ := io.ReadFull(p, buf)
			if n > maxFieldBytes {
				writeError(w, http.StatusRequestEntityTooLarge, "field_too_large", "field too large")
				return
			}
			// ... validate the field value with the same rules as §25.4 ...
			continue
		}

		// 4. File part. The filename is HOSTILE — never use it as a path.
		limited := io.LimitReader(p, maxFilePerPart+1)
		// Stream `limited` to a temp file in a controlled dir, then hand off to §24
		// (magic-byte sniff, AV scan, image re-encode) before accepting it.
		if err := storeAndScanTempFile(r.Context(), limited, maxFilePerPart); err != nil {
			writeError(w, http.StatusRequestEntityTooLarge, "file_too_large", "file too large or rejected")
			return
		}
	}

	writeJSON(w, http.StatusOK, map[string]string{"status": "accepted"})
}
```

If you parse with `ParseMultipartForm` instead of streaming, always `defer r.MultipartForm.RemoveAll()` (after a nil check) so the spilled temp files are deleted even on the error path. And configure the server itself:

```go
srv := &http.Server{
	Addr:              ":8443",
	Handler:           mux,
	ReadHeaderTimeout: 5 * time.Second,  // defeat slowloris-style header drips
	ReadTimeout:       30 * time.Second,
	WriteTimeout:      30 * time.Second,
	MaxHeaderBytes:    1 << 16,          // 64 KiB of headers is generous
}
```

**Gin equivalent (note):** Gin wraps this with `c.Request.ParseMultipartForm` / `c.FormFile` / `c.MultipartForm`, but you must still set `router.MaxMultipartMemory`, wrap the body in `http.MaxBytesReader` via middleware *before* binding, and apply the same field/part/size allow-lists and temp cleanup — the convenience methods do not impose banking-grade limits for you. See `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`.

### 25.7 Consistent Error Responses & Logging

A validation layer is only as good as the signal it returns. Use a **single uniform error envelope** across every endpoint with a **stable machine-readable code** so clients can branch on `code`, not on prose. Map failures consistently: 400 for malformed/unparseable input, 422 for well-formed-but-invalid, 413 for size-limit breaches, 415 for bad content types. Crucially, **never leak internals** — no stack traces, no driver errors, no struct field names, no SQL fragments, and **never echo raw attacker input unencoded** (that is how a reflected-XSS or log-injection payload lands in a browser or a log viewer).

```go
type ErrorEnvelope struct {
	Code     string         `json:"code"`              // stable: "validation_failed", "body_too_large"
	Message  string         `json:"message"`           // safe, generic
	Problems []FieldProblem `json:"problems,omitempty"`// per-field, from §25.2
}
```

On the logging side, **record validation failures** — they are a leading indicator of probing and abuse — but **rate-limit the log output** so an attacker cannot flood your log pipeline (a log-spam DoS and a cost amplification). Log a redacted, length-capped, encoded representation of the offending field, the route, and a correlation ID; never log secrets (passwords, tokens, full PANs) and never log the raw body verbatim. Correlate with the rate-limiting and auth telemetry from §17 to spot credential-stuffing and enumeration patterns early.

> **Banking-grade input-validation checklist:**
> - [ ] All input treated as hostile; **allow-list** everywhere, deny-list nowhere.
> - [ ] Validate **syntactically** (type/shape/length/range/format) and **semantically** (business rules); **authorize separately** (§17).
> - [ ] JSON bodies capped with `http.MaxBytesReader`; `DisallowUnknownFields()` enabled; trailing data rejected via `Decoder.More()`; nesting depth bounded.
> - [ ] Bind to a **DTO**, never to a domain/DB model; map explicitly (no mass-assignment / over-posting).
> - [ ] `validator/v10` tags on every field (required, ranges, formats, `oneof`, `dive`); custom validators for domain formats; field-level **422** errors with no internal leakage.
> - [ ] Strings: hard length bounds, **NFC normalization**, control-char/NUL rejection, homoglyph defence, strict email/phone/URL validation, explicit numeric ranges, money in integer minor units.
> - [ ] URLs you fetch are **SSRF-safe**: https + host allow-list + block loopback/private/link-local; re-check after redirects/DNS.
> - [ ] Validation treated as **one layer** — parameterised SQL (`POSTGRESQL_GUIDE.md`), `html/template` output encoding, no-shell `exec.Command`.
> - [ ] Multipart: total cap with `MaxBytesReader` **before** parsing; stream via `MultipartReader()` for large uploads; limit parts/per-field/per-file size; allow-list field names; filenames hostile; content checks deferred to §24; temp files cleaned (`form.RemoveAll()`).
> - [ ] Server hardened: `MaxHeaderBytes`, `ReadHeaderTimeout`, read/write timeouts.
> - [ ] Uniform error envelope, stable codes, correct status (400/413/415/422); no stack/SQL/field-name leakage; raw input never echoed unencoded.
> - [ ] Validation failures logged (rate-limited, redacted, encoded) and correlated with auth/rate-limit telemetry (§17).

---

## 26. Quick Reference Tables

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

## 27. Study Path & Build-to-Learn Projects

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

*Guide accurate as of June 2026. `github.com/golang-jwt/jwt/v5` (v5.x) and `golang.org/x/crypto` (current), Go 1.25/1.26. Authentication is security-critical: re-read the OWASP cheat sheets and the library changelogs, and run a security review, before shipping to production.*
