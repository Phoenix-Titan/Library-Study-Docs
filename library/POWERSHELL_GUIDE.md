# PowerShell — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers, sysadmins, and power users (beginner through advanced) who want to learn PowerShell properly — not as "a slightly weirder cmd.exe" but as the **object-oriented automation shell and scripting language** it actually is. Every concept here is explained with runnable, heavily commented code you can paste straight into a console or a `.ps1` file. No internet required.
>
> **⚡ Version note (READ THIS FIRST — it matters a lot):** There are **two** PowerShells, and they are *not* the same product:
>
> | | **Windows PowerShell 5.1** | **PowerShell 7.x** |
> |---|---|---|
> | Executable | `powershell.exe` | `pwsh` / `pwsh.exe` |
> | Runtime | .NET **Framework** 4.x | .NET (modern, "Core" lineage) |
> | Platform | Windows only | Windows, Linux, macOS |
> | Status in 2026 | **Frozen** — bundled with Windows, only security fixes | **Current & recommended** — actively developed |
> | Default file encoding | UTF-16LE / system "Default" (a *huge* bug source) | **UTF-8 (no BOM)** |
> | Install | Already on every Windows box | `winget install Microsoft.PowerShell` |
>
> **Windows PowerShell 5.1 is built into Windows and will never go away**, so even in 2026 you *will* hit it (login scripts, locked-down servers, "I double-clicked and it ran 5.1"). PowerShell 7.x is what you should install and use for everything new. The two install side by side — `powershell` launches 5.1, `pwsh` launches 7. Throughout this guide, features that exist **only in 7.x** are flagged with **⚡ 7+ only**. Common 5.1-vs-7 traps get a **⚡ Version note**. If a script "works on my machine" but breaks elsewhere, the version split is the first thing to check.

---

## Table of Contents

1. [What PowerShell Is — The Object Pipeline Mental Shift](#1-what-powershell-is)
2. [Cmdlets & Syntax — Verb-Noun, Help, Get-Member](#2-cmdlets--syntax)
3. [The Pipeline Deeply — Select / Where / ForEach / Sort / Group / Measure](#3-the-pipeline-deeply)
4. [Variables, Scopes & Data Types](#4-variables-scopes--data-types)
5. [Operators](#5-operators)
6. [Strings](#6-strings)
7. [Collections — Arrays & Hashtables](#7-collections)
8. [Control Flow](#8-control-flow)
9. [Functions & Advanced Functions](#9-functions--advanced-functions)
10. [Error Handling](#10-error-handling)
11. [Filesystem & Providers](#11-filesystem--providers)
12. [Objects & Data Formats — PSCustomObject, CSV, JSON, XML, Formatting](#12-objects--data-formats)
13. [Running External Programs](#13-running-external-programs)
14. [Modules](#14-modules)
15. [Remoting & Jobs](#15-remoting--jobs)
16. [Real-World Cmdlets & Tasks](#16-real-world-cmdlets--tasks)
17. [Scripting Best Practices](#17-scripting-best-practices)
18. [The 5.1 vs 7 Gotchas Section](#18-the-51-vs-7-gotchas-section)
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

> **Difficulty markers** used throughout: 🟢 **Beginner** · 🟡 **Intermediate** · 🔴 **Advanced**

---

## 1. What PowerShell Is

🟢 **Beginner**

PowerShell is two things at once:

1. An **interactive command-line shell** (like bash or cmd.exe), and
2. A full **scripting language** built on .NET.

But the single idea that makes PowerShell *click* — and the one thing to internalize before anything else — is this:

> **PowerShell pipes pass .NET objects, not text.**

### The object pipeline vs the text pipeline

In bash and cmd, every command's output is a **stream of text**. To get, say, the names of running processes, you run a command and then parse its text with `grep`, `awk`, `cut`, `sed` — chopping columns out of formatted strings. The format is for humans; programs have to reverse-engineer it.

In PowerShell, a cmdlet outputs **real objects** with **properties** and **methods**. The text you see on screen is just a *rendering* of those objects produced at the very end. You filter and transform by reaching into properties directly — no string parsing.

```powershell
# bash-style thinking (DON'T do this in PowerShell):
#   ps aux | grep chrome | awk '{print $2}'    # parse text columns

# PowerShell thinking — work with the OBJECT's properties directly:
Get-Process chrome | Select-Object -ExpandProperty Id
#   Get-Process returns Process OBJECTS.
#   Each object already HAS an .Id property — no parsing required.
```

Here is the proof that the pipeline carries objects, not strings:

```powershell
# Get-Process returns objects of type System.Diagnostics.Process.
Get-Process | Get-Member -MemberType Property | Select-Object Name -First 5
#   Get-Member shows the TYPE and all properties/methods of whatever
#   flows into it. You'll see Id, Name, CPU, WorkingSet, etc. —
#   these are .NET properties, not text columns.
```

Because objects flow through the pipe, you can do things that are painful in text shells:

```powershell
# Sort running processes by memory, take the top 3, show name + MB.
Get-Process |
    Sort-Object WorkingSet -Descending |        # sort by a real numeric property
    Select-Object -First 3 Name,
        @{ Name = 'MB'; Expression = { [math]::Round($_.WorkingSet / 1MB, 1) } } |
    Format-Table -AutoSize
#   No awk, no cut, no fragile column math. We reached into .WorkingSet,
#   did arithmetic on it, and rounded — because it's a number, not text.
```

### Cmdlets, functions, scripts, aliases

PowerShell commands come in several flavors:

| Kind | What it is | Example |
|---|---|---|
| **Cmdlet** | Compiled .NET command, always `Verb-Noun` | `Get-Process` |
| **Function** | Command written in PowerShell | `function Get-Thing { ... }` |
| **Script** | A `.ps1` file you run | `.\deploy.ps1` |
| **Alias** | Short name for a command | `ls` → `Get-ChildItem` |
| **Native command** | An external `.exe` | `git`, `ipconfig`, `node` |

### Running PowerShell

```powershell
# Launch the shells (they install side by side):
powershell        # Windows PowerShell 5.1  (powershell.exe)
pwsh              # PowerShell 7.x           (the modern one — use this)

# Check which one you're in — ALWAYS verify when debugging weird behavior:
$PSVersionTable
#   PSVersion 5.1.x  -> Windows PowerShell
#   PSVersion 7.x.x  -> PowerShell 7
$PSVersionTable.PSVersion        # just the version number
$PSVersionTable.PSEdition        # 'Desktop' = 5.1, 'Core' = 7+
```

### Running scripts (.ps1)

A script is just a text file of PowerShell commands saved with a `.ps1` extension.

```powershell
# Run a script. Note the .\ — PowerShell does NOT run scripts from the
# current directory without an explicit path (a security feature; unlike cmd).
.\hello.ps1

# Run with an absolute path:
& "C:\Scripts\hello.ps1"          # & is the call operator (more in section 13)

# Run a script in another PowerShell process, bypassing policy for that one call:
pwsh -File .\hello.ps1
pwsh -ExecutionPolicy Bypass -File .\hello.ps1
```

### Execution policy — *"why won't my script run?!"*

🟢 The #1 first-day frustration: you write a `.ps1`, double-click or run it, and get

```
File C:\...\hello.ps1 cannot be loaded because running scripts is disabled on this system.
```

This is the **execution policy** — a safety setting (NOT a real security boundary; it's a guardrail against accidentally running a script you downloaded). Common values:

| Policy | Meaning |
|---|---|
| `Restricted` | No scripts at all (default on client Windows for 5.1). Interactive commands still work. |
| `AllSigned` | Scripts must be signed by a trusted publisher. |
| `RemoteSigned` | Local scripts run freely; **downloaded** scripts must be signed. **The sensible default.** |
| `Unrestricted` | Everything runs (warns on downloaded files). |
| `Bypass` | Nothing is blocked, no warnings. |
| `Undefined` | No policy set at this scope; falls through to next scope. |

```powershell
# See current effective policy and the policy at each scope:
Get-ExecutionPolicy
Get-ExecutionPolicy -List

# Recommended one-time fix for your own user (no admin needed):
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
#   -Scope CurrentUser writes to your profile, not the whole machine.

# Run a single script WITHOUT changing your policy at all:
powershell -ExecutionPolicy Bypass -File .\hello.ps1
pwsh       -ExecutionPolicy Bypass -File .\hello.ps1
```

**Why "downloaded" scripts are special:** Windows tags files from the internet with a "Mark of the Web" (a Zone.Identifier alternate data stream). `RemoteSigned` checks that mark. You can clear it:

```powershell
Unblock-File .\downloaded-script.ps1   # removes the "downloaded from internet" mark
```

**Signing basics (just so you know it exists):** `AllSigned`/`RemoteSigned` rely on Authenticode signatures. You sign with `Set-AuthenticodeSignature -FilePath .\x.ps1 -Certificate $cert` using a code-signing cert. Most individuals never sign; they use `RemoteSigned` and `Unblock-File`. Enterprises sign.

### The profile (`$PROFILE`)

Your **profile** is a `.ps1` that runs automatically every time you start PowerShell — the place for aliases, functions, prompt tweaks, and module imports you always want.

```powershell
$PROFILE          # path to your "current user, current host" profile
#   e.g. C:\Users\You\Documents\PowerShell\Microsoft.PowerShell_profile.ps1   (pwsh)
#        C:\Users\You\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1 (5.1)
#   NOTE: 5.1 and 7 use DIFFERENT profile paths — they do not share config!

# There are several profiles (all-users, all-hosts, etc.):
$PROFILE | Format-List -Force   # shows AllUsersAllHosts, CurrentUserCurrentHost, etc.

# Create & edit your profile:
if (-not (Test-Path $PROFILE)) {
    New-Item -ItemType File -Path $PROFILE -Force   # also creates parent folders
}
notepad $PROFILE        # or: code $PROFILE

# Example contents of a profile:
#   Set-Alias ll Get-ChildItem
#   function gs { git status }
#   Import-Module posh-git
#   $PSStyle.FileInfo.Directory = "`e[1;34m"   # (7+) colorize directories
```

---

## 2. Cmdlets & Syntax

🟢 **Beginner**

### Verb-Noun naming

Every well-behaved command is `Verb-Noun`, singular noun, with an **approved verb**. This consistency is why PowerShell is discoverable: once you know `Get-`, you can guess `Get-Process`, `Get-Service`, `Get-Item`.

```powershell
Get-Verb                         # the official list of approved verbs
Get-Verb | Where-Object Group -eq 'Lifecycle'   # verbs in a category
```

Common verb meanings: `Get` (read), `Set` (modify), `New` (create), `Remove` (delete), `Start`/`Stop`, `Add`/`Clear`, `Import`/`Export`, `ConvertTo`/`ConvertFrom`, `Out` (send to a destination), `Write` (emit), `Invoke` (run), `Test` (check), `Find`/`Search`.

### Parameters: named, positional, switch

```powershell
# Named parameter (explicit -Name):  ALWAYS prefer this in scripts.
Get-ChildItem -Path C:\Windows -Filter *.dll

# Positional parameter: the first argument goes to -Path implicitly.
Get-ChildItem C:\Windows          # C:\Windows binds to -Path by position

# Switch parameter: a boolean flag — its presence means $true. No value needed.
Get-ChildItem -Recurse            # -Recurse is a switch
Get-ChildItem -Recurse:$false     # you CAN force a switch off with :$false

# Parameter values with spaces need quotes:
Get-ChildItem -Path "C:\Program Files"
```

### `Get-Help` — your built-in manual

```powershell
Get-Help Get-Process                 # summary
Get-Help Get-Process -Examples       # JUST the examples (most useful day to day)
Get-Help Get-Process -Full           # everything: every parameter, types, notes
Get-Help Get-Process -Online         # opens the web docs (needs internet)
Get-Help Get-Process -Parameter Name # help for ONE parameter

# Help content ships separately and may be a stub until you update it:
Update-Help                          # downloads full help (needs internet + admin sometimes)
Update-Help -Scope CurrentUser       # 7+ can update to your user dir without admin

# about_* topics are conceptual docs — gold for learning the language itself:
Get-Help about_*                     # list them
Get-Help about_Operators
Get-Help about_Splatting
```

### `Get-Command` — discover commands

```powershell
Get-Command Get-Process              # what is this? where's it from?
Get-Command -Verb Get -Noun Service  # find by verb/noun
Get-Command -Module Microsoft.PowerShell.Management
Get-Command *process*                # wildcard search by name
Get-Command notepad                  # also finds native .exe's on PATH
(Get-Command Get-Process).Parameters.Keys   # list a cmdlet's parameter names
```

### `Get-Member` — *learn this early and use it constantly*

`Get-Member` (alias `gm`) tells you the **type** of an object and **every property and method** it has. When you don't know how to get a piece of data, pipe the object to `Get-Member` and read what's available. This is the single most important learning/debugging tool.

```powershell
Get-Process | Get-Member             # all members of a Process object
Get-Date    | Get-Member -MemberType Method      # just methods
(Get-Date)  | Get-Member -MemberType Property     # just properties

# Real workflow: "I have a file object — how do I get its size?"
Get-ChildItem .\notes.txt | Get-Member
#   ...scan output... you spot a 'Length' property (size in bytes).
(Get-ChildItem .\notes.txt).Length

# What TYPE is this thing?
(Get-Date).GetType().FullName        # System.DateTime
'hello'.GetType().Name               # String
```

### Aliases (and why to avoid them in scripts)

```powershell
Get-Alias                            # all aliases
Get-Alias ls                         # ls -> Get-ChildItem
Get-Alias -Definition Get-ChildItem  # all aliases FOR a command (ls, dir, gci)
Set-Alias ll Get-ChildItem           # make your own (put in $PROFILE)
```

**Rule:** aliases are fine *interactively* (fast to type). In **scripts, always use full cmdlet names and named parameters.** Aliases like `ls`/`cat`/`curl` may be redefined, differ across machines (`curl` is `Invoke-WebRequest` in 5.1 but the real curl.exe in 7 on many systems), or simply confuse the next reader. `PSScriptAnalyzer` (section 17) flags aliases in scripts.

### Tab completion

Press **Tab** to complete cmdlet names, parameter names (type `-` then Tab), parameter *values* (e.g. service names), file paths, and even enum values. In PowerShell 7 with PSReadLine, you also get **predictive IntelliSense** (ghost-text suggestions from history) — press **→** to accept. Type `Get-Pr<Tab>`, `-Pa<Tab>`, etc.

---

## 3. The Pipeline Deeply

🟢🟡 **Beginner → Intermediate**

The pipeline (`|`) sends the output objects of one command as input to the next. Master these six cmdlets and you can do 80% of everyday work.

### `Select-Object` — pick properties / rows

```powershell
# Pick specific PROPERTIES (returns a new object with only those):
Get-Process | Select-Object Name, Id, CPU

# -First / -Last / -Skip  (rows):
Get-Process | Select-Object -First 5
Get-Process | Select-Object -Last 3
Get-Process | Select-Object -Skip 10 -First 5

# -Unique  (distinct objects/values):
1,1,2,3,3,3 | Select-Object -Unique          # 1 2 3

# -ExpandProperty  — pull ONE property OUT as raw values (not wrapped in an object).
#   This is the cmdlet most beginners miss. Compare:
Get-Process | Select-Object Name             # objects, each with a .Name property
Get-Process | Select-Object -ExpandProperty Name   # plain strings — usable directly
(Get-Process | Select-Object -ExpandProperty Id) | Measure-Object -Sum   # works: numbers

# Calculated property — invent a NEW property with @{Name=...; Expression={...}}:
Get-ChildItem -File |
    Select-Object Name,
        @{ Name = 'SizeKB'; Expression = { [math]::Round($_.Length / 1KB, 2) } },
        @{ Name = 'Age';    Expression = { (Get-Date) - $_.LastWriteTime } }
#   'n'/'e' are accepted shorthands for Name/Expression:  @{ n='SizeKB'; e={...} }
```

### `Where-Object` — filter rows

```powershell
# Script-block syntax (full power; $_ is the current object):
Get-Process | Where-Object { $_.CPU -gt 100 }
Get-Process | Where-Object { $_.Name -like 'pw*' -and $_.WorkingSet -gt 50MB }

# Simplified ("comparison") syntax — cleaner for ONE simple condition (PS 3+):
Get-Process | Where-Object CPU -gt 100
Get-Process | Where-Object Name -like 'pw*'
Get-Service  | Where-Object Status -eq 'Running'

# Common comparison operators inside Where-Object (full list in section 5):
#   -eq -ne -gt -lt -ge -le   -like (wildcards)   -match (regex)
#   -contains / -in (membership)
Get-Process | Where-Object { $_.Name -match '^chrome' }   # regex
Get-ChildItem | Where-Object { $_.Extension -in '.md', '.txt' }
```

`?` is the alias for `Where-Object` (`gps | ? CPU -gt 100`). Fine interactively; avoid in scripts.

### `ForEach-Object` — run code per item

```powershell
# Run a script block for EACH object. $_ (or $PSItem) is the current item.
1..5 | ForEach-Object { $_ * $_ }            # 1 4 9 16 25

Get-ChildItem -File | ForEach-Object {
    "$($_.Name) is $($_.Length) bytes"
}

# -Begin / -Process / -End : setup, per-item, teardown.
1..3 | ForEach-Object -Begin   { $total = 0 } `
                      -Process { $total += $_ } `
                      -End     { "Sum = $total" }     # Sum = 6

# Shorthand: -MemberName calls a property or method on each item (PS 4+):
'a','bb','ccc' | ForEach-Object Length               # 1 2 3 (property)
'a','b','c'    | ForEach-Object ToUpper              # A B C (method)
```

> **`foreach` statement vs `ForEach-Object` cmdlet** — they are different (see section 8). The `foreach (...)` *statement* loops over a collection already in memory (faster, no pipeline). `ForEach-Object` is a *cmdlet* that processes a streaming pipeline one object at a time (lower memory, works with huge/streamed input). Both alias to `%` confusingly — `%` is `ForEach-Object`.

### `Sort-Object`, `Group-Object`, `Measure-Object`

```powershell
# Sort:
Get-Process | Sort-Object CPU -Descending
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 5
Get-ChildItem | Sort-Object Length, Name           # sort by multiple keys
Get-Process | Sort-Object Name -Unique             # sort AND dedupe by a key

# Group: bucket objects by a property; .Count and .Group per bucket.
Get-ChildItem -File | Group-Object Extension | Sort-Object Count -Descending
Get-Process | Group-Object Company -NoElement      # just counts, drop the items
#   Each output is a GroupInfo with .Name (the key), .Count, .Group (the items).

# Measure: aggregate numbers / count / text stats.
Get-ChildItem -File | Measure-Object Length -Sum -Average -Maximum -Minimum
(Get-ChildItem).Count                              # just how many
Get-Content .\log.txt | Measure-Object -Line -Word -Character
```

### Putting it together — a tiny real report

```powershell
# Top 5 largest files under the current folder, recursively, as a table.
Get-ChildItem -Recurse -File |
    Sort-Object Length -Descending |
    Select-Object -First 5 FullName,
        @{ n = 'MB'; e = { [math]::Round($_.Length / 1MB, 2) } } |
    Format-Table -AutoSize
```

---

## 4. Variables, Scopes & Data Types

🟢🟡 **Beginner → Intermediate**

### Variables

```powershell
$name = 'Ada'           # variables start with $
$count = 42
$items = 1, 2, 3        # comma makes an array
$nothing = $null

# Variable names can have spaces/odd chars via {curly braces} (rarely needed):
${my var} = 10

# The Variable: provider lets you treat variables like files:
Get-Variable                    # list all variables
Get-Variable name -ValueOnly    # value of $name
Remove-Variable name            # delete $name
Clear-Variable count            # set to $null
```

### Scopes: global / script / local / private

```powershell
$x = 'global-ish'   # at the prompt, this is in the global scope

function Test-Scope {
    $y = 'local'              # local to this function
    $x                        # functions can READ parent scope variables
    $script:shared = 'hi'     # write to the SCRIPT scope explicitly
    $global:g = 'set globally'# write to GLOBAL scope explicitly
}
```

| Scope modifier | Meaning |
|---|---|
| `$global:x` | Visible everywhere in the session |
| `$script:x` | Visible throughout the current script/module file |
| `$local:x` | Current scope only (the default) |
| `$private:x` | Current scope, **not** even readable by child scopes |
| `$using:x` | Pull a *local* variable into a remote/job/parallel scope (sections 15) |

**Key rule:** a child scope can **read** a parent's variable, but assigning `$x = ...` inside a function creates a *new local* `$x` (copy-on-write). To modify the parent's, use `$script:` or `$global:`.

### `$env:` — environment variables

```powershell
$env:PATH                       # read PATH
$env:USERNAME
$env:COMPUTERNAME
$env:TEMP

$env:MY_FLAG = 'true'           # set for THIS process (and children) only
Remove-Item Env:\MY_FLAG        # remove (Env: is a provider — section 11)
Get-ChildItem Env:              # list ALL environment variables

# Persisting machine/user env vars (survives reboot) uses .NET:
[Environment]::SetEnvironmentVariable('MY_VAR', 'x', 'User')   # 'User' or 'Machine'
[Environment]::GetEnvironmentVariable('MY_VAR', 'User')
```

### Automatic variables (PowerShell sets these for you)

| Variable | Meaning |
|---|---|
| `$_` / `$PSItem` | Current pipeline object (inside `Where`/`ForEach`/etc.) |
| `$?` | `$true` if the **last** command succeeded (boolean) |
| `$LASTEXITCODE` | Exit code of the last **native** (.exe) command |
| `$args` | Unbound arguments passed to a function/script |
| `$PSScriptRoot` | Folder the running script lives in (great for relative paths) |
| `$PSCommandPath` | Full path of the running script |
| `$null` | The null value |
| `$true` / `$false` | Booleans |
| `$PID` | Process ID of the current PowerShell |
| `$Host` | The host program (console, ISE, VS Code) info |
| `$PSVersionTable` | Version details (your 5.1-vs-7 check) |
| `$Error` | Array of recent errors (`$Error[0]` is the latest) |
| `$Matches` | Capture groups from the last `-match` |
| `$input` | Pipeline input enumerator (in functions without `process{}`) |
| `$MyInvocation` | Info about how the current command was called |
| `$PROFILE` | Path to your profile script |
| `$OFS` | Output Field Separator used when an array is interpolated into a string |

```powershell
# $PSScriptRoot — build robust paths relative to the script (NOT the cwd):
$config = Join-Path $PSScriptRoot 'settings.json'
#   This works no matter what directory the user runs the script from.

# $? vs $LASTEXITCODE — they answer DIFFERENT questions (see section 10/13):
Get-Item C:\nope 2>$null; $?        # $false  (cmdlet error)
cmd /c exit 5;            $LASTEXITCODE   # 5  (native exe exit code)
```

### Typing: `[int]`, `[string]`, `[datetime]`, etc.

PowerShell is dynamically typed but you can (and in scripts *should*) annotate types. Annotating a variable makes PowerShell **coerce** assignments to that type.

```powershell
[int]    $age   = '42'          # the string '42' is converted to int 42
[string] $label = 123           # becomes '123'
[double] $pi    = 3.14159
[datetime] $when = '2026-06-21' # parsed into a DateTime object
[bool]   $flag  = 1             # $true

# Casting on the fly:
[int]'0x1F'                     # 31 (hex literal cast)
[int][char]'A'                  # 65
[datetime]'2026-01-01' - (Get-Date)   # a TimeSpan (date math!)

# Type accelerators you'll use a lot:
#   [int] [long] [double] [decimal] [string] [char] [bool]
#   [datetime] [timespan] [guid] [version] [regex] [xml]
#   [array] [hashtable] [pscustomobject] [scriptblock]
[guid]::NewGuid()
[math]::Round(3.14159, 2)       # 3.14   ([math] is the .NET System.Math class)

# Numeric multipliers are built in: KB MB GB TB PB
2GB                              # 2147483648
```

### `$null` handling (and a famous gotcha)

```powershell
$x = $null
$null -eq $x                     # $true  — CORRECT way to test for null
#   ⚠️ ALWAYS put $null on the LEFT. Why? If $x is an ARRAY, `$x -eq $null`
#      FILTERS the array element-by-element instead of testing for null.
@(1, $null, 2) -eq $null         # returns the $null element(s) — NOT a boolean!
$null -eq @(1, $null, 2)         # $false — the safe, scalar comparison

# Empty vs null vs whitespace:
[string]::IsNullOrEmpty($x)
[string]::IsNullOrWhiteSpace('   ')   # $true
```

### Strict mode — catch bugs early

🟡 By default PowerShell silently returns `$null` for a typo'd or undefined variable. `Set-StrictMode` turns those into errors. **Use it in every serious script.**

```powershell
Set-StrictMode -Version Latest
#   Now: referencing an UNINITIALIZED variable throws,
#        referencing a NON-EXISTENT property throws,
#        calling a function like a method  f(1,2)  throws.

$undefined        # without strict mode: $null silently. With it: ERROR.

Set-StrictMode -Off    # turn it back off (rarely needed)
```

---

## 5. Operators

🟡 **Intermediate**

PowerShell operators are **words with a dash**, not symbols like `==` or `&&` (mostly). Comparison operators are **case-insensitive by default**; prefix with `c` for case-sensitive (`-ceq`) or `i` to be explicit (`-ieq`).

### Comparison

```powershell
5 -eq 5            # equal           -> True
5 -ne 3            # not equal       -> True
5 -gt 3            # greater than    -> True
5 -lt 3            # less than       -> False
5 -ge 5            # >=              -> True
5 -le 4            # <=              -> False

'ABC' -eq 'abc'    # True  (case-INSENSITIVE by default)
'ABC' -ceq 'abc'   # False (case-SENSITIVE)
'ABC' -ieq 'abc'   # True  (explicitly case-insensitive)

# On the RIGHT of an array, comparison FILTERS (returns matching elements):
1,2,3,4 -gt 2      # 3 4    (filter, not boolean!)
```

### `-like` / `-notlike` — wildcards

```powershell
'PowerShell' -like 'Power*'      # True   (* = any chars)
'cat' -like 'c?t'                # True   (? = exactly one char)
'report.txt' -notlike '*.log'    # True
'abc' -like '[a-c]*'             # True   ([] = char set/range)
```

### `-match` / `-notmatch` — regex + `$Matches`

```powershell
'2026-06-21' -match '\d{4}-\d{2}-\d{2}'   # True
$Matches                                   # auto-populated on a successful -match
$Matches[0]                                # the whole match: 2026-06-21

# Named capture groups land in $Matches by name:
'name: Ada' -match 'name:\s*(?<who>\w+)'
$Matches.who                               # Ada

# -match on an ARRAY filters (returns matching elements):
'apple','banana','avocado' -match '^a'     # apple, avocado
```

### `-contains` / `-in` — membership

```powershell
1,2,3 -contains 2          # True  — does the COLLECTION contain the value?
2 -in 1,2,3                # True  — same test, value first (reads nicer)
'x' -notin 'a','b'         # True
#   ⚠️ Don't confuse -contains with -like. -contains tests collection membership
#      (exact element), NOT substrings. For substring use -like '*x*' or -match.
```

### Logical, arithmetic, assignment

```powershell
($true -and $false)        # False
($true -or  $false)        # True
(-not $true)               # False    (also: !$true)
($true -xor $true)         # False

7 + 3; 7 - 3; 7 * 3; 7 / 2; 7 % 3       # +,-,*,/, modulus
2 + 2 * 2                                # 6 (normal precedence)
[math]::Pow(2, 10)                       # 1024 (no ** operator in PS)

$n = 5
$n += 1; $n -= 1; $n *= 2; $n /= 2       # compound assignment
$n++; $n--                               # increment/decrement
```

### Type operators `-is` / `-as`

```powershell
(Get-Date) -is [datetime]      # True  — type test
'5' -is [int]                  # False
'5' -as [int]                  # 5     — try-convert; returns $null if it can't
'abc' -as [int]                # $null (no error, unlike a [int] cast)
```

### `-replace`, `-split`, `-join`

```powershell
'hello world' -replace 'o', '0'           # hell0 w0rld   (regex replace!)
'a1b2c3' -replace '\d', ''                # abc          (-replace uses REGEX)
'a1b2c3' -replace '(\w)(\d)', '$2$1'      # 1a2b3c       (backrefs $1,$2)

'a,b,c' -split ','                        # array: a b c
'one  two   three' -split '\s+'           # split on regex whitespace runs
'a,b,c' -split ',', 2                     # max 2 pieces: 'a', 'b,c'

'a','b','c' -join '-'                     # a-b-c
1..3 -join ', '                           # 1, 2, 3
```

### Ternary, `??`, `??=`, `&&`/`||` — **⚡ 7+ only**

These modern operators **do not exist in Windows PowerShell 5.1** and will cause parse errors there.

```powershell
# Ternary  condition ? ifTrue : ifFalse           (7+ only)
$status = ($code -eq 0) ? 'OK' : 'FAIL'

# Null-coalescing: use right side if left is $null  (7+ only)
$name = $userInput ?? 'default'

# Null-coalescing ASSIGNMENT: assign only if currently $null  (7+ only)
$config ??= @{ retries = 3 }

# Pipeline chain operators (like bash && / ||)     (7+ only)
git pull && npm install              # run npm install only if git pull SUCCEEDED
mkdir build || Write-Host 'exists'   # run right side only if left FAILED

# ⚡ In 5.1 you must write the long form instead:
$status = if ($code -eq 0) { 'OK' } else { 'FAIL' }
$name   = if ($null -ne $userInput) { $userInput } else { 'default' }
```

### Null-conditional `?.` and `?[]` — **⚡ 7+ only**

```powershell
# Safely access a member that might be on a $null object (7+ only).
# NOTE: the variable name MUST be wrapped in ${} for this syntax:
${obj}?.Property
${arr}?[0]
```

---

## 6. Strings

🟡 **Intermediate**

### Single vs double quotes

```powershell
$name = 'Ada'

'Hello $name'                  # literal:        Hello $name   (NO interpolation)
"Hello $name"                  # interpolated:   Hello Ada

# Inside double quotes, $() runs an EXPRESSION (subexpression operator):
"Today is $(Get-Date -Format 'yyyy-MM-dd')"
"2 + 2 = $(2 + 2)"
"Path has $($env:PATH.Split(';').Count) entries"

# A bare $obj.Prop in a string DOESN'T expand the property — wrap it in $():
$p = Get-Process | Select-Object -First 1
"Bad:  $p.Name"                # prints the object's ToString() then ".Name"
"Good: $($p.Name)"             # prints the actual Name

# Escaping: the backtick ` is PowerShell's escape char (NOT backslash).
"Line1`nLine2"                 # `n newline, `t tab, `r CR, `` literal backtick
"A literal dollar: `$home"     # `$ escapes the $
'In single quotes, double the quote to escape '' like this'
"In double quotes, double the `" to escape"
```

### Here-strings (multi-line literals)

```powershell
# Double-quoted here-string: interpolates. Note the EXACT rules:
#   - opening @" must be the LAST thing on its line
#   - closing "@ must be at the START of its own line (column 0, no indentation!)
$report = @"
User:  $name
Date:  $(Get-Date)
Path:  $PWD
"@

# Single-quoted here-string: pure literal, no interpolation — perfect for
# JSON/SQL/XML templates with $ and quotes in them:
$json = @'
{ "name": "$placeholder", "tags": ["a", "b"] }
'@
#   ⚠️ Indentation gotcha: the closing "@ / '@ MUST be flush-left.
#      If it's indented, you get a parse error. (See section 18.)
```

### Formatting & common string methods

```powershell
# -f  format operator (.NET composite formatting):
'{0} scored {1:P0}' -f 'Ada', 0.85          # Ada scored 85%
'{0,10}' -f 'right'                          # right-justify in width 10
'{0,-10}|' -f 'left'                         # left-justify in width 10
'{0:N2}'  -f 1234.5                          # 1,234.50  (thousands + 2 dp)
'{0:X}'   -f 255                             # FF (hex)
'{0:yyyy-MM-dd}' -f (Get-Date)               # date formatting

# String methods (these are real .NET System.String methods):
'  trim me  '.Trim()                         # 'trim me'
'  pad'.PadLeft(8)                           # '     pad'
'a-b-c'.Split('-')                           # array a b c   (.NET split)
'a-b-c'.Replace('-', '_')                    # a_b_c   (LITERAL replace, not regex)
'Hello'.ToUpper(); 'Hello'.ToLower()
'Hello'.Substring(1, 3)                      # ell
'Hello'.Contains('ell')                      # True
'Hello'.StartsWith('He'); 'Hello'.EndsWith('lo')
'Hello'.IndexOf('l')                         # 2
'Hello'.Length                               # 5

# ⚡ .Replace() is LITERAL; -replace is REGEX. Pick on purpose.
'a.b.c'.Replace('.', '_')   # a_b_c  (literal)
'a.b.c' -replace '.', '_'   # _____  (regex: . matches everything!)
'a.b.c' -replace '\.', '_'  # a_b_c  (escaped)
```

### Regex with `$Matches` and capture groups

```powershell
$log = '2026-06-21 14:03:22 ERROR Disk full'
if ($log -match '^(?<date>\S+)\s+(?<time>\S+)\s+(?<level>\w+)\s+(?<msg>.+)$') {
    $Matches.date     # 2026-06-21
    $Matches.level    # ERROR
    $Matches.msg      # Disk full
}

# [regex] class for more power (all matches, options):
[regex]::Matches('a1 b2 c3', '\d') | ForEach-Object { $_.Value }   # 1 2 3
[regex]::Replace('Hello', 'l', 'L')                                # HeLLo
'CamelCase' -creplace '([a-z])([A-Z])', '$1 $2'                    # Camel Case
```

---

## 7. Collections

🟡 **Intermediate**

### Arrays

```powershell
$a = 1, 2, 3                     # comma operator builds an array
$a = @(1, 2, 3)                  # @() is the array subexpression — explicit & safe
$empty = @()                     # empty array
$single = , 'x'                  # @('x') or ,'x' forces a ONE-element array

$a[0]                            # first element  -> 1
$a[-1]                           # last element   -> 3
$a[0..2]                         # slice indices 0..2
$a[1..-1]                        # ⚠️ this is NOT "to the end" — it's 1,0,-1!
$a[1..($a.Count-1)]             # correct "from index 1 to end"
$a.Count                         # length
$a -contains 2                   # membership test (section 5)

$a += 4                          # append... BUT SEE THE WARNING BELOW
```

**⚡ The `+=` cost gotcha (important for performance):** PowerShell arrays are **fixed size**. `$a += $x` does **not** grow the array — it **creates a brand-new array** and copies every element each time. In a loop over thousands of items this is O(n²) and crawls. Use a generic `List` instead:

```powershell
# SLOW — O(n^2), rebuilds the whole array every iteration:
$result = @()
foreach ($i in 1..100000) { $result += $i }     # don't do this for big N

# FAST — a real growable list (.NET List<T>):
$result = [System.Collections.Generic.List[int]]::new()
foreach ($i in 1..100000) { $result.Add($i) }    # O(1) amortized append
$result.ToArray()                                 # convert back if you need an array

# Idiomatic FAST alternative — let the pipeline/loop emit and capture:
$result = foreach ($i in 1..100000) { $i }        # collects all emitted values
```

### Hashtables

```powershell
$h = @{ Name = 'Ada'; Age = 42; City = 'London' }

$h['Name']                       # index access      -> Ada
$h.Age                           # dot access        -> 42
$h.Keys                          # all keys
$h.Values                        # all values
$h.Count                         # number of entries
$h.ContainsKey('City')           # True

$h['Country'] = 'UK'             # add / update a key
$h.Remove('Age')                 # delete a key
$h.Clear()                       # empty it

# Iterate a hashtable (use .GetEnumerator() to get key/value pairs):
foreach ($pair in $h.GetEnumerator()) {
    "$($pair.Key) = $($pair.Value)"
}

# ⚠️ Plain @{} is UNORDERED. For predictable order use [ordered]:
$ordered = [ordered]@{ First = 1; Second = 2; Third = 3 }
#   [ordered] preserves insertion order — important for tables, JSON, splatting.
```

Hashtables are the backbone of **splatting** (section 17), **calculated properties** (section 3), and quick lookups. They're not the same as `[PSCustomObject]` (section 12) — hashtables are key/value maps; PSCustomObjects are records with properties for the pipeline.

---

## 8. Control Flow

🟡 **Intermediate**

### `if / elseif / else`

```powershell
$n = 7
if ($n -gt 10) {
    'big'
} elseif ($n -gt 5) {
    'medium'
} else {
    'small'
}

# if is an EXPRESSION — you can assign its result (no ternary needed in 5.1):
$label = if ($n % 2 -eq 0) { 'even' } else { 'odd' }
```

### `switch` — far more powerful than C's switch

```powershell
switch ($n) {
    1       { 'one' }
    7       { 'seven'; break }     # break stops after a match
    default { 'other' }
}

# ⚠️ Without break, switch tests EVERY case and runs ALL that match:
switch (5) {
    { $_ -gt 0 } { 'positive' }   # script-block case = a condition test
    { $_ -lt 10 } { 'small' }     # this ALSO runs (no break) -> both print
}

# -Wildcard, -Regex, -CaseSensitive modifiers:
switch -Wildcard ('report.txt') {
    '*.txt' { 'text file' }
    '*.log' { 'log file' }
}
switch -Regex ('user42') {
    '^\d+$'      { 'all digits' }
    '\d+$'       { "ends in $($Matches[0])" }   # $Matches works in -Regex switch!
}

# -File reads a file LINE BY LINE (memory-friendly log scanning):
switch -Regex -File '.\app.log' {
    'ERROR'   { "ERR : $_" }
    'WARN'    { "WARN: $_" }
}

# Switch over an ARRAY processes each element:
switch (1, 2, 3) { { $_ % 2 } { "$_ is odd" } }
```

### Loops

```powershell
# for — classic counter loop
for ($i = 0; $i -lt 5; $i++) { $i }

# foreach STATEMENT — iterate an existing collection (fast, in-memory)
$nums = 1..5
foreach ($n in $nums) { $n * 2 }

# while — test BEFORE each iteration
$i = 0
while ($i -lt 3) { $i; $i++ }

# do/while — run, THEN test (runs at least once)
$i = 0
do { $i; $i++ } while ($i -lt 3)

# do/until — run, then loop UNTIL the condition becomes true
$i = 0
do { $i; $i++ } until ($i -ge 3)

# break / continue
foreach ($n in 1..10) {
    if ($n -eq 8) { break }       # exit the loop entirely
    if ($n % 2)   { continue }    # skip to next iteration (odd numbers)
    $n                            # prints 2 4 6
}

# 1..N range operator, and reverse:
1..5            # 1 2 3 4 5
5..1            # 5 4 3 2 1
'a'..'e'        # a b c d e  (7+; char ranges)
```

### `foreach` statement vs `ForEach-Object` cmdlet — the difference

```powershell
# foreach STATEMENT: needs the whole collection in memory; faster per-item;
#   NOT usable in the middle of a pipeline.
foreach ($p in Get-Process) { $p.Name }

# ForEach-Object CMDLET: streams the pipeline one object at a time; lower memory;
#   the only option mid-pipeline.
Get-Process | ForEach-Object { $_.Name }

# Rule of thumb:
#   - Already have an array and want speed     -> foreach statement
#   - Processing a streaming/huge pipeline     -> ForEach-Object
#   - Need parallelism                          -> ForEach-Object -Parallel (7+)
```

---

## 9. Functions & Advanced Functions

🟡🔴 **Intermediate → Advanced**

### Basic functions

```powershell
function Get-Greeting {
    param(
        [string] $Name = 'World'      # typed param with a default value
    )
    "Hello, $Name!"
}

Get-Greeting                 # Hello, World!
Get-Greeting -Name 'Ada'     # Hello, Ada!
Get-Greeting Ada             # positional also works
```

### The `param()` block, typing, defaults

```powershell
function New-User {
    param(
        [string]   $Name,
        [int]      $Age = 18,                 # default value
        [string[]] $Roles = @('reader'),      # array parameter
        [switch]   $Active                    # switch flag: -Active sets it $true
    )

    [PSCustomObject]@{          # return an OBJECT (section 12)
        Name   = $Name
        Age    = $Age
        Roles  = $Roles
        Active = [bool]$Active
    }
}

New-User -Name 'Ada' -Age 36 -Roles 'admin','editor' -Active
```

### Output vs `return`

```powershell
function Get-Numbers {
    1            # ANY expression that isn't captured becomes OUTPUT
    2            # so this function outputs 1, 2, 3 (a stream of three objects)
    3
    return 4     # 'return' outputs 4 AND exits — it does NOT replace the above!
}
Get-Numbers      # 1 2 3 4   <- common surprise: stray output leaks into results

# ⚠️ "Accidental output" is a top PowerShell bug. If a line produces a value and
#    you don't assign/suppress it, it goes down the pipeline. Suppress with:
$null = Some-Command           # preferred
Some-Command | Out-Null        # also works (slightly slower)
[void](Some-Command)           # also works
```

### Advanced functions — `[CmdletBinding()]`

Adding `[CmdletBinding()]` turns a plain function into an **advanced function** that behaves like a real cmdlet: it gets the **common parameters** for free (`-Verbose`, `-Debug`, `-ErrorAction`, `-ErrorVariable`, `-WhatIf`/`-Confirm` if you opt in), proper pipeline support, and stricter parameter binding.

```powershell
function Get-BigFile {
    [CmdletBinding()]                          # makes this an advanced function
    param(
        [Parameter(Mandatory)]                 # -Path is required; prompts if missing
        [string] $Path,

        [Parameter()]
        [ValidateRange(1, 1024)]               # only 1..1024 accepted
        [int] $MinMB = 100,

        [ValidateSet('Name', 'Size', 'Date')]  # restrict to these literal values (tab-completes!)
        [string] $SortBy = 'Size'
    )

    Write-Verbose "Scanning $Path for files >= $MinMB MB"   # only shows with -Verbose

    Get-ChildItem -Path $Path -Recurse -File |
        Where-Object { $_.Length -ge ($MinMB * 1MB) } |
        Sort-Object $SortBy
}

Get-BigFile -Path C:\Temp -MinMB 50 -Verbose
```

### Pipeline-aware functions — `ValueFromPipeline` + `begin/process/end`

```powershell
function Convert-ToUpper {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline)]        # accept input FROM the pipeline
        [string] $Text
    )

    begin   { Write-Verbose 'Starting' }      # runs ONCE before any input
    process { $Text.ToUpper() }               # runs ONCE PER pipeline item ($Text = current)
    end     { Write-Verbose 'Done' }          # runs ONCE after all input
}

'a', 'b', 'c' | Convert-ToUpper               # A B C
#   Without process{}, only the LAST piped item is processed — classic beginner trap.

# Bind a whole property by name from incoming objects:
function Stop-ByName {
    param([Parameter(ValueFromPipelineByPropertyName)][string] $Name)
    process { "Would stop: $Name" }
}
Get-Process | Select-Object Name | Stop-ByName   # .Name auto-binds to -Name
```

### Validation attributes

```powershell
function Set-Level {
    param(
        [ValidateNotNullOrEmpty()] [string] $Name,
        [ValidateRange(0, 100)]    [int]    $Percent,
        [ValidateSet('Low','Med','High')]  [string] $Priority,
        [ValidatePattern('^\d{4}-\d{2}-\d{2}$')] [string] $Date,
        [ValidateScript({ Test-Path $_ })] [string] $File,
        [ValidateCount(1, 5)]      [string[]] $Tags,
        [ValidateLength(3, 20)]    [string] $Label
    )
    "OK"
}
#   These run automatically at bind time and give clean errors — far better than
#   hand-written `if (...) { throw }` checks scattered in the body.
```

### Comment-based help

```powershell
function Get-Square {
<#
.SYNOPSIS
    Returns the square of a number.
.DESCRIPTION
    Multiplies the input by itself. Demonstrates comment-based help, which
    Get-Help reads automatically — no external files needed.
.PARAMETER Number
    The number to square.
.EXAMPLE
    Get-Square -Number 5
    Returns 25.
.NOTES
    Author: you
#>
    param([Parameter(Mandatory)][double] $Number)
    $Number * $Number
}

Get-Help Get-Square -Full        # your help block shows up just like a cmdlet's!
```

---

## 10. Error Handling

🔴 **Advanced**

### Terminating vs non-terminating errors — *the* concept to grasp

PowerShell has **two kinds of error**:

- **Non-terminating** (the default for most cmdlets): the cmdlet reports an error to the error stream and **keeps going** to the next pipeline item. `try/catch` **will NOT catch these by default**.
- **Terminating**: stops execution. Caught by `try/catch`. Produced by `throw`, by .NET exceptions, and by non-terminating errors **promoted** via `-ErrorAction Stop`.

```powershell
# This does NOT enter catch — Get-Item raises a NON-terminating error:
try { Get-Item 'C:\nope' } catch { 'caught!' }     # "caught!" does NOT print

# Promote it to terminating with -ErrorAction Stop, and NOW catch works:
try { Get-Item 'C:\nope' -ErrorAction Stop } catch { 'caught!' }   # prints
```

### `$ErrorActionPreference` and `-ErrorAction`

```powershell
# Per-command (best practice — local and explicit):
Get-Item 'C:\nope' -ErrorAction Stop            # Stop = make it terminating
Get-Item 'C:\nope' -ErrorAction SilentlyContinue # swallow it, keep going
Get-Item 'C:\nope' -ErrorAction Ignore          # swallow AND don't add to $Error

# Common -ErrorAction values:
#   Stop · Continue (default) · SilentlyContinue · Ignore · Inquire

# Global default for a whole script (use sparingly — affects everything):
$ErrorActionPreference = 'Stop'
#   Many people set this at the top of scripts so cmdlet errors actually halt.
```

### `try / catch / finally`

```powershell
try {
    $data = Get-Content 'C:\config.json' -Raw -ErrorAction Stop
    $obj  = $data | ConvertFrom-Json
}
catch [System.IO.FileNotFoundException] {     # catch a SPECIFIC exception type
    Write-Warning "Config missing: $($_.Exception.Message)"
}
catch [System.Management.Automation.ItemNotFoundException] {
    Write-Warning "Path not found"
}
catch {                                        # catch-all (must come LAST)
    Write-Error "Unexpected: $($_.Exception.Message)"
    # $_ inside catch is the ErrorRecord:
    #   $_.Exception          -> the .NET exception
    #   $_.Exception.GetType().FullName  -> use this to discover the type to catch
    #   $_.ScriptStackTrace   -> where it happened
    #   $_.InvocationInfo     -> line/command context
}
finally {
    'Always runs — cleanup goes here'          # runs whether or not there was an error
}
```

### `throw`, `Write-Error`, `$Error`

```powershell
throw 'Something broke'                         # raise a TERMINATING error
throw [System.ArgumentException]::new('bad arg')

Write-Error 'Soft, non-terminating error'      # error stream; doesn't stop by default
Write-Error 'stop me' -ErrorAction Stop        # ...unless you say so

$Error                                          # array of recent errors this session
$Error[0]                                        # most recent error (the ErrorRecord)
$Error[0].Exception.GetType().FullName           # find the type to catch it next time
$Error.Clear()                                   # wipe the history
```

### `-ErrorVariable`

```powershell
# Capture errors into a variable WITHOUT using try/catch (note: no $ on the name):
Get-Item 'C:\nope','C:\Windows' -ErrorAction SilentlyContinue -ErrorVariable myErrs
$myErrs.Count          # how many failed
$myErrs[0]             # the first error record
#   Use +name to APPEND to an existing error variable: -ErrorVariable +myErrs
```

### Native commands: `$LASTEXITCODE` vs `$?` vs exceptions

🔴 This trips up everyone. External `.exe`s don't throw PowerShell exceptions and don't set `$?` the way cmdlets do — they return an **integer exit code** in `$LASTEXITCODE`.

```powershell
# A native command FAILED but try/catch won't catch it (it didn't "throw"):
try { cmd /c exit 1 } catch { 'never reached' }
$LASTEXITCODE          # 1  <- THIS is how you detect native failure

# Correct pattern for native commands:
git pull
if ($LASTEXITCODE -ne 0) { throw "git pull failed with code $LASTEXITCODE" }

# $? is a BOOLEAN about the last operation succeeding; subtle differences:
#   - For cmdlets: $? reflects whether an error occurred.
#   - For native exes: $? is $true if exit code is 0, else $false.
#     But $? gets reset by the very next command, so capture it IMMEDIATELY.
ping bad.host.invalid > $null
$ok = $?               # capture right away
$code = $LASTEXITCODE  # the actual exit code

# ⚡ 7.4+ adds  $PSNativeCommandUseErrorActionPreference = $true  which can make
#    native non-zero exits raise terminating errors that try/catch CAN catch.
#    Not available in 5.1.
```

---

## 11. Filesystem & Providers

🟡 **Intermediate**

### `Get-ChildItem` (the workhorse — `ls`/`dir`/`gci`)

```powershell
Get-ChildItem                              # current dir
Get-ChildItem C:\Logs -Recurse             # recurse into subfolders
Get-ChildItem -File                        # files only
Get-ChildItem -Directory                   # folders only
Get-ChildItem -Hidden                      # hidden items only
Get-ChildItem -Force                       # include hidden + system items
Get-ChildItem -Filter *.log                # -Filter is FAST (provider-level)
Get-ChildItem -Include *.log,*.txt -Recurse  # -Include needs -Recurse or a wildcard path
Get-ChildItem -Exclude *.tmp
Get-ChildItem -Depth 2 -Recurse            # limit recursion depth (PS 5+)

# Combine with the pipeline:
Get-ChildItem -Recurse -File |
    Where-Object Length -gt 10MB |
    Sort-Object Length -Descending
```

> **⚡ `-Filter` vs `-Include`:** `-Filter` is handled by the underlying provider/OS and is much faster, but supports only a single simple wildcard. `-Include`/`-Exclude` are applied by PowerShell *after* enumeration (slower) but support multiple patterns. Prefer `-Filter` when you can.

### Reading & writing files — *watch the encoding!*

```powershell
# READ:
Get-Content .\file.txt                      # array of LINES (one string per line)
Get-Content .\file.txt -Raw                 # the WHOLE file as ONE string (often what you want)
Get-Content .\file.txt -TotalCount 10       # first 10 lines (like head)
Get-Content .\file.txt -Tail 10             # last 10 lines (like tail)
Get-Content .\app.log -Wait -Tail 0         # follow/stream new lines (like tail -f)

# WRITE:
Set-Content  .\out.txt -Value 'replace whole file'
Add-Content  .\out.txt -Value 'append a line'
'data' | Out-File .\out.txt                 # Out-File renders objects like the console

# ⚡⚡ ENCODING — the single biggest cross-version bug:
#   - Windows PowerShell 5.1 defaults to UTF-16LE ('Unicode') or system ANSI.
#   - PowerShell 7 defaults to UTF-8 (no BOM).
#   A file written in 5.1 can look corrupted/garbled when read elsewhere.
# ALWAYS specify -Encoding explicitly when it matters:
Set-Content .\out.txt -Value 'hi' -Encoding utf8
Out-File   .\out.txt -InputObject 'hi' -Encoding utf8
#   In 5.1, 'utf8' WRITES A BOM. For no-BOM UTF-8 in 5.1 you need utf8NoBOM-style
#   workarounds (e.g. [System.IO.File]::WriteAllText). In 7, 'utf8' is already no-BOM
#   and 'utf8BOM' opts into a BOM.
```

### Creating / removing / copying / moving

```powershell
New-Item -ItemType File      -Path .\new.txt
New-Item -ItemType Directory -Path .\logs\2026 -Force   # -Force makes parent dirs (like mkdir -p)
#   ⚠️ NEVER use New-Item -Force on an EXISTING file — it truncates its contents.

Remove-Item .\old.txt
Remove-Item .\temp -Recurse -Force          # delete a folder tree
Remove-Item .\temp -Recurse -WhatIf         # PREVIEW what would be deleted (safe!)

Copy-Item   .\a.txt .\b.txt
Copy-Item   .\src -Destination .\dst -Recurse
Move-Item   .\a.txt .\archive\
Rename-Item .\old.txt -NewName new.txt
```

### Path helpers — use these, never string-concat paths

```powershell
Test-Path .\maybe.txt                       # does it exist? -> bool
Test-Path .\folder -PathType Container       # is it a directory?
Test-Path .\file   -PathType Leaf            # is it a file?

Join-Path C:\Users 'Ada' 'docs'             # C:\Users\Ada\docs (handles slashes; cross-platform)
Resolve-Path .\..\sibling                    # turn relative into absolute (must exist)
Split-Path C:\a\b\c.txt                      # C:\a\b      (parent)
Split-Path C:\a\b\c.txt -Leaf                # c.txt       (filename)
Split-Path C:\a\b\c.txt -Extension           # .txt        (PS 6+)
Split-Path C:\a\b\c.txt -Qualifier           # C:          (drive)

# Robust, cross-platform path building (do this, NOT "$dir\$file"):
$full = Join-Path $PSScriptRoot 'config' 'app.json'
```

### The provider model — drives that aren't disks

PowerShell exposes many data stores through the *same* cmdlets (`Get-ChildItem`, `Get-Item`, etc.) via **providers**.

```powershell
Get-PSProvider                              # FileSystem, Registry, Environment, ...
Get-PSDrive                                  # C:, HKLM:, HKCU:, Env:, Variable:, Function:, Alias:

# Environment as a drive:
Get-ChildItem Env:
$env:PATH                                    # shortcut for the Env: provider

# Variables / functions / aliases as drives:
Get-ChildItem Variable:
Get-ChildItem Function:
Get-ChildItem Alias:
```

### Registry provider (Windows; `HKLM:` / `HKCU:`)

🔴 **Advanced (Windows-only)** — registry "folders" are keys; "files" are values.

```powershell
# Navigate the registry like a filesystem:
Get-ChildItem HKCU:\Software | Select-Object -First 10
Get-ChildItem HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion

# Read VALUES with Get-ItemProperty (a key's values are its "properties"):
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName, CurrentBuild, DisplayVersion

# Read a single value:
(Get-ItemProperty 'HKCU:\Environment').Path

# Write (needs the right permissions; HKLM usually needs admin):
New-Item        -Path 'HKCU:\Software\MyApp' -Force
New-ItemProperty -Path 'HKCU:\Software\MyApp' -Name 'Version' -Value '1.0' -PropertyType String
Set-ItemProperty -Path 'HKCU:\Software\MyApp' -Name 'Version' -Value '2.0'
Remove-Item     -Path 'HKCU:\Software\MyApp' -Recurse
```

---

## 12. Objects & Data Formats

🟡🔴 **Intermediate → Advanced**

### `[PSCustomObject]` — the modern way to make objects

When you need to produce structured records (for the pipeline, CSV, JSON, tables), build a `[PSCustomObject]`. It preserves property order and behaves like a first-class object.

```powershell
$person = [PSCustomObject]@{
    Name = 'Ada'
    Age  = 36
    City = 'London'
}
$person.Name                                 # Ada
$person | Get-Member                          # it has real properties

# Emit one per item to build a table/dataset:
$report = Get-ChildItem -File | ForEach-Object {
    [PSCustomObject]@{
        Name    = $_.Name
        SizeKB  = [math]::Round($_.Length / 1KB, 1)
        Modified = $_.LastWriteTime
    }
}
$report | Format-Table -AutoSize
$report | Export-Csv .\report.csv -NoTypeInformation -Encoding utf8

# Add a property to an existing object later:
$person | Add-Member -NotePropertyName Country -NotePropertyValue 'UK'
```

### CSV

```powershell
# Export objects -> CSV:
Get-Process | Select-Object Name, Id, CPU |
    Export-Csv .\procs.csv -NoTypeInformation -Encoding utf8
#   -NoTypeInformation drops the "#TYPE" header line (default already in 7; needed in 5.1).

# Import CSV -> objects (every column becomes a STRING property):
$rows = Import-Csv .\procs.csv
$rows[0].Name
$rows | Where-Object { [int]$_.Id -gt 1000 }   # cast strings back to numbers as needed

# Different delimiter:
Import-Csv .\data.tsv -Delimiter "`t"

# Convert to/from CSV text without touching disk:
$csvText = Get-Process | ConvertTo-Csv -NoTypeInformation
$csvText | ConvertFrom-Csv
```

### JSON — and the depth gotcha

```powershell
# Object -> JSON:
$obj = [PSCustomObject]@{ name = 'Ada'; tags = @('a','b'); meta = @{ x = 1 } }
$obj | ConvertTo-Json
$obj | ConvertTo-Json -Compress              # no whitespace

# ⚡⚡ DEPTH GOTCHA: ConvertTo-Json only serializes nested objects up to -Depth
#    levels (DEFAULT = 2 in 5.1!). Deeper structures get cut off and rendered
#    as a type name string. ALWAYS set -Depth for nested data:
$deep | ConvertTo-Json -Depth 10
#    (PS 7 default depth is also 2 but it WARNS when truncating; 5.1 silently truncates.)

# JSON -> object:
$data = Get-Content .\config.json -Raw | ConvertFrom-Json
$data.name
$data.tags[0]

# ⚡ -AsHashtable returns a hashtable instead of PSCustomObject — 7+ ONLY.
#    In 5.1, ConvertFrom-Json ALWAYS returns PSCustomObject (no hashtable option).
$h = Get-Content .\config.json -Raw | ConvertFrom-Json -AsHashtable   # 7+ only
```

### XML

```powershell
[xml]$xml = Get-Content .\data.xml -Raw       # cast to XmlDocument
$xml.root.item                                 # dot-navigate the tree
$xml.SelectNodes('//item[@id="3"]')            # XPath
$xml.root.item | ForEach-Object { $_.InnerText }

# Import-Clixml / Export-Clixml: round-trip FULL PowerShell objects to disk
# (preserves types — great for caching credentials/objects between runs):
Get-Process | Export-Clixml .\snapshot.xml
$restored = Import-Clixml .\snapshot.xml
```

### Formatting — and why it must come **LAST**

`Format-*` cmdlets produce **formatting instruction objects**, not data. Once you format, the real objects are gone — anything after a `Format-*` in the pipeline (Export-Csv, Where-Object, etc.) sees garbage.

```powershell
Get-Process | Format-Table Name, Id, CPU -AutoSize    # tabular (default for many objects)
Get-Process | Format-List  *                          # one property per line (great for inspecting)
Get-ChildItem | Format-Wide  Name -Column 4           # multi-column names

# ❌ WRONG — formatting before exporting corrupts the data:
Get-Process | Format-Table | Export-Csv .\bad.csv     # produces useless format objects

# ✅ RIGHT — select/compute first, FORMAT (or export) last:
Get-Process | Select-Object Name, Id, CPU | Export-Csv .\good.csv -NoTypeInformation

# Other "Out"/sink cmdlets:
Get-Process | Out-GridView                # interactive grid (Windows; needs the module)
Get-Process | Out-String                  # render to a single multi-line string
Get-Process | Out-Host                    # explicit send to screen
Get-Process | Out-File .\procs.txt        # send rendered text to a file
```

---

## 13. Running External Programs

🟡🔴 **Intermediate → Advanced**

### Calling native exes

```powershell
git status                       # just type it — exes on PATH run directly
ipconfig /all
node --version

# The call operator & runs a command whose NAME is in a variable or has spaces:
$tool = 'C:\Program Files\Git\bin\git.exe'
& $tool status
& 'C:\Program Files\App\app.exe' --flag value

# Capture output (it comes back as an array of STRINGS, one per line):
$out = git status
$out = ipconfig | Select-String 'IPv4'
```

### Argument-passing pitfalls (quoting) and `--%`

🔴 PowerShell parses your line *before* handing args to the exe, which mangles things like `=`, `;`, embedded quotes, and `@`. Two tools help:

```powershell
# 1) Pass arguments as an array — each element is one argument, no re-splitting:
$args = @('clone', '--depth', '1', 'https://example.com/repo.git')
& git $args

# 2) The stop-parsing token --% : everything AFTER it is passed VERBATIM to the
#    exe (PowerShell stops interpreting it). Use for tricky native syntax:
icacls C:\data --% /grant Users:(OI)(CI)F
cmd --% /c echo %PATH%               # %PATH% is left alone for cmd to expand

# ⚡ PowerShell 7.3+ greatly improved native argument passing (PSNativeCommandArgumentPassing).
#    5.1 has the legacy, quirkier behavior — when in doubt, use an array or --%.
```

### Output vs `$LASTEXITCODE`

```powershell
# Output (stdout) flows down the pipeline as strings; the EXIT CODE is separate:
$json = az account show --output json     # capture stdout
if ($LASTEXITCODE -ne 0) { throw 'az failed' }   # check success separately

# Redirect streams:
some.exe 1> out.txt        # stdout to file
some.exe 2> err.txt        # stderr to file
some.exe 2>&1              # merge stderr INTO stdout
#   ⚡ In 5.1, 2>&1 with native commands wraps each stderr line in an ErrorRecord
#      (NativeCommandError), which can flip $? to $false even on success. (Section 18.)
```

### `Start-Process` — launch, wait, elevate

```powershell
# Fire-and-forget launch:
Start-Process notepad

# Wait for it to finish, keep it in the same window, capture exit code:
$p = Start-Process ping -ArgumentList 'localhost' -NoNewWindow -Wait -PassThru
$p.ExitCode

# Run elevated (UAC prompt) — RunAs verb:
Start-Process pwsh -Verb RunAs -ArgumentList '-File', 'C:\admin-task.ps1'

# Redirect output to files:
Start-Process git -ArgumentList 'status' -NoNewWindow -Wait `
    -RedirectStandardOutput out.txt -RedirectStandardError err.txt
```

| `Start-Process` switch | Use |
|---|---|
| `-Wait` | Block until the process exits |
| `-NoNewWindow` | Run in the current console window |
| `-PassThru` | Return the Process object (so you can read `.ExitCode`) |
| `-Verb RunAs` | Elevate (UAC) |
| `-ArgumentList` | Arguments (array recommended) |
| `-WorkingDirectory` | Set the cwd for the child |
| `-RedirectStandard*` | Capture stdout/stderr/stdin to files |

---

## 14. Modules

🟡🔴 **Intermediate → Advanced**

A **module** packages related functions, cmdlets, and variables for reuse and distribution.

### Using modules

```powershell
Get-Module                       # currently LOADED modules
Get-Module -ListAvailable        # all INSTALLED modules on disk
Import-Module Pester             # explicitly load one (often auto-loads on first use)
Remove-Module Pester             # unload from the session

# Where PowerShell looks for modules:
$env:PSModulePath -split ';'     # (use -split ':' on Linux/macOS)
```

### Installing from the PowerShell Gallery

```powershell
# Classic (PowerShellGet v2 — works in 5.1 and 7):
Install-Module -Name Pester -Scope CurrentUser           # no admin needed
Install-Module -Name PSScriptAnalyzer -Scope CurrentUser
Find-Module *aws*                                         # search the gallery
Update-Module Pester
Uninstall-Module Pester

# ⚡ Modern (PSResourceGet / Microsoft.PowerShell.PSResourceGet) — 7+, faster:
Install-PSResource Pester
Find-PSResource *azure*
Update-PSResource Pester
#   PSResourceGet is the successor; it ships in recent PowerShell 7 and can be
#   installed into 5.1. Both pull from the same PowerShell Gallery.

# First time you may be asked to trust the gallery or install NuGet provider:
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted   # (PowerShellGet v2)
```

### Writing a simple module (`.psm1` + `.psd1` manifest)

```powershell
# --- File: MyTools\MyTools.psm1  (the code) ---
function Get-Square {
    param([Parameter(Mandatory)][double] $Number)
    $Number * $Number
}

function Get-Cube {
    param([Parameter(Mandatory)][double] $Number)
    $Number * $Number * $Number
}

# Explicitly control what the module exposes (best practice — keeps helpers private):
Export-ModuleMember -Function Get-Square, Get-Cube
```

```powershell
# --- Generate the manifest: MyTools\MyTools.psd1  (metadata) ---
New-ModuleManifest -Path .\MyTools\MyTools.psd1 `
    -RootModule 'MyTools.psm1' `
    -ModuleVersion '1.0.0' `
    -Author 'You' `
    -Description 'Math helper functions' `
    -FunctionsToExport 'Get-Square', 'Get-Cube' `
    -PowerShellVersion '5.1'

# A .psd1 is just a hashtable of metadata: version, author, RootModule,
# FunctionsToExport, RequiredModules, etc. It's what `Get-Module` reads.
```

```powershell
# Load and use your module:
Import-Module .\MyTools\MyTools.psd1
Get-Square 7        # 49
Get-Command -Module MyTools

# To make it auto-discoverable, place the MyTools folder in one of the
# $env:PSModulePath directories (e.g. Documents\PowerShell\Modules\MyTools\).
```

---

## 15. Remoting & Jobs

🔴 **Advanced**

### Remoting with `Invoke-Command` and PSSessions

PowerShell Remoting runs commands on other machines (over WinRM by default; SSH-based remoting also exists, especially cross-platform in 7).

```powershell
# One-off command on a remote machine:
Invoke-Command -ComputerName Server01 -ScriptBlock { Get-Service spooler }

# Fan out to MANY machines at once (runs in parallel across them):
Invoke-Command -ComputerName Server01, Server02, Server03 -ScriptBlock {
    [PSCustomObject]@{ Host = $env:COMPUTERNAME; Free = (Get-PSDrive C).Free }
}

# Pass LOCAL variables into the remote block with $using:
$threshold = 100MB
Invoke-Command -ComputerName Server01 -ScriptBlock {
    Get-Process | Where-Object WorkingSet -gt $using:threshold
}

# Persistent sessions (reuse state across calls):
$s = New-PSSession -ComputerName Server01
Invoke-Command -Session $s -ScriptBlock { $x = 1 }
Invoke-Command -Session $s -ScriptBlock { $x + 1 }   # 2 (state persisted)
Remove-PSSession $s

# Interactive remote shell:
Enter-PSSession -ComputerName Server01      # your prompt is now on the remote box
Exit-PSSession                               # come back

# SSH-based remoting (great cross-platform; 6+):
Invoke-Command -HostName host.example.com -UserName ada -ScriptBlock { hostname }
```

### Background jobs

```powershell
# Start-Job runs in a SEPARATE process (heavier, full isolation):
$job = Start-Job -ScriptBlock { Start-Sleep 3; Get-Date }
Get-Job                                   # list jobs and their State
Wait-Job $job                             # block until done
Receive-Job $job                          # get the results (drains the buffer)
Remove-Job $job                           # clean up

# Thread jobs — lighter weight, same process (module ThreadJob; built into 7):
$j = Start-ThreadJob -ScriptBlock { 1..3 | ForEach-Object { $_; Start-Sleep 1 } }
$j | Wait-Job | Receive-Job
```

### `ForEach-Object -Parallel` — **⚡ 7+ only**

The easiest parallelism in PowerShell — but it **does not exist in 5.1**.

```powershell
# Process items concurrently. $_ is the item. Each iteration runs in its own runspace.
1..10 | ForEach-Object -Parallel {
    Start-Sleep 1
    "Done $_ on thread $([System.Threading.Thread]::CurrentThread.ManagedThreadId)"
} -ThrottleLimit 5            # at most 5 at a time

# ⚠️ Variables from outside are NOT automatically visible — use $using: :
$prefix = 'item'
1..5 | ForEach-Object -Parallel { "$using:prefix-$_" } -ThrottleLimit 3

# In 5.1 you'd reach for Start-Job / Start-ThreadJob / runspaces instead.
```

---

## 16. Real-World Cmdlets & Tasks

🟡🔴 **Intermediate → Advanced**

### Processes & services

```powershell
Get-Process                              # all processes
Get-Process chrome                        # by name
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
Stop-Process -Name notepad               # kill by name
Stop-Process -Id 1234 -Force
Get-Process -Id $PID                      # the current PowerShell process

Get-Service                               # all services (Windows)
Get-Service -Name spooler
Get-Service | Where-Object Status -eq 'Stopped'
Start-Service spooler                      # needs admin
Stop-Service  spooler
Restart-Service spooler
Set-Service spooler -StartupType Automatic
```

### System info with `Get-CimInstance` (modern WMI)

🔴 `Get-CimInstance` replaces the old `Get-WmiObject` (don't use WMI cmdlets in new code). It queries the OS, hardware, disks, etc.

```powershell
Get-CimInstance Win32_OperatingSystem |
    Select-Object Caption, Version, OSArchitecture, LastBootUpTime

Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" |    # 3 = local fixed disk
    Select-Object DeviceID,
        @{ n='SizeGB'; e={ [math]::Round($_.Size/1GB, 1) } },
        @{ n='FreeGB'; e={ [math]::Round($_.FreeSpace/1GB, 1) } }

Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model, TotalPhysicalMemory
Get-CimInstance Win32_Processor      | Select-Object Name, NumberOfCores, MaxClockSpeed
Get-CimInstance Win32_BIOS           | Select-Object SerialNumber, SMBIOSBIOSVersion

# Remote CIM:
Get-CimInstance Win32_OperatingSystem -ComputerName Server01
```

### HTTP / REST with `Invoke-RestMethod` & `Invoke-WebRequest`

```powershell
# Invoke-RestMethod (irm): for APIs — auto-parses JSON/XML into OBJECTS:
$repo = Invoke-RestMethod 'https://api.github.com/repos/PowerShell/PowerShell'
$repo.stargazers_count          # already an object, no manual ConvertFrom-Json

# GET with headers:
$headers = @{ Authorization = "Bearer $token"; Accept = 'application/json' }
$me = Invoke-RestMethod 'https://api.example.com/me' -Headers $headers

# POST JSON:
$body = @{ name = 'Ada'; role = 'admin' } | ConvertTo-Json
Invoke-RestMethod 'https://api.example.com/users' -Method Post `
    -Body $body -ContentType 'application/json'

# Invoke-WebRequest (iwr): for raw responses — status code, headers, raw content:
$resp = Invoke-WebRequest 'https://example.com'
$resp.StatusCode                 # 200
$resp.Headers['Content-Type']
$resp.Content                    # raw body string

# Download a file:
Invoke-WebRequest 'https://example.com/file.zip' -OutFile .\file.zip

# ⚡ In 5.1, iwr uses Internet Explorer's engine for parsing (-UseBasicParsing
#    avoids that and is recommended). In 7, -UseBasicParsing is the default/no-op.
Invoke-WebRequest 'https://example.com' -UseBasicParsing   # safe in both
```

### Dates & date math

```powershell
Get-Date                                          # now
Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
(Get-Date).AddDays(7)                              # a week from now
(Get-Date).AddHours(-3)                            # 3 hours ago
(Get-Date) - (Get-Date '2026-01-01')              # a TimeSpan (difference)
((Get-Date) - $start).TotalSeconds                 # elapsed seconds
[datetime]'2026-12-25' -gt (Get-Date)              # date comparison
Get-Date -UFormat '%Y-%m-%dT%H:%M:%SZ'             # unix-style format
(Get-Date).ToString('o')                           # ISO 8601 round-trip
[datetimeoffset]::UtcNow                            # UTC with offset
```

### Archives, clipboard

```powershell
Compress-Archive -Path .\logs\* -DestinationPath .\logs.zip
Compress-Archive -Path .\a.txt -Update -DestinationPath .\logs.zip   # add to existing
Expand-Archive   -Path .\logs.zip -DestinationPath .\unzipped -Force

Set-Clipboard 'copied text'                        # put text on the clipboard
Get-Process | Set-Clipboard                         # copy rendered output
$x = Get-Clipboard                                  # read clipboard text
```

### Scheduled tasks (Windows)

```powershell
# Run a script every day at 2am:
$action  = New-ScheduledTaskAction  -Execute 'pwsh.exe' -Argument '-File C:\Scripts\backup.ps1'
$trigger = New-ScheduledTaskTrigger -Daily -At 2am
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -RunLevel Highest
Register-ScheduledTask -TaskName 'NightlyBackup' -Action $action -Trigger $trigger -Principal $principal

Get-ScheduledTask -TaskName 'NightlyBackup'
Start-ScheduledTask     -TaskName 'NightlyBackup'    # run it now
Unregister-ScheduledTask -TaskName 'NightlyBackup' -Confirm:$false
```

### Credentials & secrets — never hardcode

🔴 Don't put passwords/tokens in scripts. Use `Get-Credential`, `SecureString`, env vars, or a secret vault.

```powershell
# Prompt securely for username + password:
$cred = Get-Credential
$cred.UserName
# $cred.Password is a SecureString (not plain text); pass it to cmdlets that accept -Credential:
Invoke-Command -ComputerName Server01 -Credential $cred -ScriptBlock { whoami }

# Save a credential ENCRYPTED to disk (DPAPI: only the SAME user on the SAME machine
# can read it back — 5.1/Windows). Good for unattended scripts on one box:
$cred | Export-Clixml .\cred.xml
$cred = Import-Clixml .\cred.xml

# The cross-platform, recommended modern way — SecretManagement + a vault:
#   Install-Module Microsoft.PowerShell.SecretManagement, Microsoft.PowerShell.SecretStore
#   Set-Secret -Name ApiToken -Secret 'xyz'
#   $token = Get-Secret -Name ApiToken -AsPlainText

# ⚠️ ConvertTo-SecureString -AsPlainText -Force exists but storing the plaintext
#    in the script defeats the purpose. Prefer prompts, env vars, or a vault.
```

---

## 17. Scripting Best Practices

🔴 **Advanced**

### A well-formed script skeleton

```powershell
#requires -Version 7.0                 # refuse to run on older PowerShell (great guard!)
[CmdletBinding(SupportsShouldProcess)] # gives -WhatIf / -Confirm AND -Verbose/-Debug
param(
    [Parameter(Mandatory)]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string] $Path,

    [ValidateRange(1, 365)]
    [int] $OlderThanDays = 30
)

Set-StrictMode -Version Latest          # catch typos / undefined vars
$ErrorActionPreference = 'Stop'         # make cmdlet errors actually terminate

# Use Write-Verbose for progress (shows only with -Verbose), NOT Write-Host:
Write-Verbose "Cleaning files older than $OlderThanDays days in $Path"

$cutoff = (Get-Date).AddDays(-$OlderThanDays)
Get-ChildItem -Path $Path -Recurse -File |
    Where-Object LastWriteTime -lt $cutoff |
    ForEach-Object {
        # ShouldProcess respects -WhatIf and prompts with -Confirm automatically:
        if ($PSCmdlet.ShouldProcess($_.FullName, 'Delete')) {
            Remove-Item $_.FullName
        }
    }
```

```powershell
# Try it safely first — -WhatIf shows what WOULD happen, changing nothing:
.\Clear-OldFiles.ps1 -Path C:\Temp -OlderThanDays 90 -WhatIf
.\Clear-OldFiles.ps1 -Path C:\Temp -OlderThanDays 90 -Confirm   # prompt per item
```

### Output streams: use the right one

```powershell
Write-Output  $data        # the DATA / success stream — pipeline-able, capturable
Write-Verbose 'progress'   # verbose stream — only with -Verbose
Write-Debug   'state=...'  # debug stream — only with -Debug
Write-Warning 'careful'    # warning stream
Write-Error   'failed'     # error stream
Write-Information 'fyi'    # information stream (6>)
Write-Progress -Activity 'Copying' -PercentComplete 50   # progress bar

# ⚠️ Write-Host writes straight to the host (screen) and is NOT capturable as
#    data (pre-5.1) — never use it to RETURN values from a function. It's fine
#    ONLY for deliberately user-facing console messages (colored banners, prompts).
Write-Host 'User-facing message' -ForegroundColor Green
```

| Stream | Cmdlet | Shown by default? | Redirect # |
|---|---|---|---|
| Success/Output | `Write-Output` | Yes | 1 |
| Error | `Write-Error` | Yes | 2 |
| Warning | `Write-Warning` | Yes | 3 |
| Verbose | `Write-Verbose` | No (`-Verbose`) | 4 |
| Debug | `Write-Debug` | No (`-Debug`) | 5 |
| Information | `Write-Information` / `Write-Host` | Yes | 6 |

### Splatting — clean, readable parameter passing

Instead of a long line with backtick continuations, put parameters in a hashtable and **splat** with `@`.

```powershell
# Hard to read:
Copy-Item -Path .\src -Destination .\dst -Recurse -Force -Verbose -ErrorAction Stop

# Splatted — a hashtable, then @name (note: @ not $ at the call site):
$params = @{
    Path        = '.\src'
    Destination = '.\dst'
    Recurse     = $true
    Force       = $true
    Verbose     = $true
    ErrorAction = 'Stop'
}
Copy-Item @params

# Splat an ARRAY for positional args:
$files = @('a.txt', 'b.txt')
Remove-Item @files

# Conditionally include a parameter (super handy):
$opt = @{}
if ($Recurse) { $opt['Recurse'] = $true }
Get-ChildItem $Path @opt
```

### General rules

- **Use approved verbs** (`Get-Verb`). Unapproved verbs cause an import warning and hurt discoverability.
- **Full cmdlet names + named parameters** in scripts. No `gci`, `?`, `%`, or positional guessing.
- **`Set-StrictMode -Version Latest`** at the top of serious scripts.
- **`[CmdletBinding()]` + `param()` with validation** on every reusable function.
- **Return objects, not formatted text.** Let the caller format. Never `Format-*` inside a function that returns data.
- **Comment-based help** on public functions.
- **Idempotency & `-WhatIf`** for anything destructive.
- **`#requires`** to assert version/modules/admin (`#requires -RunAsAdministrator`).

### `PSScriptAnalyzer` — the linter (strongly recommended)

```powershell
Install-Module PSScriptAnalyzer -Scope CurrentUser

Invoke-ScriptAnalyzer -Path .\myscript.ps1          # lint one file
Invoke-ScriptAnalyzer -Path .\src -Recurse          # lint a project
Invoke-ScriptAnalyzer -Path .\myscript.ps1 -Fix     # auto-fix what it can

# It flags aliases in scripts, unapproved verbs, plaintext passwords, missing
# ShouldProcess on state-changing verbs, $null on the wrong side of -eq, etc.
# Pester (Install-Module Pester) is the standard TESTING framework — pair them.
```

---

## 18. The 5.1 vs 7 Gotchas Section

🔴 The cross-version traps that waste the most time. Keep this list handy.

**1. File encoding defaults differ.**
5.1 writes UTF-16LE / system "Default" (ANSI); 7 writes UTF-8 no-BOM. A CSV/JSON/text file produced in one can be garbled in the other or in non-PowerShell tools. **Fix:** always pass `-Encoding utf8` on `Set-Content`/`Out-File`/`Export-Csv`. Note `utf8` *adds a BOM in 5.1* but is *no-BOM in 7* — for guaranteed no-BOM UTF-8 in 5.1, use `[System.IO.File]::WriteAllText($path, $text, [System.Text.UTF8Encoding]::new($false))`.

**2. `2>&1` with native commands wraps stderr as ErrorRecords.**
In 5.1, redirecting a native program's stderr produces `NativeCommandError` objects (not plain strings) and can set `$?` to `$false` even on success. **Fix:** check `$LASTEXITCODE` for native success; in 7, native stderr handling is cleaner. To get plain text either way, capture with `2>&1 | ForEach-Object { "$_" }` or compare on exit code.

**3. `ConvertFrom-Json -AsHashtable` is 7-only.**
In 5.1 `ConvertFrom-Json` *always* returns `[PSCustomObject]` — there is no hashtable option, and very deep JSON can hit recursion limits. **Fix:** treat results as objects (`.prop`), or write a manual converter; upgrade to 7 if you need hashtables.

**4. `ConvertTo-Json` default depth = 2.**
Nested structures get truncated. 5.1 truncates **silently**; 7 emits a warning. **Fix:** always `-Depth N` for nested data.

**5. Parallelism is 7-only.**
`ForEach-Object -Parallel` and the `-Parallel` option don't exist in 5.1. **Fix on 5.1:** `Start-Job`, `Start-ThreadJob` (install the ThreadJob module), or runspaces.

**6. Operators missing in 5.1.**
Ternary `? :`, null-coalescing `??` / `??=`, null-conditional `?.` / `?[]`, and the pipeline chain operators `&&` / `||` are **7+ only** and cause **parse errors** in 5.1 (the whole script fails to load, not just that line). **Fix:** use `if/else`, explicit `$null` checks, and `$LASTEXITCODE` tests on 5.1.

**7. Here-string indentation rules.**
The closing `"@` / `'@` must be at the **very start of its line** (column 0). If you indent it to match surrounding code, you get a parse error. The opening `@"` / `@'` must be the last token on its line. (Same in both versions, but a frequent stumble.)

**8. `Where`/`ForEach` method syntax differs.**
The `.Where({...})` and `.ForEach({...})` *methods* on collections exist in 5.1+ (PS 4+), but advanced modes and parallel are 7-only. Don't confuse the *method* `.Where({})` with the *cmdlet* `Where-Object`.

**9. `Invoke-WebRequest` parser.**
5.1's `iwr` uses the IE engine and can hang/error without `-UseBasicParsing`. 7 always uses basic parsing. **Fix:** always pass `-UseBasicParsing` for portability (it's a harmless no-op in 7).

**10. Module/profile paths are separate.**
5.1 uses `WindowsPowerShell\`; 7 uses `PowerShell\` under Documents. Your aliases/functions/profile do **not** carry over automatically. Configure each, or dot-source a shared file from both profiles.

**11. Case sensitivity on Linux/macOS (7).**
PowerShell *operators* stay case-insensitive by default, but the **filesystem** on Linux/macOS is case-sensitive — `./MyScript.ps1` ≠ `./myscript.ps1`. Watch this when scripts move cross-platform.

**12. Some Windows-only cmdlets aren't in 7 natively.**
A few classic modules (e.g., older WMI cmdlets, some `*-Ws*`) target Windows .NET Framework. 7 covers most via the `WindowsCompatibility`/`CimCmdlets` path, and `Get-WmiObject` is gone in 7 (use `Get-CimInstance`).

---

## 19. Study Path & Build-to-Learn Projects

### Suggested learning order

1. **Week 1 — Think in objects.** Live in the console. For every command, pipe to `Get-Member` and `Get-Help -Examples`. Master `Get-`/`Select-Object`/`Where-Object`/`Sort-Object`. Set your `$PROFILE` and execution policy. (Sections 1–4.)
2. **Week 2 — Language fluency.** Operators, strings/regex, arrays & hashtables, control flow. Write small one-off scripts. (Sections 5–8.)
3. **Week 3 — Real functions.** `param()`, `[CmdletBinding()]`, validation, pipeline input with `begin/process/end`, comment-based help. (Section 9.)
4. **Week 4 — Robustness.** Error handling (terminating vs not, `try/catch`, `$LASTEXITCODE`), filesystem & providers, objects/CSV/JSON, calling external tools. (Sections 10–13.)
5. **Week 5 — Scale up.** Modules, jobs/remoting/parallel, real-world cmdlets (CIM, REST, scheduled tasks, credentials). (Sections 14–16.)
6. **Ongoing — Professionalize.** Best practices, `PSScriptAnalyzer`, Pester tests, splatting, `-WhatIf`. Re-read the gotchas (Section 18) every time something behaves oddly. (Sections 17–18.)

### Build-to-learn projects (in rough difficulty order)

1. **System info report (`Get-SystemReport.ps1`)** 🟢
   Collect OS, CPU, RAM, disk free space, uptime, top-5 processes by memory via `Get-CimInstance` + `Get-Process`. Output `[PSCustomObject]`s; export to CSV/JSON/HTML (`ConvertTo-Html`). *Teaches:* objects, CIM, formatting vs export, calculated properties.

2. **Bulk file renamer (`Rename-Batch.ps1`)** 🟢🟡
   Rename/normalize files in a folder by regex (e.g., prefix dates, lowercase, strip spaces). Support `-WhatIf`. *Teaches:* `Get-ChildItem`, `-replace`/regex, `Rename-Item`, `ShouldProcess`, validation.

3. **Log parser (`Parse-Log.ps1`)** 🟡
   Stream a large log with `switch -Regex -File` or `Get-Content -ReadCount`, extract level/timestamp/message into objects, then group/count errors by hour and emit a summary. *Teaches:* regex `$Matches`, streaming for memory, `Group-Object`/`Measure-Object`.

4. **REST API client module (`GitHubTools.psm1`)** 🟡🔴
   Wrap `Invoke-RestMethod` into functions like `Get-GhRepo`, `Get-GhIssues` with `[CmdletBinding()]`, headers/auth from a secret, pagination, and typed output. Ship it as a module with a `.psd1`. *Teaches:* REST, modules, secrets, advanced functions, error handling.

5. **Backup / cleanup automation (`Invoke-Backup.ps1`)** 🟡🔴
   Compress a folder with a timestamped name (`Compress-Archive`), copy to a target, then prune backups older than N days. Register it as a scheduled task. Full `[CmdletBinding(SupportsShouldProcess)]`, `Write-Verbose`, strict mode, robust error handling, `-Encoding`. *Teaches:* archives, dates, scheduled tasks, production scripting.

6. **Dev-setup automation (`Setup-DevBox.ps1`)** 🔴
   Idempotently install tools via `winget`/`Install-Module`, set env vars, write a `$PROFILE`, configure git. Detect 5.1 vs 7 and branch (`#requires -Version 7.0`), check `$LASTEXITCODE` after every native call, and lint the result with `PSScriptAnalyzer`. *Teaches:* native commands, idempotency, version detection, end-to-end best practices.

### Quick reference — the cmdlets you'll reach for daily

| Task | Cmdlet(s) |
|---|---|
| Discover | `Get-Command`, `Get-Help`, `Get-Member`, `Get-Verb` |
| Shape data | `Select-Object`, `Where-Object`, `Sort-Object`, `Group-Object`, `Measure-Object`, `ForEach-Object` |
| Files | `Get-ChildItem`, `Get-Content`, `Set-Content`, `Test-Path`, `Join-Path`, `Copy/Move/Remove-Item` |
| Objects/data | `[PSCustomObject]`, `Import/Export-Csv`, `ConvertTo/From-Json`, `Format-Table/List` |
| System | `Get-Process`, `Get-Service`, `Get-CimInstance`, `Start-Process` |
| Web | `Invoke-RestMethod`, `Invoke-WebRequest` |
| Quality | `Set-StrictMode`, `Invoke-ScriptAnalyzer`, `Get-Help -Examples` |

---

> **Final advice:** Whenever a command surprises you, do three things: (1) pipe it to `Get-Member` to see what it *really* returns, (2) check `$PSVersionTable` to confirm 5.1 vs 7, and (3) read `Get-Help <cmd> -Examples`. Those three habits will carry you from beginner to advanced faster than anything else. Think in **objects**, write **functions that emit objects**, and let formatting happen last.
