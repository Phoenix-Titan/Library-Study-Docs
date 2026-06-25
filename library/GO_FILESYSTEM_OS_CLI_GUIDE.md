# Go — File System, OS Operations & CLI Tools — Complete Offline Reference

> **Who this is for:** Go developers (beginner to intermediate) who want to read and write files, manipulate the filesystem, shell out to external programs, inspect the host system, handle OS signals, and build real command-line tools — using mostly the **Go standard library**, with the modern CLI ecosystem (Cobra, urfave/cli, Viper, charm) covered where it earns its place. Every concept comes with runnable, commented code you can drop into a `main.go` and run with `go run .`.
>
> **Version note:** This guide targets **Go 1.25 / 1.26** (current in 2026). Two recent additions matter a lot here:
> - **`os.Root`** (Go 1.24) — a sandboxed filesystem handle that prevents path-traversal escapes (`..`, absolute paths, symlinks pointing outside the root). Covered in §4.
> - **Range-over-func iterators** (Go 1.23) — `for x := range fn` over functions, which the standard library now exposes in places like `strings.Lines`, `bytes.Lines`, etc. Mentioned where relevant.
>
> Other relevant baselines: `io/fs` and `embed` (Go 1.16+), `os.ReadDir`/`os.ReadFile`/`os.WriteFile` (Go 1.16+), `signal.NotifyContext` (Go 1.16+), `errors.Is`/`As` (Go 1.13+), `log/slog` (Go 1.21+). Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform differences (path separators, `.exe`/`.bat`, permissions, signals) are called out throughout. Confirm exact APIs at pkg.go.dev.

---

## Table of Contents

1. [Reading & Writing Files](#1-reading--writing-files)
2. [File & Directory Operations](#2-file--directory-operations)
3. [Paths — `path/filepath` Deeply](#3-paths--pathfilepath-deeply)
4. [The `io/fs` Package, `embed`, and `os.Root`](#4-the-iofs-package-embed-and-osroot)
5. [File Permissions & Metadata](#5-file-permissions--metadata)
6. [Watching the Filesystem](#6-watching-the-filesystem)
7. [Executing External Commands (`os/exec`)](#7-executing-external-commands-osexec)
8. [Environment Variables](#8-environment-variables)
9. [Accessing System Information](#9-accessing-system-information)
10. [Signals & Process Control](#10-signals--process-control)
11. [Building CLIs — `flag`, Cobra, urfave/cli, Viper, charm](#11-building-clis)
12. [Reading stdin & Interactive Input](#12-reading-stdin--interactive-input)
13. [Archives & Compression — zip, tar, gzip](#13-archives--compression)
14. [Cross-Platform Gotchas, Build Tags & Cross-Compilation](#14-cross-platform-gotchas-build-tags--cross-compilation)
15. [Gotchas & Best Practices](#15-gotchas--best-practices)
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects)

---

## 1. Reading & Writing Files

The two packages you live in are **`os`** (open/create/stat files, the `*os.File` type) and **`io`** (the `Reader`/`Writer`/`Closer` interfaces and helpers like `io.Copy`). Layer **`bufio`** on top for buffered, line-oriented reads and writes.

### 1.1 The quick way — `os.ReadFile` and `os.WriteFile`

For small files (config, JSON, a few KB to a few MB) read or write the whole thing in one call.

```go
package main

import (
    "fmt"
    "log"
    "os"
)

func main() {
    // WriteFile creates the file if it doesn't exist, TRUNCATES it if it does,
    // then writes all the bytes. The third arg is the permission mode (see §5).
    // 0o644 = owner read/write, group + others read. (Mostly ignored on Windows.)
    data := []byte("hello\nworld\n")
    if err := os.WriteFile("greeting.txt", data, 0o644); err != nil {
        log.Fatal(err)
    }

    // ReadFile slurps the ENTIRE file into memory and returns the bytes.
    // Great for small files; DON'T use it on multi-gigabyte files (see §1.6).
    content, err := os.ReadFile("greeting.txt")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%d bytes:\n%s", len(content), content)
}
```

⚡ **Version note:** `os.ReadFile` / `os.WriteFile` / `os.ReadDir` were promoted from `io/ioutil` into `os` in **Go 1.16**. The `ioutil` package is deprecated — use the `os` versions.

### 1.2 Opening files — `os.Open`, `os.Create`, `os.OpenFile`

`os.ReadFile`/`WriteFile` are convenience wrappers. Under the hood they call `os.Open`/`os.Create`, which give you an `*os.File` you can stream from.

```go
// os.Open opens a file READ-ONLY. Equivalent to OpenFile(name, O_RDONLY, 0).
f, err := os.Open("data.txt")
if err != nil {
    // Inspect the error — is it "not found", "permission denied", etc.?
    if os.IsNotExist(err) {
        log.Fatal("file does not exist")
    }
    log.Fatal(err)
}
defer f.Close() // ALWAYS defer Close immediately after a successful open.

// os.Create creates (or truncates) a file for WRITING, mode 0o666 before umask.
// Equivalent to OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0o666).
out, err := os.Create("out.txt")
if err != nil {
    log.Fatal(err)
}
defer out.Close()
```

#### `os.OpenFile` — full control with flags & permissions

`OpenFile` is the general form. You combine **open flags** (bitwise OR) and pass a permission mode (used only when the file is created).

```go
f, err := os.OpenFile("app.log",
    os.O_CREATE|os.O_WRONLY|os.O_APPEND, // create if missing, write-only, append
    0o644)
if err != nil {
    log.Fatal(err)
}
defer f.Close()

fmt.Fprintln(f, "log line appended at the end") // never truncates existing content
```

**Open flag reference (`os.O_*`):**

| Flag | Meaning |
|---|---|
| `os.O_RDONLY` | Open read-only |
| `os.O_WRONLY` | Open write-only |
| `os.O_RDWR` | Open read+write |
| `os.O_APPEND` | Writes go to the end of the file |
| `os.O_CREATE` | Create the file if it doesn't exist |
| `os.O_EXCL` | With `O_CREATE`, fail if the file already exists (atomic "create new") |
| `os.O_TRUNC` | Truncate to zero length on open |
| `os.O_SYNC` | Open for synchronous I/O (writes flush to disk) |

```go
// "Create exclusively" — fails if the file exists. Useful for lock files
// and avoiding accidental overwrites (this combination is atomic).
lock, err := os.OpenFile("app.lock", os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0o644)
if err != nil {
    if os.IsExist(err) {
        log.Fatal("already locked — another instance is running")
    }
    log.Fatal(err)
}
defer lock.Close()
defer os.Remove("app.lock")
```

### 1.3 `io.Reader` / `io.Writer` — the universal interfaces

Almost everything that produces or consumes bytes in Go implements these:

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }
```

`*os.File` implements all three, but so do `bytes.Buffer`, `strings.Reader`, network connections, HTTP bodies, gzip streams, etc. This is why the same `io.Copy` works for files, network sockets, and compressed streams alike.

### 1.4 `io.Copy` — stream from any reader to any writer

```go
// Copy a file the streaming way — constant memory regardless of file size.
func copyFile(src, dst string) error {
    in, err := os.Open(src)
    if err != nil {
        return err
    }
    defer in.Close()

    out, err := os.Create(dst)
    if err != nil {
        return err
    }
    // We need to check the error from Close on the WRITE side specifically,
    // because buffered data may only be flushed on Close.
    defer out.Close()

    // io.Copy reads from `in` and writes to `out` in 32KB chunks until EOF.
    if _, err := io.Copy(out, in); err != nil {
        return err
    }

    // Ensure data is flushed to stable storage and surface any close error.
    return out.Close()
}
```

> **Gotcha — the double Close:** `defer out.Close()` plus a manual `out.Close()` means Close runs twice. The second returns an error (file already closed) which we ignore via the explicit return. A cleaner pattern uses a named return and a deferred closure — shown in §15.

Handy `io` helpers:

```go
io.Copy(dst, src)              // copy everything
io.CopyN(dst, src, 1024)       // copy at most 1024 bytes
io.ReadAll(r)                  // read everything into a []byte (like os.ReadFile but any Reader)
io.WriteString(w, "text")      // write a string to any Writer
io.Discard                     // a Writer that throws bytes away (drain a body)
io.LimitReader(r, 1<<20)       // wrap r so it returns EOF after 1 MB (防 DoS)
io.MultiReader(r1, r2, r3)     // concatenate readers
io.MultiWriter(w1, w2)         // write the same bytes to several writers (tee)
io.TeeReader(r, w)             // read from r while copying what's read into w
```

### 1.5 `bufio` — buffered reading, writing, and scanning

Raw `Read`/`Write` syscalls are expensive when done byte-by-byte or line-by-line. `bufio` batches them.

#### Line-by-line reading with `bufio.Scanner`

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "os"
)

func main() {
    f, err := os.Open("big.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    scanner := bufio.NewScanner(f)
    // Default split is by line (bufio.ScanLines). Other options:
    //   scanner.Split(bufio.ScanWords)  -> token per whitespace-separated word
    //   scanner.Split(bufio.ScanRunes)  -> token per UTF-8 rune
    //   scanner.Split(bufio.ScanBytes)  -> token per byte

    lineNo := 0
    for scanner.Scan() {
        lineNo++
        line := scanner.Text() // the current line WITHOUT the trailing newline
        fmt.Printf("%4d: %s\n", lineNo, line)
    }

    // Scanner stops on EOF OR error. ALWAYS check Err() afterwards — the loop
    // condition returning false does not tell you which happened.
    if err := scanner.Err(); err != nil {
        log.Fatal(err)
    }
}
```

> **Gotcha — the 64KB line limit:** `bufio.Scanner` rejects tokens longer than `bufio.MaxScanTokenSize` (64 KB) with `bufio.ErrTooLong`. For files with very long lines, grow the buffer or use `bufio.Reader.ReadString` instead.

```go
scanner := bufio.NewScanner(f)
buf := make([]byte, 0, 1024*1024)   // 1 MB working buffer
scanner.Buffer(buf, 10*1024*1024)   // allow tokens up to 10 MB
```

#### Line reading without the size limit — `bufio.Reader.ReadString`

```go
reader := bufio.NewReader(f)
for {
    // ReadString reads until (and including) the delimiter '\n'.
    line, err := reader.ReadString('\n')
    if len(line) > 0 {
        // line still contains the trailing '\n' — trim if you don't want it.
        fmt.Print(line)
    }
    if err != nil {
        if err == io.EOF {
            break // normal end of file
        }
        log.Fatal(err)
    }
}
```

⚡ **Version note (Go 1.24):** `bufio.Reader` now has a `ReadByte`-friendly companion and, more usefully, the standard library added range-over-func line iterators. For an in-memory string you can do `for line := range strings.Lines(s) { ... }` (Go 1.24). For files, `bufio.Scanner` is still the idiomatic choice.

#### Buffered writing — `bufio.Writer`

```go
out, err := os.Create("report.txt")
if err != nil {
    log.Fatal(err)
}
defer out.Close()

w := bufio.NewWriter(out)
for i := 0; i < 100_000; i++ {
    // Each Fprintf goes into the in-memory buffer, NOT a syscall, until the
    // buffer fills (default 4096 bytes) or you Flush.
    fmt.Fprintf(w, "line %d\n", i)
}

// CRITICAL: you MUST Flush, or buffered bytes are lost when the program exits.
if err := w.Flush(); err != nil {
    log.Fatal(err)
}
```

> **Gotcha:** `defer w.Flush()` is good, but you should still capture its error — a failed flush means data didn't make it to disk. Pair it with `defer out.Close()` so the order is: Flush (deferred last → runs first), then Close.

### 1.6 Streaming large files

The golden rule: **never `os.ReadFile` a file you can't fit in memory.** Stream it.

```go
// Count lines in an arbitrarily large file with constant memory.
func countLines(path string) (int, error) {
    f, err := os.Open(path)
    if err != nil {
        return 0, err
    }
    defer f.Close()

    // Read in fixed-size chunks and count newline bytes — the fastest portable way.
    buf := make([]byte, 64*1024)
    count := 0
    for {
        n, err := f.Read(buf)
        for _, b := range buf[:n] {
            if b == '\n' {
                count++
            }
        }
        if err != nil {
            if err == io.EOF {
                return count, nil
            }
            return count, err
        }
    }
}
```

```go
// Hash a huge file without loading it into memory: io.Copy streams bytes
// straight through the hash writer.
import "crypto/sha256"

func fileHash(path string) (string, error) {
    f, err := os.Open(path)
    if err != nil {
        return "", err
    }
    defer f.Close()

    h := sha256.New() // hash.Hash implements io.Writer
    if _, err := io.Copy(h, f); err != nil {
        return "", err
    }
    return fmt.Sprintf("%x", h.Sum(nil)), nil
}
```

### 1.7 Seeking and random access

```go
f, _ := os.Open("data.bin")
defer f.Close()

// Seek(offset, whence): whence is io.SeekStart, io.SeekCurrent, or io.SeekEnd.
f.Seek(10, io.SeekStart)        // jump to byte 10
buf := make([]byte, 4)
f.Read(buf)                     // read 4 bytes starting at offset 10

// ReadAt / WriteAt do positioned I/O WITHOUT moving the file offset —
// safe to call concurrently from multiple goroutines on the same *os.File.
f.ReadAt(buf, 100)              // read 4 bytes at offset 100
```

---

## 2. File & Directory Operations

### 2.1 Creating directories

```go
// Mkdir creates ONE directory. It fails if the parent doesn't exist
// or if the directory already exists.
err := os.Mkdir("logs", 0o755)

// MkdirAll creates the directory AND any missing parents (like `mkdir -p`).
// It returns nil (no error) if the directory already exists — very convenient.
err = os.MkdirAll("data/2026/06/reports", 0o755)
```

### 2.2 Removing files & directories

```go
// Remove deletes a single file OR an empty directory. Fails on a non-empty dir.
err := os.Remove("temp.txt")

// RemoveAll deletes a path and EVERYTHING under it recursively (like `rm -rf`).
// It returns nil even if the path doesn't exist — so it's safe to call blindly.
err = os.RemoveAll("build/")
```

> **⚠️ Danger:** `os.RemoveAll` is `rm -rf`. If you build the path from user input or a config value and it ends up being `"/"` or `"C:\\"`, you will destroy the machine. Validate paths and never pass an empty string (which on some setups resolves to the current directory).

### 2.3 Renaming & moving

```go
// Rename moves/renames. On the same filesystem it's atomic and instant.
// ACROSS filesystems (e.g. C: to D:, or / to a mounted volume) it may fail with
// a cross-device error — in that case copy + delete instead.
err := os.Rename("old.txt", "new.txt")
err = os.Rename("draft.txt", "archive/draft.txt") // also "moves"
```

```go
// Robust move that falls back to copy+delete across filesystems.
func moveFile(src, dst string) error {
    if err := os.Rename(src, dst); err == nil {
        return nil // fast path worked
    }
    // Fall back: copy then remove the original.
    if err := copyFile(src, dst); err != nil { // copyFile from §1.4
        return err
    }
    return os.Remove(src)
}
```

### 2.4 Inspecting files — `os.Stat`, `os.Lstat`, `FileInfo`

```go
info, err := os.Stat("greeting.txt")
if err != nil {
    log.Fatal(err)
}

// FileInfo is an interface with these methods:
fmt.Println("name:   ", info.Name())     // base name, e.g. "greeting.txt"
fmt.Println("size:   ", info.Size())     // bytes (for regular files)
fmt.Println("mode:   ", info.Mode())     // FileMode (perms + type bits) — see §5
fmt.Println("modtime:", info.ModTime())  // time.Time of last modification
fmt.Println("isDir:  ", info.IsDir())    // true for directories
```

**`Stat` vs `Lstat`:** `os.Stat` follows symbolic links (returns info about the *target*). `os.Lstat` does **not** follow them (returns info about the *link itself*). Use `Lstat` when walking trees to detect symlinks.

### 2.5 Checking existence (the idiomatic way)

There is no `os.Exists`. You check the error from `os.Stat`:

```go
func exists(path string) (bool, error) {
    _, err := os.Stat(path)
    if err == nil {
        return true, nil
    }
    if errors.Is(err, os.ErrNotExist) { // preferred over os.IsNotExist(err)
        return false, nil
    }
    // Some OTHER error (e.g. permission denied) — report it.
    return false, err
}
```

⚡ **Version note:** Prefer `errors.Is(err, os.ErrNotExist)` / `os.ErrExist` (works with wrapped errors) over the older `os.IsNotExist(err)` / `os.IsExist(err)` predicate functions, which don't unwrap.

> **Gotcha — TOCTOU:** "Check then act" (`if exists { open }`) is a race: the file can be created or deleted between the check and the act. Where it matters (lock files, atomic create), use `os.OpenFile` with `O_CREATE|O_EXCL` and handle the error, rather than checking first.

### 2.6 Listing a directory — `os.ReadDir`

⚡ **Version note (Go 1.16+):** `os.ReadDir` returns `[]fs.DirEntry`, which is cheaper than the old `ioutil.ReadDir` because it doesn't `Stat` every entry up front (it lazily fetches `Info()` only when you ask).

```go
entries, err := os.ReadDir(".")
if err != nil {
    log.Fatal(err)
}

for _, e := range entries {
    // DirEntry is cheap. Call Info() only if you need size/modtime.
    if e.IsDir() {
        fmt.Printf("[DIR]  %s\n", e.Name())
    } else {
        info, _ := e.Info() // fs.FileInfo, same interface as os.Stat returns
        fmt.Printf("%8d  %s\n", info.Size(), e.Name())
    }
}
```

Entries are returned **sorted by filename**. For a quick low-level alternative, `os.Open(dir)` then `f.ReadDir(n)` / `f.Readdirnames(n)` lets you page through huge directories.

### 2.7 Copying a directory recursively

The standard library has no `CopyDir`. Build one with `filepath.WalkDir` (§3.7):

```go
import (
    "io"
    "os"
    "path/filepath"
)

// copyTree copies everything under src into dst, preserving structure.
func copyTree(src, dst string) error {
    return filepath.WalkDir(src, func(path string, d os.DirEntry, err error) error {
        if err != nil {
            return err // propagate walk errors (e.g. permission denied)
        }

        // Compute the path of this entry RELATIVE to src, then re-root under dst.
        rel, err := filepath.Rel(src, path)
        if err != nil {
            return err
        }
        target := filepath.Join(dst, rel)

        if d.IsDir() {
            // Recreate the directory (MkdirAll is idempotent).
            return os.MkdirAll(target, 0o755)
        }

        // Skip non-regular files (symlinks, devices) — handle them explicitly
        // if you need to (see §5 for symlinks).
        if !d.Type().IsRegular() {
            return nil
        }
        return copyFile(path, target) // copyFile from §1.4
    })
}
```

### 2.8 Temporary files & directories

⚡ **Version note (Go 1.16+):** Use `os.CreateTemp` and `os.MkdirTemp` (the `ioutil.TempFile`/`TempDir` versions are deprecated).

```go
// CreateTemp makes a NEW file with a random name. The "*" in the pattern is
// replaced by a random string; here we get something like "upload-839172.tmp".
// An empty first arg means "use the OS temp dir" (os.TempDir() — on Windows
// typically C:\Users\You\AppData\Local\Temp).
tmp, err := os.CreateTemp("", "upload-*.tmp")
if err != nil {
    log.Fatal(err)
}
// Clean up: remove the file when done. Defer both in this order.
defer os.Remove(tmp.Name())
defer tmp.Close()

fmt.Println("temp file:", tmp.Name())
io.WriteString(tmp, "some scratch data")

// MkdirTemp makes a NEW unique directory and returns its path.
dir, err := os.MkdirTemp("", "build-*")
if err != nil {
    log.Fatal(err)
}
defer os.RemoveAll(dir) // remove the whole scratch dir and its contents
fmt.Println("temp dir:", dir)
```

> **Best practice:** In tests, use `t.TempDir()` instead — the testing framework creates a temp dir and **automatically removes it** when the test finishes, with no manual cleanup.

### 2.9 Truncating & syncing

```go
os.Truncate("log.txt", 0)   // chop the file to 0 bytes (empty it without deleting)
f.Truncate(1024)            // resize an open file to exactly 1024 bytes
f.Sync()                    // flush OS buffers to physical disk (durability)
```

---

## 3. Paths — `path/filepath` Deeply

There are **two** path packages and choosing the wrong one is a classic Windows bug.

| Package | Use for | Separator |
|---|---|---|
| **`path/filepath`** | Real filesystem paths | OS-native (`\` on Windows, `/` on Unix) |
| **`path`** | URLs, slash-separated virtual paths, `io/fs` / `embed` paths | Always `/` |

**Rule:** touching the disk → `path/filepath`. Manipulating a URL or an `embed.FS`/`io/fs` path → `path`. On Windows this is not academic: `filepath.Join("a", "b")` gives `a\b`, while `path.Join("a", "b")` gives `a/b`.

### 3.1 Joining paths — `filepath.Join`

```go
// Join cleans the result and uses the OS separator. NEVER build paths with
// string concatenation or fmt.Sprintf("%s/%s", ...) — it breaks on Windows.
p := filepath.Join("data", "2026", "report.csv")
// Windows: data\2026\report.csv
// Unix:    data/2026/report.csv

// Join also cleans ".." and redundant separators:
filepath.Join("a", "b", "..", "c") // "a/c"  (or "a\c" on Windows)
```

### 3.2 Splitting paths apart

```go
full := "/home/user/docs/report.pdf" // (use a Windows path on Windows)

filepath.Dir(full)   // "/home/user/docs"   — everything but the last element
filepath.Base(full)  // "report.pdf"        — the last element
filepath.Ext(full)   // ".pdf"              — the extension INCLUDING the dot

// Split returns dir + file in one call (dir keeps its trailing separator).
dir, file := filepath.Split(full) // dir="/home/user/docs/", file="report.pdf"

// Strip the extension to get the stem:
name := filepath.Base(full)
stem := name[:len(name)-len(filepath.Ext(name))] // "report"
```

### 3.3 Absolute, clean, and relative

```go
// Abs turns a relative path into an absolute one, anchored at the CWD.
abs, err := filepath.Abs("data/x.txt") // e.g. C:\proj\data\x.txt

// Clean normalises: removes ., resolves .., collapses double separators.
filepath.Clean("a/b/../c/./d")  // "a/c/d"
filepath.Clean("a//b/")         // "a/b"

// Rel computes the path to reach `target` FROM `base`.
rel, err := filepath.Rel("/a/b", "/a/b/c/d") // "c/d"
rel, err = filepath.Rel("/a/b", "/a/x")      // "../x"
```

> **Security note:** `filepath.Clean` resolves `..` lexically but does **not** stop a cleaned path from escaping a directory (e.g. `Clean("../../etc/passwd")` stays `../../etc/passwd`). For untrusted paths use `os.Root` (§4.4) or validate with `filepath.IsLocal` (below).

### 3.4 `filepath.IsLocal` — guard against traversal

⚡ **Version note (Go 1.20+):** `filepath.IsLocal(p)` reports whether `p` stays within the current directory: it must be relative, must not start with `..`, and (on Windows) must not be a reserved name like `CON`/`NUL` or use a drive letter.

```go
// Reject path-traversal attempts from user/archive input BEFORE joining.
func safeJoin(root, untrusted string) (string, error) {
    if !filepath.IsLocal(untrusted) {
        return "", fmt.Errorf("unsafe path: %q", untrusted)
    }
    return filepath.Join(root, untrusted), nil
}
```

⚡ **Version note (Go 1.20+):** `filepath.Localize` converts a slash-separated `io/fs` path into a safe OS-local path, rejecting anything that would escape.

### 3.5 `ToSlash` / `FromSlash` — converting separators

```go
// ToSlash converts OS separators to '/' — useful for printing portable output
// or storing paths in JSON/config.
filepath.ToSlash(`C:\Users\me\file.txt`) // "C:/Users/me/file.txt"

// FromSlash converts '/' to the OS separator.
filepath.FromSlash("a/b/c") // "a\b\c" on Windows, "a/b/c" on Unix
```

### 3.6 Glob matching — `filepath.Glob`

```go
// Glob returns all paths matching a shell-style pattern. Patterns support
// * (any non-separator chars), ? (one char), and [abc] character classes.
// It does NOT support ** (recursive). For recursion use WalkDir + Match.
matches, err := filepath.Glob("data/*.csv")
matches, err = filepath.Glob("logs/2026-*.log")

// filepath.Match tests a single name against a pattern (no disk access).
ok, _ := filepath.Match("*.go", "main.go") // true
```

### 3.7 Walking a tree — `filepath.WalkDir` (preferred) vs `filepath.Walk`

⚡ **Version note (Go 1.16+):** Prefer `filepath.WalkDir` over `filepath.Walk`. `WalkDir`'s callback receives a cheap `fs.DirEntry` instead of a `fs.FileInfo`, so it doesn't `Stat` every file — much faster on large trees.

```go
import (
    "fmt"
    "io/fs"
    "path/filepath"
)

func main() {
    root := "."
    err := filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error {
        // The callback gets called once per file/dir, depth-first.
        if err != nil {
            // An error reaching THIS path (e.g. permission denied).
            // Returning the err aborts the walk; return nil to skip & continue.
            fmt.Printf("skip %s: %v\n", path, err)
            return nil
        }

        // Skip whole directories with the sentinel filepath.SkipDir.
        if d.IsDir() && d.Name() == ".git" {
            return filepath.SkipDir // don't descend into .git
        }

        // ⚡ Version note (Go 1.20+): return filepath.SkipAll to stop the ENTIRE
        // walk immediately (not just the current directory).

        if !d.IsDir() {
            info, _ := d.Info()
            fmt.Printf("%10d  %s\n", info.Size(), path)
        }
        return nil
    })
    if err != nil {
        log.Fatal(err)
    }
}
```

**`filepath.SkipDir` semantics:** returned from a **directory** entry → that directory is not descended into. Returned from a **file** entry → the rest of the *containing directory* is skipped. `filepath.SkipAll` (Go 1.20+) stops the whole walk.

### 3.8 Cross-platform path summary

```go
// DO use filepath for disk paths:
cfg := filepath.Join(os.Getenv("HOME"), ".config", "app.toml") // wrong on Windows
home, _ := os.UserHomeDir()                                     // RIGHT — see §9
cfg = filepath.Join(home, ".config", "app.toml")

// DON'T hardcode separators:
bad := "data" + "/" + "file.txt"        // breaks tooling expectations on Windows
good := filepath.Join("data", "file.txt")

// DON'T assume case sensitivity: Windows & macOS are usually case-INSENSITIVE,
// Linux is case-SENSITIVE. "File.txt" and "file.txt" may or may not collide.
```

---

## 4. The `io/fs` Package, `embed`, and `os.Root`

### 4.1 What `io/fs` is for

⚡ **Version note (Go 1.16+):** `io/fs` defines a **read-only, abstract filesystem interface**. Code written against `fs.FS` works identically whether the files live on disk, inside the binary (`embed.FS`), in a zip archive (`zip.Reader` implements `fs.FS`), or in a test fake (`fstest.MapFS`). This decouples your logic from where bytes actually live.

```go
type FS interface {
    Open(name string) (File, error) // name is ALWAYS slash-separated, never "C:\..."
}
```

Key sub-interfaces a filesystem may also implement: `fs.ReadDirFS`, `fs.ReadFileFS`, `fs.StatFS`, `fs.GlobFS`, `fs.SubFS`. The package provides helpers that use them: `fs.ReadFile`, `fs.ReadDir`, `fs.Stat`, `fs.Glob`, `fs.WalkDir`, `fs.Sub`.

### 4.2 `os.DirFS` — view a real directory as an `fs.FS`

```go
// os.DirFS returns an fs.FS rooted at the given directory. Paths used with it
// are slash-separated and CANNOT escape the root (no ".." traversal). Good for
// passing "a directory" to library code in a portable, sandboxed-ish way.
fsys := os.DirFS("/var/www/static")

data, err := fs.ReadFile(fsys, "css/app.css") // note: forward slashes always
entries, err := fs.ReadDir(fsys, "images")
```

> Note: `os.DirFS` blocks `..` but historically could still be tricked via absolute paths/symlinks in some cases. For *hard* sandboxing of writes too, use `os.Root` (§4.4).

### 4.3 `embed` — bake files into the binary with `//go:embed`

⚡ **Version note (Go 1.16+):** The `embed` package compiles files into your executable, so you can ship a single binary with templates, static assets, migrations, etc.

```go
package main

import (
    "embed"
    "fmt"
    "io/fs"
)

// Embed a single file into a string or []byte:
//
//go:embed version.txt
var version string

//go:embed banner.txt
var bannerBytes []byte

// Embed a whole directory tree into an embed.FS (which implements fs.FS):
//
//go:embed templates/* static/*
var assets embed.FS

func main() {
    fmt.Println("version:", version)

    // Read an embedded file via the fs.FS API.
    data, _ := assets.ReadFile("templates/index.html")
    fmt.Println(len(data), "bytes of template")

    // Walk the embedded tree just like a real one.
    fs.WalkDir(assets, ".", func(p string, d fs.DirEntry, err error) error {
        if err == nil {
            fmt.Println(p)
        }
        return nil
    })
}
```

**`//go:embed` rules to remember:**
- The directive must sit **immediately above** the `var` (no blank line) and the var must be package-level.
- Patterns are relative to the `.go` file's directory and use **forward slashes** even on Windows.
- `embed.FS` paths are always slash-separated — manipulate them with `path`, not `path/filepath`.
- By default, files starting with `.` or `_` are excluded. Use `all:` prefix (`//go:embed all:static`) to include them.
- You can embed into `string`, `[]byte`, or `embed.FS` only.

### 4.4 `os.Root` — sandboxed filesystem access (Go 1.24)

⚡ **Version note (Go 1.24):** `os.Root` is the big new filesystem feature. It opens a directory and gives you an `*os.Root` whose methods (`Open`, `Create`, `Mkdir`, `Stat`, `Remove`, `OpenFile`, …) **cannot escape that directory** — not via `..`, not via absolute paths, and not via symlinks that point outside. This is exactly what you want when extracting archives, serving user files, or processing untrusted paths, and it closes the "Zip Slip" class of vulnerabilities.

```go
package main

import (
    "io"
    "log"
    "os"
)

func main() {
    // Open a sandbox rooted at ./userdata. All operations are confined here.
    root, err := os.OpenRoot("userdata")
    if err != nil {
        log.Fatal(err)
    }
    defer root.Close()

    // These all resolve WITHIN the root. Any attempt to escape returns an error.
    f, err := root.Create("notes/today.txt") // creates userdata/notes/today.txt
    if err != nil {
        log.Fatal(err) // (parent dir must exist; use root.Mkdir / MkdirAll-style loop)
    }
    io.WriteString(f, "confined write")
    f.Close()

    // Traversal attempts FAIL instead of escaping:
    if _, err := root.Open("../../etc/passwd"); err != nil {
        log.Println("blocked as expected:", err) // *os.PathError, escapes root
    }

    // Get an fs.FS view of the sandbox for read-only library code:
    fsys := root.FS()
    _ = fsys
}
```

**Why `os.Root` over manual `filepath.Clean` checks:** `Clean` is purely lexical and can't see symlinks. `os.Root` performs the checks **at each path component during resolution**, using OS-level openat-style semantics, so a symlink inside the root that points to `/etc` still can't be followed out. Use it whenever a path comes from outside your program.

### 4.5 `fs.Sub` — narrow an FS to a subtree

```go
// fs.Sub returns an fs.FS rooted at a subdirectory of another fs.FS. Handy to
// strip a prefix from embedded assets (e.g. serve "static/" as the web root).
static, err := fs.Sub(assets, "static")
// Now fs.ReadFile(static, "app.css") reads embedded "static/app.css".
```

### 4.6 Testing with `fstest.MapFS`

```go
import "testing/fstest"

// A fake in-memory filesystem — perfect for unit tests, no disk needed.
fsys := fstest.MapFS{
    "config.json":      {Data: []byte(`{"port":8080}`)},
    "data/users.csv":   {Data: []byte("id,name\n1,alice\n")},
}
data, _ := fs.ReadFile(fsys, "config.json")
```

---

## 5. File Permissions & Metadata

### 5.1 `os.FileMode` — bits for type and permission

`info.Mode()` returns an `os.FileMode` (alias of `fs.FileMode`): a `uint32` where the low 9 bits are Unix permissions and the high bits encode the file *type* (dir, symlink, device, …).

```go
info, _ := os.Stat("script.sh")
mode := info.Mode()

fmt.Println(mode)               // e.g. "-rwxr-xr-x"  (string form like `ls -l`)
fmt.Println(mode.Perm())        // just the permission bits: -rwxr-xr-x
fmt.Printf("%o\n", mode.Perm()) // octal: 755
fmt.Println(mode.IsDir())       // is it a directory?
fmt.Println(mode.IsRegular())   // is it a normal file?

// Type bits you can test against (mode & os.ModeXxx != 0):
//   os.ModeDir, os.ModeSymlink, os.ModeNamedPipe, os.ModeSocket,
//   os.ModeDevice, os.ModeCharDevice, os.ModeSetuid, os.ModeSticky
if mode&os.ModeSymlink != 0 {
    fmt.Println("it's a symlink")
}
```

**Octal permission cheat sheet (Unix):**

| Octal | Symbolic | Meaning |
|---|---|---|
| `0o644` | `rw-r--r--` | Files: owner write, everyone read |
| `0o600` | `rw-------` | Private file (secrets, keys) |
| `0o755` | `rwxr-xr-x` | Executables / directories |
| `0o700` | `rwx------` | Private directory |
| `0o777` | `rwxrwxrwx` | World-writable (avoid) |

> Write octal literals with the `0o` prefix (Go 1.13+); the bare `0755` form still works but `0o755` is clearer.

### 5.2 `os.Chmod` and the Windows caveat

```go
// Make a file executable (Unix). On WINDOWS this is largely a no-op:
// NTFS doesn't use Unix permission bits. Chmod on Windows can only toggle the
// read-only attribute (clearing the write bit makes the file read-only).
os.Chmod("script.sh", 0o755)
```

> **Windows reality:** Go reports a synthesised mode for files on Windows (e.g. `-rw-rw-rw-` for a writable file, `-r--r--r--` for read-only), and "executable" is determined by file *extension* (`.exe`, `.bat`, `.cmd`), not a permission bit. Don't rely on Unix permission semantics in cross-platform code.

### 5.3 `os.Chown` (Unix only)

```go
// Change owner/group by numeric UID/GID. Unix only — returns an error or is a
// no-op on Windows. -1 means "leave unchanged".
err := os.Chown("file.txt", 1000, 1000)
```

To resolve names to IDs, use `os/user` (§9). Guard Chown with a build tag or a runtime `runtime.GOOS != "windows"` check.

### 5.4 Modification & access times

```go
info, _ := os.Stat("file.txt")
fmt.Println("modified:", info.ModTime())

// Chtimes sets access time (atime) and modification time (mtime).
// Pass the zero time to leave one unchanged.
atime := time.Now()
mtime := time.Now().Add(-24 * time.Hour) // pretend it was modified yesterday
os.Chtimes("file.txt", atime, mtime)
```

> **Note:** Creation time and richer timestamps require platform-specific code via `info.Sys()` (a `*syscall.Stat_t` on Unix, `*syscall.Win32FileAttributeData` on Windows). The portable `FileInfo` only guarantees `ModTime()`.

### 5.5 Symbolic links

```go
// Create a symlink: Symlink(target, linkname). On Windows this requires either
// admin privileges OR Developer Mode enabled (since Windows 10 build 14972).
err := os.Symlink("real.txt", "alias.txt")

// Read where a symlink points (does NOT follow it).
dest, err := os.Readlink("alias.txt") // "real.txt"

// Lstat sees the link itself; Stat follows it to the target (§2.4).
li, _ := os.Lstat("alias.txt")
fmt.Println(li.Mode()&os.ModeSymlink != 0) // true

// EvalSymlinks resolves ALL symlinks in a path to the real underlying path.
real, err := filepath.EvalSymlinks("alias.txt")
```

> **Windows caveat:** symlink creation commonly fails with "a required privilege is not held by the client" unless Developer Mode is on or the program runs elevated. Junctions and hard links have different rules. Test on the target OS.

### 5.6 Hard links

```go
// Link(oldname, newname) creates a hard link: a second directory entry pointing
// at the SAME inode/data. Both names are equal; deleting one keeps the data
// until all links are gone. Must be on the same filesystem.
err := os.Link("original.txt", "hardlink.txt")
```

---

## 6. Watching the Filesystem

The standard library has **no** file-watching API. You have two options.

### 6.1 Polling (no dependencies, fully portable)

Poll `os.Stat` on an interval and compare modification times. Simple, robust, and good enough for config reloads.

```go
package main

import (
    "fmt"
    "log"
    "os"
    "time"
)

// watchFile calls onChange whenever the file's size or mod-time changes.
func watchFile(path string, interval time.Duration, onChange func()) error {
    var lastMod time.Time
    var lastSize int64

    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for range ticker.C {
        info, err := os.Stat(path)
        if err != nil {
            if os.IsNotExist(err) {
                continue // file deleted or not created yet — keep waiting
            }
            return err
        }
        // Compare against the last seen state.
        if info.ModTime() != lastMod || info.Size() != lastSize {
            if !lastMod.IsZero() { // skip the very first observation
                onChange()
            }
            lastMod = info.ModTime()
            lastSize = info.Size()
        }
    }
    return nil
}

func main() {
    err := watchFile("config.toml", time.Second, func() {
        fmt.Println("config changed — reloading…")
    })
    log.Fatal(err)
}
```

**Polling trade-offs:** dead simple and portable, but introduces latency (up to one interval) and can miss rapid create-then-delete events. CPU cost is trivial at 1s intervals. For watching whole trees, walk the tree each tick and diff a `map[string]time.Time`.

### 6.2 `fsnotify` — real OS-level notifications

For low-latency, event-driven watching use the de-facto library **`github.com/fsnotify/fsnotify`**, which wraps inotify (Linux), kqueue (macOS/BSD), and ReadDirectoryChangesW (Windows) behind one API.

```go
// go get github.com/fsnotify/fsnotify
import "github.com/fsnotify/fsnotify"

func main() {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        log.Fatal(err)
    }
    defer watcher.Close()

    // Add directories (NOT files individually for whole-dir watching).
    if err := watcher.Add("./config"); err != nil {
        log.Fatal(err)
    }

    for {
        select {
        case event, ok := <-watcher.Events:
            if !ok {
                return
            }
            // event.Op is a bitmask: Create, Write, Remove, Rename, Chmod.
            if event.Op&fsnotify.Write == fsnotify.Write {
                fmt.Println("modified:", event.Name)
            }
        case err, ok := <-watcher.Errors:
            if !ok {
                return
            }
            log.Println("watch error:", err)
        }
    }
}
```

> **fsnotify gotchas:** watching a directory is not recursive — add each subdirectory yourself (walk the tree, add dirs, and add new dirs as Create events arrive). Editors often fire several events per save (write + chmod + rename via temp file), so **debounce** before reacting.

---

## 7. Executing External Commands (`os/exec`)

This is the section you came for. `os/exec` runs other programs: `git`, `go`, `npm`, `ffmpeg`, anything. It is **not** a shell — it executes a binary directly with an argument list, which is safer and more predictable.

### 7.1 The mental model

```go
import "os/exec"

// exec.Command builds a *Cmd. It does NOT run anything yet.
// First arg = program name (looked up on PATH). Rest = arguments, each as a
// SEPARATE string. There is NO shell, so no glob expansion, no quoting, no
// pipes, no "&&" — those are shell features (see §7.9).
cmd := exec.Command("git", "commit", "-m", "my message with spaces")
```

> **Key point:** Each argument is a distinct slice element. `exec.Command("git", "commit -m msg")` would pass the literal string `"commit -m msg"` as ONE argument to git — wrong. Split arguments yourself.

### 7.2 `Run`, `Output`, `CombinedOutput` — the common cases

```go
// (a) Run: execute and wait; you don't care about output (it goes nowhere
//     unless you wire cmd.Stdout/Stderr). Returns an error if exit code != 0.
cmd := exec.Command("go", "build", "./...")
if err := cmd.Run(); err != nil {
    log.Fatalf("build failed: %v", err)
}

// (b) Output: run and CAPTURE stdout as []byte. stderr is NOT captured (but
//     on failure it's attached to the *ExitError, see §7.7).
out, err := exec.Command("git", "rev-parse", "HEAD").Output()
if err != nil {
    log.Fatal(err)
}
fmt.Printf("commit: %s", out) // includes trailing newline

// (c) CombinedOutput: capture stdout AND stderr interleaved into one []byte.
//     Great for logging a tool's full output on failure.
out, err = exec.Command("go", "vet", "./...").CombinedOutput()
if err != nil {
    log.Printf("vet output:\n%s", out)
}
```

### 7.3 `Start` + `Wait` — run asynchronously

```go
cmd := exec.Command("long-running-task")
if err := cmd.Start(); err != nil { // returns immediately after the process starts
    log.Fatal(err)
}
fmt.Println("started PID", cmd.Process.Pid)

// Do other work here while it runs…

if err := cmd.Wait(); err != nil { // block until it exits, reap the process
    log.Fatal(err)
}
```

> `Run()` is literally `Start()` then `Wait()`. Use `Start`/`Wait` when you need the PID, want to do work concurrently, or want to send signals while it runs.

### 7.4 Wiring stdout/stderr to your own program

```go
// Stream the child's output straight to your terminal in real time.
cmd := exec.Command("npm", "install")
cmd.Stdout = os.Stdout // child stdout → your stdout (live)
cmd.Stderr = os.Stderr // child stderr → your stderr (live)
cmd.Stdin = os.Stdin   // forward your stdin to the child (for interactive tools)
if err := cmd.Run(); err != nil {
    log.Fatal(err)
}
```

Capture into a buffer instead:

```go
import "bytes"

var stdout, stderr bytes.Buffer
cmd := exec.Command("git", "status", "--porcelain")
cmd.Stdout = &stdout
cmd.Stderr = &stderr
err := cmd.Run()
fmt.Println("OUT:", stdout.String())
fmt.Println("ERR:", stderr.String())
```

Tee it (show live **and** capture) with `io.MultiWriter`:

```go
var captured bytes.Buffer
cmd := exec.Command("go", "test", "./...")
cmd.Stdout = io.MultiWriter(os.Stdout, &captured) // print AND save
cmd.Stderr = io.MultiWriter(os.Stderr, &captured)
cmd.Run()
```

### 7.5 Pipes — `StdinPipe`, `StdoutPipe`, streaming output line-by-line

```go
// Stream a command's output line-by-line AS IT RUNS (don't wait for it to finish).
func streamCommand(name string, args ...string) error {
    cmd := exec.Command(name, args...)

    // Get a pipe connected to the child's stdout.
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return err
    }
    cmd.Stderr = os.Stderr

    if err := cmd.Start(); err != nil { // start BEFORE reading the pipe
        return err
    }

    // Scan the pipe live. Each line prints the moment the child emits it.
    scanner := bufio.NewScanner(stdout)
    for scanner.Scan() {
        fmt.Println(">>", scanner.Text())
    }
    if err := scanner.Err(); err != nil {
        return err
    }

    // Wait AFTER the pipe is fully drained, or you can deadlock.
    return cmd.Wait()
}
```

> **Pipe ordering rules:** call `Start()` *before* reading `StdoutPipe`, and call `Wait()` only *after* you've finished reading the pipe (`Wait` closes the pipe). Do the reverse for `StdinPipe`: write to it, then **close it** so the child sees EOF, then `Wait`.

```go
// Feed data to a program's stdin (e.g. pipe text into `grep`).
cmd := exec.Command("grep", "error")
stdin, _ := cmd.StdinPipe()
cmd.Stdout = os.Stdout
cmd.Start()

io.WriteString(stdin, "info: ok\nerror: boom\ninfo: done\n")
stdin.Close() // IMPORTANT: close so grep sees end-of-input and exits

cmd.Wait()    // prints: error: boom
```

### 7.6 Environment & working directory

```go
cmd := exec.Command("node", "build.js")

// Cmd.Dir sets the working directory for the CHILD only.
// SET THIS instead of running `cd` — there is no `cd` to run (no shell), and
// changing your own process CWD with os.Chdir is global, racy, and affects
// every goroutine. Cmd.Dir is scoped, safe, and the correct tool.
cmd.Dir = "/path/to/project"

// Cmd.Env REPLACES the child's environment entirely. nil = inherit yours.
// To ADD to the inherited env, start from os.Environ():
cmd.Env = append(os.Environ(),
    "NODE_ENV=production",
    "CI=true",
)
cmd.Run()
```

> **Why `Cmd.Dir`, not `cd`:** `cd` is a shell builtin, so `exec.Command("cd", "x")` fails (there's no `cd` binary). Even with a shell, the directory change wouldn't persist. And `os.Chdir` mutates your whole process. `Cmd.Dir` is the only correct, isolated way to set a child's directory.

### 7.7 Checking exit codes — `*exec.ExitError`

```go
cmd := exec.Command("git", "diff", "--quiet") // exits 1 if there ARE changes
err := cmd.Run()
if err != nil {
    var exitErr *exec.ExitError
    if errors.As(err, &exitErr) {
        // The program ran but returned a non-zero exit code.
        code := exitErr.ExitCode()
        fmt.Printf("git exited with code %d\n", code)
        // exitErr.Stderr holds captured stderr — but ONLY when you used Output()
        // (not Run), and only if you didn't set cmd.Stderr yourself.
        fmt.Printf("stderr: %s\n", exitErr.Stderr)
    } else {
        // A DIFFERENT failure: binary not found, permission denied, etc.
        fmt.Printf("failed to run git: %v\n", err)
    }
}
```

### 7.8 Finding binaries — `exec.LookPath`

```go
// LookPath searches PATH for an executable and returns its full path.
// On Windows it also honours PATHEXT (.exe, .bat, .cmd, .com, .ps1…), so
// LookPath("git") may return "C:\Program Files\Git\cmd\git.exe".
path, err := exec.LookPath("git")
if err != nil {
    log.Fatal("git is not installed or not on PATH")
}
fmt.Println("found git at:", path)
```

⚡ **Version note (Go 1.19):** `os/exec` stopped resolving a program found in the **current directory** unless you opt in (security fix for "look up relative to CWD"). If you intend to run `./mytool`, pass an explicit relative path; a bare name like `"mytool"` only searches PATH. A program found only via `.` yields `exec.ErrDot`.

### 7.9 Running through a shell (and when NOT to)

`exec.Command` runs a binary directly — no globbing, pipes, `&&`, `~`, or env-var expansion. To use shell features, invoke the shell explicitly:

```go
// Unix shell:
exec.Command("sh", "-c", "ls -la | grep .go | wc -l")

// Windows command prompt:
exec.Command("cmd", "/C", "dir /b *.go")

// Windows PowerShell:
exec.Command("powershell", "-NoProfile", "-Command", "Get-ChildItem *.go | Measure-Object")
```

A portable helper:

```go
import "runtime"

func shell(command string) *exec.Cmd {
    if runtime.GOOS == "windows" {
        return exec.Command("cmd", "/C", command)
    }
    return exec.Command("sh", "-c", command)
}
```

> **🔒 SECURITY — command injection:** **Never** build a shell string from untrusted input. `exec.Command("sh", "-c", "git clone "+userURL)` lets an attacker pass `userURL = "x; rm -rf ~"`. The whole point of `exec.Command`'s argument list is that it does **not** interpret metacharacters — so prefer `exec.Command("git", "clone", userURL)` (direct, no shell) over any `sh -c` with concatenation. Reach for a shell only with **hardcoded** commands, never user data.

### 7.10 Timeouts & cancellation — `exec.CommandContext`

```go
import (
    "context"
    "time"
)

// CommandContext kills the process if the context is cancelled or times out.
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

cmd := exec.CommandContext(ctx, "go", "test", "-run", "Slow", "./...")
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr

err := cmd.Run()
if ctx.Err() == context.DeadlineExceeded {
    log.Fatal("tests timed out after 30s")
}
if err != nil {
    log.Fatal(err)
}
```

⚡ **Version note (Go 1.20+):** `Cmd.Cancel` and `Cmd.WaitDelay` let you customise cancellation. By default the context kills the process group with SIGKILL; set `cmd.Cancel = func() error { return cmd.Process.Signal(os.Interrupt) }` to send a gentler signal first, and `cmd.WaitDelay = 5 * time.Second` to force-kill if it doesn't exit in time.

### 7.11 Concrete recipes against real tools

```go
// --- go build / go get ---
exec.Command("go", "build", "-o", "bin/app", "./cmd/app").Run()
exec.Command("go", "get", "github.com/spf13/cobra@latest").Run()

// --- npm / pnpm install (Windows: npm is npm.cmd — see the gotcha below) ---
npm := exec.Command("npm", "install")
npm.Dir = "frontend"
npm.Stdout, npm.Stderr = os.Stdout, os.Stderr
npm.Run()

// --- pip install ---
exec.Command("pip", "install", "requests").Run()

// --- git: capture current branch ---
branch, _ := exec.Command("git", "rev-parse", "--abbrev-ref", "HEAD").Output()
fmt.Println("branch:", strings.TrimSpace(string(branch)))

// --- list directory portably ---
func listDir(dir string) ([]byte, error) {
    if runtime.GOOS == "windows" {
        return exec.Command("cmd", "/C", "dir", "/b", dir).Output()
    }
    return exec.Command("ls", "-la", dir).Output()
}

// --- make a directory (just use os.MkdirAll, but to demo exec) ---
exec.Command("mkdir", "-p", "a/b/c").Run() // Unix; on Windows prefer os.MkdirAll

// --- capture and PARSE JSON output from a subcommand ---
func goModInfo() (map[string]any, error) {
    out, err := exec.Command("go", "mod", "edit", "-json").Output()
    if err != nil {
        return nil, err
    }
    var mod map[string]any
    if err := json.Unmarshal(out, &mod); err != nil {
        return nil, err
    }
    return mod, nil
}
```

> **⚡ Windows gotcha — `.cmd`/`.bat` shims:** Many Node tools (`npm`, `npx`, `yarn`, `tsc`, `vite`) are installed as `npm.cmd`, not `npm.exe`. `exec.LookPath` resolves these via PATHEXT, so `exec.Command("npm", ...)` usually works. But if you pass an explicit path, include the extension (`npm.cmd`). Also: batch files are run by `cmd.exe`, which has its own quoting rules — Go's `exec` quotes arguments for the typical case, but unusual characters (`&`, `^`, `%`, quotes) in arguments to `.cmd` files have historically been a source of injection bugs. ⚡ **Version note (Go 1.23):** `os/exec` tightened `.bat`/`.cmd` argument escaping on Windows; very odd characters may now error rather than be silently mis-quoted. Validate inputs.

### 7.12 Chaining commands without a shell (real pipes)

You can connect two Go-spawned processes with an `io.Pipe`-style hookup — no shell needed:

```go
// Emulate `cat file | grep error` using two *Cmd connected via a pipe.
cat := exec.Command("cat", "log.txt")
grep := exec.Command("grep", "error")

// Wire cat's stdout to grep's stdin.
pipe, _ := cat.StdoutPipe()
grep.Stdin = pipe
grep.Stdout = os.Stdout

grep.Start() // start the consumer first
cat.Run()    // run the producer to completion (closes the pipe at EOF)
grep.Wait()  // wait for the consumer to finish
```

---

## 8. Environment Variables

### 8.1 Reading & writing

```go
// Getenv returns "" if the variable is unset — you can't distinguish
// "unset" from "set to empty string" with it.
home := os.Getenv("HOME") // on Windows you usually want USERPROFILE — see §9

// LookupEnv distinguishes unset (ok=false) from empty (ok=true, value="").
if val, ok := os.LookupEnv("DEBUG"); ok {
    fmt.Println("DEBUG is set to:", val)
} else {
    fmt.Println("DEBUG is not set")
}

// Setenv / Unsetenv mutate THIS process's environment (and are inherited by
// child processes you spawn afterwards). They do NOT affect your shell.
os.Setenv("LOG_LEVEL", "debug")
os.Unsetenv("LOG_LEVEL")

// Environ returns the whole environment as "KEY=value" strings.
for _, kv := range os.Environ() {
    fmt.Println(kv)
}
```

> **⚡ Windows note:** environment variable names are **case-insensitive** on Windows (`PATH`, `Path`, `path` are the same) but **case-sensitive** on Unix. Pick a canonical case and stick to it. Also, the "home" variable is `USERPROFILE` on Windows, not `HOME` — use `os.UserHomeDir()` (§9) instead of reading either directly.

### 8.2 Expansion — `os.ExpandEnv` and `os.Expand`

```go
// ExpandEnv replaces ${VAR} and $VAR using the current environment.
os.Setenv("USER", "alice")
fmt.Println(os.ExpandEnv("hello $USER, home is ${HOME}")) // "hello alice, home is /home/alice"

// Expand is the general form: you supply the lookup function. Useful for
// templating from a map or providing defaults.
vars := map[string]string{"NAME": "world"}
s := os.Expand("hi ${NAME}!", func(key string) string {
    return vars[key] // return "" for unknown keys
})
fmt.Println(s) // "hi world!"
```

### 8.3 Loading a `.env` file (manual, zero dependencies)

```go
import (
    "bufio"
    "os"
    "strings"
)

// loadDotenv parses KEY=VALUE lines and sets them in the environment.
// Does NOT override variables already set in the real environment (common .env
// convention: real env wins). Handles comments and simple quoting.
func loadDotenv(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()

    sc := bufio.NewScanner(f)
    for sc.Scan() {
        line := strings.TrimSpace(sc.Text())
        if line == "" || strings.HasPrefix(line, "#") {
            continue // skip blank lines and comments
        }
        key, val, ok := strings.Cut(line, "=") // Go 1.18+ strings.Cut
        if !ok {
            continue
        }
        key = strings.TrimSpace(key)
        val = strings.Trim(strings.TrimSpace(val), `"'`) // strip surrounding quotes
        if _, exists := os.LookupEnv(key); !exists {
            os.Setenv(key, val)
        }
    }
    return sc.Err()
}
```

For anything beyond the basics (multiline values, variable interpolation), the standard library plus the popular **`github.com/joho/godotenv`** package is the usual answer:

```go
// import "github.com/joho/godotenv"
// godotenv.Load()                 // loads ".env" into the environment
// godotenv.Load("config/.env")    // a specific file
// godotenv.Overload()             // override existing vars
```

---

## 9. Accessing System Information

### 9.1 Process & host identity

```go
hostname, _ := os.Hostname()       // the machine's network name
pid := os.Getpid()                 // this process's ID
ppid := os.Getppid()               // the parent process's ID
fmt.Printf("host=%s pid=%d ppid=%d\n", hostname, pid, ppid)
```

### 9.2 Standard user directories (cross-platform!)

⚡ **Version note:** these resolve the correct per-OS location. **Always use them instead of hand-building `$HOME/...`** — that's the #1 portability bug for config files.

```go
home, _ := os.UserHomeDir()    // Win: C:\Users\You   Unix: /home/you   macOS: /Users/you
cfg, _  := os.UserConfigDir()  // Win: %AppData%      Unix: ~/.config    macOS: ~/Library/Application Support
cache, _ := os.UserCacheDir()  // Win: %LocalAppData% Unix: ~/.cache     macOS: ~/Library/Caches
tmp := os.TempDir()            // Win: %TEMP%         Unix: /tmp

// Idiomatic: store your app's config under the OS config dir.
appCfgDir := filepath.Join(cfg, "myapp")
os.MkdirAll(appCfgDir, 0o755)
cfgFile := filepath.Join(appCfgDir, "config.json")
```

### 9.3 Working directory

```go
wd, _ := os.Getwd()   // current working directory (absolute)

// os.Chdir changes the process CWD — GLOBAL and racy. Prefer absolute paths or
// Cmd.Dir (§7.6). Use Chdir sparingly, e.g. in a CLI that intentionally `cd`s.
os.Chdir("/some/dir")
```

### 9.4 Runtime info — `runtime` package

```go
import "runtime"

fmt.Println("OS:        ", runtime.GOOS)        // "windows", "linux", "darwin"
fmt.Println("Arch:      ", runtime.GOARCH)      // "amd64", "arm64"
fmt.Println("CPUs:      ", runtime.NumCPU())    // logical CPU count
fmt.Println("Goroutines:", runtime.NumGoroutine())
fmt.Println("Go version:", runtime.Version())   // e.g. "go1.24.0"

// Memory statistics:
var m runtime.MemStats
runtime.ReadMemStats(&m)
fmt.Printf("HeapAlloc: %d KB\n", m.HeapAlloc/1024) // currently allocated heap
fmt.Printf("Sys:       %d KB\n", m.Sys/1024)       // total from the OS
fmt.Printf("NumGC:     %d\n", m.NumGC)             // completed GC cycles
```

> **`runtime.GOOS` is a compile-time constant.** The compiler can eliminate dead branches, so `if runtime.GOOS == "windows" { ... }` is both idiomatic and zero-cost — a good lightweight alternative to build tags for small differences.

### 9.5 User & group info — `os/user`

```go
import "os/user"

u, err := user.Current()
if err == nil {
    fmt.Println("username:", u.Username) // "DOMAIN\\you" on Windows
    fmt.Println("name:    ", u.Name)     // display name
    fmt.Println("uid:     ", u.Uid)      // SID string on Windows, number on Unix
    fmt.Println("home:    ", u.HomeDir)
}

// Look up by name or id:
other, _ := user.Lookup("alice")
byID, _ := user.LookupId("1000")

// Groups (Unix-centric):
grp, _ := user.LookupGroup("staff")
_ = grp
```

> **⚡ Note:** `os/user` may need cgo on some platforms for full lookups; pure-Go builds (`CGO_ENABLED=0`) fall back to parsing `/etc/passwd` on Unix, which can miss LDAP/AD users. On Windows it uses the Win32 API. `user.Current()` is reliable everywhere.

### 9.6 `os.Args`, stdin/stdout/stderr

```go
// os.Args[0] is the program path; os.Args[1:] are the arguments.
fmt.Println("program:", os.Args[0])
fmt.Println("args:   ", os.Args[1:])

// The three standard streams are *os.File values you can read/write directly.
fmt.Fprintln(os.Stdout, "to stdout")
fmt.Fprintln(os.Stderr, "to stderr (errors/logs go here so stdout stays clean)")
input, _ := io.ReadAll(os.Stdin) // read everything piped in
_ = input
```

### 9.7 Disk & detailed memory metrics — `gopsutil`

The standard library does **not** expose disk free space, per-process CPU%, network counters, or system-wide memory portably. The community standard is **`github.com/shirou/gopsutil/v4`**:

```go
// import (
//     "github.com/shirou/gopsutil/v4/cpu"
//     "github.com/shirou/gopsutil/v4/disk"
//     "github.com/shirou/gopsutil/v4/mem"
// )

// v, _ := mem.VirtualMemory()
// fmt.Printf("RAM: %d MB used of %d MB (%.1f%%)\n",
//     v.Used/1024/1024, v.Total/1024/1024, v.UsedPercent)

// d, _ := disk.Usage("/")            // on Windows: disk.Usage("C:\\")
// fmt.Printf("Disk free: %d GB\n", d.Free/1024/1024/1024)

// pct, _ := cpu.Percent(time.Second, false)
// fmt.Printf("CPU: %.1f%%\n", pct[0])
```

`gopsutil` abstracts `/proc` (Linux), sysctl (macOS), and WMI/PDH (Windows) so you don't have to write per-OS code for system metrics.

---

## 10. Signals & Process Control

### 10.1 Catching signals — `os/signal`

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    // Notify delivers matching signals to a channel instead of the default
    // action (which for SIGINT is "terminate"). Buffer of 1 avoids missing a
    // signal that arrives before we're in the receive.
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

    fmt.Println("running… press Ctrl+C to stop")
    sig := <-sigs
    fmt.Println("\nreceived:", sig, "— cleaning up")
}
```

**Cross-platform signal reality:**

| Signal | Unix | Windows |
|---|---|---|
| `os.Interrupt` / `SIGINT` | Ctrl+C | Ctrl+C (supported) |
| `SIGTERM` | graceful kill | Defined but **not delivered** by the OS |
| `SIGHUP` | terminal closed / reload | Not supported |
| `SIGKILL` / `SIGSTOP` | cannot be caught | n/a |

> On Windows, only `os.Interrupt` (Ctrl+C) is genuinely deliverable. Code that relies on `SIGTERM`/`SIGHUP` is Unix-specific — keep portable code to `os.Interrupt`.

### 10.2 `signal.NotifyContext` — the modern, idiomatic pattern

⚡ **Version note (Go 1.16+):** prefer `signal.NotifyContext` over a raw channel. It returns a context that's cancelled on the first matching signal — integrating cleanly with everything else that takes a `context.Context`.

```go
import (
    "context"
    "os/signal"
    "syscall"
)

func main() {
    // ctx is cancelled when SIGINT or SIGTERM arrives.
    ctx, stop := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer stop() // release signal handling resources

    // Do work that respects ctx…
    if err := run(ctx); err != nil {
        log.Fatal(err)
    }
}
```

### 10.3 Graceful shutdown of a worker loop

```go
func run(ctx context.Context) error {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            // Signal received. Finish current work, flush, release resources.
            fmt.Println("shutting down gracefully…")
            return nil
        case <-ticker.C:
            fmt.Println("doing work…")
        }
    }
}
```

> A second Ctrl+C: after `signal.NotifyContext` fires once, the handler is still installed until `stop()`. To make a second Ctrl+C force-quit immediately, call `stop()` as soon as the first signal arrives — this restores default behaviour for the next signal.

### 10.4 Sending signals & killing processes

```go
// Find a process by PID (on Unix this never errors; the process may not exist —
// signaling reveals that). On Windows it opens a real handle.
p, err := os.FindProcess(pid)
if err != nil {
    log.Fatal(err)
}

p.Signal(os.Interrupt) // ask politely (Unix: SIGINT)
p.Signal(syscall.SIGTERM) // Unix graceful terminate
p.Kill()                  // forceful (SIGKILL on Unix, TerminateProcess on Windows)

// For a child you spawned with exec, signal via cmd.Process:
cmd := exec.Command("server")
cmd.Start()
// …later…
cmd.Process.Signal(os.Interrupt)
```

### 10.5 `os.Exit` and the deferred-cleanup trap

```go
// ⚠️ os.Exit terminates IMMEDIATELY. Deferred functions DO NOT run, buffered
// writers are NOT flushed, and other goroutines are abandoned.
func bad() {
    f, _ := os.Create("out.txt")
    defer f.Close()        // ❌ NEVER RUNS
    w := bufio.NewWriter(f)
    w.WriteString("data")  // ❌ buffer never flushed → file is empty
    os.Exit(1)             // deferred Close + Flush skipped
}

// Pattern: keep main thin; do work in a function that RETURNS an error, and
// only os.Exit at the very top after deferreds have run.
func main() {
    if err := realMain(); err != nil {
        fmt.Fprintln(os.Stderr, "error:", err)
        os.Exit(1) // safe: all of realMain's defers already ran on return
    }
}

func realMain() error {
    f, err := os.Create("out.txt")
    if err != nil {
        return err
    }
    defer f.Close() // ✅ runs because we return, not Exit
    _, err = f.WriteString("data")
    return err
}
```

> `log.Fatal` calls `os.Exit(1)` internally — so it has the **same** "defers don't run" problem. Use it only at the top level of `main`, never deep in helpers.

---

## 11. Building CLIs

We'll go bottom-up: the standard `flag` package (zero dependencies), then **Cobra** (the industry standard for big CLIs like `kubectl`, `gh`, `hugo`), then alternatives (**urfave/cli v3**), config (**Viper**), and interactive UX (**charm**, **survey**, **pterm**).

### 11.1 The standard `flag` package

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // Each flag.* call defines a flag and returns a POINTER to where the parsed
    // value will be stored. Args: name, default, usage text.
    name := flag.String("name", "world", "who to greet")
    count := flag.Int("count", 1, "how many times")
    verbose := flag.Bool("verbose", false, "enable verbose output")

    // flag.Parse() reads os.Args[1:], fills the pointers, and stops at the first
    // non-flag argument. CALL IT before reading any flag value.
    flag.Parse()

    if *verbose {
        fmt.Println("verbose mode on")
    }
    for i := 0; i < *count; i++ {
        fmt.Printf("Hello, %s!\n", *name)
    }

    // Positional (non-flag) arguments left after the flags:
    fmt.Println("positional args:", flag.Args())
}
```

```bash
go run . -name Alice -count 3 -verbose extra1 extra2
# Flags accept -name, --name, -name=Alice, --name=Alice (single & double dash both work).
# Boolean flags: -verbose or -verbose=true. Note: -verbose false does NOT work for bools
# (the "false" would be a positional arg); use -verbose=false.
```

#### Binding to existing variables with `flag.XxxVar`

```go
var cfg struct {
    Host string
    Port int
}
flag.StringVar(&cfg.Host, "host", "localhost", "server host")
flag.IntVar(&cfg.Port, "port", 8080, "server port")
flag.DurationVar(&timeout, "timeout", 30*time.Second, "request timeout") // accepts "5s","2m"
flag.Parse()
```

#### Custom usage and required flags

```go
flag.Usage = func() {
    fmt.Fprintf(os.Stderr, "Usage: %s [options] <file>\n\n", os.Args[0])
    fmt.Fprintln(os.Stderr, "Options:")
    flag.PrintDefaults() // auto-generated from your flag definitions
}

flag.Parse()

// flag has NO "required" concept — enforce it yourself.
if flag.NArg() < 1 {
    flag.Usage()
    os.Exit(2) // exit code 2 is the conventional "usage error"
}
```

#### Subcommands with `flag.NewFlagSet`

The stdlib has no subcommand system, but you can build one with separate `FlagSet`s:

```go
package main

import (
    "flag"
    "fmt"
    "os"
)

func main() {
    // Define one FlagSet per subcommand.
    addCmd := flag.NewFlagSet("add", flag.ExitOnError)
    addName := addCmd.String("name", "", "item name")

    listCmd := flag.NewFlagSet("list", flag.ExitOnError)
    listAll := listCmd.Bool("all", false, "show all items")

    if len(os.Args) < 2 {
        fmt.Println("expected 'add' or 'list' subcommand")
        os.Exit(2)
    }

    // Dispatch on os.Args[1]; parse the REMAINING args with that subcommand's set.
    switch os.Args[1] {
    case "add":
        addCmd.Parse(os.Args[2:])
        fmt.Println("adding:", *addName, "positional:", addCmd.Args())
    case "list":
        listCmd.Parse(os.Args[2:])
        fmt.Println("listing, all =", *listAll)
    default:
        fmt.Printf("unknown subcommand %q\n", os.Args[1])
        os.Exit(2)
    }
}
```

```bash
go run . add -name widget
go run . list -all
```

**`flag` error-handling modes:** `flag.ExitOnError` (print + `os.Exit(2)` on parse error — usual), `flag.ContinueOnError` (return the error so you can handle it), `flag.PanicOnError`.

**When `flag` is enough:** single-purpose tools, internal scripts, no nested subcommands. When you need a deep command tree, auto-generated help, shell completion, and persistent flags — reach for Cobra.

### 11.2 Cobra — the production CLI framework

`github.com/spf13/cobra` powers `kubectl`, `hugo`, `gh`, `docker` (partly), and most large Go CLIs. Core concept: a tree of `*cobra.Command`s, each with its own flags, args, and `Run` function.

```bash
go get github.com/spf13/cobra@latest
# Optional scaffolding generator:
go install github.com/spf13/cobra-cli@latest
```

#### Minimal Cobra app

```go
package main

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

func main() {
    // The root command: the program itself.
    rootCmd := &cobra.Command{
        Use:   "scaffold",                       // how it's invoked
        Short: "A project scaffolding tool",     // one-line help
        Long:  "scaffold creates project skeletons from templates.",
        // Run executes when you call `scaffold` with no subcommand.
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Println("run a subcommand — try `scaffold --help`")
        },
    }

    if err := rootCmd.Execute(); err != nil {
        // Execute prints the error & usage already; just set the exit code.
        os.Exit(1)
    }
}
```

#### Adding subcommands, flags, and arg validation

```go
func main() {
    rootCmd := &cobra.Command{Use: "scaffold", Short: "Project scaffolding tool"}

    // --- persistent flag: available to THIS command AND all its children ---
    var verbose bool
    rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")

    // --- `scaffold new <name>` ---
    var template string
    newCmd := &cobra.Command{
        Use:   "new <name>",
        Short: "Create a new project",
        // Args validators run before Run. cobra.ExactArgs(1) requires exactly one
        // positional arg; others: NoArgs, MinimumNArgs(n), MaximumNArgs(n),
        // RangeArgs(min,max), OnlyValidArgs, or a custom func.
        Args: cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error { // RunE returns an error
            name := args[0]
            if verbose {
                fmt.Printf("using template %q\n", template)
            }
            fmt.Printf("creating project %q…\n", name)
            return os.MkdirAll(name, 0o755)
        },
    }
    // --- local flag: only this command ---
    newCmd.Flags().StringVarP(&template, "template", "t", "default", "template to use")
    // Mark it required:
    // newCmd.MarkFlagRequired("template")

    // --- `scaffold list` ---
    listCmd := &cobra.Command{
        Use:   "list",
        Short: "List available templates",
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Println("default\nweb\ncli\nlibrary")
        },
    }

    // Build the command tree.
    rootCmd.AddCommand(newCmd, listCmd)

    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

```bash
scaffold new myapp --template web -v
scaffold list
scaffold --help          # auto-generated, recursive help
scaffold new --help      # per-command help
```

**Cobra concepts cheat sheet:**

| Concept | API | Notes |
|---|---|---|
| Command | `&cobra.Command{Use, Short, Long, Run/RunE}` | `RunE` lets you return errors |
| Subcommand | `parent.AddCommand(child)` | Builds the tree |
| Local flag | `cmd.Flags().StringVarP(...)` | Only that command |
| Persistent flag | `cmd.PersistentFlags()...` | That command + all children |
| Shorthand | the `P` variants (`BoolVarP`) take a 1-char alias | `-v` for `--verbose` |
| Required flag | `cmd.MarkFlagRequired("name")` | Errors if missing |
| Arg validation | `Args: cobra.ExactArgs(1)` | Pre-run check |
| Lifecycle hooks | `PreRun`, `PersistentPreRun`, `PostRun` | Run around `Run` |
| Completion | built-in `completion` subcommand | bash/zsh/fish/powershell |

#### Lifecycle hooks (e.g. init logging once)

```go
rootCmd.PersistentPreRunE = func(cmd *cobra.Command, args []string) error {
    // Runs before EVERY command. Good place to configure logging, load config,
    // open a DB, etc., based on the now-parsed persistent flags.
    return nil
}
```

### 11.3 urfave/cli v3 — the lighter alternative

⚡ **Version note:** `github.com/urfave/cli/v3` is the current major version (v3 reworked the API and added native context support). It's a popular, less "frameworky" alternative to Cobra — declarative command/flag structs, no code-gen.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/urfave/cli/v3"
)

func main() {
    cmd := &cli.Command{
        Name:  "greet",
        Usage: "say hello",
        Flags: []cli.Flag{
            &cli.StringFlag{
                Name:    "name",
                Aliases: []string{"n"},
                Value:   "world",
                Usage:   "who to greet",
            },
        },
        Commands: []*cli.Command{
            {
                Name:  "shout",
                Usage: "greet loudly",
                Action: func(ctx context.Context, c *cli.Command) error {
                    fmt.Printf("HELLO, %s!\n", c.String("name"))
                    return nil
                },
            },
        },
        Action: func(ctx context.Context, c *cli.Command) error {
            fmt.Printf("Hello, %s!\n", c.String("name"))
            return nil
        },
    }

    // v3 takes a context — wire signal handling straight in (§10.2).
    if err := cmd.Run(context.Background(), os.Args); err != nil {
        log.Fatal(err)
    }
}
```

**Cobra vs urfave/cli:** Cobra has the bigger ecosystem (Viper integration, generators, completion), heavier boilerplate, struct-then-AddCommand wiring. urfave/cli is more declarative and lighter, with everything in nested structs. Both are solid; Cobra if you want the kubectl-style experience, urfave for smaller tools.

### 11.4 Viper — configuration management

`github.com/spf13/viper` (same author as Cobra) merges config from **flags, env vars, config files (JSON/TOML/YAML), and defaults** with a clear precedence order, and integrates with Cobra's pflag.

```go
import "github.com/spf13/viper"

func initConfig() {
    viper.SetConfigName("config")          // config.yaml / config.toml / …
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")               // look in CWD
    home, _ := os.UserHomeDir()
    viper.AddConfigPath(filepath.Join(home, ".myapp")) // and in ~/.myapp

    viper.SetDefault("port", 8080)         // fallback if nothing else sets it

    // Read env vars: viper.AutomaticEnv() maps PORT, DATABASE_URL, etc.
    viper.SetEnvPrefix("MYAPP")            // only MYAPP_* are considered
    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err != nil {
        // Not fatal if the file is simply absent.
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            log.Fatal(err)
        }
    }
}

// Read values (precedence: explicit Set > flag > env > config file > default):
port := viper.GetInt("port")
dbURL := viper.GetString("database_url")
```

Bind a Cobra flag to Viper so `--port` overrides the file:

```go
rootCmd.PersistentFlags().Int("port", 8080, "server port")
viper.BindPFlag("port", rootCmd.PersistentFlags().Lookup("port"))
```

### 11.5 Interactive UX — colors, spinners, prompts, TUIs

For polished CLIs, the **charm** ecosystem dominates in 2026:

| Library | Use |
|---|---|
| `github.com/charmbracelet/lipgloss` | Style/layout terminal text (colors, borders, alignment) |
| `github.com/charmbracelet/bubbletea` | Full TUI framework (Elm-style model/update/view) |
| `github.com/charmbracelet/bubbles` | Pre-built TUI components (text input, list, spinner, progress) |
| `github.com/charmbracelet/huh` | Interactive forms/prompts (the modern survey replacement) |
| `github.com/AlecAivazis/survey/v2` | Classic interactive prompts (select, confirm, input) |
| `github.com/pterm/pterm` | Spinners, progress bars, tables, colored output — batteries included |
| `github.com/fatih/color` | Simple ANSI colors |
| `github.com/schollz/progressbar/v3` | Progress bars |

```go
// lipgloss styling example:
// import "github.com/charmbracelet/lipgloss"
// style := lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("#04B575"))
// fmt.Println(style.Render("Success!"))

// huh form example (interactive prompt):
// import "github.com/charmbracelet/huh"
// var name string
// huh.NewInput().Title("Project name?").Value(&name).Run()

// pterm spinner:
// import "github.com/pterm/pterm"
// spinner, _ := pterm.DefaultSpinner.Start("Installing…")
// time.Sleep(2 * time.Second)
// spinner.Success("Done!")
```

> **Color gotcha:** detect whether output is a terminal before emitting ANSI codes — piping `mytool | grep x` should not get raw escape sequences. See §12.2 for terminal detection; most of these libraries auto-detect and respect the `NO_COLOR` environment variable.

### 11.6 A complete `flag`-based CLI (file manager)

```go
// fm: a tiny cross-platform file manager. Subcommands: ls, cp, rm, mkdir.
// Build: go build -o fm .   Run: ./fm ls .
package main

import (
    "flag"
    "fmt"
    "io"
    "os"
    "path/filepath"
)

func main() {
    if len(os.Args) < 2 {
        usage()
        os.Exit(2)
    }
    var err error
    switch os.Args[1] {
    case "ls":
        err = cmdLs(os.Args[2:])
    case "cp":
        err = cmdCp(os.Args[2:])
    case "rm":
        err = cmdRm(os.Args[2:])
    case "mkdir":
        err = cmdMkdir(os.Args[2:])
    default:
        usage()
        os.Exit(2)
    }
    if err != nil {
        fmt.Fprintln(os.Stderr, "error:", err)
        os.Exit(1)
    }
}

func usage() {
    fmt.Fprintln(os.Stderr, "usage: fm <ls|cp|rm|mkdir> [args]")
}

func cmdLs(args []string) error {
    fs := flag.NewFlagSet("ls", flag.ExitOnError)
    long := fs.Bool("l", false, "long format (show size)")
    fs.Parse(args)

    dir := "."
    if fs.NArg() > 0 {
        dir = fs.Arg(0)
    }
    entries, err := os.ReadDir(dir)
    if err != nil {
        return err
    }
    for _, e := range entries {
        if *long {
            info, _ := e.Info()
            fmt.Printf("%10d  %s\n", info.Size(), e.Name())
        } else {
            fmt.Println(e.Name())
        }
    }
    return nil
}

func cmdCp(args []string) error {
    fs := flag.NewFlagSet("cp", flag.ExitOnError)
    fs.Parse(args)
    if fs.NArg() != 2 {
        return fmt.Errorf("usage: fm cp <src> <dst>")
    }
    src, dst := fs.Arg(0), fs.Arg(1)

    in, err := os.Open(src)
    if err != nil {
        return err
    }
    defer in.Close()
    out, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer out.Close()
    if _, err := io.Copy(out, in); err != nil {
        return err
    }
    return out.Close()
}

func cmdRm(args []string) error {
    fs := flag.NewFlagSet("rm", flag.ExitOnError)
    recursive := fs.Bool("r", false, "remove directories recursively")
    fs.Parse(args)
    if fs.NArg() < 1 {
        return fmt.Errorf("usage: fm rm [-r] <path>...")
    }
    for _, p := range fs.Args() {
        var err error
        if *recursive {
            err = os.RemoveAll(p)
        } else {
            err = os.Remove(p)
        }
        if err != nil {
            return err
        }
    }
    return nil
}

func cmdMkdir(args []string) error {
    fs := flag.NewFlagSet("mkdir", flag.ExitOnError)
    parents := fs.Bool("p", false, "create parent directories")
    fs.Parse(args)
    if fs.NArg() < 1 {
        return fmt.Errorf("usage: fm mkdir [-p] <dir>")
    }
    for _, d := range fs.Args() {
        d = filepath.Clean(d)
        var err error
        if *parents {
            err = os.MkdirAll(d, 0o755)
        } else {
            err = os.Mkdir(d, 0o755)
        }
        if err != nil {
            return err
        }
    }
    return nil
}
```

### 11.7 The same tool in Cobra (project scaffolder)

```go
// scaffold: create a project skeleton. Demonstrates Cobra command tree,
// persistent + local flags, RunE error handling, and embedded templates.
package main

import (
    "embed"
    "fmt"
    "io/fs"
    "os"
    "path/filepath"

    "github.com/spf13/cobra"
)

//go:embed templates
var templates embed.FS

func main() {
    var verbose bool

    root := &cobra.Command{
        Use:   "scaffold",
        Short: "Scaffold new projects from embedded templates",
    }
    root.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")

    var lang string
    newCmd := &cobra.Command{
        Use:   "new <name>",
        Short: "Create a new project directory from a template",
        Args:  cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            name := args[0]
            srcDir := filepath.ToSlash(filepath.Join("templates", lang)) // embed = '/'
            // Confirm the template exists in the embedded FS.
            if _, err := fs.Stat(templates, srcDir); err != nil {
                return fmt.Errorf("unknown template %q", lang)
            }
            // Copy every embedded file under templates/<lang> into ./<name>.
            return fs.WalkDir(templates, srcDir, func(p string, d fs.DirEntry, err error) error {
                if err != nil {
                    return err
                }
                rel, _ := filepath.Rel(srcDir, p) // p uses '/', Rel handles it
                target := filepath.Join(name, filepath.FromSlash(rel))
                if d.IsDir() {
                    return os.MkdirAll(target, 0o755)
                }
                data, err := templates.ReadFile(p)
                if err != nil {
                    return err
                }
                if verbose {
                    fmt.Println("writing", target)
                }
                return os.WriteFile(target, data, 0o644)
            })
        },
    }
    newCmd.Flags().StringVarP(&lang, "lang", "l", "go", "template language (go|node|python)")

    root.AddCommand(newCmd)

    if err := root.Execute(); err != nil {
        os.Exit(1)
    }
}
```

---

## 12. Reading stdin & Interactive Input

### 12.1 Reading piped input vs prompting

```go
// Read EVERYTHING from stdin (e.g. `echo data | mytool`).
all, err := io.ReadAll(os.Stdin)

// Read line-by-line from stdin.
sc := bufio.NewScanner(os.Stdin)
for sc.Scan() {
    line := sc.Text()
    fmt.Println("got:", line)
}

// Prompt for a single line of input (interactive).
func prompt(question string) (string, error) {
    fmt.Print(question + " ")
    reader := bufio.NewReader(os.Stdin)
    line, err := reader.ReadString('\n')
    return strings.TrimSpace(line), err // trim the trailing newline (and \r on Windows)
}
```

> **⚡ Windows note:** lines typed at a Windows console end in `\r\n`. `strings.TrimSpace` removes both; if you only `TrimSuffix(line, "\n")` you'll leave a stray `\r`. Always `TrimSpace` or `TrimRight(line, "\r\n")`.

### 12.2 Detecting whether stdin/stdout is a terminal

A well-behaved CLI behaves differently when piped (no colors, no prompts) than when run interactively. The portable way is to `Stat` the stream and check the `ModeCharDevice` bit.

```go
// isTerminal reports whether f is connected to an interactive terminal
// (as opposed to a pipe, file, or /dev/null).
func isTerminal(f *os.File) bool {
    info, err := f.Stat()
    if err != nil {
        return false
    }
    // A terminal is a character device. A pipe/file is not.
    return info.Mode()&os.ModeCharDevice != 0
}

func main() {
    if isTerminal(os.Stdin) {
        fmt.Println("interactive — I can prompt the user")
    } else {
        fmt.Println("piped — reading data from stdin")
        data, _ := io.ReadAll(os.Stdin)
        _ = data
    }

    if !isTerminal(os.Stdout) {
        // Output is redirected/piped → disable ANSI colors.
    }
}
```

> For richer needs (raw mode, terminal size, password input without echo) use **`golang.org/x/term`**: `term.IsTerminal(int(os.Stdin.Fd()))`, `term.GetSize(fd)`, and `term.ReadPassword(fd)` for hidden input. It works on Windows too.

```go
// Read a password without echoing it (cross-platform via x/term).
// import "golang.org/x/term"
// fmt.Print("Password: ")
// pw, _ := term.ReadPassword(int(os.Stdin.Fd()))
// fmt.Println() // newline after the hidden input
```

---

## 13. Archives & Compression

### 13.1 gzip a single stream — `compress/gzip`

```go
import "compress/gzip"

// Compress a file to file.gz.
func gzipFile(src, dst string) error {
    in, err := os.Open(src)
    if err != nil {
        return err
    }
    defer in.Close()

    out, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer out.Close()

    // gzip.Writer wraps any io.Writer. Everything written to it is compressed.
    gz := gzip.NewWriter(out)
    gz.Name = filepath.Base(src) // optional metadata stored in the gzip header
    if _, err := io.Copy(gz, in); err != nil {
        return err
    }
    return gz.Close() // CRITICAL: Close flushes the gzip footer/checksum
}

// Decompress file.gz back to a file.
func gunzipFile(src, dst string) error {
    in, err := os.Open(src)
    if err != nil {
        return err
    }
    defer in.Close()

    gz, err := gzip.NewReader(in)
    if err != nil {
        return err
    }
    defer gz.Close()

    out, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer out.Close()

    _, err = io.Copy(out, gz)
    return err
}
```

> gzip compresses a **single stream**, not multiple files. For "many files in one archive" you need `tar` (which bundles) often combined with `gzip` (which compresses) — the classic `.tar.gz`.

### 13.2 Create a zip archive — `archive/zip`

```go
import "archive/zip"

// zipDir walks `srcDir` and writes all files into a .zip at `zipPath`.
func zipDir(srcDir, zipPath string) error {
    zf, err := os.Create(zipPath)
    if err != nil {
        return err
    }
    defer zf.Close()

    zw := zip.NewWriter(zf)
    defer zw.Close() // Close writes the central directory — don't skip it

    return filepath.WalkDir(srcDir, func(path string, d os.DirEntry, err error) error {
        if err != nil {
            return err
        }
        if d.IsDir() {
            return nil // zip stores files; directories are implied by paths
        }
        // The name inside the zip must be slash-separated and relative.
        rel, err := filepath.Rel(srcDir, path)
        if err != nil {
            return err
        }
        rel = filepath.ToSlash(rel) // IMPORTANT on Windows: zip uses '/'

        w, err := zw.Create(rel)
        if err != nil {
            return err
        }
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close()
        _, err = io.Copy(w, f)
        return err
    })
}
```

### 13.3 Extract a zip safely (defeating "Zip Slip")

A malicious archive can contain entries like `../../etc/passwd`. Validate every entry before writing — or use `os.Root` (§4.4) so escapes are impossible.

```go
// unzip extracts archive `zipPath` into `destDir`, rejecting path traversal.
func unzip(zipPath, destDir string) error {
    r, err := zip.OpenReader(zipPath) // *zip.ReadCloser also implements fs.FS
    if err != nil {
        return err
    }
    defer r.Close()

    // Resolve destDir to an absolute, cleaned path for the prefix check.
    destAbs, err := filepath.Abs(destDir)
    if err != nil {
        return err
    }

    for _, f := range r.File {
        // SAFETY: build the target and confirm it stays under destDir.
        target := filepath.Join(destAbs, f.Name)
        if !strings.HasPrefix(target, destAbs+string(os.PathSeparator)) && target != destAbs {
            return fmt.Errorf("zip slip detected: %q escapes destination", f.Name)
        }

        if f.FileInfo().IsDir() {
            if err := os.MkdirAll(target, f.Mode()); err != nil {
                return err
            }
            continue
        }

        // Ensure the parent directory exists.
        if err := os.MkdirAll(filepath.Dir(target), 0o755); err != nil {
            return err
        }

        rc, err := f.Open()
        if err != nil {
            return err
        }
        out, err := os.OpenFile(target, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, f.Mode())
        if err != nil {
            rc.Close()
            return err
        }
        // Limit decompressed size to defend against zip bombs.
        if _, err := io.Copy(out, io.LimitReader(rc, 1<<30)); err != nil {
            out.Close()
            rc.Close()
            return err
        }
        out.Close()
        rc.Close()
    }
    return nil
}
```

> **Even simpler & safer (Go 1.24):** open the destination with `os.OpenRoot(destDir)` and use `root.OpenFile`/`root.Mkdir` for every entry. Traversal becomes structurally impossible — no manual prefix check needed.

### 13.4 tar (and tar.gz) — `archive/tar`

```go
import (
    "archive/tar"
    "compress/gzip"
)

// tarGz creates a .tar.gz of a directory tree.
func tarGz(srcDir, outPath string) error {
    out, err := os.Create(outPath)
    if err != nil {
        return err
    }
    defer out.Close()

    // Layer the writers: tar → gzip → file. Order of Close matters (reverse).
    gz := gzip.NewWriter(out)
    defer gz.Close()
    tw := tar.NewWriter(gz)
    defer tw.Close()

    return filepath.WalkDir(srcDir, func(path string, d os.DirEntry, err error) error {
        if err != nil {
            return err
        }
        info, err := d.Info()
        if err != nil {
            return err
        }
        // Build a tar header from the FileInfo.
        hdr, err := tar.FileInfoHeader(info, "") // 2nd arg: symlink target if any
        if err != nil {
            return err
        }
        rel, _ := filepath.Rel(srcDir, path)
        hdr.Name = filepath.ToSlash(rel) // tar uses '/' separators

        if err := tw.WriteHeader(hdr); err != nil {
            return err
        }
        if d.IsDir() {
            return nil // directories have a header but no body
        }
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close()
        _, err = io.Copy(tw, f)
        return err
    })
}
```

Extraction mirrors the zip case: read headers in a loop with `tw := tar.NewReader(gz)`, `for { hdr, err := tr.Next(); ... }`, applying the same path-traversal guard (or `os.Root`).

---

## 14. Cross-Platform Gotchas, Build Tags & Cross-Compilation

### 14.1 The portability checklist (especially for Windows devs)

| Concern | Windows | Unix | Portable approach |
|---|---|---|---|
| Path separator | `\` | `/` | `filepath.Join`, never string concat |
| Path list separator | `;` | `:` | `os.PathListSeparator`, `filepath.SplitList` |
| Line endings | `\r\n` | `\n` | `strings.TrimSpace`; write `\n` and let tools cope |
| Home dir env | `USERPROFILE` | `HOME` | `os.UserHomeDir()` |
| Executable suffix | `.exe` | none | `exec.LookPath`; add `.exe` when naming output |
| Script shims | `npm.cmd`, `.bat` | shebang scripts | `exec.LookPath` (honours PATHEXT) |
| Permissions | NTFS ACLs | Unix mode bits | Don't rely on `chmod` semantics |
| Symlinks | need admin/Dev Mode | normal | guard creation, handle failure |
| Case sensitivity | insensitive | sensitive | normalise names; don't rely on either |
| Signals | only Ctrl+C | full set | use `os.Interrupt` for portable code |
| Shell | `cmd /C`, `powershell` | `sh -c` | branch on `runtime.GOOS` |

```go
// Portable PATH splitting:
for _, dir := range filepath.SplitList(os.Getenv("PATH")) {
    fmt.Println(dir)
}

// Name an output binary correctly per OS:
binName := "myapp"
if runtime.GOOS == "windows" {
    binName += ".exe"
}
```

### 14.2 Build tags — compile platform-specific files

Two ways to make code OS/arch-specific:

**(a) Filename suffix convention** — the Go toolchain auto-selects by name:

```
storage_windows.go   // compiled ONLY on Windows
storage_linux.go     // compiled ONLY on Linux
storage_darwin.go    // compiled ONLY on macOS
storage_unix.go      // requires a //go:build unix tag (see below)
helper_amd64.go      // arch-specific
```

**(b) `//go:build` constraints** at the top of a file (Go 1.17+ syntax):

```go
//go:build linux || darwin

package storage

// This file compiles only on Linux or macOS. The build line MUST be the first
// line (before package), followed by a blank line.

import "syscall"

func diskFree(path string) (uint64, error) {
    var st syscall.Statfs_t
    if err := syscall.Statfs(path, &st); err != nil {
        return 0, err
    }
    return st.Bavail * uint64(st.Bsize), nil
}
```

```go
//go:build windows

package storage

// A Windows implementation of the SAME function signature lives here, so the
// rest of the program calls diskFree() without caring about the OS.
func diskFree(path string) (uint64, error) {
    // ... Windows API call (GetDiskFreeSpaceEx) ...
    return 0, nil
}
```

⚡ **Version note:** the modern syntax is `//go:build linux` (Go 1.17+). The legacy `// +build linux` form still works but is being phased out; `gofmt` keeps them in sync. Common pseudo-tags: `unix` (any Unix-like), `cgo`, `ignore`.

### 14.3 Cross-compilation — build for other OSes from Windows

Go's killer feature: set `GOOS`/`GOARCH` and compile a binary for any target from your machine, no toolchain install needed.

```bash
# From Windows (Git Bash / PowerShell). Examples build a Linux/macOS binary.

# Bash syntax:
GOOS=linux   GOARCH=amd64 go build -o dist/app-linux-amd64   ./cmd/app
GOOS=darwin  GOARCH=arm64 go build -o dist/app-darwin-arm64  ./cmd/app
GOOS=windows GOARCH=amd64 go build -o dist/app-windows.exe   ./cmd/app
```

```powershell
# PowerShell syntax (env vars set per-command):
$env:GOOS="linux";  $env:GOARCH="amd64"; go build -o dist/app-linux ./cmd/app
$env:GOOS="windows";$env:GOARCH="amd64"; go build -o dist/app.exe   ./cmd/app
# Reset:
$env:GOOS=""; $env:GOARCH=""
```

```bash
# List all supported OS/ARCH combinations:
go tool dist list

# Pure-Go static binary (no libc dependency) — great for tiny Docker images.
# CGO_ENABLED=0 disables cgo; the result runs on scratch/alpine without glibc.
CGO_ENABLED=0 GOOS=linux go build -o app ./cmd/app
```

> **cgo caveat:** cross-compiling code that uses cgo (e.g. some SQLite drivers, parts of `os/user`) needs a cross C toolchain. Pure-Go code cross-compiles with zero setup. Prefer pure-Go dependencies (`CGO_ENABLED=0`) for painless cross-builds.

---

## 15. Gotchas & Best Practices

### 1. Always `defer Close()` immediately after a successful open

```go
f, err := os.Open(path)
if err != nil {
    return err // don't defer Close — there's nothing to close
}
defer f.Close() // right after the error check
```

### 2. Check the error from `Close()` on files you *wrote*

Buffered data may only flush on `Close`. For writes, the close error matters. The clean pattern uses a named return:

```go
func writeReport(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    // This deferred closure overwrites err if Close fails AND err was nil —
    // so a flush failure isn't silently swallowed.
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = cerr
        }
    }()
    _, err = f.Write(data)
    return err
}
```

### 3. Don't leak file descriptors in loops

```go
// WRONG: every iteration opens a file but defer only runs when the FUNCTION
// returns — you'll exhaust file descriptors on a big directory.
for _, name := range names {
    f, _ := os.Open(name)
    defer f.Close() // ❌ piles up until function end
    process(f)
}

// RIGHT: close within each iteration, ideally via a helper func so defer scopes.
for _, name := range names {
    if err := func() error {
        f, err := os.Open(name)
        if err != nil {
            return err
        }
        defer f.Close() // ✅ runs at the end of THIS iteration
        return process(f)
    }(); err != nil {
        return err
    }
}
```

### 4. `WriteFile` truncates — it does not append

`os.WriteFile` always replaces the whole file. To append, use `os.OpenFile(..., os.O_APPEND|os.O_CREATE|os.O_WRONLY, ...)` (§1.2).

### 5. Atomic writes: write to a temp file, then rename

To avoid a half-written file if the program crashes mid-write:

```go
func atomicWrite(path string, data []byte) error {
    dir := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".tmp-*") // same dir → rename is atomic
    if err != nil {
        return err
    }
    tmpName := tmp.Name()
    defer os.Remove(tmpName) // clean up if we bail before the rename

    if _, err := tmp.Write(data); err != nil {
        tmp.Close()
        return err
    }
    if err := tmp.Sync(); err != nil { // flush to disk before rename
        tmp.Close()
        return err
    }
    if err := tmp.Close(); err != nil {
        return err
    }
    // Rename over the destination — atomic on the same filesystem.
    return os.Rename(tmpName, path)
}
```

### 6. Never trust paths from outside your program

Archive entries, HTTP params, config values: validate with `filepath.IsLocal`, or — best — confine with `os.Root` (§4.4). A single unchecked `..` is a security hole.

### 7. Use `errors.Is` with the sentinel errors

```go
if errors.Is(err, os.ErrNotExist) { /* missing */ }
if errors.Is(err, os.ErrExist)    { /* already there */ }
if errors.Is(err, os.ErrPermission){ /* denied */ }
if errors.Is(err, fs.ErrNotExist) { /* same as os.ErrNotExist */ }
```

### 8. `os.Exit` / `log.Fatal` skip deferred cleanup

Covered in §10.5 — keep them out of helper functions; return errors up to `main`.

### 9. `exec.Command` is not a shell

No globs, pipes, `~`, env expansion, or `&&`. Use a shell explicitly (§7.9) — and never with untrusted input.

### 10. On Windows, an external tool may be a `.cmd`/`.bat`, not `.exe`

`exec.LookPath` handles PATHEXT, but be careful quoting arguments to batch files (§7.11).

### 11. Buffered writers MUST be flushed

A `bufio.Writer` or `gzip.Writer` loses unflushed data unless you `Flush()`/`Close()`. Defer the close *and* check its error.

### 12. Prefer `WalkDir` over `Walk`, `ReadDir` over `ReadDir`-via-`Open`+`Stat`

The `DirEntry`-based APIs (Go 1.16+) skip needless `Stat` calls and are markedly faster on large trees.

### 13. Send program output to stdout, logs/errors to stderr

Pipelines (`mytool | other`) expect data on stdout. Diagnostics on stderr keep stdout parseable.

### 14. Quote nothing, escape nothing — pass args as a slice

When shelling out, pass each argument as its own slice element; let `os/exec` handle quoting. Manual quoting is where injection bugs live.

### 15. Clean up temp files & dirs (and use `t.TempDir()` in tests)

`defer os.Remove(tmp.Name())` / `defer os.RemoveAll(dir)`. In tests, `t.TempDir()` auto-cleans.

---

## 16. Study Path & Build-to-Learn Projects

Work top to bottom. Each phase produces a working tool you can keep.

### Phase 1 — Files & paths (1–2 days)

1. **Read/write** — `os.ReadFile`/`WriteFile`, then `os.Open`/`Create`/`OpenFile` with flags.
2. **Streaming** — `bufio.Scanner` line-by-line; `io.Copy` to copy a file; hash a large file.
3. **Paths** — `filepath.Join/Dir/Base/Ext/Rel/Clean`; understand `path` vs `path/filepath` on Windows.
4. **Directories** — `os.Mkdir/MkdirAll`, `os.ReadDir`, `os.Remove/RemoveAll`, `os.Rename`.

**Mini-project — `wc` clone:** count lines, words, and bytes of files passed as args, reading stdin when no file is given. Practice scanners and `os.Args`.

### Phase 2 — Trees, fs, metadata (2–3 days)

5. **Walking** — `filepath.WalkDir` with `SkipDir`; build the recursive directory copier from §2.7.
6. **`io/fs` & embed** — embed a folder with `//go:embed`; walk it with `fs.WalkDir`; serve it.
7. **`os.Root`** — extract a zip into a sandboxed root (Go 1.24); confirm `..` is blocked.
8. **Metadata** — `Stat`/`Lstat`, `FileMode`, mod-times; create and read a symlink.

**Mini-project — `tree` clone:** recursively print a directory as an indented tree (├──, └──), with flags for max depth (`-L`), showing hidden files (`-a`), and dirs-only (`-d`). Uses `WalkDir`, `filepath`, and the `flag` package.

### Phase 3 — Processes & system (2–3 days)

9. **`os/exec`** — `Output`, `CombinedOutput`, `Run`; `Cmd.Dir` and `Cmd.Env`; capture & parse JSON from `go mod edit -json`.
10. **Streaming subprocess output** — `StdoutPipe` + `bufio.Scanner` live; `CommandContext` timeout.
11. **System info** — `runtime.GOOS/NumCPU/MemStats`, `os.UserConfigDir`, `os/user`.
12. **Signals** — `signal.NotifyContext`; graceful shutdown of a loop; the `os.Exit`/defer trap.

**Mini-project — task runner:** read a `tasks.yaml` (or simple text) of named commands and run them, streaming each command's output live, honouring a `--timeout`, stopping cleanly on Ctrl+C, and reporting exit codes. Heavy `os/exec` + `os/signal` + context practice.

### Phase 4 — CLIs & archives (3–5 days)

13. **`flag` deeply** — flags, defaults, `NewFlagSet` subcommands, custom usage. Build the §11.6 file manager.
14. **Cobra** — command tree, persistent vs local flags, `Args` validators, `RunE`. Build the §11.7 scaffolder.
15. **Viper** — merge flags + env + config file with precedence.
16. **Archives** — create/extract zip and tar.gz with the Zip-Slip guard (or `os.Root`).
17. **Interactive UX** — add a spinner/progress bar and terminal detection (§12.2); try `huh`/`lipgloss`.

**Capstone — a backup CLI (`bkup`):**
- `bkup create <srcDir> --out backup.tar.gz` — walk the tree, tar+gzip it, show a progress bar.
- `bkup restore backup.tar.gz --to <dir>` — extract safely into an `os.Root` sandbox.
- `bkup list backup.tar.gz` — read headers and print contents without extracting.
- Cobra command tree; `--exclude` glob patterns via `filepath.Match`; `--verbose` persistent flag.
- Graceful Ctrl+C cancellation with `signal.NotifyContext`; atomic write of the output file.
- Cross-compile it for Windows, Linux, and macOS from your machine (§14.3).

This single project exercises nearly every topic in the guide: streaming I/O, `filepath`, `WalkDir`, archives, `os.Root`, Cobra, signals, context, and cross-compilation.

---

### Reference — standard library packages used in this guide

| Package | Purpose |
|---|---|
| `os` | Files, dirs, env, args, process, signals' source |
| `io` | `Reader`/`Writer`/`Closer`, `Copy`, `ReadAll`, `LimitReader` |
| `bufio` | Buffered I/O, `Scanner`, `Reader`, `Writer` |
| `path/filepath` | OS-native filesystem paths |
| `path` | Slash-separated virtual paths (URLs, embed, io/fs) |
| `io/fs` | Abstract read-only filesystem interface |
| `embed` | Embed files into the binary (Go 1.16+) |
| `os/exec` | Run external programs |
| `os/signal` | OS signal handling |
| `os/user` | User & group lookup |
| `runtime` | GOOS/GOARCH, CPU count, mem stats |
| `archive/zip` | Zip create/extract |
| `archive/tar` | Tar create/extract |
| `compress/gzip` | Gzip compression |
| `context` | Cancellation & timeouts (subprocesses, signals) |
| `errors` | `errors.Is`/`As` with `os.ErrNotExist` etc. |
| `flag` | Standard-library CLI flags & subcommands |
| `syscall` | Signal constants, low-level platform calls |

### Reference — popular third-party libraries

| Library | Purpose |
|---|---|
| `github.com/spf13/cobra` | CLI command framework |
| `github.com/spf13/viper` | Layered configuration |
| `github.com/urfave/cli/v3` | Lighter CLI framework |
| `github.com/fsnotify/fsnotify` | OS-level filesystem watching |
| `github.com/joho/godotenv` | `.env` loading |
| `github.com/shirou/gopsutil/v4` | Cross-platform system metrics |
| `github.com/charmbracelet/{lipgloss,bubbletea,huh}` | Styling, TUIs, prompts |
| `github.com/pterm/pterm` | Spinners, progress, tables |
| `golang.org/x/term` | Terminal detection, raw mode, password input |

---

> **Final note:** The Go standard library covers the overwhelming majority of filesystem, OS, and subprocess work without any dependencies — and Go 1.24's `os.Root` finally makes untrusted-path handling safe by construction. Reach for Cobra/Viper when your CLI grows a real command tree and config story, and for charm/pterm when UX matters. Master the primitives here first: `os`, `io`, `bufio`, `filepath`, `io/fs`, and `os/exec`. They compose cleanly, behave predictably across Windows and Unix, and will carry you through almost anything you build.
