# Windows CMD & Batch Scripting — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone who has to read, write, fix, or live with `cmd.exe` batch scripts (`.bat` / `.cmd` files) on Windows — sysadmins, developers wiring up build/CI scripts, people maintaining legacy automation, and anyone who needs a double-clickable script that "just works" on a vanilla Windows box. No prior batch knowledge is assumed; the guide goes from "what is a command prompt" all the way to delayed expansion, escaping hell, and self-elevation. Every concept comes with runnable, commented code.
>
> **Version note:** This guide targets **Windows 11 and the classic `cmd.exe` command interpreter as of 2026**. Be honest with yourself about what CMD is: it is **legacy and effectively frozen**. Microsoft has not added meaningful features to the batch language in well over a decade and has publicly positioned **PowerShell** as the modern, supported successor for new automation. CMD/batch is *not* deprecated-and-removed — it ships in every Windows install, runs build scripts, CI steps, double-clickable `.bat` installers, and decades of legacy automation — but you should not reach for it to start a *new*, non-trivial project. If you are writing something new and complex, **use PowerShell** (see the PowerShell guide in this library). Use batch when: you need a tiny, dependency-free, double-clickable script; you are patching existing batch; or you are constrained to an environment where PowerShell is locked down. Version-specific quirks (delayed expansion, Unicode / `chcp 65001`, locale-dependent `%date%`) are flagged with **⚡ Version note**.

---

## Table of Contents

1. [CMD vs PowerShell vs bash — and Batch Basics](#1-cmd-vs-powershell-vs-bash--and-batch-basics)
2. [Essential Built-in Commands](#2-essential-built-in-commands)
3. [Paths, Drives, Wildcards & the Working Directory](#3-paths-drives-wildcards--the-working-directory)
4. [Variables — `set`, `set /a`, `set /p`, setlocal](#4-variables--set-set-a-set-p-setlocal)
5. [Script Arguments & Argument Modifiers](#5-script-arguments--argument-modifiers)
6. [Conditionals — `if` in All Its Forms](#6-conditionals--if-in-all-its-forms)
7. [Loops — `for` in All Its Forms](#7-loops--for-in-all-its-forms)
8. [Delayed Expansion — The Big One](#8-delayed-expansion--the-big-one)
9. [Labels, `goto`, Subroutines & `call`](#9-labels-goto-subroutines--call)
10. [Redirection & Pipes](#10-redirection--pipes)
11. [Errorlevels & Error Handling](#11-errorlevels--error-handling)
12. [String Manipulation](#12-string-manipulation)
13. [Capturing Command Output & Reading Files](#13-capturing-command-output--reading-files)
14. [Escaping & Special Characters](#14-escaping--special-characters)
15. [Useful System Commands & One-Liners](#15-useful-system-commands--one-liners)
16. [Practical Recipes](#16-practical-recipes)
17. [Unicode & Encoding](#17-unicode--encoding)
18. [Gotchas & Best Practices](#18-gotchas--best-practices)
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. CMD vs PowerShell vs bash — and Batch Basics

### 1.1 What is `cmd.exe`?

`cmd.exe` is the **command-line interpreter** that has shipped with Windows since the NT line began (it is the successor to MS-DOS's `COMMAND.COM`). When you open a "Command Prompt", you are running `cmd.exe`. It reads commands you type interactively, *or* it reads them from a script file (a **batch file**). The batch "language" is whatever `cmd.exe` understands — it is small, quirky, and full of historical accidents.

### 1.2 CMD vs PowerShell vs bash

| | **CMD / batch** | **PowerShell** | **bash** |
|---|---|---|---|
| Platform | Windows only | Windows + cross-platform (pwsh 7+) | Linux/macOS (+ WSL/Git Bash on Windows) |
| Status in 2026 | Legacy, frozen, ubiquitous | Modern, supported, recommended for new work | Standard Unix shell |
| Data model | Plain text only | **Objects** (.NET) | Plain text |
| Scripting power | Weak; many quirks | Strong; real language | Strong |
| Startup speed | Instant | Slower (loads .NET) | Fast |
| Best for | Tiny double-clickable scripts, legacy, build steps | Real Windows automation | Unix automation |

**Rule of thumb in 2026:** new, non-trivial Windows automation → **PowerShell**. A 10-line "delete temp files and restart a service" double-clickable that must run on any box with zero setup → **batch is fine**. Reading/fixing the 10,000 batch files already out there → you need this guide.

### 1.3 `.bat` vs `.cmd` — what's the difference?

Both are batch files run by `cmd.exe`. For 99% of scripts **they behave identically**. The only documented difference involves how `ERRORLEVEL` is set by a few built-in commands (`set`, `path`, `assoc`, `prompt`, `append`...):

- In a **`.cmd`** file, those built-ins **reset `ERRORLEVEL`** (to 0 on success).
- In a **`.bat`** file, they generally **leave `ERRORLEVEL` unchanged** on success.

This is an obscure edge case. The practical guidance: **prefer `.cmd`** for new files (slightly saner errorlevel behavior), but if you inherit `.bat` files, don't bother renaming them. Both extensions are associated with `cmd.exe` and are double-clickable.

### 1.4 Opening cmd

- Press **Win+R**, type `cmd`, Enter.
- Start menu → type "cmd" or "Command Prompt".
- In File Explorer, type `cmd` in the address bar and press Enter (opens cmd *in that folder* — handy).
- From PowerShell or another terminal: `cmd`.
- **Run as Administrator:** right-click → "Run as administrator" (needed for system changes — see §16.9).

### 1.5 Running commands interactively

Type a command, press Enter. Example:

```bat
echo Hello, world
```

```
Hello, world
```

`cmd` is **case-insensitive** for commands and (mostly) for filenames. `ECHO`, `echo`, `Echo` are the same.

### 1.6 Running a `.bat` file

Three ways:

1. **Double-click** it in Explorer. The window opens, runs the script, and — unless you `pause` — slams shut when done. (This is why beginners' scripts "flash and vanish": add `pause` at the end to see the output.)
2. **From cmd:** type its name (with or without extension) while in its folder, or give a path:
   ```bat
   myscript.bat
   C:\tools\myscript.bat
   ```
3. **With arguments:**
   ```bat
   myscript.bat foo bar "arg with spaces"
   ```
   Inside the script those become `%1`, `%2`, `%3` (see §5).

> **cmd /C vs a .bat file.** You can run a one-liner without making a file using `cmd /C "command"`. `/C` runs the command then exits; `/K` runs it then **keeps** the window open. Throughout this guide, snippets marked *(one-liner)* can be pasted at a prompt or run via `cmd /C`; snippets marked *(.bat)* are meant to be saved to a file because they use `%%` loop variables, labels, or multiple lines.

### 1.7 The script skeleton — `@echo off`, comments, `pause`, `cls`, `title`, `exit`

```bat
@echo off
REM ^ @echo off must be the FIRST line. The leading @ suppresses the
REM   echoing of THIS line; "echo off" then stops cmd from printing each
REM   subsequent command before running it. Without it, your script spews
REM   every line to the console. Almost every real .bat starts this way.

REM This is a comment. REM = "remark". Everything after REM is ignored.

:: This is ALSO a comment. "::" is really an invalid label that cmd skips.
:: It is shorter and visually distinct, BUT see the gotcha below.

title My First Script     REM sets the console window title bar text
cls                       REM clears the screen

echo Doing some work...
echo Another line of output.

pause                     REM prints "Press any key to continue . . ." and waits.
                          REM Essential for double-clicked scripts so the window
                          REM stays open long enough to read output.

exit /b 0
REM exit /b 0  => leave THIS script with errorlevel 0 (success), but do NOT
REM             close the cmd window if the script was launched from a prompt.
REM exit       => without /b, terminates the entire cmd.exe process (closes
REM             the window). Use exit /b in scripts; use exit to close a shell.
```

#### The `::`-inside-a-block gotcha ⚡

`::` works fine on its own line. But **inside a parenthesised block** (an `if` or `for` body), `::` can break parsing because cmd tries to process the "label" in a context where labels are illegal. **Inside `( ... )` blocks, use `REM` for comments, never `::`.**

```bat
@echo off
if 1==1 (
    echo this is fine
    REM this comment is safe inside a block
    :: THIS LINE CAN CAUSE: "The system cannot find the batch label ..." or
    ::   a syntax error, because :: inside () confuses the parser.
    echo after the bad comment
)
```

Also note: a `::` line right before a `goto :label` or as the last line of a block has bitten many people. **Default to `REM` when in doubt.**

---

## 2. Essential Built-in Commands

These are the commands you'll use constantly. Most are **built into `cmd.exe`** (no `.exe` on disk); a few (like `robocopy`, `where`, `find`, `findstr`, `attrib`) are separate executables on the `PATH`. Every one has `/?` help: `dir /?`, `robocopy /?`, etc.

### 2.1 Common command reference table

| Command | Purpose | Key switches / notes |
|---|---|---|
| `cd` / `chdir` | Change / show current directory | `cd /d D:\path` to change **drive + dir** at once; `cd ..` up one; `cd \` to drive root |
| `dir` | List directory contents | `/a` all (incl. hidden), `/b` bare (names only), `/s` recurse, `/o:n` sort by name, `/o:-d` newest first, `/t:w` by write time |
| `copy` | Copy a few files | Simple; can't recurse. `copy a.txt b.txt` |
| `xcopy` | Copy files/trees (legacy) | `/s` subdirs, `/e` incl. empty dirs, `/i` assume dest is dir, `/y` no overwrite prompt. Largely superseded by robocopy |
| `robocopy` | **Robust** copy/mirror (best) | `/MIR` mirror, `/E` subdirs incl empty, `/Z` resumable, `/R:n` retries, `/W:n` wait. Returns nonstandard exit codes (see §11) |
| `move` | Move/rename files & dirs | `move a.txt D:\dst\`; `/y` no prompt |
| `del` / `erase` | Delete files | `/q` quiet, `/s` recurse, `/f` force read-only, `/a:h` by attribute. Cannot delete dirs |
| `md` / `mkdir` | Make directory | Creates intermediate dirs automatically: `md a\b\c` |
| `rd` / `rmdir` | Remove directory | `/s` remove tree, `/q` quiet (no confirm). `rd /s /q folder` deletes everything |
| `ren` / `rename` | Rename | `ren old.txt new.txt`; supports wildcards |
| `type` | Print a file to screen | `type file.txt` (the `cat` of cmd) |
| `more` | Page through output | `more file.txt`, or `dir | more` |
| `find` | Search for a literal string | Case-sensitive by default; `/i` ignore case, `/c` count, `/v` invert, `/n` line numbers |
| `findstr` | Search with **regex** | `/i` ignore case, `/r` regex, `/s` recurse, `/c:"phrase"` literal phrase, `/v` invert, `/n` line nums |
| `where` | Find a program on PATH | `where node` → prints full path(s); errorlevel 1 if not found |
| `tree` | ASCII directory tree | `/f` include files, `/a` ASCII chars |
| `attrib` | View/change file attributes | `+r/-r` read-only, `+h/-h` hidden, `+s` system, `/s` recurse |
| `cls` | Clear screen | — |
| `echo` | Print text / toggle echo | `echo.` prints blank line; `echo off/on` |
| `set` | Variables & environment | See §4 |
| `call` | Call sub-script / subroutine | See §9 |
| `start` | Launch program / file / URL | See §15 |
| `ver` | Show Windows version | — |
| `assoc` / `ftype` | File-type associations | See §15 |

### 2.2 `copy` vs `xcopy` vs `robocopy` — which to use?

- **`copy`** — one or a few files, same folder/simple paths. Cannot recurse directories.
- **`xcopy`** — older tool for copying directory trees. Still works, but quirky (e.g., it prompts "is dest a file or directory?" unless you pass `/i`). Considered legacy.
- **`robocopy`** — **use this for anything serious.** It mirrors trees, retries on failure, resumes large files, multithreads (`/MT`), logs, and is rock solid. The one wart: its exit codes are a bitmask, not 0=success (see §11.6). Built into Windows since Vista.

```bat
:: (one-liner) Mirror C:\src into D:\backup (deletes extras in dest!), with
:: retry-once and 1-second wait, skipping junctions:
robocopy C:\src D:\backup /MIR /R:1 /W:1 /XJ
```

### 2.3 `find` vs `findstr`

`find` does plain literal substring search. `findstr` is the more powerful one — it supports (limited) **regular expressions**, recursion, and searching multiple files. Prefer `findstr` for new work.

```bat
:: (one-liner) Count non-empty matches of "error" (case-insensitive) in a log:
findstr /i /c:"error" app.log

:: (one-liner) Regex: lines starting with a date like 2026-:
findstr /r /c:"^2026-" app.log

:: (one-liner) Search recursively through a tree:
findstr /s /i /c:"TODO" *.txt
```

---

## 3. Paths, Drives, Wildcards & the Working Directory

### 3.1 Drives and the current directory

Windows has **per-drive** current directories. `C:` doesn't just mean "drive C root" — it means "the *current* directory on drive C". This is why `cd D:\foo` from a `C:` prompt sometimes seems to "do nothing": it changes D's current dir but you're still *on* C. Use **`cd /d`** to change drive and directory together:

```bat
cd /d D:\projects\app      :: switches to D: AND into that folder
```

To see your current directory: `cd` (with no args prints it), or the built-in variable `%CD%`:

```bat
echo Current dir is: %CD%
```

### 3.2 Path quoting — spaces are the enemy

Windows paths frequently contain spaces (`C:\Program Files\...`). **Always quote paths that might contain spaces.**

```bat
:: GOOD
cd /d "C:\Program Files\My App"
copy "C:\my docs\a.txt" "D:\backup\a.txt"

:: BAD — cmd sees "C:\Program" as the command and "Files\My" "App" as args:
cd /d C:\Program Files\My App     :: breaks
```

### 3.3 Wildcards

- `*` — matches any number of characters.
- `?` — matches exactly one character.

```bat
dir *.txt          :: all .txt files
del temp_??.log    :: temp_01.log, temp_99.log, ... (exactly 2 chars)
copy *.jpg D:\pics\
```

Note: wildcards work in most file commands, **not** in arbitrary string contexts. `if exist *.txt` works; string comparisons do not expand wildcards.

### 3.4 Long paths

Classic Windows had a 260-character path limit (`MAX_PATH`). Modern Windows 11 supports long paths if enabled (registry/Group Policy `LongPathsEnabled`), but **`cmd.exe` itself does not reliably honor long paths**. For very deep paths, robocopy handles them better, or use the `\\?\C:\...` extended-length prefix with tools that support it. For most scripts, just keep paths reasonable.

### 3.5 Relative vs absolute, and `%CD%` vs script dir

A crucial distinction: **`%CD%` is the *working directory* (wherever the user was when they ran the script), NOT the folder the script lives in.** If a user double-clicks a script in `C:\tools\` while their cmd was in `C:\Users\Me`, then `%CD%` may be `C:\Users\Me`. To reliably reference files *next to your script*, use `%~dp0` (see §5.4) — this is one of the most important idioms in batch.

---

## 4. Variables — `set`, `set /a`, `set /p`, setlocal

### 4.1 Setting and reading variables

```bat
@echo off
set name=World
echo Hello, %name%
:: → Hello, World
```

Variables are **strings**. Reading is `%name%` in a script or at the prompt.

### 4.2 The `set "name=value"` quoting form — avoid trailing-space bugs ⚡

This is one of the **most important habits in batch.** Consider:

```bat
set name=World      &  echo done   :: BAD
```

Everything *after* the `=` up to the end of the line (or `&`) becomes the value — **including trailing spaces**. The classic disaster:

```bat
set name=World          REM the spaces before REM are now PART of the value!
```

The fix is to **wrap the whole assignment in quotes**: `set "name=value"`. The quotes are *not* stored; they just delimit exactly what the value is, killing trailing-space and special-char bugs:

```bat
set "name=World"
echo [%name%]      :: → [World]   (no stray spaces)
```

**Always use `set "name=value"`.** It is the single best defensive habit in batch scripting.

```bat
:: Show a variable's value safely (brackets reveal hidden spaces):
set "x= padded "
echo [%x%]         :: → [ padded ]   (you can SEE the spaces)
```

Delete a variable by setting it to nothing:

```bat
set "name="        :: now %name% is undefined
```

### 4.3 `set /a` — integer arithmetic

`set /a` evaluates an integer expression. **Inside `/a`, variable names don't need `%`.** It supports `+ - * / %` (modulo — note: literal `%` in arithmetic; in a script context you may need `%%`), `( )`, bit ops (`& | ^ ~ << >>`), and assignment operators (`+=`, etc.).

```bat
@echo off
set /a sum = 2 + 3 * 4         :: → 14 (normal precedence)
echo %sum%

set /a x = 10
set /a x += 5                  :: x is now 15
echo %x%

set /a half = 7 / 2           :: integer division → 3 (no floats in cmd!)
set /a rem = 7 %% 2           :: modulo. Use %% in a .bat; just % at the prompt
echo %half% remainder %rem%   :: → 3 remainder 1

:: Hex/octal: 0x prefix = hex, leading 0 = OCTAL (a classic trap!)
set /a hexval = 0xFF          :: → 255
set /a oops = 011            :: → 9  (011 is OCTAL, not eleven!)
echo %hexval% %oops%
```

> **No floating point.** `cmd` arithmetic is integer-only. For decimals you must use PowerShell, or scale by 100 and format manually. Leading-zero numbers are interpreted as **octal**, so `08` and `09` are *errors* — strip leading zeros before doing math on things like dates/times.

### 4.4 `set /p` — prompt for input

```bat
@echo off
set /p "userName=Enter your name: "
echo Hello, %userName%!

set /p "answer=Continue (y/n)? "
if /i "%answer%"=="y" echo Continuing...
```

`set /p` reads one line from the user (or from redirected stdin — see §13.4 for reading files this way). If the user just presses Enter, the variable keeps its **previous** value (it is *not* cleared) — a subtle gotcha; pre-clear it if that matters.

### 4.5 Environment variables vs local — `setlocal` / `endlocal`

By default, `set` in a script modifies the environment of the **current cmd session**, and those changes *persist* after the script ends if run in an existing window. To keep your script's variables from leaking, use `setlocal`:

```bat
@echo off
setlocal
:: Everything from here uses a PRIVATE copy of the environment.
set "TEMP_VAR=only visible inside"
set "PATH=%PATH%;C:\mytools"
:: ... do work ...
endlocal
:: After endlocal, TEMP_VAR is gone and PATH is restored to what it was.
```

`setlocal`/`endlocal` also scopes the **current directory** and is required to enable **delayed expansion** (§8): `setlocal EnableDelayedExpansion`. When a script ends, an implicit `endlocal` happens. **Start almost every non-trivial script with `setlocal`** (often `setlocal EnableExtensions EnableDelayedExpansion`).

> **Passing a value OUT past `endlocal`.** Because `endlocal` discards your variables, returning a value is tricky. The trick:
> ```bat
> setlocal
> set "result=42"
> endlocal & set "result=%result%"
> :: The right side of & is parsed BEFORE endlocal runs, so %result% expands
> :: to 42, and the new set survives into the outer scope.
> ```

### 4.6 Useful built-in / environment variables

| Variable | Meaning |
|---|---|
| `%CD%` | Current directory |
| `%DATE%` / `%TIME%` | Current date/time (**locale-dependent format!** see §15) |
| `%RANDOM%` | Random int 0–32767 |
| `%ERRORLEVEL%` | Exit code of last command (§11) |
| `%PATH%` | Executable search path |
| `%PATHEXT%` | Extensions tried for commands (`.COM;.EXE;.BAT;.CMD;...`) |
| `%USERNAME%` / `%USERPROFILE%` | Current user / their home folder |
| `%COMPUTERNAME%` | Machine name |
| `%TEMP%` / `%TMP%` | Temp folder |
| `%APPDATA%` / `%LOCALAPPDATA%` | App data folders |
| `%WINDIR%` / `%SystemRoot%` | Windows folder |
| `%ProgramFiles%` / `%ProgramFiles(x86)%` | Program Files folders |
| `%NUMBER_OF_PROCESSORS%` | CPU count |
| `%CMDCMDLINE%` | The full command line that started cmd |

Persisting an environment variable beyond the session requires **`setx`** (writes to the registry; affects *new* shells, not the current one) — not `set`:

```bat
:: (one-liner) Permanently add to YOUR user PATH (takes effect in new shells):
setx PATH "%PATH%;C:\mytools"
:: WARNING: setx truncates values over 1024 chars and is clumsy for PATH.
:: For real PATH editing, prefer the System Properties UI or PowerShell.
```

---

## 5. Script Arguments & Argument Modifiers

### 5.1 `%0`–`%9` and `%*`

When a script is called as `myscript.bat alpha beta gamma`:

| Token | Value |
|---|---|
| `%0` | The script name as invoked (`myscript.bat` or its path) |
| `%1` | `alpha` |
| `%2` | `beta` |
| `%3` | `gamma` |
| `%*` | **All** args as one string: `alpha beta gamma` |

```bat
@echo off
echo Script : %0
echo Arg 1  : %1
echo Arg 2  : %2
echo All    : %*
```

```
> myscript.bat alpha beta
Script : myscript.bat
Arg 1  : alpha
Arg 2  : beta
All    : alpha beta
```

There are only `%0`–`%9` (ten positional slots). For more arguments, use `shift`.

### 5.2 `shift`

`shift` moves every argument down by one: `%1`←`%2`, `%2`←`%3`, etc. Use it to walk an unbounded list:

```bat
@echo off
:loop
if "%~1"=="" goto done       :: stop when no more args (note the ~ to strip quotes)
echo Processing: %~1
shift
goto loop
:done
echo All arguments processed.
```

> **Note:** `shift` does **not** affect `%*`, and it does not shift `%0` unless you use `shift /1` etc. Use the loop pattern above for "process every argument."

### 5.3 Argument modifiers — the full reference table

You can transform an argument with a `~` modifier. These also work on `for` loop variables (§7). For argument `%1` referring to a file `C:\My Docs\report.final.txt`:

| Modifier | Expands to | Example result |
|---|---|---|
| `%1` | The argument as-is (with quotes if present) | `"C:\My Docs\report.final.txt"` |
| `%~1` | …with surrounding **quotes removed** | `C:\My Docs\report.final.txt` |
| `%~f1` | **Full** (absolute) path | `C:\My Docs\report.final.txt` |
| `%~d1` | **Drive** letter | `C:` |
| `%~p1` | **Path** (dirs, no drive, no filename) | `\My Docs\` |
| `%~n1` | File **name** (no extension) | `report.final` |
| `%~x1` | File e**x**tension (with dot) | `.txt` |
| `%~nx1` | Name **+** extension | `report.final.txt` |
| `%~dp1` | Drive **+** path | `C:\My Docs\` |
| `%~dpnx1` | Drive+path+name+ext (full, normalized) | `C:\My Docs\report.final.txt` |
| `%~s1` | **Short** (8.3) path form | `C:\MYDOCS~1\REPORT~1.TXT` |
| `%~a1` | File **attributes** | `--a------` |
| `%~t1` | File's **timestamp** (last write) | `06/21/2026 10:30 AM` |
| `%~z1` | File **size** in bytes | `1048576` |
| `%~$PATH:1` | Searches `PATH` for the file; full path or empty | `C:\Windows\system32\cmd.exe` |

Modifiers combine: `%~dpn1` = drive+path+name. Order is flexible but stick to the documented forms above.

### 5.4 `%~dp0` — the script's own directory (critical idiom)

`%0` is the script itself, so `%~dp0` is **the drive+path of the script** — i.e., the folder the script lives in, with a trailing backslash. This is *the* way to reference files bundled next to your script, regardless of where the user ran it from:

```bat
@echo off
setlocal
echo This script lives in: %~dp0

:: Reliably reference a file next to the script:
set "configFile=%~dp0config.ini"
echo Will read config from: %configFile%

:: Reliably cd into the script's folder (use pushd so it auto-quotes & works
:: even for UNC paths; popd restores later):
pushd "%~dp0"
echo Now working in: %CD%
:: ... do work relative to the script ...
popd
endlocal
```

> **⚡ Note:** `%~dp0` already ends with `\`, so write `%~dp0config.ini`, **not** `%~dp0\config.ini` (the double backslash usually works but is sloppy). `%~f0` gives the full script path including filename.

---

## 6. Conditionals — `if` in All Its Forms

### 6.1 Basic `if` / `if not`

```bat
@echo off
set "x=5"
if %x%==5 echo x is five
if not %x%==9 echo x is not nine
```

### 6.2 Quote your strings — the empty-variable syntax trap ⚡

If a variable is **empty or undefined**, `if %var%==value` expands to `if ==value`, which is a **syntax error** that aborts the block. **Always quote both sides:**

```bat
:: BAD — if %answer% is empty this becomes  if ==yes  → syntax error
if %answer%==yes echo ok

:: GOOD — quotes keep it valid even when empty:  if ""=="yes"
if "%answer%"=="yes" echo ok
```

Quoting is mandatory defensive practice for any string comparison involving variables.

### 6.3 Case-insensitive compare — `/i`

String comparison is **case-sensitive** by default. Add `/i`:

```bat
if /i "%answer%"=="YES" echo matched yes/Yes/yEs/...
```

### 6.4 Numeric comparison operators

For numbers, use the word operators (they do numeric comparison if both sides look numeric, else string):

| Operator | Meaning |
|---|---|
| `equ` | equal |
| `neq` | not equal |
| `lss` | less than |
| `leq` | less than or equal |
| `gtr` | greater than |
| `geq` | greater than or equal |

```bat
@echo off
set /a n=7
if %n% gtr 5 echo n is greater than 5
if %n% leq 10 echo n is at most 10
if /i "%n%" equ "7" echo n equals 7
```

> **Caution:** `==` does a *string* compare, so `if "%n%"=="7"` is fine for exact strings, but `if 07 == 7` is FALSE (string mismatch) while `if 07 equ 7` is TRUE (numeric). Use `equ`/`neq`/etc. for numbers.

### 6.5 `if exist`, `if defined`, `if errorlevel`

```bat
@echo off
:: Does a file or folder exist?
if exist "C:\data\report.txt" echo file is there
if not exist "C:\data\" echo data folder missing

:: Is a variable defined? (don't use %var% here — use the NAME)
if defined MYVAR echo MYVAR is set to: %MYVAR%
if not defined MYVAR echo MYVAR is not set

:: Errorlevel test — see the gotcha in 6.6
if errorlevel 1 echo last command failed (errorlevel >= 1)
```

> **`if exist` for directories:** `if exist "C:\folder\"` (trailing backslash) tests specifically for a directory. Without the backslash it matches files too.

### 6.6 The `if errorlevel N` ">=" gotcha ⚡

`if errorlevel N` means **"is errorlevel greater than OR EQUAL to N"**, *not* "equals N". This trips up everyone.

```bat
:: WRONG mental model: this fires for errorlevel 1, 2, 3, ... AND higher:
if errorlevel 1 echo something failed

:: To test EXACTLY 1, either test the boundary:
if errorlevel 1 if not errorlevel 2 echo errorlevel is exactly 1

:: ...or just compare the variable directly (clearer, preferred):
if "%errorlevel%"=="1" echo exactly 1
if %errorlevel% neq 0 echo nonzero (i.e., failed)
```

For checking "did it fail?", `if %errorlevel% neq 0` is the cleanest. See §11 for the deep dive.

### 6.7 `else` and block syntax rules

`if ... ( ) else ( )` must follow strict bracket placement. **The `)` and `else` and `(` must be on the same line** (or use careful line continuation). The canonical layout:

```bat
@echo off
set "score=85"
if %score% geq 90 (
    echo Grade: A
) else if %score% geq 80 (
    echo Grade: B
) else if %score% geq 70 (
    echo Grade: C
) else (
    echo Grade: F
)
```

Rules that bite people:
- The opening `(` must be on the **same line** as `if` / `else`.
- `) else (` must be on **one line** — `)` then newline then `else` is a syntax error.
- **Inside blocks, use `REM` not `::`** (see §1.7).
- Don't put a bare `)` inside an `echo` without escaping — `echo Done)` may end the block early. Use `echo Done^)` or rearrange.
- Variables read with `%var%` inside a block are expanded **once, when the block is parsed** — this is the delayed-expansion problem (§8).

---

## 7. Loops — `for` in All Its Forms

`for` is the most powerful — and most confusing — construct in batch. Master it and you can do almost anything. The general shape is:

```
for {options} %%variable in (set) do (command)
```

> **⚡ `%%` in scripts vs `%` at the prompt.** Inside a **`.bat` file** the loop variable is written with **two** percent signs: `%%a`. When you type a `for` loop **interactively at the prompt**, use **one**: `%a`. This is the #1 "why doesn't my loop work" mistake. All `.bat` examples below use `%%`.

> **Loop variable names** are single letters and **case-sensitive**: `%%a` and `%%A` are different. Conventionally use `%%a`, `%%b`, `%%i`, etc. Avoid `%%n`/`%%x` clashing with modifiers in your head.

### 7.1 Plain `for` — iterate over a set

```bat
@echo off
:: Iterate over a literal list of words:
for %%w in (apple banana cherry) do (
    echo Fruit: %%w
)

:: Iterate over files matching a wildcard (this is hugely common):
for %%f in (*.txt) do (
    echo Found text file: %%f
    echo   full path: %%~ff
    echo   size: %%~zf bytes
)
```

All the `~` modifiers from §5.3 work on `%%f`: `%%~nxf` (name+ext), `%%~dpf` (drive+path), etc.

### 7.2 `for /L` — counting loops

`for /L %%i in (start,step,end) do ...` counts.

```bat
@echo off
:: Count 1 to 5:
for /L %%i in (1,1,5) do echo Iteration %%i

:: Count down 10,8,6,...,0:
for /L %%i in (10,-2,0) do echo Countdown: %%i

:: Build a row of stars (combine with delayed expansion §8 to accumulate):
setlocal EnableDelayedExpansion
set "line="
for /L %%i in (1,1,10) do set "line=!line!*"
echo !line!
endlocal
```

### 7.3 `for /D` — iterate over directories

```bat
@echo off
:: List immediate subdirectories of the current folder:
for /D %%d in (*) do echo Subdir: %%d

:: With a path & wildcard:
for /D %%d in ("C:\projects\*") do echo Project: %%~nxd
```

### 7.4 `for /R` — recurse a directory tree

`for /R [path] %%f in (pattern) do ...` walks `path` and all subfolders.

```bat
@echo off
:: Every .log file under C:\logs (recursively):
for /R "C:\logs" %%f in (*.log) do (
    echo %%~ff  (%%~zf bytes)
)

:: List every subfolder recursively (combine /R with /D-like via *):
for /R "C:\projects" %%d in (.) do echo Folder: %%~fd
```

### 7.5 `for /F` — parse files, strings, and command output (the powerful one)

`for /F` is the workhorse for **parsing text**: file contents, literal strings, or the **output of a command**. The options go in quotes:

| Option | Meaning |
|---|---|
| `delims=xyz` | Characters that separate tokens (default: space + tab). `delims=` (empty) = whole line |
| `tokens=2,3` | Which token(s) to grab. `tokens=1,*` = token 1, then rest of line as one |
| `skip=N` | Skip the first N lines (e.g., headers) |
| `eol=;` | Lines starting with this char are ignored (default `;`) |
| `usebackq` | Changes quoting: backquotes run commands, double-quotes are literal filenames, single-quotes are literal strings |

#### 7.5.1 Parse a file line by line

```bat
@echo off
:: Read names.txt, one whole line at a time (delims= grabs the entire line):
for /F "usebackq delims=" %%L in ("names.txt") do (
    echo Line: %%L
)
```

#### 7.5.2 Split into columns with `tokens` and `delims`

```bat
@echo off
:: Parse a CSV: "Alice,30,NYC" → name, age, city
for /F "usebackq tokens=1,2,3 delims=," %%a in ("people.csv") do (
    echo Name=%%a Age=%%b City=%%c
)
```

`tokens=1,2,3` assigns token 1 to `%%a`, token 2 to the **next letter** `%%b`, token 3 to `%%c` — the variable letters auto-increment.

#### 7.5.3 `tokens=N,*` — grab one column plus "the rest"

```bat
@echo off
:: A line like:  2026-06-21  ERROR  Disk full on volume D:
:: Grab the date (token 1) and EVERYTHING after the 2nd field as one chunk:
for /F "tokens=1,2,* delims= " %%d in ("2026-06-21 ERROR Disk full on D:") do (
    echo Date=%%d Level=%%e Message=%%f
)
```

#### 7.5.4 Parse a **literal string** (no file)

Quote the string in the `in (...)` clause **without** `usebackq`:

```bat
@echo off
for /F "tokens=1,2 delims=:" %%h in ("10:30") do echo Hour=%%h Minute=%%m
```

#### 7.5.5 Capture **command output** — `usebackq` + backticks

This is THE way to put a command's output into your script (see §13 for more):

```bat
@echo off
:: Get today's date from the 'date /t' command:
for /F "usebackq delims=" %%d in (`date /t`) do set "today=%%d"
echo Today is: %today%

:: Without usebackq you use single-quotes for the command instead:
for /F "delims=" %%d in ('date /t') do set "today=%%d"
```

> **`usebackq` quoting summary:** With `usebackq`: `` `cmd` `` runs a command, `"file"` reads a file, `'string'` is a literal string. Without `usebackq`: `'cmd'` runs a command, `"string"` is a literal string, and reading a file with spaces in its path is awkward. **Use `usebackq` whenever your filename/path has spaces or you want clarity.**

#### 7.5.6 `skip` and `eol`

```bat
@echo off
:: Skip a 1-line header, ignore comment lines starting with #:
for /F "usebackq skip=1 eol=# tokens=1,2 delims=," %%a in ("data.csv") do (
    echo Col1=%%a Col2=%%b
)
```

> **Gotcha — empty lines & `for /F`:** `for /F` **silently skips blank lines**. If you need to preserve blank lines (e.g., reproducing a file exactly), `for /F` is the wrong tool — prepend line numbers with `findstr /n ^^` and strip them, or use a different approach.

---

## 8. Delayed Expansion — The Big One

This is the single most important advanced concept in batch. Misunderstanding it causes the most "my loop variable doesn't update" bugs.

### 8.1 Why `%var%` breaks inside loops and blocks

`cmd` reads and **expands `%var%` references at *parse time*** — i.e., when it reads the *entire* block `( ... )` or compound line, **before executing any of it**. So inside a loop, every `%var%` is replaced with whatever the variable was *before the loop started*, and it **never changes during the loop**, no matter how many times you `set` it.

```bat
@echo off
set "count=0"
for %%f in (a b c) do (
    set /a count+=1
    echo %count%        :: ALWAYS prints 0 !  (parsed once, before the loop)
)
echo Final: %count%      :: this DOES print 3 (parsed fresh on this line)
```

Output:
```
0
0
0
Final: 3
```

The `set /a` *does* work — but `echo %count%` inside the block was frozen at `0` during parsing.

### 8.2 The fix — `EnableDelayedExpansion` and `!var!`

Enable delayed expansion, then use **exclamation marks** `!var!` instead of `%var%` for anything that changes inside a block. `!var!` is expanded at **run time**, each iteration:

```bat
@echo off
setlocal EnableDelayedExpansion
set "count=0"
for %%f in (a b c) do (
    set /a count+=1
    echo !count!        :: prints 1, then 2, then 3  ✓
)
echo Final: !count!
endlocal
```

Output:
```
1
2
3
Final: 3
```

### 8.3 Classic counter / accumulator example

```bat
@echo off
setlocal EnableDelayedExpansion

set "total=0"
set "biggest="
:: Sum the sizes of all .log files and track the largest name:
for %%f in (*.log) do (
    set /a total += %%~zf
    set "biggest=%%f"        :: %%f is a loop var, fine to read with %% — but
                             :: if we wanted the PREVIOUS biggest we'd need !biggest!
)
echo Total bytes: !total!
echo Last file:   !biggest!
endlocal
```

### 8.4 Pitfalls of delayed expansion

1. **You still use `%%f` for loop variables** — only your *own* variables that change need `!...!`. Loop vars are substituted by `for` itself, not by normal expansion.
2. **`!` becomes special when delayed expansion is on.** A literal `!` in text now needs escaping as `^^!` (yes, *two* carets in some contexts) or you must toggle delayed expansion off. This is the start of "escaping hell" (§14).
3. **Values containing `!`** read into a variable while delayed expansion is on can get mangled. If you must handle data with `!`, read it with delayed expansion *off*, or carefully escape.
4. **Don't enable it globally if you don't need it** — but in practice most scripts with loops want it. A common safe header is:
   ```bat
   setlocal EnableExtensions EnableDelayedExpansion
   ```

> **Mental model:** `%var%` = "value when this line was *read*." `!var!` = "value right *now* as this line runs." Use `!...!` inside any `( )` block or loop where the variable changes.

---

## 9. Labels, `goto`, Subroutines & `call`

### 9.1 Labels and `goto`

A label is `:name` on its own line. `goto name` jumps to it. This is how batch does loops without `for` and how it builds menus.

```bat
@echo off
set /a i=0
:loop
set /a i+=1
echo i is %i%
if %i% lss 5 goto loop
echo Done.
```

### 9.2 `goto :eof` — the clean "return"

`:eof` is a **special built-in label** meaning "end of file." `goto :eof` (with the colon) jumps to the end of the current script/subroutine without you needing to add an `:end` label. Use it to exit a subroutine.

```bat
@echo off
echo Start
if not exist "data.txt" (
    echo No data file, aborting.
    goto :eof
)
echo Processing data...
```

### 9.3 Subroutines with `call :label`

`call :label args` runs the code at `:label` like a function, then **returns to the line after the `call`** when it hits `goto :eof`. Arguments become `%1`, `%2`, ... *inside the subroutine*.

```bat
@echo off
setlocal EnableDelayedExpansion

call :greet "Alice"
call :greet "Bob"

call :add 3 4
echo The sum returned was: %sumResult%

goto :eof
:: ^ This goto :eof stops the MAIN flow from falling into the subroutines.

:: ---------- subroutines ----------

:greet
    echo Hello, %~1!
    goto :eof        :: return to caller

:add
    :: %1 and %2 are the args. Return a value via a named variable.
    set /a sumResult=%1 + %2
    goto :eof
```

> **Returning values.** A subroutine can return:
> - An **errorlevel**, via `exit /b N` (good for success/failure codes).
> - A **variable** the caller reads afterward (as `sumResult` above). If you used `setlocal` *inside* the subroutine, use the `endlocal & set` trick (§4.5) to pass it out.

```bat
:add
    setlocal
    set /a result=%1 + %2
    endlocal & set "sumResult=%result%"   :: pass result past endlocal
    exit /b 0
```

### 9.4 `call` another batch file

`call other.bat arg1 arg2` runs another script and **returns** to yours afterward. **Without `call`**, invoking another `.bat` from inside a `.bat` transfers control and **never comes back** (the first script ends). Always use `call` when you want to continue:

```bat
@echo off
call setup-env.bat              :: runs it, then continues here
echo Back in main script.
call build.bat Release          :: pass an argument
if %errorlevel% neq 0 echo Build failed!
```

> **Also use `call` for variable-name indirection** and to force re-parsing — e.g., `call set "x=%%y%%"` is an old trick for double-expansion (mostly superseded by delayed expansion).

---

## 10. Redirection & Pipes

### 10.1 The streams

Every command has **stdout (stream 1)** for normal output and **stderr (stream 2)** for errors. Redirection sends them to files or to the void.

| Syntax | Effect |
|---|---|
| `cmd > file` | stdout → file (**overwrite**) |
| `cmd >> file` | stdout → file (**append**) |
| `cmd 2> file` | stderr → file |
| `cmd 2>> file` | stderr → file (append) |
| `cmd > out.txt 2> err.txt` | stdout and stderr to separate files |
| `cmd > file 2>&1` | **both** stdout and stderr → file (order matters!) |
| `cmd 2>&1` | merge stderr into stdout (e.g., before a pipe) |
| `cmd < file` | feed file to stdin |
| `cmd > nul` | discard stdout (`nul` = Windows /dev/null) |
| `cmd 2> nul` | discard errors only |
| `cmd > nul 2>&1` | discard **everything** (silent) |
| `cmdA | cmdB` | pipe stdout of A into stdin of B |

### 10.2 `nul` — the Windows /dev/null

`nul` is a special device that swallows whatever you send it. Use it to silence commands.

```bat
:: (one-liner) Make a directory, ignore "already exists" error noise:
md "C:\temp\work" 2>nul

:: (one-liner) Check if a program exists WITHOUT printing anything:
where git >nul 2>&1
if %errorlevel%==0 (echo git is installed) else (echo git is NOT installed)
```

### 10.3 `2>&1` ordering ⚡

`> file 2>&1` works, but **`2>&1 > file` does NOT** do what you'd expect. Redirections are processed **left to right**: `2>&1` first points stderr at *wherever stdout currently is* (the console), *then* `> file` moves stdout to the file — leaving stderr still on the console. **Always put `2>&1` LAST:**

```bat
:: CORRECT — both go to the file:
mytool.exe > output.log 2>&1

:: WRONG — stderr stays on console:
mytool.exe 2>&1 > output.log
```

### 10.4 Piping

```bat
:: (one-liner) Page a long listing:
dir /s | more

:: (one-liner) Find processes whose name contains "chrome":
tasklist | findstr /i chrome

:: (one-liner) Count matching lines:
findstr /i error app.log | find /c /v ""
```

> **Pipe gotcha:** each side of a `|` runs in its **own** `cmd` subshell. Variables you `set` on the left side of a pipe **do not survive** to your script. Also, delayed expansion behaves oddly across pipes. Avoid `set`-ing variables inside a piped command; capture with `for /F` instead (§13).

### 10.5 Grouping with parentheses

`( ... )` groups commands so you can redirect them all together:

```bat
:: Write multiple lines to a file in one redirect:
(
    echo Line 1
    echo Line 2
    echo Line 3
) > out.txt

:: Append a "section" to a log:
(
    echo === Run at %date% %time% ===
    echo Status: OK
) >> run.log
```

> **Echoing tricky characters:** to write a blank line use `echo.` (the dot). To write a line with leading spaces, `echo  text` may strip them — use `echo. text` patterns carefully. To echo special chars like `|`, `>`, `&`, `(`, `)`, escape them with `^` (§14).

---

## 11. Errorlevels & Error Handling

### 11.1 What is `ERRORLEVEL`?

After (almost) every command, `cmd` sets `ERRORLEVEL` to that command's exit code: **0 = success**, nonzero = some failure. You read it as `%errorlevel%`.

```bat
@echo off
ping -n 1 8.8.8.8 >nul 2>&1
if %errorlevel%==0 (echo Online) else (echo Offline)
```

### 11.2 `if errorlevel` vs `%errorlevel%` — the distinction ⚡

There are **two different things** named "errorlevel" and they behave differently:

- **`if errorlevel N`** — a special syntax meaning "errorlevel **≥** N" (see §6.6). It reads the *real* internal errorlevel.
- **`%errorlevel%`** — the *environment variable*. Usually mirrors the real value, BUT: if someone did `set "errorlevel=foo"`, the variable shadows the real one and lies. Also it expands at parse time like any `%var%`.

For clarity, most modern scripts use `if %errorlevel% neq 0` or `if %errorlevel% equ 0`. Just never define a variable literally named `errorlevel`.

### 11.3 `&&`, `||`, and `&`

| Operator | Meaning |
|---|---|
| `a && b` | run `b` **only if** `a` succeeded (errorlevel 0) |
| `a \|\| b` | run `b` **only if** `a` failed (errorlevel ≠ 0) |
| `a & b` | run `b` **regardless** (unconditional sequence) |

```bat
:: (one-liner) Build, and only run if build succeeded:
build.bat && run.bat

:: (one-liner) Try primary, fall back on failure:
ping -n 1 primary.local || ping -n 1 backup.local

:: (one-liner) Do both no matter what:
echo step 1 & echo step 2

:: Combine: build, then either succeed-message or fail-message:
build.bat && (echo BUILD OK) || (echo BUILD FAILED)
```

> **Subtle:** `a && b || c` is not a clean if/else — if `a` succeeds but `b` then *fails*, `c` runs too. Use explicit `if` blocks when the logic matters.

### 11.4 `exit /b N` — setting your script's exit code

Return a code from your script so callers (CI, other scripts) can check it:

```bat
@echo off
build.bat
if %errorlevel% neq 0 (
    echo Build failed.
    exit /b 1            :: propagate failure to whoever called this script
)
echo All good.
exit /b 0
```

`exit /b N` leaves the script with code `N` (and keeps the shell open). Plain `exit N` closes the whole shell — don't use it in scripts meant to be `call`ed.

### 11.5 Capturing errorlevel before it changes

`%errorlevel%` reflects the **last** command. If you run anything between the command and your check — even `echo` — you may clobber it. Capture it immediately:

```bat
@echo off
mytool.exe
set "rc=%errorlevel%"     :: snapshot it RIGHT AWAY
echo Tool finished.       :: this echo would otherwise reset errorlevel to 0
if %rc% neq 0 echo Tool failed with code %rc%
```

> **⚡ Inside a block**, `%errorlevel%` is parsed once. To read a live errorlevel inside an `if`/`for` body, enable delayed expansion and use `!errorlevel!`.

### 11.6 Robocopy's nonstandard exit codes ⚡

`robocopy` does **not** use 0=success. Its exit code is a bitmask; **anything < 8 is actually success** (0 = nothing copied, 1 = files copied, etc.); **8 or higher = real failure**. CI scripts that treat any nonzero robocopy code as failure will break. Handle it explicitly:

```bat
@echo off
robocopy "C:\src" "D:\dst" /MIR /R:1 /W:1
:: Robocopy: 0-7 = success-ish, >=8 = failure.
if %errorlevel% geq 8 (
    echo Robocopy FAILED with code %errorlevel%
    exit /b 1
)
echo Robocopy OK (code %errorlevel%)
exit /b 0
```

### 11.7 A reusable "die on error" helper

```bat
@echo off
setlocal

call :run git pull        || goto :fail
call :run npm install     || goto :fail
call :run npm run build   || goto :fail
echo All steps succeeded.
exit /b 0

:run
    echo + %*
    %*                    :: run the command passed as arguments
    exit /b %errorlevel%  :: bubble up its exit code

:fail
    echo.
    echo *** A step failed (errorlevel %errorlevel%). Aborting. ***
    exit /b 1
```

---

## 12. String Manipulation

`cmd` has surprisingly capable (if cryptic) string operations on **variables** via `%var:...%` syntax. These work on the *value* of a variable, not on arbitrary text.

### 12.1 Substring — `%var:~start,len%`

```bat
@echo off
set "s=Hello, World"
echo %s:~0,5%      :: → Hello        (start 0, length 5)
echo %s:~7%        :: → World        (from index 7 to end)
echo %s:~-5%       :: → World        (last 5 chars — negative start)
echo %s:~7,3%      :: → Wor          (3 chars from index 7)
echo %s:~-5,3%     :: → Wor          (3 chars, starting 5 from the end)
echo %s:~0,-1%     :: → Hello, Worl  (all but the last char — negative length)
```

- Start is **0-based**. Negative start counts from the end.
- A negative length means "stop that many chars from the end."

### 12.2 Search and replace — `%var:old=new%`

```bat
@echo off
set "path=C:\Users\Me\Documents"
echo %path:\=/%               :: → C:/Users/Me/Documents  (swap \ for /)
echo %path:Documents=Desktop% :: → C:\Users\Me\Desktop

set "csv=a,b,c,d"
echo %csv:,= %                :: → a b c d   (commas to spaces)

:: Delete a substring by replacing with nothing:
set "noisy=  trim   me  "
echo [%noisy: =%]             :: → [trimme]  (removes ALL spaces, not just ends!)
```

> **Note:** `%var:old=new%` replaces **all** occurrences and is case-insensitive in the *search*. There is no regex here. To replace starting only at the beginning use `%var:*old=new%` (the `*` deletes everything up to and including the first `old`).

### 12.3 Case conversion — no built-in (workarounds)

`cmd` has **no** built-in upper/lower-casing. Options:

```bat
:: Workaround A: brute-force replace each letter (ugly but pure batch).
:: Workaround B: shell out to PowerShell (cleanest, needs PS available):
for /f "usebackq delims=" %%u in (`powershell -NoProfile -Command "'%s%'.ToUpper()"`) do set "upper=%%u"
echo %upper%
```

For real case work, this is a sign you should be in PowerShell.

### 12.4 String length — the trick

There's no length operator. The classic technique uses a `for /L` loop probing substrings (works but verbose), or shelling to PowerShell. The loop version:

```bat
@echo off
setlocal EnableDelayedExpansion
set "str=Hello, World"
set "len=0"
:lenLoop
if not "!str:~%len%,1!"=="" (
    set /a len+=1
    goto lenLoop
)
echo Length of "%str%" is %len%
endlocal
```

It keeps grabbing the 1-char substring at index `len`; when that's empty, you've passed the end.

### 12.5 Trimming whitespace

No built-in trim. Common approaches:

```bat
@echo off
setlocal EnableDelayedExpansion
set "v=   hello   "

:: Trim LEADING spaces:
for /f "tokens=* delims= " %%a in ("!v!") do set "v=%%a"

:: Trim TRAILING spaces (loop removing one trailing space at a time):
:trimTrail
if "!v:~-1!"==" " set "v=!v:~0,-1!" & goto trimTrail

echo [!v!]      :: → [hello]
endlocal
```

`tokens=*` skips leading delimiters, which conveniently trims the front.

---

## 13. Capturing Command Output & Reading Files

### 13.1 The `for /F` capture idiom — the only real way

`cmd` cannot do `var=$(command)` like bash. The **only** general way to put a command's output into a variable is `for /F`:

```bat
@echo off
:: Capture single-line output:
for /F "usebackq delims=" %%v in (`hostname`) do set "host=%%v"
echo This machine is: %host%

:: Capture a specific token (e.g., 2nd word of a command's output):
for /F "usebackq tokens=2" %%v in (`some-command`) do set "field=%%v"
```

> **⚡ Multi-line output:** the loop runs once per line, so a multi-line command leaves your variable holding the **last** line. To capture all lines, accumulate with delayed expansion, or use a counter.

### 13.2 Counting files / lines

```bat
@echo off
setlocal EnableDelayedExpansion

:: Count files matching a pattern:
set "n=0"
for %%f in (*.txt) do set /a n+=1
echo There are !n! .txt files.

:: Count lines in a file (find /c /v "" counts all lines incl. blanks):
for /F %%c in ('type "data.txt" ^| find /c /v ""') do set "lines=%%c"
echo data.txt has %lines% lines.
:: NOTE: the | inside the for command must be escaped as ^| (see §14).
endlocal
```

### 13.3 Reading a file line by line

```bat
@echo off
setlocal EnableDelayedExpansion
set "ln=0"
for /F "usebackq delims=" %%L in ("config.txt") do (
    set /a ln+=1
    echo [!ln!] %%L
)
endlocal
```

Remember `for /F` **skips blank lines** and strips/handles `eol` lines. To preserve blanks, prefix line numbers first:

```bat
@echo off
:: Read EVERY line including blanks by numbering them with findstr, then stripping:
for /F "tokens=1* delims=:" %%a in ('findstr /n "^" "file.txt"') do (
    echo Line %%a: %%b
)
:: findstr /n "^" numbers every line "N:text"; tokens=1* splits at the FIRST colon.
:: (A truly-empty line becomes "N:" so %%b is empty but the line is still seen.)
```

### 13.4 Reading input with `set /p` from a file

```bat
:: Read the FIRST line of a file into a variable:
set /p firstline=<"data.txt"
echo First line: %firstline%
```

`set /p var=<file` is a neat way to grab just line one (e.g., a version number stored in a file).

---

## 14. Escaping & Special Characters

This is where batch earns its reputation. Several characters are special to `cmd` and must be escaped depending on context.

### 14.1 Special characters

| Char | Meaning | How to make it literal |
|---|---|---|
| `^` | Escape character | `^^` |
| `%` | Variable expansion | `%%` (in a `.bat`); `%%`also for literal % |
| `&` | Command separator | `^&` |
| `|` | Pipe | `^|` |
| `<` `>` | Redirection | `^<` `^>` |
| `(` `)` | Grouping | `^(` `^)` (often only needed inside blocks) |
| `"` | Quote (turns off most escaping inside) | `""` (context-dependent) |
| `!` | Delayed expansion (when enabled) | `^^!` or `^!` (see below) |

### 14.2 `^` — the escape character

`^` tells cmd "treat the next char literally."

```bat
echo This ^& that          :: → This & that  (without ^, & would split commands)
echo 3 ^> 2 is true        :: → 3 > 2 is true  (without ^, > would redirect!)
echo Use the ^| pipe       :: → Use the | pipe
```

`^` at the **end of a line** is the line-continuation character (it escapes the newline):

```bat
robocopy "C:\src" "D:\dst" ^
    /MIR /R:1 /W:1 ^
    /XJ /NFL /NDL
:: This is ONE command split across three lines. Don't put anything after ^.
```

### 14.3 `%%` — literal percent in scripts

Inside a `.bat`, a single `%` may start a variable lookup. To print a literal `%`, double it:

```bat
echo Progress: 100%%      :: → Progress: 100%
```

At the **command prompt** (not a script) you use a single `%` to print a literal percent. (Yet another script-vs-prompt difference.)

### 14.4 Escaping inside `for` command blocks

When a `for /F` runs a command containing a pipe or redirection, those chars belong to the *inner* command and must be escaped so `for` doesn't grab them:

```bat
:: Count lines: the pipe is part of the command run by for, so escape it:
for /F %%c in ('type "f.txt" ^| find /c /v ""') do echo %%c lines
```

### 14.5 The nightmare: escaping + delayed expansion

When `EnableDelayedExpansion` is on, `!` becomes special, and the escaping rules for `!` and `^` interact in confusing ways. Rough guide:

- With delayed expansion **off**, escape `!` is not needed; escape `^` as `^^`.
- With delayed expansion **on**, a literal `!` needs `^^!` (the `^` itself may need escaping because two phases of parsing occur).
- Strings read into variables that contain `!` or `^` can be corrupted when delayed expansion is on.

```bat
@echo off
setlocal EnableDelayedExpansion
echo Warning^^! Something happened^^!     :: → Warning! Something happened!
echo A caret looks like ^^^^             :: messy — prints a single ^
endlocal
```

**Practical advice:** if you find yourself fighting `!`/`^` escaping inside delayed expansion, that's the universe telling you to switch to PowerShell for this task. For pure-batch code, **avoid putting `!` and `^` into data** whenever you can, and toggle delayed expansion off for the lines that handle such data.

### 14.6 Quoting reduces escaping

Inside double quotes, many special chars (`&`, `|`, `<`, `>`) lose their power, so you often don't need `^`:

```bat
echo "This & that | works fine inside quotes"
:: But the quotes are now PART of the echoed text. Tradeoffs everywhere.
```

---

## 15. Useful System Commands & One-Liners

These are external tools you'll constantly call from scripts. All have `/?` help.

### 15.1 Networking

```bat
ipconfig                       :: show network config
ipconfig /all                  :: detailed (MAC, DNS, DHCP)
ipconfig /flushdns             :: clear DNS cache
ping -n 4 example.com          :: 4 pings (Windows uses -n, not -c)
ping -n 1 -w 1000 host         :: 1 ping, 1000ms timeout
nslookup example.com           :: DNS lookup
netstat -ano | findstr :443    :: who's listening on port 443 (PID in last col)
```

### 15.2 Processes

```bat
tasklist                       :: list running processes
tasklist | findstr /i node     :: filter
tasklist /FI "IMAGENAME eq chrome.exe"  :: filter syntax
taskkill /IM notepad.exe       :: kill by image name
taskkill /F /IM chrome.exe     :: /F force-kills
taskkill /PID 1234 /T          :: kill PID 1234 and its child tree
```

### 15.3 Services

```bat
sc query wuauserv              :: query a service's state
sc start  wuauserv             :: start  (needs admin)
sc stop   wuauserv             :: stop   (needs admin)
sc config wuauserv start= auto :: set startup type (note the SPACE after =)
net start                      :: list running services
net stop  "Print Spooler"      :: stop by display name (admin)
net start "Print Spooler"      :: start it again
```

### 15.4 System info

```bat
systeminfo                     :: big dump of OS/hardware info
systeminfo | findstr /C:"Total Physical Memory"
ver                            :: Windows version
hostname                       :: machine name
whoami                         :: current user (DOMAIN\user)
whoami /groups                 :: group memberships (check for admin)
```

### 15.5 `wmic` — deprecated; use PowerShell/CIM ⚡

`wmic` (Windows Management Instrumentation command-line) was the old way to query system data. **It is deprecated** and being removed from default Windows installs (it's now an optional "feature on demand"). **Do not write new scripts on `wmic`.** Use PowerShell's CIM cmdlets instead:

```bat
:: OLD (deprecated, may not exist on a 2026 box):
wmic cpu get name

:: NEW — call PowerShell from your batch file:
powershell -NoProfile -Command "Get-CimInstance Win32_Processor | Select-Object Name"
powershell -NoProfile -Command "(Get-CimInstance Win32_OperatingSystem).Caption"
```

### 15.6 Registry — `reg`

```bat
:: Query a value:
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"

:: Add a value (string):
reg add "HKCU\Software\MyApp" /v Version /t REG_SZ /d "1.0" /f
:: /v name  /t type  /d data  /f = force (no prompt)

:: Delete a value:
reg delete "HKCU\Software\MyApp" /v Version /f

:: Read one value into a variable:
for /F "tokens=2,*" %%a in ('reg query "HKCU\Software\MyApp" /v Version 2^>nul ^| findstr Version') do set "ver=%%b"
echo App version: %ver%
```

> **⚡ Editing HKLM and most system keys requires Administrator.** Mistakes in the registry can break Windows — back up first (`reg export`).

### 15.7 Scheduled tasks — `schtasks`

```bat
:: Create a daily task at 02:00 that runs a backup script:
schtasks /Create /SC DAILY /ST 02:00 /TN "NightlyBackup" /TR "C:\scripts\backup.cmd" /F
schtasks /Query /TN "NightlyBackup"     :: inspect it
schtasks /Run   /TN "NightlyBackup"     :: run it now
schtasks /Delete /TN "NightlyBackup" /F :: remove it
```

### 15.8 File associations — `assoc` / `ftype`

```bat
assoc .txt                     :: → .txt=txtfile  (extension → file type)
ftype txtfile                  :: → txtfile=%SystemRoot%\system32\NOTEPAD.EXE %1
:: Together they control what double-clicking a file does. Changing them needs admin.
```

### 15.9 `start` — launch programs, files, URLs

`start` launches something and (by default) **returns immediately** without waiting.

```bat
start notepad.exe                 :: launch Notepad, don't wait
start "" "C:\Program Files\App\app.exe"  :: the "" is a (dummy) WINDOW TITLE — needed
                                  :: when the path is quoted, or start treats the
                                  :: quoted path as a title! Classic gotcha.
start "" "https://example.com"    :: open a URL in the default browser
start "" "C:\docs\readme.pdf"     :: open a file with its default app
start /wait setup.exe /silent     :: WAIT for it to finish before continuing
start /min "" mytool.exe          :: start minimized; /max maximized
start /b mytool.exe               :: run in the SAME window (no new window)
```

> **⚡ The empty-title trap:** `start "C:\path with spaces\x.exe"` fails because `start` reads the first quoted string as the **window title**. Always pass an explicit (often empty) title first: `start "" "C:\path...\x.exe"`.

### 15.10 `timeout` and `choice`

```bat
timeout /t 5                   :: wait 5 seconds, user can skip with a key
timeout /t 5 /nobreak          :: wait 5s, ignore keypresses
timeout /t 5 /nobreak >nul     :: ...silently

:: choice: present options, set errorlevel to the chosen index (1-based):
choice /C YN /M "Proceed"
if errorlevel 2 goto cancel    :: errorlevel 2 = second choice (N)
if errorlevel 1 goto proceed   :: errorlevel 1 = first choice (Y)
:: NOTE: test HIGHER numbers FIRST because errorlevel N means ">= N" (§6.6)!
```

> `choice` is the **right** tool for menus/confirmations — it returns a numeric errorlevel and supports timeouts (`/T 10 /D Y` = default Y after 10s). Prefer it over `set /p` for fixed choices.

### 15.11 Date and time — and locale dependence ⚡

```bat
date /t                        :: print date (format DEPENDS ON LOCALE)
time /t                        :: print time
echo %date% %time%             :: built-in vars (also locale-dependent!)
```

> **⚡ BIG warning:** `%date%` and `%time%` formats are **locale-dependent**. On a US machine `%date%` might be `Sat 06/21/2026`; on a UK machine `21/06/2026`; elsewhere with different separators. **Never** parse them with fixed `tokens`/`delims` assuming a format, or your script breaks on other machines. For a stable, locale-independent timestamp, use PowerShell or `wmic os get localdatetime` (deprecated) — the modern way:
> ```bat
> for /F "usebackq delims=" %%t in (`powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd_HH-mm-ss"`) do set "stamp=%%t"
> echo %stamp%      :: → 2026-06-21_10-30-45  (ALWAYS this format, any locale)
> ```

---

## 16. Practical Recipes

### 16.1 Get the script's directory & run from anywhere (beginner)

```bat
@echo off
setlocal
:: %~dp0 = folder of THIS script (with trailing \). pushd makes it the CWD.
pushd "%~dp0"
echo Working from the script's own folder: %CD%
:: ... your file operations are now relative to the script ...
popd
endlocal
```

### 16.2 A menu with `choice` + `goto` (intermediate)

```bat
@echo off
setlocal EnableDelayedExpansion
:menu
cls
echo ==============================
echo   PROJECT LAUNCHER
echo ==============================
echo   1. Start dev server
echo   2. Run tests
echo   3. Open project folder
echo   4. Quit
echo.
choice /C 1234 /N /M "Choose an option: "
:: choice sets errorlevel = position of chosen key (1..4). Test HIGH to LOW.
if errorlevel 4 goto quit
if errorlevel 3 goto openfolder
if errorlevel 2 goto runtests
if errorlevel 1 goto devserver

:devserver
    echo Starting dev server...
    REM npm run dev
    pause
    goto menu
:runtests
    echo Running tests...
    REM npm test
    pause
    goto menu
:openfolder
    start "" "%~dp0"
    goto menu
:quit
    echo Bye!
endlocal
exit /b 0
```

### 16.3 A yes/no confirm prompt (beginner)

```bat
@echo off
choice /C YN /M "Delete all temp files"
if errorlevel 2 (
    echo Cancelled.
    exit /b 0
)
echo Deleting...
del /q "%TEMP%\*.tmp" 2>nul
echo Done.
```

### 16.4 Logging with a locale-safe timestamp (intermediate)

```bat
@echo off
setlocal
:: Get a sortable, locale-independent timestamp via PowerShell:
for /F "usebackq delims=" %%t in (`powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd_HH-mm-ss"`) do set "stamp=%%t"

set "logdir=%~dp0logs"
if not exist "%logdir%" md "%logdir%"
set "logfile=%logdir%\run_%stamp%.log"

call :log "Script started"
call :log "Doing work..."
REM ... real work, redirect tool output too:  mytool.exe >> "%logfile%" 2>&1
call :log "Script finished"
echo Log written to: %logfile%
endlocal
exit /b 0

:log
    :: Append a timestamped line to the log AND echo to console.
    for /F "usebackq delims=" %%n in (`powershell -NoProfile -Command "Get-Date -Format HH:mm:ss"`) do set "now=%%n"
    echo [%now%] %~1
    echo [%now%] %~1>>"%logfile%"
    goto :eof
```

### 16.5 Check a program exists before using it — `where` (intermediate)

```bat
@echo off
setlocal
call :need git   || exit /b 1
call :need node  || exit /b 1
echo All required tools are present.
node --version
exit /b 0

:need
    where %1 >nul 2>&1
    if errorlevel 1 (
        echo ERROR: required program "%1" was not found on PATH.
        exit /b 1
    )
    exit /b 0
```

### 16.6 Wait/retry loop (intermediate)

```bat
@echo off
setlocal EnableDelayedExpansion
set "maxTries=5"
set "try=0"
:retry
set /a try+=1
echo Attempt !try! of %maxTries%...
ping -n 1 myserver.local >nul 2>&1
if not errorlevel 1 (
    echo Server is up.
    goto :ok
)
if !try! geq %maxTries% (
    echo Giving up after %maxTries% attempts.
    exit /b 1
)
echo Not ready, waiting 3s...
timeout /t 3 /nobreak >nul
goto retry
:ok
echo Proceeding.
endlocal
exit /b 0
```

### 16.7 Backup script with robocopy (intermediate)

```bat
@echo off
setlocal
:: --- config ---
set "SRC=C:\Users\%USERNAME%\Documents"
set "DST=D:\Backups\Documents"
:: ---------------

if not exist "%DST%" md "%DST%"

echo Backing up:
echo   FROM %SRC%
echo   TO   %DST%
echo.

robocopy "%SRC%" "%DST%" /MIR /R:2 /W:5 /XJ /NP /NFL /NDL /LOG+:"%~dp0backup.log"
:: /MIR mirror, /R:2 retry twice, /W:5 wait 5s, /XJ skip junctions,
:: /NP no per-file %, /NFL /NDL quiet file/dir lists, /LOG+ append to a log.

:: Robocopy: <8 = success-ish, >=8 = failure (see §11.6).
if %errorlevel% geq 8 (
    echo BACKUP FAILED (code %errorlevel%). See backup.log.
    exit /b 1
)
echo Backup complete (robocopy code %errorlevel%).
exit /b 0
```

### 16.8 Dev-environment setup script (intermediate)

```bat
@echo off
setlocal
echo === Setting up project ===

call :need git  || exit /b 1
call :need node || exit /b 1

pushd "%~dp0"
echo Installing dependencies...
call npm install || (echo npm install failed & popd & exit /b 1)
echo Creating .env if missing...
if not exist ".env" copy ".env.example" ".env" >nul
popd

echo Setup complete. Run "npm run dev" to start.
exit /b 0

:need
    where %1 >nul 2>&1 || (echo Missing required tool: %1 & exit /b 1)
    exit /b 0
```

### 16.9 Self-elevating to Administrator (advanced)

A script that needs admin rights can relaunch itself elevated using a PowerShell `Start-Process -Verb RunAs` trick (modern, reliable):

```bat
@echo off
:: --- Self-elevation block: put at the very top of the script ---
net session >nul 2>&1
if %errorlevel% neq 0 (
    echo Requesting administrator privileges...
    powershell -NoProfile -Command "Start-Process -FilePath '%~f0' -ArgumentList '%*' -Verb RunAs"
    exit /b
)
:: --- From here on, we are running ELEVATED ---
echo Now running as administrator.
whoami /groups | find "S-1-16-12288" >nul && echo (High integrity confirmed)

:: ... admin-only work, e.g.:
:: net stop "SomeService"
:: reg add "HKLM\..." ...

pause
exit /b 0
```

How it works: `net session` only succeeds when elevated, so a nonzero errorlevel means "not admin" → we relaunch ourselves (`%~f0` = full path of this script) via PowerShell's `RunAs` verb (which triggers the UAC prompt), then exit the non-elevated instance. The elevated copy passes the `net session` check and proceeds.

> **⚡ Caveats:** the elevated instance starts in `C:\Windows\System32` (not your folder) — use `%~dp0`/`pushd` for paths. Arguments with spaces/quotes can be mangled in the relaunch; keep them simple or rework the quoting.

### 16.10 Reading config values from an .ini-style file (advanced)

```bat
@echo off
setlocal EnableDelayedExpansion
set "cfg=%~dp0settings.ini"

:: settings.ini contains lines like:   key=value   (and # comments)
:: Parse key=value pairs into variables named CFG_<key>:
for /F "usebackq eol=# tokens=1,* delims==" %%k in ("%cfg%") do (
    set "CFG_%%k=%%v"
)

echo Server : !CFG_server!
echo Port   : !CFG_port!
echo User   : !CFG_user!
endlocal
```

For a single value, the `:getval` subroutine pattern:

```bat
:getval
    :: %1 = key name. Returns value in variable "value".
    set "value="
    for /F "usebackq eol=# tokens=1,* delims==" %%k in ("%cfg%") do (
        if /i "%%k"=="%~1" set "value=%%v"
    )
    goto :eof
```

---

## 17. Unicode & Encoding

### 17.1 Code pages and `chcp`

`cmd.exe` historically uses an **OEM code page** (e.g., `437` US, `850` Western Europe, `932` Japanese) — *not* UTF-8. This is why accented or non-Latin characters often display as garbage (`├®` instead of `é`). The active code page is shown/set with `chcp`:

```bat
chcp                  :: show current code page (e.g., "Active code page: 437")
chcp 65001            :: switch to UTF-8
```

### 17.2 `chcp 65001` — UTF-8 mode ⚡

To handle UTF-8 text correctly, switch the code page at the top of your script:

```bat
@echo off
chcp 65001 >nul       :: UTF-8; >nul hides the "Active code page" message
echo café — naïve — 日本語
```

> **⚡ Caveats with 65001:**
> - The **batch file itself** should be saved as **UTF-8 *without* BOM**. A BOM at the start of a `.bat` can make `cmd` choke on the first line (you'll see a stray `´╗┐` or an error on `@echo off`).
> - Not every legacy console font renders all glyphs; the default font may show boxes even when the bytes are correct.
> - Some external tools assume the OEM code page; switching to 65001 can break *their* output parsing. Switch back with `chcp <original>` if needed.
> - Modern Windows Terminal handles UTF-8 far better than the old conhost console.

### 17.3 BOM issues, summarized

- **Batch scripts:** save as UTF-8 **no BOM** (or ANSI). BOM breaks parsing.
- **Files you generate with `echo > file`:** they'll be in the console's current code page, no BOM. If a downstream tool needs UTF-8-with-BOM, generate it with PowerShell instead.
- If accented characters break: check (1) the file's saved encoding, (2) `chcp`, (3) the console font.

---

## 18. Gotchas & Best Practices

A consolidated list of the landmines. If your script "works on my machine" but breaks elsewhere, the cause is almost certainly on this list.

### 18.1 The big gotchas

1. **Trailing spaces in `set`.** `set x=value   ` stores the trailing spaces. **Always** `set "x=value"`. (§4.2)
2. **Delayed expansion.** `%var%` is frozen at parse time; inside loops/blocks use `setlocal EnableDelayedExpansion` and `!var!`. The #1 source of "my counter is always 0." (§8)
3. **`::` inside `()` blocks** can cause syntax errors. Use `REM` inside blocks. (§1.7)
4. **`if errorlevel N` means `>= N`**, not `== N`. Test high-to-low, or use `%errorlevel%`. (§6.6, §15.10)
5. **Empty variables in `if`.** `if %x%==y` is a syntax error when `x` is empty. Always quote: `if "%x%"=="y"`. (§6.2)
6. **`%date%`/`%time%` are locale-dependent.** Never parse them with fixed delimiters. Use PowerShell `Get-Date -Format`. (§15.11)
7. **`%%` in scripts vs `%` at the prompt** for `for` variables and literal percent. (§7, §14.3)
8. **`2>&1` must come AFTER `> file`.** (§10.3)
9. **`robocopy` exit codes**: `<8` = success, `>=8` = failure. Not 0=success. (§11.6)
10. **`start "title" prog`**: a quoted first argument is the *title*. Use `start "" "prog"`. (§15.9)
11. **Parentheses in variable values** break `if`/`for` blocks: `set "msg=Done (ok)"` then `if 1==1 (echo %msg%)` — the `)` in the value closes the block early. Use delayed expansion (`!msg!`) or escape, or restructure. (§10.5)
12. **Paths with spaces** must be quoted everywhere. (§3.2)
13. **`call` is required** to run another `.bat` and come back; without it, control never returns. (§9.4)
14. **`for /F` skips blank lines** and applies `eol`. Use `findstr /n "^"` to read every line. (§13.3)
15. **Errorlevel gets clobbered** by intervening commands (even `echo`). Snapshot it immediately: `set "rc=%errorlevel%"`. (§11.5)
16. **Don't name a variable `errorlevel`, `cd`, `random`, `date`, etc.** — you'll shadow the dynamic built-ins.
17. **Pipes spawn subshells** — `set` on the left of `|` doesn't persist. (§10.4)
18. **BOM in the .bat file** breaks `@echo off`. Save UTF-8 *without* BOM. (§17.2)

### 18.2 Best-practice script header

Start non-trivial scripts with:

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion
:: EnableExtensions: ensures command extensions are on (usually default, but
::                   guarantees for /F, set /a, etc. behave as documented).
:: EnableDelayedExpansion: turns on !var! for use inside loops/blocks.
:: Then: cd into the script dir if your files are relative:
pushd "%~dp0"

:: ... script body ...

popd
endlocal
exit /b 0
```

### 18.3 General good habits

- **Quote everything** that could contain spaces or be empty (paths and variables in `if`).
- **Check errorlevels** after important commands; `exit /b N` to propagate failures so CI notices.
- **Comment with `REM`**; reserve `::` for top-level lines only.
- **Use `%~dp0`** for files bundled with the script; never assume `%CD%`.
- **Use `choice`** for menus/confirms, not fragile `set /p` parsing.
- **Pull date/time, math beyond integers, JSON/regex, and case conversion** out to PowerShell — those are where batch is weakest.
- **Test on a clean machine / different locale** if the script will be distributed.

### 18.4 When to just use PowerShell instead

Reach for PowerShell (it's preinstalled on every Windows 11 box) when your task involves any of: real data structures (objects, arrays, JSON, CSV with quoting), floating-point or precise math, robust string/regex work, dates you need to format or compute with, error handling beyond errorlevels (`try/catch`), HTTP requests, working with the Windows API/WMI/CIM, or anything where you're spending more time fighting cmd's quirks than solving the problem. A good heuristic: **if your batch script has grown past ~50 lines or needs delayed-expansion gymnastics, rewrite it in PowerShell.** You can even call PowerShell from batch for just the hard parts, as shown throughout §15–16.

---

## 19. Study Path & Build-to-Learn Projects

### 19.1 Suggested learning order

1. **Survive the prompt (§1–3).** Open cmd, navigate with `cd /d`, list with `dir`, copy/move/delete files, run a `.bat`. Add `@echo off` and `pause`. Learn `cls`, `title`, `echo`.
2. **Variables (§4).** Practice `set "x=value"`, `set /a` math, `set /p` input, `setlocal`. Internalize the trailing-space habit.
3. **Arguments (§5).** Write scripts that take `%1`/`%*`; learn `%~dp0` and the modifier table — this unlocks reusable scripts.
4. **Conditionals & loops (§6–7).** This is the core. Drill every `for` form, especially `for /F` parsing. Memorize the quoting rules for `if`.
5. **Delayed expansion (§8).** Do the counter-in-a-loop exercise until it clicks. This is the gateway to non-trivial batch.
6. **Subroutines & flow (§9), redirection (§10), errorlevels (§11).** Now you can structure real scripts with functions and error handling.
7. **Strings, capturing output, escaping (§12–14).** The advanced, fiddly bits — learn them by need.
8. **System commands & recipes (§15–16).** Build real tools.
9. **Encoding & gotchas (§17–18).** Read these before you ship anything to other machines.

### 19.2 Build-to-learn projects

Do these in order; each forces you to use the previous section's skills.

1. **"Hello, args" launcher (beginner).** A `.bat` that prints each argument, its full path (`%~f1`), and whether it exists. Practice `%1`–`%9`, `shift`, `if exist`, modifiers.

2. **Project launcher menu (beginner→intermediate).** A `choice`-driven menu that can start a dev server, run tests, open the project folder (`start ""`), and quit — looping back to the menu. Practice `choice`, `goto`, labels, `%~dp0`. (Skeleton in §16.2.)

3. **Backup script with robocopy (intermediate).** Mirror a source folder to a backup drive, log to a timestamped file, and report success/failure honoring robocopy's exit codes. Practice robocopy, logging, errorlevel handling, locale-safe timestamps. (Skeleton in §16.7.)

4. **Dev-environment setup script (intermediate).** Check that `git`/`node`/`python` exist (`where`), clone or pull a repo, install deps, copy `.env.example`→`.env` if missing, and `exit /b` with a clear code on any failure. Practice `where`, `call`, `&&`/`||`, subroutines.

5. **Log rotator (intermediate→advanced).** Given a folder of `*.log` files, delete or archive (rename with timestamp / move to an `archive\` subfolder) any older than N days or beyond a count limit. Practice `for /R`, `%%~t`/`%%~z` modifiers, delayed expansion counters, date handling via PowerShell.

6. **Config-driven task runner (advanced).** Read a `tasks.ini` of `name=command` pairs (§16.10), present them as a `choice` menu, run the selected command, capture and report its errorlevel, and log everything. This combines parsing, delayed expansion, menus, subroutines, redirection, and error handling — essentially everything in this guide.

7. **Self-elevating maintenance script (advanced).** Auto-relaunch as admin (§16.9), then stop a service, clear a temp folder, tweak a registry value, restart the service, and log each step. Practice elevation, `sc`/`net`, `reg`, robust logging, and `%~dp0` from the System32 working directory.

By the time you finish project 6 or 7, you'll be able to read and confidently modify almost any batch file you encounter — and you'll have a sharp sense of *when the right move is to stop and write it in PowerShell instead.*

---

*End of reference. CMD is old, quirky, and not going anywhere — knowing it well is a quietly valuable skill. But always weigh it against PowerShell for new work, and never trust a batch script you haven't tested on a second machine.*
