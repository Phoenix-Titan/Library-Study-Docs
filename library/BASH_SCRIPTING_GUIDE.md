# Bash Scripting — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone who wants to go from "I can type `ls`" to writing robust, production-grade shell scripts — automation, deploy scripts, log analysis, dotfiles, CI glue. Every concept comes with runnable, heavily commented code you can paste into a `.sh` file and run. No internet needed: this document is self-contained.
>
> **Version note:** This guide targets **Bash 5.2+** (released 2022, current and stable in 2026). Bash is the default login shell on most Linux distributions and is available everywhere. Where a feature is **Bash-specific** (arrays, `[[ ]]`, advanced `${...}` parameter expansions, `(( ))`, process substitution) versus **portable POSIX `sh`** (works in `dash`, `ash`, BusyBox), it is flagged. **macOS ships ancient Bash 3.2** (frozen at that version since 2007 for GPLv3 licensing reasons) and uses **zsh** as the default interactive shell — so bashisms written against 5.2 may fail on a stock Mac. Install modern Bash via Homebrew (`brew install bash`) if you target macOS. Fast-moving or version-specific details are flagged with **⚡ Version note**.

---

## Table of Contents

1. [What Bash Is — Shells, Shebangs, Running Scripts](#1-what-bash-is)
2. [Commands, Arguments & Quoting](#2-commands-arguments--quoting)
3. [Variables, Environment & Special Parameters](#3-variables-environment--special-parameters)
4. [Output & Input — echo, printf, read, here-docs](#4-output--input)
5. [Conditionals — test, `[ ]`, `[[ ]]`, if, case](#5-conditionals)
6. [Arithmetic & Math](#6-arithmetic--math)
7. [Loops — for, while, until](#7-loops)
8. [Parameter Expansion In Depth](#8-parameter-expansion-in-depth)
9. [Globbing & Pathname Expansion](#9-globbing--pathname-expansion)
10. [Functions](#10-functions)
11. [Arrays — Indexed & Associative](#11-arrays)
12. [I/O Redirection & File Descriptors](#12-io-redirection--file-descriptors)
13. [Pipelines, Subshells & Command Substitution](#13-pipelines-subshells--command-substitution)
14. [Robust Scripting — set -euo pipefail, trap, shellcheck](#14-robust-scripting)
15. [Text Processing Toolkit — grep, sed, awk, and friends](#15-text-processing-toolkit)
16. [Job Control, Processes & Parallelism](#16-job-control-processes--parallelism)
17. [Debugging](#17-debugging)
18. [Argument Parsing — getopts & long options](#18-argument-parsing)
19. [Regular Expressions in Bash — `=~` and BASH_REMATCH](#19-regular-expressions-in-bash)
20. [Practical Patterns & Recipes](#20-practical-patterns--recipes)
21. [Portability & Gotchas](#21-portability--gotchas)
22. [Study Path & Build-to-Learn Projects](#22-study-path--build-to-learn-projects)

---

## 1. What Bash Is

### 1.1 What is a shell, and what is Bash?

A **shell** is a program that reads commands you type (or that a script contains), interprets them, and runs other programs. It is both a **command-line interpreter** (the interactive prompt) and a **programming language** (loops, variables, functions).

**Bash** = the **B**ourne-**A**gain **SH**ell. It is a superset of the original Bourne shell (`sh`) from 1979, written for the GNU project in 1989. It is the most widely installed shell and the de-facto scripting standard on Linux.

| Shell | What it is | Typical role | Notes |
|---|---|---|---|
| `sh` | The POSIX shell standard / interface | Portable scripts | On most Linux it's a *symlink* to `dash` or `bash`. |
| `bash` | Bourne-Again Shell (GNU) | Default Linux login shell, scripting | Rich features: arrays, `[[ ]]`, `(( ))`. |
| `dash` | Debian Almquist Shell | `/bin/sh` on Debian/Ubuntu | Tiny, fast, **POSIX-only** — no bashisms. |
| `ash` / BusyBox | Minimal shells | Embedded systems, Alpine Linux | POSIX-ish, very limited. |
| `zsh` | Z shell | Default on macOS (since Catalina) | Bash-compatible-ish, more interactive features. |
| `ksh` | Korn shell | Some Unix systems | Influenced Bash's arrays. |
| `fish` | Friendly Interactive Shell | Interactive use | **Not** POSIX/Bash compatible — don't write portable scripts in it. |

**Key takeaway:** "Bash scripting" and "shell scripting" overlap but are not identical. A *portable* shell script uses only POSIX features so it runs under `dash`, `ash`, BusyBox, etc. A *Bash script* may use bashisms (arrays, `[[ ]]`) that only Bash understands.

### 1.2 The shebang line

The first line of a script tells the OS which interpreter to use. It's called the **shebang** (`#!`).

```bash
#!/usr/bin/env bash
# ^ The recommended portable shebang.
# `env` searches your $PATH for `bash`, so it works whether bash is in
# /bin, /usr/bin, /usr/local/bin (Homebrew on macOS!), etc.

#!/bin/bash
# ^ Common, but hardcodes the path. Fails if bash lives elsewhere
#   (e.g. /usr/local/bin/bash on macOS Homebrew, or NixOS).

#!/bin/sh
# ^ Use this ONLY for strictly POSIX scripts. On Debian/Ubuntu this is
#   dash, NOT bash — so bashisms here will break.
```

**Rules for the shebang:**
- It must be the **very first line**, no blank lines or spaces before `#!`.
- It must be a real path to an executable.
- You can pass **one** argument: `#!/usr/bin/env bash` works, but `#!/usr/bin/env bash -e` is unreliable across systems (some `env` don't split args). Prefer `set -e` inside the script instead.

> **⚡ Version note:** `env -S` (split string) since coreutils 8.30 lets you pass multiple args: `#!/usr/bin/env -S bash -e`. Still not universal — avoid in portable scripts.

### 1.3 Making a script executable and running it

```bash
# Create a script
cat > hello.sh <<'EOF'
#!/usr/bin/env bash
echo "Hello, world!"
EOF

# Method 1: make it executable, then run it directly.
chmod +x hello.sh        # add the "execute" permission bit
./hello.sh               # the ./ is REQUIRED — current dir is not in $PATH

# Method 2: pass it to an interpreter (no execute bit or shebang needed).
bash hello.sh            # runs with bash regardless of shebang
sh hello.sh              # runs with sh (POSIX) — bashisms may fail

# Method 3: source it into the CURRENT shell (no new process).
source hello.sh          # variables/functions it defines persist in your shell
. hello.sh               # `.` is the POSIX synonym for `source`
```

**Why `./hello.sh` and not just `hello.sh`?** For security, the current directory (`.`) is **not** in your `$PATH`. If it were, a malicious `ls` script dropped in a directory could hijack the real `ls`. So you must explicitly say "run the file here": `./`.

### 1.4 Interactive vs non-interactive, login vs non-login

These four concepts control **which startup files run** — a frequent source of "why isn't my alias/PATH working?" confusion.

| Type | Meaning |
|---|---|
| **Interactive** | A shell connected to a terminal where you type commands. |
| **Non-interactive** | A shell running a script (no human typing). |
| **Login** | The first shell when you log in (SSH, console, `su -`). |
| **Non-login** | A shell started from within another (opening a new terminal tab, running `bash`). |

**Which files get read** (Bash):

| Situation | Files sourced (in order) |
|---|---|
| Interactive **login** | `/etc/profile`, then the **first** found of `~/.bash_profile`, `~/.bash_login`, `~/.profile` |
| Interactive **non-login** | `/etc/bash.bashrc` (some distros), then `~/.bashrc` |
| Non-interactive (script) | **None** by default (unless `$BASH_ENV` is set) |

```bash
# Detect if a shell is interactive (inside a script):
case $- in
  *i*) echo "interactive" ;;   # $- lists the current option flags; 'i' = interactive
  *)   echo "non-interactive" ;;
esac

# Detect login shell:
shopt -q login_shell && echo "login shell" || echo "non-login shell"
```

**The classic pattern** so your config works in *both* login and non-login shells: put everything in `~/.bashrc`, then source it from `~/.bash_profile`:

```bash
# ~/.bash_profile
# Put environment setup and PATH here, then pull in interactive config:
if [[ -f ~/.bashrc ]]; then
  source ~/.bashrc
fi
```

| File | Purpose / convention |
|---|---|
| `~/.bash_profile` | Login-shell config: set `PATH`, exported env vars, run-once setup. |
| `~/.bashrc` | Interactive non-login config: aliases, prompt (`PS1`), shell options, functions. |
| `~/.profile` | POSIX-portable login config (read by `sh`/`dash` too, and by Bash if no `.bash_profile`). |
| `~/.bash_logout` | Runs when a login shell exits (cleanup). |
| `/etc/profile`, `/etc/bash.bashrc` | System-wide equivalents. |

---

## 2. Commands, Arguments & Quoting

### 2.1 Anatomy of a command line

```bash
#         command   options/flags   arguments
#         vvvvv      vvvvvvvv         vvvvvvvvvvvv
          grep       -i -n            "error" app.log
```

The shell splits the line into **words** (tokens) separated by whitespace. The first word is the command; the rest are arguments passed to it. This splitting step is exactly why **quoting** matters so much.

### 2.2 Why quoting matters: word splitting & globbing

After variable/command expansion, the shell performs **word splitting** (on `$IFS` — space/tab/newline by default) and **pathname expansion (globbing)** on the *unquoted* result. This silently breaks scripts that handle filenames with spaces.

```bash
# BEGINNER — the classic bug:
file="my report.txt"        # a filename containing a space
rm $file                    # WRONG: expands to `rm my report.txt`
                            #   → tries to delete TWO files: "my" and "report.txt"
rm "$file"                  # RIGHT: `rm "my report.txt"` — one argument

# Globbing surprise:
pattern="*.txt"
echo $pattern               # if any .txt files exist, prints their NAMES (glob expands)
echo "$pattern"             # prints the literal string: *.txt
```

> **Rule of thumb:** **Always double-quote your variable expansions** (`"$var"`, `"$@"`, `"${arr[@]}"`) unless you have a deliberate, specific reason not to. This single habit prevents the majority of shell bugs. ShellCheck (Section 14) will nag you about every unquoted one.

### 2.3 The three (and a half) kinds of quoting

```bash
name="Ada"

# 1) Double quotes: PREVENT word-splitting & globbing,
#    but ALLOW expansion of $variables, $(commands), and `\` escapes.
echo "Hello $name, today is $(date +%A)"   # → Hello Ada, today is Sunday

# 2) Single quotes: LITERAL. Nothing is expanded. No way to embed a single quote.
echo 'Hello $name'                          # → Hello $name  (literal)
echo 'It'\''s tricky'                       # embed a ' by closing, escaping, reopening
                                            # → It's tricky

# 3) Backslash: escapes the very next character.
echo \$name        # → $name   (dollar is literal)
echo a\ b          # → a b     (space is literal, not a separator)

# 3.5) ANSI-C quoting $'...': interprets backslash escapes like a C string.
echo $'line1\nline2\ttabbed'                # real newline and tab
printf '%s\n' $'✓ done'                # → ✓ done   (\u for unicode)
```

| Construct | `$var` expand? | `$(cmd)` expand? | `\` escapes? | Globbing? |
|---|---|---|---|---|
| `"double"` | ✅ | ✅ | ✅ (limited) | ❌ |
| `'single'` | ❌ | ❌ | ❌ | ❌ |
| `$'ansi-c'` | ❌ | ❌ | ✅ (C-style) | ❌ |
| unquoted | ✅ | ✅ | ✅ | ✅ |

**Inside double quotes**, backslash only escapes: `$`, `` ` ``, `"`, `\`, and newline. Elsewhere it's literal. E.g. `echo "a\b"` prints `a\b`.

### 2.4 Comments and line continuation

```bash
# This is a comment — everything after # (when # starts a word) is ignored.
echo "hi"  # trailing comment

echo not#acomment   # NOT a comment — # is mid-word, so it's literal text

# Line continuation: a backslash at END of line joins the next line.
long_command --option-one \
             --option-two \
             argument
```

---

## 3. Variables, Environment & Special Parameters

### 3.1 Assignment — the no-spaces rule

```bash
# BEGINNER
name="Ada"          # CORRECT — no spaces around =
name = "Ada"        # WRONG — shell thinks `name` is a command with args `=` and `Ada`
name ="Ada"         # WRONG too
count=42            # numbers are just strings; no type declaration needed
empty=              # an empty (but defined) variable
```

The space rule trips up everyone once. `=` in an assignment must touch the name on the left and the value on the right with **no spaces**, because `foo = bar` parses as "run command `foo` with arguments `=` and `bar`".

### 3.2 Using variables: `$var` vs `${var}`

```bash
greeting="Hello"
echo $greeting           # Hello
echo ${greeting}         # Hello — braces are optional here...
echo "${greeting}world"  # Helloworld — but REQUIRED to delimit the name
echo "$greetingworld"    # (empty) — shell looks for a var named "greetingworld"

# Braces are also required for array elements and parameter expansion:
echo "${arr[2]}"         # array element
echo "${name:-default}"  # parameter expansion (Section 8)
```

**Best practice:** use `${var}` braces routinely; they're never wrong and they prevent the "merged name" bug.

### 3.3 Shell variables vs environment variables; `export`

A **shell variable** lives only in the current shell. An **environment variable** is exported, so it's inherited by child processes (programs you run).

```bash
myvar="local-only"        # shell variable
export PATH="$HOME/bin:$PATH"   # environment variable — child processes see it

# Export an existing variable:
config="prod"
export config

# Set an env var for ONE command only (no export needed):
DEBUG=1 ./myscript.sh     # DEBUG is set only inside that script's process

# Inspect:
env            # list all environment variables
printenv PATH  # print one env var
set            # list ALL shell vars + functions (huge output)
declare -p X   # show how X is declared (type + value)
unset myvar    # remove a variable
```

```bash
# Demonstration of inheritance:
parent_var="seen?"
export parent_var
bash -c 'echo "child sees: $parent_var"'   # child sees: seen?

not_exported="hidden"
bash -c 'echo "child sees: $not_exported"' # child sees:   (empty)
```

### 3.4 `readonly` and `local`

```bash
readonly PI=3.14159     # constant — cannot be reassigned or unset
PI=3                    # → bash: PI: readonly variable (error)

myfunc() {
  local count=0        # `local` scopes the variable to this function only
  count=$((count + 1))
  echo "$count"
}
# Without `local`, count would leak into the global scope. ALWAYS use local
# inside functions. (Bash-specific; `local` is not strict POSIX but is in dash too.)
```

### 3.5 Special parameters

| Parameter | Meaning |
|---|---|
| `$0` | Name of the script (or shell). |
| `$1`, `$2`, … `$9` | Positional arguments. `${10}` and up need braces. |
| `$#` | Number of positional arguments. |
| `$@` | All positional args, as **separate** words (when quoted). |
| `$*` | All positional args, as **one** word joined by first char of `$IFS` (when quoted). |
| `$?` | Exit status of the last command (0 = success). |
| `$$` | PID of the current shell — handy for unique temp names. |
| `$!` | PID of the last background command. |
| `$-` | Current shell option flags. |
| `$_` | Last argument of the previous command. |

```bash
#!/usr/bin/env bash
echo "Script name: $0"
echo "First arg:   $1"
echo "Arg count:   $#"
echo "All args:    $@"
echo "My PID:      $$"

# Run: ./args.sh apple banana
# Script name: ./args.sh
# First arg:   apple
# Arg count:   2
# All args:    apple banana
```

### 3.6 `"$@"` vs `"$*"` — the critical difference

This is one of the most important distinctions in shell scripting. **Almost always use `"$@"`.**

```bash
#!/usr/bin/env bash
# Save as demo.sh, run: ./demo.sh "first arg" second

show() {
  echo "--- $1 ---"
  local i=1
  for x in "$@"; do          # iterate over the args passed to show()
    echo "  [$i] $x"
    ((i++))
  done
}

show "with quoted \$@"  "$@"   # NOTE: we'll re-demo below properly

# The four forms:
set -- "alpha beta" "gamma"    # set positional params to two args

echo "Quoted \$@ — preserves each arg as a separate word:"
for a in "$@"; do echo "  <$a>"; done
#   <alpha beta>
#   <gamma>

echo 'Quoted $* — joins ALL args into ONE word (separated by IFS first char):'
for a in "$*"; do echo "  <$a>"; done
#   <alpha beta gamma>

echo 'Unquoted $@ or $* — both word-split (BAD, breaks on spaces):'
for a in $@; do echo "  <$a>"; done
#   <alpha>
#   <beta>
#   <gamma>
```

**Summary:** `"$@"` = N words (the right one for passing args through). `"$*"` = 1 joined word (useful only for printing/joining). Unquoted = broken.

---

## 4. Output & Input

### 4.1 `echo` vs `printf` — why `printf` is safer

```bash
echo "Hello"              # simple, but inconsistent across shells/systems
echo -n "no newline"      # -n suppresses the trailing newline (NOT portable!)
echo -e "tab\there"       # -e enables escapes (NOT portable — sh echo differs)

# The PROBLEM: `echo` behaviour varies. On some systems `echo -n` prints "-n".
# Flags like -e/-n are not in POSIX. Variables starting with - confuse it:
var="-n"
echo "$var"               # might print nothing, or "-n", depending on shell

# printf is consistent everywhere and far more powerful:
printf 'Hello\n'                       # explicit \n — no surprises
printf '%s\n' "$var"                   # ALWAYS prints -n correctly
printf '%s = %d\n' "count" 42          # → count = 42
printf '%-10s|%5s\n' "left" "right"    # field widths: → left      |right
printf '%.2f\n' 3.14159                # → 3.14
printf '%x\n' 255                      # → ff (hex)
printf '%q\n' "a b'c"                  # shell-quoted (safe for re-input): a\ b\'c
printf '%s\n' a b c                    # format is REUSED for each arg → a / b / c (3 lines)
```

| Reason `printf` wins | Detail |
|---|---|
| Portable | Same behaviour in bash, dash, sh. |
| Safe with `-` | `printf '%s'` never misinterprets values starting with `-`. |
| Formatting | Field widths, precision, hex, padding. |
| Reuse | Format string cycles over extra args (great for tables). |

**Rule:** use `printf` in scripts. Reserve `echo` for quick interactive one-liners.

### 4.2 Reading input with `read`

```bash
# BEGINNER — basic prompt:
read -p "What is your name? " name      # -p shows a prompt (Bash-specific)
echo "Hi, $name"

# ALWAYS use -r in scripts: without it, backslashes are interpreted (mangling input).
read -r line                            # read one line into `line`, backslashes literal

# Read into multiple variables (split on $IFS):
read -r first last <<< "Ada Lovelace"   # first=Ada  last=Lovelace

# Read silently (passwords) with -s:
read -rs -p "Password: " pass; echo     # -s = no echo; trailing echo adds newline

# Read with a timeout (-t) and a single char (-n):
if read -rt 5 -n 1 -p "Continue? [y/N] " ans; then
  echo
else
  echo " (timed out)"
fi

# Read into an array with -a (Bash):
read -ra words <<< "one two three"      # words=(one two three)
echo "${words[1]}"                      # two
```

**Controlling splitting with `IFS`:** `read` splits its input using `$IFS`. To read a whole line verbatim (no leading/trailing whitespace trimming, no splitting), set `IFS=` for that command:

```bash
IFS= read -r line                       # the canonical "read exact line" idiom

# Parse a CSV-ish line on commas:
IFS=',' read -r col1 col2 col3 <<< "a,b,c"
echo "$col2"                            # b
```

### 4.3 Here-documents and here-strings

A **here-document** (`<<`) feeds a multi-line block to a command's stdin. A **here-string** (`<<<`) feeds a single string.

```bash
# Here-doc: variables and $(...) ARE expanded (delimiter unquoted):
name="Ada"
cat <<EOF
Hello, $name!
Today is $(date +%F).
EOF

# Here-doc with QUOTED delimiter: NOTHING is expanded (literal):
cat <<'EOF'
This $name is literal, and $(date) is not run.
EOF

# Here-doc with <<- : leading TABS (only tabs!) are stripped, so you can indent:
if true; then
	cat <<-EOF
		Indented with tabs in the source,
		but the tabs are stripped from output.
	EOF
fi

# Here-string: feed one string to stdin:
grep "foo" <<< "foo bar baz"            # → foo bar baz
wc -c <<< "hello"                       # → 6  (5 chars + newline read adds one)

# Common use: write a config file
cat > /tmp/app.conf <<'EOF'
[server]
port = 8080
EOF
```

---

## 5. Conditionals

### 5.1 Exit status: the foundation of all conditionals

Every command returns an **exit status**: `0` = success, non-zero (1–255) = failure. Bash conditionals branch on this, **not** on a boolean type.

```bash
ls /etc > /dev/null         # run a command
echo $?                     # 0  → it succeeded

ls /nonexistent 2>/dev/null
echo $?                     # non-zero → it failed

# `true` and `false` are commands that just set exit status:
true;  echo $?              # 0
false; echo $?             # 1
```

### 5.2 `if` / `elif` / `else`

```bash
# The thing after `if` is a COMMAND. if runs it and branches on its exit status.
if grep -q "ERROR" app.log; then        # -q = quiet, just sets exit status
  echo "Errors found"
elif grep -q "WARN" app.log; then
  echo "Only warnings"
else
  echo "All clean"
fi

# `[ ... ]` is literally a command named `test`. The brackets are arguments!
if [ "$count" -gt 10 ]; then            # spaces around [ and ] are MANDATORY
  echo "big"
fi
```

### 5.3 `test` / `[ ]` vs `[[ ]]`

`[ ]` is the POSIX `test` command (portable). `[[ ]]` is a **Bash keyword** (not portable to `dash`) that is safer and more powerful.

```bash
# [[ ]] does NOT word-split or glob its operands, so unquoted vars are safe-ish:
file="my file.txt"
[[ -f $file ]] && echo "exists"         # works even unquoted (still quote for clarity)
[ -f $file ]   && echo "exists"         # BREAKS — splits into two args → syntax error

# [[ ]] supports && and || directly; [ ] needs -a/-o (deprecated) or chaining:
[[ $age -ge 18 && $age -lt 65 ]] && echo "working age"
[ "$age" -ge 18 ] && [ "$age" -lt 65 ] && echo "working age"   # POSIX way

# [[ ]] supports pattern matching with == and regex with =~ :
[[ $name == A* ]]   && echo "starts with A"      # glob pattern (no quotes on RHS!)
[[ $name == "A*" ]] && echo "literally A*"       # quotes → literal match
[[ $email =~ ^[^@]+@[^@]+$ ]] && echo "looks like email"   # regex (Section 19)
```

| Feature | `[ ]` (test, POSIX) | `[[ ]]` (Bash keyword) |
|---|---|---|
| Portable to `sh`/`dash` | ✅ | ❌ |
| Word-splitting of operands | Yes (must quote!) | No (safer) |
| `&&` / `||` inside | No (use `-a`/`-o`, deprecated) | ✅ |
| `==` glob matching | No | ✅ |
| `=~` regex matching | No | ✅ |
| `<` `>` string compare | Needs escaping `\<` | ✅ literal |

**Recommendation:** use `[[ ]]` in Bash scripts; use `[ ]` only when you need POSIX portability.

### 5.4 String, numeric, and file tests

```bash
# --- String tests ---
[[ -z "$s" ]]        # true if string is empty (zero length)
[[ -n "$s" ]]        # true if string is non-empty
[[ "$a" == "$b" ]]   # equal (string)
[[ "$a" != "$b" ]]   # not equal
[[ "$a" < "$b" ]]    # lexicographically less (in [[ ]]; locale-aware)

# --- Numeric tests (use these, NOT < > which are string compares) ---
[[ "$x" -eq "$y" ]]  # equal
[[ "$x" -ne "$y" ]]  # not equal
[[ "$x" -lt "$y" ]]  # less than
[[ "$x" -le "$y" ]]  # less than or equal
[[ "$x" -gt "$y" ]]  # greater than
[[ "$x" -ge "$y" ]]  # greater than or equal
# (Or use arithmetic context: (( x < y )) — see Section 6.)

# --- File tests ---
[[ -e path ]]   # exists (any type)
[[ -f path ]]   # exists and is a regular file
[[ -d path ]]   # exists and is a directory
[[ -L path ]]   # is a symbolic link
[[ -r path ]]   # readable
[[ -w path ]]   # writable
[[ -x path ]]   # executable
[[ -s path ]]   # exists and size > 0 (non-empty)
[[ a -nt b ]]   # a is newer than b (modification time)
[[ a -ot b ]]   # a is older than b
```

### 5.5 `&&` and `||` — short-circuit logic

```bash
# && runs the right side ONLY IF the left succeeded (exit 0).
# || runs the right side ONLY IF the left failed (exit non-zero).

mkdir -p build && cd build              # cd only if mkdir succeeded
command -v git >/dev/null || echo "git not installed"

# Common one-liner if/else:
[[ -f config.yml ]] && echo "found" || echo "missing"
# ⚠️ GOTCHA: this is NOT a true if/else. If the `&&` part fails, the `||`
#   part ALSO runs. Use a real if/then/else when the success branch can fail.
```

### 5.6 `case` — pattern-based branching

```bash
# Cleaner than a long if/elif chain when matching one value against patterns.
read -rp "Choose [start|stop|restart]: " action
case "$action" in
  start)
    echo "Starting..."
    ;;                          # ;; ends a branch
  stop)
    echo "Stopping..."
    ;;
  restart|reload)               # | = OR multiple patterns
    echo "Restarting..."
    ;;
  st*)                          # glob patterns are allowed
    echo "Ambiguous st-something"
    ;;
  *)                            # default case (matches anything)
    echo "Unknown action: $action"
    exit 1
    ;;
esac

# ⚡ Version note: Bash 4+ supports ;& (fall through to next) and ;;& (test next pattern).
case "$x" in
  a) echo "a";;&                # ;;& : also test the following patterns
  [a-z]) echo "lowercase";;
esac
```

---

## 6. Arithmetic & Math

### 6.1 Integer arithmetic — `(( ))` and `$(( ))`

Bash does **integer-only** arithmetic natively. Use `$(( ))` to get a *value*; use `(( ))` as a *command* (sets exit status, used in conditions/loops).

```bash
# $(( )) — arithmetic EXPANSION, yields the result as text:
x=$(( 3 + 4 ))           # 7
y=$(( x * 2 ))           # 14 — note: NO $ needed on vars inside (( ))
z=$(( (1 + 2) * 3 ))     # 9
echo "$(( 2 ** 10 ))"    # 1024 — exponentiation
echo "$(( 17 % 5 ))"     # 2 — modulo
echo "$(( 10 / 3 ))"     # 3 — INTEGER division (truncates!)

# (( )) — arithmetic COMMAND, used for its exit status (0 if result != 0):
count=5
(( count++ ))            # post-increment
(( count += 10 ))        # compound assignment → 16
if (( count > 10 )); then echo "big"; fi    # clean numeric comparison
(( count > 100 )) && echo "huge" || echo "not huge"

# ⚠️ GOTCHA: (( )) returns exit 1 when the result is 0. With `set -e` this kills
#   your script! Example: (( i = 0 )) "fails". Use ((i = 0)) || true, or i=0.
i=0
(( i++ )) || true        # i++ evaluates to 0 (the pre-value), so exit is 1 — guard it
```

### 6.2 `let` (older syntax) and other notes

```bash
let "a = 5 + 3"          # older; (( )) is preferred and cleaner
let a++                  # works but quirky

# Bases:
echo $(( 0xff ))         # 255 (hex)
echo $(( 0o17 ))         # 15  (octal, Bash 5+) — also 017 works
echo $(( 2#1010 ))       # 10  (binary: base#number)
echo $(( 16#1F ))        # 31  (arbitrary base#number)

# ⚠️ Leading-zero gotcha: numbers with leading zeros are OCTAL in arithmetic!
n="08"
echo $(( n + 1 ))        # ERROR: 8 is not a valid octal digit
echo $(( 10#$n + 1 ))    # 9 — force base 10 with 10#
```

### 6.3 Floating point — use `bc` or `awk`

Bash has **no native floats**. Shell out to `bc` (basic calculator) or `awk`.

```bash
# --- bc (arbitrary precision) ---
echo "scale=4; 10 / 3" | bc           # 3.3333  (scale = decimal places)
echo "3.14 * 2" | bc -l               # -l loads math lib (and sets scale=20)
result=$(echo "scale=2; $a / $b" | bc)

# bc for comparisons (returns 1/0):
if (( $(echo "$price > 9.99" | bc -l) )); then echo "expensive"; fi

# --- awk (often faster, no extra process for simple jobs) ---
awk 'BEGIN { printf "%.2f\n", 10/3 }'              # 3.33
awk "BEGIN { print ($a > $b) ? \"a bigger\" : \"b bigger\" }"
echo "10 3" | awk '{ printf "%.3f\n", $1/$2 }'     # pipe values in

# --- printf can ROUND but not compute ---
printf '%.2f\n' "$(echo "10/3" | bc -l)"           # 3.33
```

| Tool | Best for |
|---|---|
| `(( ))` / `$(( ))` | Integer math (fast, built-in). |
| `bc -l` | Arbitrary-precision floats, `scale=`. |
| `awk` | Float math + formatting in one step; processing columns. |

---

## 7. Loops

### 7.1 `for` loops

```bash
# BEGINNER — iterate over a literal list:
for color in red green blue; do
  echo "$color"
done

# Iterate over arguments — ALWAYS quote "$@":
for arg in "$@"; do
  echo "arg: $arg"
done

# Iterate over command output (careful with word splitting):
for user in $(cut -d: -f1 /etc/passwd); do   # splits on IFS — OK for no-space tokens
  echo "user: $user"
done

# Brace expansion (Bash) — numeric and char sequences:
for i in {1..5};     do echo "$i"; done        # 1 2 3 4 5
for i in {0..20..5}; do echo "$i"; done        # 0 5 10 15 20 (step)
for c in {a..e};     do echo "$c"; done        # a b c d e

# ⚠️ Brace ranges can't use variables: {1..$n} does NOT work. Use seq or C-style.
n=5
for i in $(seq 1 "$n");   do echo "$i"; done    # seq with a variable
for i in $(seq 1 2 10);   do echo "$i"; done    # seq start step end

# C-style for (Bash-specific):
for (( i = 0; i < 5; i++ )); do
  echo "index $i"
done

# Glob loop — the SAFE way to iterate over files (no ls parsing!):
for f in ./*.txt; do
  [[ -e "$f" ]] || continue       # guard: if no .txt files, glob stays literal *.txt
  echo "Processing: $f"
done
```

### 7.2 `while` and `until`

```bash
# while: repeat while the command succeeds (exit 0).
count=0
while (( count < 3 )); do
  echo "count = $count"
  (( count++ ))
done

# until: repeat until the command succeeds (the inverse of while).
until ping -c1 -W1 example.com &>/dev/null; do
  echo "Waiting for network..."
  sleep 2
done

# Infinite loop:
while true; do
  echo "tick"; sleep 1
done
```

### 7.3 Reading a file line by line — the CORRECT way

This is one of the most-gotten-wrong patterns. **Do not** use `for line in $(cat file)` — it word-splits and globs every line.

```bash
# THE canonical, safe pattern:
while IFS= read -r line; do
  #     ^^^^ IFS= prevents trimming leading/trailing whitespace
  #          read -r prevents backslash interpretation
  echo "Line: $line"
done < "$file"               # redirect the file into the loop's stdin

# Handle a final line with no trailing newline ([[ -n "$line" ]] catches it):
while IFS= read -r line || [[ -n "$line" ]]; do
  echo "Line: $line"
done < "$file"

# Read a command's output line by line via process substitution (Section 12)
# — this keeps the loop in the CURRENT shell so variables persist:
count=0
while IFS= read -r line; do
  (( count++ ))
done < <(grep "ERROR" app.log)
echo "Found $count errors"

# ⚠️ PIPE GOTCHA: `cmd | while read ...` runs the while in a SUBSHELL,
#   so any variables it sets are LOST after the loop:
count=0
grep ERROR app.log | while read -r l; do (( count++ )); done
echo "$count"                # → 0 !! (the subshell's count was discarded)
# Fixes: use < <(...) process substitution, OR `shopt -s lastpipe` (Bash 4.2+,
#   non-interactive only), OR redirect from a file.

# Split each line into fields:
while IFS=',' read -r name age city; do
  printf 'name=%s age=%s city=%s\n' "$name" "$age" "$city"
done < data.csv
```

### 7.4 `break` and `continue`

```bash
for i in {1..10}; do
  (( i == 3 )) && continue     # skip the rest of THIS iteration
  (( i == 6 )) && break        # exit the loop entirely
  echo "$i"                    # prints 1 2 4 5
done

# break N / continue N affect N levels of nested loops:
for i in {1..3}; do
  for j in {1..3}; do
    (( j == 2 )) && break 2    # break out of BOTH loops
    echo "$i.$j"
  done
done
```

---

## 8. Parameter Expansion In Depth

Parameter expansion is `${...}` — a powerful mini-language for manipulating variable values **without** calling external tools like `sed`/`cut`. Most of these are **Bash/ksh features**, not strictly POSIX (defaults `:-` `:=` `:?` `:+` ARE POSIX; substring/replace/case are not).

### 8.1 Default values and presence tests

```bash
unset name; greeting=""

echo "${name:-Anonymous}"   # USE default if unset OR empty → Anonymous
echo "${name-Anonymous}"    # USE default only if UNSET (empty stays empty)
echo "${name:=Anonymous}"   # ASSIGN default to name if unset/empty, then use it
echo "${name:?must be set}" # ERROR & exit if unset/empty (great for required vars)
echo "${greeting:+yes}"     # use "yes" only IF greeting is SET & non-empty (else empty)

# Practical: required argument with a helpful message
: "${1:?Usage: $0 <input-file>}"   # the leading : is a no-op that triggers expansion
```

| Form | If var is set & non-empty | If empty | If unset |
|---|---|---|---|
| `${v:-d}` | `$v` | `d` | `d` |
| `${v-d}` | `$v` | `""` (empty) | `d` |
| `${v:=d}` | `$v` | assign+use `d` | assign+use `d` |
| `${v:?msg}` | `$v` | error | error |
| `${v:+d}` | `d` | `""` | `""` |

### 8.2 Length, substring, and indexing

```bash
str="Hello, World"
echo "${#str}"            # 12 — length in characters
echo "${str:7}"           # World — substring from offset 7 to end
echo "${str:7:3}"         # Wor — offset 7, length 3
echo "${str: -5}"         # World — negative offset (NOTE the space! else it's :-default)
echo "${str:(-5):3}"      # Wor — parentheses also disambiguate negative offset

arr=(a b c d e)
echo "${arr[@]:1:2}"      # b c — array slice: from index 1, take 2
echo "${#arr[@]}"         # 5 — number of elements
```

### 8.3 Pattern removal (prefix/suffix stripping)

`#` removes from the **front**, `%` removes from the **back**. Doubled (`##`/`%%`) = greedy (longest match).

```bash
path="/home/ada/report.tar.gz"

echo "${path##*/}"        # report.tar.gz   — basename (strip longest leading */)
echo "${path%/*}"         # /home/ada       — dirname  (strip shortest trailing /*)
echo "${path#*/}"         # home/ada/report.tar.gz  (strip shortest leading */)

file="report.tar.gz"
echo "${file%.gz}"        # report.tar      — strip ".gz"
echo "${file%%.*}"        # report          — strip everything from FIRST dot (greedy)
echo "${file%.*}"         # report.tar      — strip from LAST dot (extension off)
echo "${file##*.}"        # gz              — the extension only

# Memory aid: on a US keyboard, # is left of $ (front), % is right (back).
```

### 8.4 Search and replace

```bash
s="foo bar foo baz"
echo "${s/foo/XXX}"       # XXX bar foo baz   — replace FIRST match
echo "${s//foo/XXX}"      # XXX bar XXX baz   — replace ALL (//)
echo "${s/#foo/XXX}"      # XXX bar foo baz   — anchor at START (#)
echo "${s/%baz/XXX}"      # foo bar foo XXX   — anchor at END (%)
echo "${s// /_}"          # foo_bar_foo_baz   — replace spaces with underscores
echo "${s//o}"            # f bar f baz       — delete all 'o' (empty replacement)

# Patterns are GLOBS, not regex:
files="a.txt b.log c.txt"
echo "${files//*.txt/REDACTED}"   # globs apply

# ⚡ Version note: Bash 5.2 added & in replacement = the matched text, and
#   ${var/#/prefix} style anchoring is stable. Older bash differs slightly.
```

### 8.5 Case modification (Bash 4+)

```bash
name="Ada Lovelace"
echo "${name^^}"          # ADA LOVELACE  — uppercase all
echo "${name,,}"          # ada lovelace  — lowercase all
echo "${name^}"           # Ada Lovelace  — uppercase first char
echo "${name,}"           # ada Lovelace  — lowercase first char
echo "${name^^[al]}"      # uppercase only matching chars (a and l) → AdA LoveLAce-ish

# ⚡ Version note: ^^ , , were added in Bash 4.0. Not in Bash 3.2 (macOS!) or POSIX.
#   Portable alternative: tr '[:lower:]' '[:upper:]'
echo "$name" | tr '[:lower:]' '[:upper:]'
```

### 8.6 Indirection and other tricks

```bash
# Indirect expansion: use the VALUE of one var as the NAME of another.
color="favorite"
favorite="blue"
echo "${!color}"          # blue — ${!name} dereferences

# List variable names matching a prefix:
prefix_one=1; prefix_two=2
echo "${!prefix_@}"       # prefix_one prefix_two

# ⚡ Version note: nameref (declare -n) is the modern alternative to ${!x}:
declare -n ref=favorite
echo "$ref"               # blue
ref="green"; echo "$favorite"   # green — writes through the reference
```

---

## 9. Globbing & Pathname Expansion

### 9.1 Globs are NOT regex

The shell expands **glob patterns** (a.k.a. wildcards) into matching filenames **before** running the command. Globs are simpler than regular expressions and mean different things.

| Glob | Matches | Regex equivalent (roughly) |
|---|---|---|
| `*` | any string (incl. empty) | `.*` |
| `?` | exactly one character | `.` |
| `[abc]` | one of a, b, c | `[abc]` |
| `[a-z]` | one char in range | `[a-z]` |
| `[!abc]` / `[^abc]` | one char NOT in set | `[^abc]` |
| `[[:digit:]]` | POSIX class | `[0-9]` |

```bash
ls *.txt          # all files ending in .txt
ls report?.log    # report1.log, reportA.log, but not report10.log
ls [A-Z]*         # files starting with an uppercase letter
ls *.{jpg,png}    # brace expansion (NOT a glob) → *.jpg *.png

# ⚠️ If NO files match, by default the glob is left LITERAL (passed unchanged):
ls *.nonexistent  # → ls: cannot access '*.nonexistent' (the * was not expanded)
```

### 9.2 Important glob-related shell options (`shopt`)

```bash
shopt -s nullglob     # non-matching globs expand to NOTHING (empty) instead of literal
                      #   → for f in *.txt; do  ... safe even with zero matches
shopt -s failglob     # non-matching globs cause an ERROR (good for catching typos)
shopt -s globstar     # enable ** for recursive matching (Bash 4+)
shopt -s dotglob      # globs also match hidden dotfiles (.bashrc etc.)
shopt -s nocaseglob   # case-insensitive globbing
shopt -s extglob      # extended glob patterns (see below)

# globstar example — recurse through subdirectories:
shopt -s globstar
for f in ./**/*.log; do      # every .log at any depth
  echo "$f"
done

# Check / reset:
shopt nullglob        # show current state
shopt -u nullglob     # unset (disable)
```

### 9.3 Extended globs (`extglob`)

```bash
shopt -s extglob      # required to enable these

# ?(pat)  zero or one         *(pat)  zero or more
# +(pat)  one or more         @(pat)  exactly one of
# !(pat)  anything EXCEPT
ls !(*.txt)                   # everything that is NOT a .txt file
ls *.@(jpg|png|gif)           # image files
ls +([0-9]).log               # log files whose name is all digits
rm !(important.txt)           # delete everything except important.txt (careful!)
```

### 9.4 Brace and tilde expansion

```bash
# Brace expansion happens FIRST, even before globbing, and needs no files to exist:
echo {a,b,c}                  # a b c
echo file{1..3}.txt           # file1.txt file2.txt file3.txt
mkdir -p project/{src,test,docs}    # create three dirs at once
cp config.yml{,.bak}          # → cp config.yml config.yml.bak (handy backup idiom!)

# Tilde expansion:
echo ~                        # your home directory ($HOME)
echo ~root                    # root's home directory
echo ~/projects               # $HOME/projects

# ⚠️ Tilde only expands when UNQUOTED and at the start of a word:
echo "~/foo"                  # → ~/foo  (literal — quoting kills tilde expansion)
echo "$HOME/foo"              # use $HOME inside quotes instead
```

---

## 10. Functions

### 10.1 Defining and calling

```bash
# Two equivalent syntaxes:
greet() {            # POSIX style — PREFER this
  echo "Hello, $1"
}

function greet2 {    # Bash `function` keyword — non-POSIX, optional ()
  echo "Hi, $1"
}

greet "Ada"          # call with arguments — NO parentheses, NO commas
greet2 "Grace"
```

### 10.2 Arguments, return values, and output

Functions receive `$1`, `$2`, `$@`, `$#` just like scripts. There are **two ways to "return"**:

```bash
# 1) EXIT STATUS via `return` (0–255 only — for SUCCESS/FAILURE, not data):
is_even() {
  (( $1 % 2 == 0 ))    # the (( )) sets the exit status; return uses it implicitly
}
if is_even 4; then echo "even"; fi

is_valid() {
  [[ -n "$1" ]] && return 0
  return 1
}

# 2) DATA via stdout (echo/printf) captured with command substitution:
get_timestamp() {
  date +%Y%m%d-%H%M%S
}
ts=$(get_timestamp)        # capture the function's stdout
echo "Backup: backup-$ts.tar.gz"

# ⚠️ Don't confuse them: `return` can only give a number 0-255. To return a
#   string or large number, ECHO it and capture with $(...).
add() { echo $(( $1 + $2 )); }
sum=$(add 3 4)             # 7
```

### 10.3 Local variables & scope

```bash
counter=0                  # global

increment() {
  local counter           # SHADOW the global; changes stay local
  counter=$(( counter + 1 ))
  echo "inside: $counter"
}
increment                  # inside: 1
echo "outside: $counter"   # outside: 0  (global untouched)

# `local` can also capture a subcommand — but BEWARE this masks exit status:
bad() {
  local result=$(might_fail)   # $? is now the exit of `local`, NOT might_fail!
}
good() {
  local result               # declare first...
  result=$(might_fail) || return 1   # ...then assign so you can check $?
}
```

### 10.4 Recursion and libraries

```bash
# Recursion works (mind the depth — no tail-call optimization):
factorial() {
  if (( $1 <= 1 )); then
    echo 1
  else
    echo $(( $1 * $(factorial $(( $1 - 1 ))) ))
  fi
}
echo "$(factorial 5)"      # 120

# Libraries: put reusable functions in a file and `source` it.
# --- file: lib/log.sh ---
log_info()  { printf '[INFO]  %s\n' "$*" >&2; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }

# --- in your script ---
source "$(dirname "$0")/lib/log.sh"   # load the library (relative to the script)
log_info "Started"

# Guard a library so it can be sourced but not run directly:
# (at the top of lib.sh)
# [[ "${BASH_SOURCE[0]}" == "${0}" ]] && { echo "This is a library, source it"; exit 1; }
```

---

## 11. Arrays

Arrays are **Bash-specific** (not POSIX `sh`). Two kinds: **indexed** (integer keys) and **associative** (string keys, Bash 4+).

### 11.1 Indexed arrays

```bash
# Create:
fruits=(apple banana cherry)         # literal
fruits[5]=date                       # sparse — indices need not be contiguous
empty=()                             # empty array
declare -a nums=(1 2 3)              # explicit declaration

# Access:
echo "${fruits[0]}"                  # apple — FIRST element is index 0
echo "${fruits[-1]}"                 # last element (Bash 4.3+)
echo "${fruits[@]}"                  # all elements as separate words (QUOTE this!)
echo "${fruits[*]}"                  # all elements as one word (joined by IFS[0])
echo "${#fruits[@]}"                 # number of elements (count)
echo "${#fruits[0]}"                 # length of the FIRST element's string
echo "${!fruits[@]}"                 # all INDICES (keys) — useful for sparse arrays

# Append & modify:
fruits+=(elderberry fig)             # append one or more
fruits[1]=blueberry                  # replace element

# Slice:
echo "${fruits[@]:1:2}"              # 2 elements starting at index 1

# Iterate — ALWAYS quote "${arr[@]}":
for f in "${fruits[@]}"; do
  echo "fruit: $f"
done

# Iterate with indices (needed for sparse arrays):
for i in "${!fruits[@]}"; do
  echo "$i -> ${fruits[$i]}"
done

# Build an array from command output safely:
mapfile -t lines < file.txt          # each line → an element (Bash 4+; -t strips \n)
readarray -t lines < file.txt        # readarray is a synonym for mapfile
IFS=$'\n' read -rd '' -a arr < <(ls) # older alternative (more fiddly)
```

### 11.2 Associative arrays (Bash 4+)

```bash
declare -A color                     # MUST declare with -A before use
color[apple]=red
color[banana]=yellow
color=([grape]=purple [lime]=green)  # literal init

echo "${color[apple]}"               # red
echo "${color[@]}"                   # all values
echo "${!color[@]}"                  # all keys
echo "${#color[@]}"                  # number of pairs

# Iterate key/value:
for key in "${!color[@]}"; do
  echo "$key is ${color[$key]}"
done

# Check if a key exists:
if [[ -v color[apple] ]]; then echo "have apple"; fi   # -v: is set? (Bash 4.2+)
[[ -n "${color[apple]+x}" ]] && echo "have apple"       # portable-to-bash3 trick

# Use as a counter / set:
declare -A seen
for word in apple banana apple cherry apple; do
  (( seen[$word]++ ))                # count occurrences
done
echo "apple appeared ${seen[apple]} times"   # 3

# ⚡ Version note: associative arrays require Bash 4+. macOS stock Bash 3.2
#   does NOT have them — another reason to install modern bash on Mac.
```

| Operation | Indexed | Associative |
|---|---|---|
| Declare | `arr=()` or `declare -a` | `declare -A arr` (required) |
| Set | `arr[0]=x` | `arr[key]=x` |
| All values | `"${arr[@]}"` | `"${arr[@]}"` |
| All keys | `"${!arr[@]}"` | `"${!arr[@]}"` |
| Count | `"${#arr[@]}"` | `"${#arr[@]}"` |

---

## 12. I/O Redirection & File Descriptors

### 12.1 The three standard streams

Every process starts with three open file descriptors (fds):

| FD | Name | Default | Purpose |
|---|---|---|---|
| 0 | stdin | keyboard | input |
| 1 | stdout | terminal | normal output |
| 2 | stderr | terminal | errors / diagnostics |

### 12.2 Basic redirection

```bash
echo "hi" > out.txt          # redirect stdout to file (TRUNCATE/overwrite)
echo "more" >> out.txt       # append stdout to file
cmd < input.txt              # redirect stdin FROM a file
cmd 2> errors.txt            # redirect stderr (fd 2) to a file
cmd 2>> errors.txt           # append stderr

cmd > /dev/null              # discard stdout (the "black hole")
cmd 2> /dev/null             # discard stderr (suppress error messages)
cmd > out.txt 2>&1           # stdout to file, AND stderr to wherever stdout goes
cmd &> out.txt               # Bash shorthand: BOTH stdout & stderr to file
cmd &>> out.txt              # Bash shorthand: append both
cmd > /dev/null 2>&1         # discard everything (silence the command)
```

### 12.3 The `2>&1` ordering subtlety

`2>&1` means "make fd 2 point to **wherever fd 1 currently points**". Order matters because redirections are applied **left to right**.

```bash
# CORRECT — send both to file:
cmd > out.txt 2>&1
#   1) stdout → out.txt
#   2) stderr → (wherever stdout now points) = out.txt   ✅

# WRONG — common mistake:
cmd 2>&1 > out.txt
#   1) stderr → (wherever stdout points NOW) = the TERMINAL
#   2) stdout → out.txt
#   Result: stderr still goes to the terminal!  ❌

# Use this knowledge to swap streams or send errors to stdout for a pipe:
cmd 2>&1 | grep something     # now grep also sees stderr
```

### 12.4 Custom file descriptors with `exec`

```bash
# Open fd 3 for writing to a log, then write to it repeatedly:
exec 3> debug.log            # open fd 3 → debug.log (truncate)
echo "step 1" >&3            # write to fd 3
echo "step 2" >&3
exec 3>&-                    # close fd 3

# Open fd 4 for reading:
exec 4< data.txt
read -r line <&4             # read one line from fd 4
read -r line2 <&4            # next line
exec 4<&-                    # close

# Save and restore stdout (e.g. to tee everything for part of a script):
exec 3>&1                    # back up current stdout into fd 3
exec 1> >(tee run.log)       # redirect stdout through tee → file + screen
echo "this is logged and shown"
exec 1>&3 3>&-               # restore original stdout, close backup

# ⚡ Version note: Bash 4.1+ can auto-allocate an fd: exec {fd}> file
exec {logfd}> auto.log
echo "auto" >&"$logfd"
exec {logfd}>&-
```

### 12.5 `tee` — write to file AND stdout

```bash
echo "hello" | tee file.txt              # prints AND writes to file
echo "more"  | tee -a file.txt           # -a appends instead of overwriting
make 2>&1 | tee build.log                # capture build output to file and screen
echo "secret" | sudo tee /etc/conf >/dev/null   # write to a root-owned file via sudo
```

### 12.6 Process substitution `<(...)` and `>(...)`

Process substitution turns a command's I/O into a **filename** (`/dev/fd/63`) that other commands can read/write. **Bash-specific.**

```bash
# Compare the output of two commands without temp files:
diff <(sort file1.txt) <(sort file2.txt)

# Feed command output to a command that wants a FILENAME, not stdin:
comm -12 <(sort a.txt) <(sort b.txt)     # lines common to both

# Avoid the subshell pitfall of pipelines (Section 7.3):
count=0
while read -r line; do (( count++ )); done < <(grep ERROR log)
echo "$count"                            # works! count persists

# >(...) — send output INTO a process:
echo "data" > >(gzip > out.gz)           # pipe stdout into gzip
some_cmd > >(tee stdout.log) 2> >(tee stderr.log >&2)   # split streams to files
```

---

## 13. Pipelines, Subshells & Command Substitution

### 13.1 Pipelines

A **pipeline** connects the stdout of one command to the stdin of the next.

```bash
cat access.log | grep 404 | wc -l        # count 404s
# (Better, avoids useless cat: grep -c 404 access.log)

ps aux | grep nginx | grep -v grep       # find nginx processes
```

By default, a pipeline's exit status is the exit status of the **last** command — so a failure earlier in the pipe is hidden:

```bash
grep "nope" missing_file.txt | sort       # grep fails, but...
echo $?                                    # 0 — sort succeeded, masking grep's failure
```

### 13.2 `pipefail` and `PIPESTATUS`

```bash
set -o pipefail                # pipeline fails if ANY command in it fails
grep "nope" missing.txt | sort
echo $?                         # now non-zero (grep's failure surfaces)

# PIPESTATUS holds the exit status of EVERY command in the last pipeline:
false | true | false
echo "${PIPESTATUS[@]}"         # 1 0 1  (status of each stage, in order)
echo "${PIPESTATUS[0]}"         # 1      (the first command's status)
```

### 13.3 Subshells `( )` vs grouping `{ }`

```bash
# ( ... ) runs in a SUBSHELL — a child process. Changes (cd, vars) don't escape.
( cd /tmp && rm -rf junk )       # cd is contained; you stay in your dir afterward
echo "$PWD"                       # unchanged

x=1
( x=99; echo "in subshell: $x" ) # in subshell: 99
echo "outside: $x"                # outside: 1   (subshell change discarded)

# { ...; } groups commands in the CURRENT shell (no child process).
#   Note: needs a space after { and a ; (or newline) before }.
{ echo "line1"; echo "line2"; } > combined.txt   # redirect a group's output
x=1
{ x=99; }
echo "$x"                         # 99 (changes persist — same shell)
```

| | `( ... )` subshell | `{ ...; }` group |
|---|---|---|
| New process? | Yes | No |
| Variable changes persist? | No | Yes |
| `cd` affects parent? | No | Yes |
| Syntax fussiness | none special | needs spaces + trailing `;` |

### 13.4 Command substitution `$(...)`

```bash
today=$(date +%F)                 # capture command output into a variable
files=$(ls *.txt)                 # (careful: word-splitting if used unquoted)
echo "There are $(ls | wc -l) entries"

# PREFER $(...) over backticks `...`:
old=`date`                        # legacy backticks — hard to nest, escape-prone
new=$(date)                       # modern — nestable, readable
nested=$(echo "outer $(echo inner)")   # nesting just works with $()

# ⚠️ Command substitution STRIPS trailing newlines. Usually convenient, but
#   note it when exact bytes matter.

# ⚠️ It runs in a SUBSHELL — variable assignments inside are lost (same as 13.3).
```

---

## 14. Robust Scripting

This section is the difference between a script that works on your machine today and one that's safe in production.

### 14.1 The "unofficial strict mode": `set -euo pipefail`

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

| Option | Long form | Effect |
|---|---|---|
| `-e` | `set -o errexit` | Exit immediately if any command fails (non-zero). |
| `-u` | `set -o nounset` | Error on use of an **unset** variable (catches typos). |
| `-o pipefail` | — | A pipeline fails if **any** stage fails (not just the last). |
| (also) | `set -x` | Print each command before running (debug; Section 17). |

**Setting `IFS=$'\n\t'`** removes the space from the field separator, so word-splitting only happens on newlines and tabs — taming a lot of accidental space-splitting.

### 14.2 Caveats of strict mode (it's not magic)

```bash
# --- `set -u` and unset vars ---
echo "${MAYBE_UNSET:-default}"    # safe: provide a default
echo "${1:-}"                     # safe even with no $1
echo "${arr[@]:-}"               # safe for possibly-empty arrays

# --- `set -e` does NOT trigger in many contexts (KNOW THESE) ---
# It is ignored for commands in: if/while/until conditions, && / || chains,
# negated commands (!), and (mostly) command substitution.
if grep foo file; then :; fi      # grep failing here does NOT exit the script
some_cmd || true                  # explicitly ignore a failure
result=$(might_fail) || result="" # capture & guard

# --- (( )) returns 1 when result is 0 → can kill you under -e ---
count=0
(( count++ )) || true             # i++ returns the OLD value 0 → "fails"

# --- functions: set -e behaviour inside functions is subtle; prefer explicit
#     error handling for anything important.
```

> **The strict-mode debate:** `set -e` has many surprising exceptions and is criticized by some experts (it can give false confidence). It's still a good default for small/medium scripts. For critical code, handle errors **explicitly** with `||`, `if`, and traps rather than relying solely on `-e`.

### 14.3 `trap` — cleanup and signal handling

`trap` runs code when the script receives a signal or hits certain conditions. The most valuable use is **guaranteed cleanup**.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Create a temp dir and GUARANTEE it's removed, no matter how we exit.
workdir="$(mktemp -d)"
cleanup() {
  rm -rf "$workdir"               # remove temp files
  echo "Cleaned up $workdir" >&2
}
trap cleanup EXIT                 # runs on ANY exit: normal, error, or signal

# ... do work in "$workdir" ...
echo "Working in $workdir"

# Multiple/different handlers:
trap 'echo "Interrupted!"; exit 130' INT     # Ctrl-C (SIGINT)
trap 'echo "Error on line $LINENO" >&2' ERR  # any command failure (with set -e)
trap 'echo "Terminated"; exit 143' TERM      # SIGTERM (kill)
```

| Signal/pseudo | When |
|---|---|
| `EXIT` | The shell exits (for any reason) — best for cleanup. |
| `ERR` | A command returns non-zero (works with `set -e`). |
| `INT` | SIGINT — user pressed Ctrl-C. |
| `TERM` | SIGTERM — `kill <pid>`. |
| `HUP` | SIGHUP — terminal closed. |
| `DEBUG` | Before *every* command (advanced tracing). |

```bash
# Reset a trap to default:
trap - INT
# Ignore a signal entirely:
trap '' INT          # Ctrl-C now does nothing
```

### 14.4 ShellCheck — your static analysis safety net

[ShellCheck](https://www.shellcheck.net) is a free linter that catches the overwhelming majority of shell bugs **before** you run anything. **Install and run it on every script.**

```bash
shellcheck myscript.sh           # lint a script
shellcheck -x myscript.sh        # follow `source`d files too
shellcheck -s bash myscript.sh   # force bash dialect

# Suppress a specific check on the next line (with justification):
# shellcheck disable=SC2086  # word splitting is intentional here
echo $intentionally_unquoted
```

**Common ShellCheck codes you'll see (memorize the top ones):**

| Code | Meaning / fix |
|---|---|
| **SC2086** | Unquoted variable → word splitting/globbing. Quote it: `"$var"`. |
| **SC2046** | Unquoted `$(...)` → splitting. Quote or restructure. |
| **SC2006** | Use `$(...)` instead of legacy backticks. |
| **SC2164** | `cd` may fail; use `cd ... || exit`. |
| **SC2155** | `local x=$(cmd)` masks the exit status. Split the declaration. |
| **SC2034** | Variable assigned but never used (typo?). |
| **SC2128** | Referencing an array without an index gives only the first element. |
| **SC2207** | `arr=( $(cmd) )` splits badly; use `mapfile`/`readarray`. |
| **SC2115** | `rm -rf "$x/"` where `$x` could be empty → could nuke `/`. Guard it. |
| **SC1090/91** | Can't follow a dynamic `source`; use `-x` or a directive. |

### 14.5 Defensive habits checklist

```bash
# - Quote EVERY expansion: "$var", "$@", "${arr[@]}", "$(cmd)".
# - Use [[ ]] over [ ] in bash.
# - cd safely:           cd "$dir" || exit 1
# - Use mktemp for temp files, trap EXIT to clean up.
# - Validate inputs early:  : "${1:?need an argument}"
# - Prefer printf over echo.
# - Use long options in scripts (--verbose) for readability.
# - Set a clean PATH if running as root/cron:  PATH=/usr/bin:/bin
# - Fail fast and loud: errors to stderr, meaningful exit codes.
```

---

## 15. Text Processing Toolkit

Bash glues together small, sharp Unix tools. Mastering these is what makes the shell powerful.

### 15.1 `grep` — search lines (regex)

```bash
grep "pattern" file.txt          # print matching lines
grep -i "error" log              # -i case-insensitive
grep -n "TODO" *.py              # -n show line numbers
grep -r "func" src/             # -r recursive
grep -c "404" access.log         # -c count matches
grep -v "DEBUG" log              # -v invert: lines NOT matching
grep -l "main(" *.c              # -l just list FILES with a match
grep -o "[0-9]\+" file           # -o print only the matched part
grep -w "cat" file               # -w whole-word match (not "concatenate")
grep -A2 -B2 "panic" log         # -A/-B/-C context lines after/before/both
grep -E "foo|bar" file           # -E extended regex (ERE): |, +, ?, () unescaped
grep -F "literal.string" file    # -F fixed string (no regex, faster)
grep -q "x" file && echo found   # -q quiet — for if-conditions

# BRE vs ERE: in basic regex you must escape + ? { } ( ) | to use them as
# operators; in extended (-E) they work bare. Use -E for sanity.
grep    "ab\+"  f                # BRE: one or more b
grep -E "ab+"   f                # ERE: same thing, cleaner
```

### 15.2 `sed` — stream editor

```bash
sed 's/old/new/' file            # replace FIRST 'old' on each line
sed 's/old/new/g' file           # g = global, replace ALL on each line
sed 's/old/new/2' file           # replace only the 2nd occurrence per line
sed 's/old/new/gi' file          # g + i (case-insensitive)
sed 's#/path#/new#g' file        # use # as delimiter when pattern has /

sed -n '5p' file                 # -n quiet; p print → print only line 5
sed -n '10,20p' file             # print lines 10–20 (an address RANGE)
sed '2d' file                    # delete line 2
sed '/^#/d' file                 # delete comment lines (starting with #)
sed '/^$/d' file                 # delete blank lines
sed -n '/start/,/end/p' file     # print between two patterns

sed -E 's/([0-9]+)/<\1>/g' f     # -E ERE; \1 = first capture group
sed 's/^/    /' file             # indent every line by 4 spaces
sed '$a\appended last line' f    # append after the last line ($)

# In-place editing (modifies the file):
sed -i 's/foo/bar/g' file        # GNU sed (Linux)
sed -i.bak 's/foo/bar/g' file    # also keep a .bak backup
# ⚠️ macOS/BSD sed requires an argument: sed -i '' 's/.../.../' file
#    This single difference breaks countless scripts across Linux/macOS.
```

### 15.3 `awk` — a mini tutorial

`awk` is a whole language for column/record processing. An awk program is a series of `pattern { action }` rules. For each input line, awk runs every rule whose pattern matches.

```bash
# Fields: $1, $2, ... are columns; $0 is the whole line; NF = number of fields.
awk '{ print $1 }' file          # print the first column (default split on whitespace)
awk '{ print $NF }' file         # print the LAST column
awk '{ print $1, $3 }' file      # comma → output field separator (a space)

# Patterns select lines:
awk '/error/ { print }' log      # lines matching /error/ (like grep)
awk '$3 > 100' data              # lines where column 3 > 100 (default action: print)
awk 'NR==1' file                 # NR = record (line) number → print line 1
awk 'NR % 2 == 0' file           # every even line
awk 'NF == 0' file               # blank lines (no fields)

# Field separators:
awk -F: '{ print $1 }' /etc/passwd      # -F sets INPUT separator to :
awk -F',' '{ print $2 }' data.csv       # CSV-ish (no quoting support!)
awk 'BEGIN{FS=":"; OFS="\t"} {print $1,$3}' /etc/passwd   # FS in/OFS out

# BEGIN runs before input; END runs after — perfect for totals:
awk '{ sum += $1 } END { print "total:", sum }' numbers.txt
awk '{ count++ } END { print "lines:", count }' file       # like wc -l
awk '{ sum += $2 } END { printf "avg: %.2f\n", sum/NR }' data

# Variables, conditionals, arithmetic:
awk '{ if ($3 >= 90) print $1, "PASS"; else print $1, "FAIL" }' grades

# Associative arrays in awk — count occurrences (a classic):
awk '{ count[$1]++ } END { for (k in count) print k, count[k] }' access.log

# Pass a shell variable IN with -v:
threshold=50
awk -v t="$threshold" '$2 > t { print }' data

# Built-in functions:
awk '{ print toupper($1), length($0) }' file
awk '{ print substr($0, 1, 10) }' file     # first 10 chars
awk '{ gsub(/foo/, "bar"); print }' file   # global substitute, like sed
```

| awk variable | Meaning |
|---|---|
| `$0` | entire current line |
| `$1..$N` | fields (columns) |
| `NF` | number of fields on this line |
| `NR` | current record (line) number, overall |
| `FNR` | record number within the current file |
| `FS` / `OFS` | input / output field separator |
| `RS` / `ORS` | input / output record separator |
| `FILENAME` | name of current input file |

### 15.4 `cut`, `tr`, `sort`, `uniq`, `wc`, `head`, `tail`

```bash
# cut — extract columns/characters:
cut -d: -f1 /etc/passwd          # -d delimiter, -f field(s); usernames
cut -d, -f2,4 data.csv           # multiple fields
cut -c1-10 file                  # characters 1–10 of each line

# tr — translate/delete characters (NOT strings):
echo "HELLO" | tr 'A-Z' 'a-z'    # hello (lowercase)
echo "a-b-c" | tr '-' ' '        # a b c
echo "hello" | tr -d 'l'         # heo (delete all l)
tr -s ' ' < file                 # -s squeeze repeated spaces into one
tr -cd '[:alnum:]\n' < file      # -c complement: keep only alnum + newline

# sort — order lines:
sort file                        # lexicographic
sort -n nums                     # -n numeric
sort -rn nums                    # -r reverse (descending)
sort -k2 -t, data.csv            # -k sort key (column 2), -t field delimiter
sort -u file                     # -u unique (sort + dedupe)
sort -h sizes                    # -h human-numeric (2K, 1G) — GNU

# uniq — collapse ADJACENT duplicates (always sort first!):
sort file | uniq                 # unique lines
sort file | uniq -c              # -c prefix counts
sort file | uniq -c | sort -rn   # the classic "top N" idiom (frequency)
sort file | uniq -d              # -d show only duplicated lines

# wc — count:
wc -l file                       # lines
wc -w file                       # words
wc -c file                       # bytes  (-m for characters)

# head / tail:
head -n 5 file                   # first 5 lines
tail -n 5 file                   # last 5 lines
tail -n +10 file                 # from line 10 to end
tail -f app.log                  # FOLLOW: stream new lines as they're written
tail -F app.log                  # like -f but survives log rotation
```

### 15.5 `find` in depth + `xargs` (null-safe)

```bash
find . -name "*.txt"             # by name (glob; quote it!)
find . -iname "*.JPG"            # -iname case-insensitive
find . -type f                   # files only  (-type d = dirs, l = symlinks)
find . -type f -size +10M        # larger than 10 MB
find . -mtime -7                 # modified in the last 7 days
find . -mmin -60                 # modified in the last 60 minutes
find . -name "*.log" -mtime +30  # .log files older than 30 days
find . -empty                    # empty files/dirs
find . -maxdepth 2 -type d       # limit recursion depth
find . -path "*/node_modules" -prune -o -name "*.js" -print   # skip a dir

# Run a command per result:
find . -name "*.tmp" -delete                 # built-in delete
find . -name "*.txt" -exec wc -l {} \;       # {} = the file; \; = run once each
find . -name "*.txt" -exec wc -l {} +        # + = batch many files per call (faster)
find . -type f -exec chmod 644 {} +

# THE null-safe pattern for filenames with spaces/newlines:
find . -name "*.txt" -print0 | xargs -0 wc -l
#   -print0 ends each name with a NUL byte; xargs -0 splits on NUL → space-safe.

# xargs basics:
echo "a b c" | xargs -n1 echo   # -n1: one arg per command invocation
ls *.jpg | xargs -I{} cp {} backup/   # -I{} : substitute placeholder (avoid; prefer -0)
cat urls.txt | xargs -P4 -n1 curl -O   # -P4: run 4 in parallel (Section 16)

# ⚠️ NEVER parse `ls` in scripts (breaks on spaces/newlines/glob chars).
#    Use find -print0 | xargs -0, or a glob with `for f in ./*; do`.
```

---

## 16. Job Control, Processes & Parallelism

### 16.1 Background jobs and job control

```bash
sleep 60 &                       # & runs the command in the BACKGROUND
echo "started PID $!"            # $! = PID of the most recent background job

jobs                             # list this shell's jobs
jobs -l                          # with PIDs
fg %1                            # bring job 1 to the FOREGROUND
bg %1                            # resume a stopped job in the BACKGROUND
# (Press Ctrl-Z to suspend the foreground job, then `bg` to continue it.)

wait                             # wait for ALL background jobs to finish
wait %1                          # wait for a specific job
wait "$pid"                      # wait for a specific PID
wait -n                          # wait for the NEXT job to finish (Bash 4.3+)
```

### 16.2 Signals and `kill`

```bash
kill "$pid"                      # send SIGTERM (15) — polite "please stop"
kill -TERM "$pid"                # same, explicit
kill -INT "$pid"                 # SIGINT (2) — like Ctrl-C
kill -KILL "$pid"                # SIGKILL (9) — forceful, cannot be trapped (last resort)
kill -HUP "$pid"                 # SIGHUP (1) — often "reload config"
kill -0 "$pid"                   # signal 0: don't kill, just TEST if process exists
kill -l                          # list all signal names/numbers
pkill -f "myscript"             # kill by command-line pattern
killall nginx                    # kill by process name
```

| Signal | Num | Default | Trappable? |
|---|---|---|---|
| SIGHUP | 1 | terminate | yes |
| SIGINT | 2 | terminate (Ctrl-C) | yes |
| SIGKILL | 9 | terminate | **no** |
| SIGTERM | 15 | terminate | yes |
| SIGSTOP | 19 | stop | **no** |
| SIGCONT | 18 | continue | yes |

### 16.3 `nohup` / `disown` — surviving logout

```bash
nohup long_task.sh &             # ignore SIGHUP; survives terminal close; logs to nohup.out
nohup long_task.sh > out.log 2>&1 &   # capture output where you want

long_task.sh &
disown                           # remove the most recent job from the shell's job table
disown -h %1                     # keep the job but shield it from SIGHUP on exit
```

### 16.4 Running things in parallel

```bash
# --- Method 1: backgrounding + wait (simplest) ---
for url in "${urls[@]}"; do
  curl -sO "$url" &              # launch each download in the background
done
wait                             # block until all finish
echo "All downloads done"

# Capture individual exit statuses:
pids=()
for host in "${hosts[@]}"; do
  ping -c1 "$host" &>/dev/null & pids+=("$!")
done
fail=0
for pid in "${pids[@]}"; do
  wait "$pid" || (( fail++ ))    # collect failures
done
echo "$fail hosts unreachable"

# --- Method 2: xargs -P (controlled concurrency) ---
printf '%s\n' "${urls[@]}" | xargs -P4 -n1 curl -sO   # 4 at a time

# --- Method 3: GNU parallel (if installed — more features) ---
parallel -j4 curl -sO ::: "${urls[@]}"    # -j4 = 4 jobs; ::: provides the args
# (GNU parallel isn't always installed; xargs -P covers most needs.)
```

### 16.5 Timeouts

```bash
timeout 10 long_command          # kill long_command after 10 seconds
timeout 10s slow.sh              # units: s,m,h,d
timeout -k 5 10 cmd              # send TERM at 10s; if still alive, KILL 5s later
timeout 10 cmd; echo $?          # exit 124 means it TIMED OUT

# Manual timeout (no `timeout` available):
( sleep 30 & wait ) & pid=$!     # ...pattern; usually just use `timeout`.
```

---

## 17. Debugging

```bash
# --- Print each command as it executes (the #1 debugging tool) ---
bash -x script.sh                # trace the whole run
set -x                           # turn tracing ON mid-script
# ... suspicious code ...
set +x                           # turn tracing OFF

# --- Print lines as they're READ (before expansion) ---
bash -v script.sh
set -v ; ... ; set +v

# --- Syntax-check WITHOUT running (catch typos/unbalanced quotes) ---
bash -n script.sh                # "no exec" — parse only

# --- Customize the trace prefix with PS4 (line numbers + function names) ---
export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}() '
set -x
# Output looks like: + script.sh:42:do_thing() echo hi

# --- Send xtrace to a separate fd so it doesn't pollute output (Bash 4.1+) ---
exec 5> trace.log                # open fd 5
export BASH_XTRACEFD=5           # xtrace writes to fd 5, not stderr
set -x

# --- Trap errors with context ---
trap 'echo "ERR at line $LINENO: exit $?" >&2' ERR
set -euo pipefail

# --- A reusable logging pattern ---
log() { printf '[%s] %s\n' "$(date +%T)" "$*" >&2; }   # timestamped, to stderr
LOG_LEVEL=${LOG_LEVEL:-info}
debug() { [[ "$LOG_LEVEL" == debug ]] && log "DEBUG: $*"; return 0; }
log "starting"
debug "x = $x"                   # only prints when LOG_LEVEL=debug
```

| Tool | Use |
|---|---|
| `set -x` / `bash -x` | Trace executed commands (with expansions). |
| `set -v` / `bash -v` | Echo input lines as read. |
| `bash -n` | Syntax check, no execution. |
| `PS4` | Customize the `+` xtrace prefix (add line/func info). |
| `BASH_XTRACEFD` | Redirect trace output to a dedicated fd/file. |
| `trap ... ERR` | Report where a failure happened. |
| `shellcheck` | Catch bugs statically before running. |

---

## 18. Argument Parsing

### 18.1 Positional args and `shift`

```bash
#!/usr/bin/env bash
# Manual positional handling
input="$1"                       # first arg
output="$2"                      # second arg
[[ -n "$input" ]] || { echo "Usage: $0 <in> <out>"; exit 1; }

# shift: drop $1, renumber the rest. Useful to consume args one by one.
while [[ $# -gt 0 ]]; do
  echo "processing: $1"
  shift                          # now $2 becomes $1, etc.; $# decreases
done
```

### 18.2 `getopts` — short options (POSIX, built-in)

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF
Usage: $0 [-v] [-o OUTPUT] [-n COUNT] FILE...
  -v          verbose mode
  -o OUTPUT   output file (requires an argument — note the colon)
  -n COUNT    number of items
  -h          show this help
EOF
}

verbose=0
output=""
count=1

# The option string: letters = flags; a letter+":" = needs an argument.
# A leading ":" enables SILENT error handling (we report errors ourselves).
while getopts ":vo:n:h" opt; do
  case "$opt" in
    v) verbose=1 ;;
    o) output="$OPTARG" ;;       # OPTARG holds the option's argument
    n) count="$OPTARG" ;;
    h) usage; exit 0 ;;
    :) echo "Error: -$OPTARG requires an argument" >&2; exit 1 ;;  # missing arg
    \?) echo "Error: unknown option -$OPTARG" >&2; usage; exit 1 ;; # unknown flag
  esac
done

shift $(( OPTIND - 1 ))          # discard the parsed options; leaves FILE... in $@

echo "verbose=$verbose output=$output count=$count"
echo "remaining args: $*"

# Run: ./tool.sh -v -o out.txt -n 3 a.txt b.txt
#   → verbose=1 output=out.txt count=3 / remaining args: a.txt b.txt
```

**`getopts` limits:** only **short** options (`-v`, not `--verbose`), and no combined-arg parsing beyond the standard. For long options, hand-roll it:

### 18.3 Hand-rolled long-option parser

```bash
#!/usr/bin/env bash
set -euo pipefail

verbose=0
output=""
positional=()                    # collect non-option args here

while [[ $# -gt 0 ]]; do
  case "$1" in
    -v|--verbose)
      verbose=1
      shift
      ;;
    -o|--output)                 # option that takes a separate argument
      [[ $# -ge 2 ]] || { echo "--output needs a value" >&2; exit 1; }
      output="$2"
      shift 2                    # consume BOTH the flag and its value
      ;;
    --output=*)                  # support --output=FILE form too
      output="${1#*=}"           # strip everything up to and including =
      shift
      ;;
    -h|--help)
      echo "Usage: $0 [--verbose] [--output FILE] FILE..."
      exit 0
      ;;
    --)                          # end-of-options marker
      shift
      positional+=("$@")         # everything after -- is positional
      break
      ;;
    -*)
      echo "Unknown option: $1" >&2
      exit 1
      ;;
    *)                           # a non-option (positional) argument
      positional+=("$1")
      shift
      ;;
  esac
done

set -- "${positional[@]}"        # restore positionals as $1, $2, ...
echo "verbose=$verbose output=$output files=$*"
```

---

## 19. Regular Expressions in Bash

The `=~` operator inside `[[ ]]` matches a string against an **ERE** (extended regular expression) and captures groups into the `BASH_REMATCH` array. **Bash-specific.**

```bash
s="version 12.4.1 released"

if [[ "$s" =~ ([0-9]+)\.([0-9]+)\.([0-9]+) ]]; then
  echo "full match: ${BASH_REMATCH[0]}"   # 12.4.1  (entire match)
  echo "major:      ${BASH_REMATCH[1]}"   # 12      (first capture group)
  echo "minor:      ${BASH_REMATCH[2]}"   # 4
  echo "patch:      ${BASH_REMATCH[3]}"   # 1
fi

# ⚠️ CRITICAL: do NOT quote the regex on the right side — quoting makes it a
#   LITERAL string, not a pattern.
[[ "$s" =~ [0-9]+ ]]      # regex (matches digits)
[[ "$s" =~ "[0-9]+" ]]    # literal — looks for the actual text [0-9]+

# To match special characters literally, escape them or store the pattern in a var:
re='^[0-9]{4}-[0-9]{2}-[0-9]{2}$'    # an ISO date
date_str="2026-06-21"
[[ "$date_str" =~ $re ]] && echo "valid date"   # unquoted $re = pattern

# Validation examples:
is_int()   { [[ "$1" =~ ^-?[0-9]+$ ]]; }
is_email() { [[ "$1" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; }
is_int 42 && echo "integer"
is_email "a@b.com" && echo "email-ish"

# ⚠️ Locale affects character classes & ranges. For predictable [a-z], you may
#   need LC_ALL=C. And { } quantifiers need ERE support (present in modern bash).
```

---

## 20. Practical Patterns & Recipes

### 20.1 Temp files with `mktemp` (+ guaranteed cleanup)

```bash
tmpfile="$(mktemp)"                    # secure unique temp FILE in $TMPDIR
tmpdir="$(mktemp -d)"                  # secure unique temp DIRECTORY
tmpl="$(mktemp /tmp/myapp.XXXXXX)"     # custom template (X's become random)

trap 'rm -rf "$tmpfile" "$tmpdir"' EXIT   # ALWAYS clean up on exit
# ...use them...
echo "data" > "$tmpfile"
```

### 20.2 Locking with `flock` (prevent concurrent runs)

```bash
#!/usr/bin/env bash
# Ensure only ONE instance of this script runs at a time.
exec 9>/var/lock/myscript.lock         # open fd 9 on a lock file
if ! flock -n 9; then                  # -n = non-blocking; fail if already locked
  echo "Another instance is running." >&2
  exit 1
fi
# The lock is auto-released when fd 9 closes (i.e., when the script exits).
echo "Got the lock; doing work..."
sleep 5
```

### 20.3 Reading a simple config file

```bash
# A KEY=VALUE config, safely (no eval/source of untrusted files!):
declare -A config
while IFS='=' read -r key value; do
  [[ "$key" =~ ^[[:space:]]*# ]] && continue   # skip comments
  [[ -z "$key" ]] && continue                  # skip blank lines
  key="${key// /}"                             # trim spaces from key
  config[$key]="$value"
done < app.conf
echo "port = ${config[port]:-8080}"

# (Sourcing a config — config.sh with var=val lines — is convenient but only
#  safe if you fully trust the file, since it executes arbitrary code.)
```

### 20.4 Retry with exponential backoff

```bash
retry() {
  local max="$1"; shift          # first arg = max attempts; rest = the command
  local attempt=1 delay=1
  until "$@"; do                 # run the command; loop while it fails
    if (( attempt >= max )); then
      echo "Failed after $attempt attempts: $*" >&2
      return 1
    fi
    echo "Attempt $attempt failed; retrying in ${delay}s..." >&2
    sleep "$delay"
    (( attempt++, delay *= 2 ))  # exponential backoff: 1,2,4,8...
  done
}
retry 5 curl -fsS https://example.com/health   # try up to 5 times
```

### 20.5 Confirmation prompts

```bash
confirm() {
  local prompt="${1:-Are you sure?} [y/N] "
  local reply
  read -rp "$prompt" reply
  [[ "$reply" =~ ^[Yy]$ ]]       # exit status: 0 if yes
}
if confirm "Delete all logs?"; then
  rm -f ./*.log
else
  echo "Aborted."
fi
```

### 20.6 Colored output (with TTY detection)

```bash
# Only emit color codes when stdout is a terminal (not a pipe/file).
if [[ -t 1 ]]; then              # -t 1 : is fd 1 a terminal?
  RED=$'\033[31m'; GREEN=$'\033[32m'; YELLOW=$'\033[33m'; BOLD=$'\033[1m'
  RESET=$'\033[0m'
else
  RED=''; GREEN=''; YELLOW=''; BOLD=''; RESET=''
fi
printf '%sOK%s done\n'    "$GREEN" "$RESET"
printf '%sWARN%s careful\n' "$YELLOW" "$RESET"
printf '%s%sERROR%s failed\n' "$BOLD" "$RED" "$RESET"
# Respect the NO_COLOR convention: if [[ -n "${NO_COLOR:-}" ]]; then disable.
```

### 20.7 Spinner / progress

```bash
spinner() {
  local pid="$1" chars='|/-\' i=0
  while kill -0 "$pid" 2>/dev/null; do        # while the process is alive
    printf '\r[%c] working...' "${chars:i++%4:1}"
    sleep 0.1
  done
  printf '\rDone.        \n'
}
long_task & spinner "$!"        # run task in background, spin until it finishes

# Simple percent progress bar:
progress() {
  local current="$1" total="$2" width=40
  local pct=$(( current * 100 / total ))
  local filled=$(( current * width / total ))
  printf '\r[%-*s] %d%%' "$width" "$(printf '#%.0s' $(seq 1 "$filled"))" "$pct"
}
for i in $(seq 1 100); do progress "$i" 100; sleep 0.02; done; echo
```

### 20.8 Dates with `date`

```bash
date +%Y-%m-%d                   # 2026-06-21 (ISO date)
date +%F                         # 2026-06-21 (shortcut for %Y-%m-%d)
date +%T                         # 14:30:05 (time)
date +%s                         # Unix epoch seconds (great for timing/IDs)
date -u +%FT%TZ                  # UTC ISO 8601: 2026-06-21T14:30:05Z

# Timing a block:
start=$(date +%s)
sleep 2
echo "Took $(( $(date +%s) - start ))s"

# Date math (GNU date; macOS/BSD uses different -v flags!):
date -d "yesterday" +%F          # GNU
date -d "+3 days" +%F            # GNU
date -v-1d +%F                   # BSD/macOS equivalent of "yesterday"
# ⚠️ This GNU vs BSD difference is a top cross-platform pitfall.
```

### 20.9 Checking command existence

```bash
# command -v is the POSIX, reliable way (NOT `which`, which varies/may be absent):
if command -v git >/dev/null 2>&1; then
  echo "git is installed"
fi

require() {                      # bail out early if a dependency is missing
  command -v "$1" >/dev/null 2>&1 || { echo "Missing required: $1" >&2; exit 1; }
}
require jq
require curl
```

### 20.10 Idempotency and dry-run flags

```bash
# Idempotent: running it twice has the same effect as running it once.
mkdir -p "$dir"                  # -p: no error if it already exists
ln -sfn "$target" "$link"        # -f force, -n treat existing link as a file
grep -qxF "$line" "$file" || echo "$line" >> "$file"   # add line only if absent

# Dry-run pattern:
DRY_RUN="${DRY_RUN:-0}"
run() {
  if [[ "$DRY_RUN" == 1 ]]; then
    printf 'DRY-RUN: %s\n' "$*"  # just show what WOULD happen
  else
    "$@"                         # actually run it
  fi
}
run rm -rf "$workdir"            # DRY_RUN=1 ./script.sh  → prints, doesn't delete
```

### 20.11 A robust script skeleton (copy this to start)

```bash
#!/usr/bin/env bash
#
# myscript.sh - one-line description
#
set -euo pipefail
IFS=$'\n\t'

# --- constants ---
readonly SCRIPT_NAME="${0##*/}"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# --- logging ---
log()  { printf '[%s] %s\n' "$(date +%T)" "$*" >&2; }
die()  { log "ERROR: $*"; exit 1; }

# --- cleanup ---
TMPDIR_RUN="$(mktemp -d)"
cleanup() { rm -rf "$TMPDIR_RUN"; }
trap cleanup EXIT

usage() {
  cat <<EOF
Usage: $SCRIPT_NAME [options] <arg>
  -h, --help     show this help
  -v, --verbose  verbose output
EOF
}

main() {
  local verbose=0
  while [[ $# -gt 0 ]]; do
    case "$1" in
      -h|--help)    usage; exit 0 ;;
      -v|--verbose) verbose=1; shift ;;
      -*)           die "unknown option: $1" ;;
      *)            break ;;
    esac
  done

  [[ $# -ge 1 ]] || { usage; die "missing argument"; }
  log "starting with arg: $1 (verbose=$verbose)"
  # ... real work here ...
}

main "$@"
```

---

## 21. Portability & Gotchas

### 21.1 Bashisms vs POSIX

If your shebang is `#!/bin/sh`, you're (often) running `dash` — these **Bash-only** features will fail:

| Bashism | POSIX-portable alternative |
|---|---|
| `[[ ... ]]` | `[ ... ]` (and quote everything) |
| `(( ... ))` / `$(( ))` | `$(( ))` is POSIX; `(( ))` command is not — use `[ "$x" -lt "$y" ]` |
| arrays | none (use multiple vars, or `set --`, or a different language) |
| `${var^^}` / `${var,,}` | `tr '[:lower:]' '[:upper:]'` |
| `${var/old/new}` | `sed`/`printf`-based substitution |
| `<<<` here-string | `printf '%s\n' "$x" \| cmd` |
| `<(...)` process subst | temp files |
| `local` | works in dash, but not strictly POSIX |
| `echo -e` / `-n` | `printf` |
| `source` | `.` (dot) |
| `&>` redirect | `> file 2>&1` |

Check for accidental bashisms in a "sh" script with `checkbashisms` (from the `devscripts` package) or `shellcheck -s sh script`.

### 21.2 The unquoted-variable family of bugs

```bash
f="a b.txt"
cp $f dest/        # BUG: cp "a" "b.txt" "dest/" — wrong number of args
cp "$f" dest/      # FIX

for f in $(ls); do ...; done   # BUG: splits names, expands globs in names
for f in *; do ...; done       # FIX: glob directly (and check [[ -e "$f" ]])

arr=$(get_items)   # BUG: this is a STRING, not an array
mapfile -t arr < <(get_items)  # FIX: real array, one item per line
```

### 21.3 `[ ]` pitfalls

```bash
[ $x = $y ]          # BUG: if $x is empty → `[ = $y ]` → syntax error
[ "$x" = "$y" ]      # FIX: quote both sides
[ "$x" == "$y" ]     # == is a bashism in [ ]; POSIX [ ] uses single =
[ -z $x ]            # BUG: empty $x → `[ -z ]` which is TRUE for wrong reason
[ -z "$x" ]          # FIX
[ "$a" -eq "$b" ]    # numeric; errors if either is non-numeric — validate first
```

### 21.4 The `cd` / subshell gotcha

```bash
cd /some/dir          # BUG: if this fails, the script keeps going in the WRONG dir
rm -rf ./*            # ...and now you're deleting from the wrong place!

cd /some/dir || exit 1     # FIX: bail if cd fails
( cd /some/dir && do_stuff )   # or contain cd in a subshell so it can't affect you
pushd /some/dir >/dev/null && do_stuff; popd >/dev/null   # save/restore dir
```

### 21.5 Locale issues

```bash
# Sorting, case conversion, and ranges depend on $LC_* / $LANG.
sort file                      # locale collation may surprise you (e.g. case order)
LC_ALL=C sort file             # force byte order — reproducible, faster
LC_ALL=C grep '[A-Z]' file     # avoid locale-dependent range weirdness
# Set LC_ALL=C in scripts that must behave identically everywhere.
```

### 21.6 CRLF line endings (especially on Windows!)

> **You are on Windows** — this one bites constantly. If you edit scripts with a Windows editor (or git checks them out with CRLF), each line ends in `\r\n`. The shebang becomes `#!/usr/bin/env bash\r`, and you get cryptic errors:

```text
/usr/bin/env: 'bash\r': No such file or directory
bad interpreter: No such file or directory
$'\r': command not found
```

**Fixes:**

```bash
# Convert CRLF → LF:
dos2unix script.sh                         # if installed
sed -i 's/\r$//' script.sh                 # with sed
tr -d '\r' < script.sh > fixed.sh          # with tr
perl -i -pe 's/\r\n/\n/g' script.sh        # with perl

# Prevent it in git — add a .gitattributes:
#   *.sh text eol=lf
# And configure your editor (VS Code: "files.eol": "\n") to use LF for .sh files.

# Detect:
file script.sh                             # may say "with CRLF line terminators"
cat -A script.sh | head                    # ^M$ at line ends = CR present
```

### 21.7 Other common traps

```bash
# Useless use of cat:
cat file | grep x        # → grep x file   (or grep x < file)

# Parsing the output of `ls`: don't (see 21.2). Use globs or find -print0.

# `read` losing the last line if the file has no trailing newline:
while IFS= read -r l || [[ -n "$l" ]]; do echo "$l"; done < file

# Forgetting that command substitution strips trailing newlines.

# `$RANDOM` is NOT cryptographically secure — for security use /dev/urandom:
head -c16 /dev/urandom | base64

# Comparing floats with [[ ]] (only does integers/strings) — use bc/awk.

# `set -e` plus a function that legitimately returns non-zero in a test context.

# Trailing whitespace after \ line-continuation breaks the continuation silently.
```

---

## 22. Study Path & Build-to-Learn Projects

### Phase 1 — Foundations (2–3 days)

1. **Shells & shebang** — Write and run your first scripts three ways (`./x.sh`, `bash x.sh`, `source x.sh`). Understand login vs non-login and edit your `~/.bashrc`.
2. **Quoting** — Internalize single vs double vs `$'...'`. Reproduce the word-splitting and globbing bugs on purpose, then fix them.
3. **Variables & special params** — Master `"$@"` vs `"$*"`, `$?`, `$#`, default expansions.
4. **printf & read** — Stop using `echo`; build a small interactive prompt with `read`.

**Mini-project: a "greeter" script** that takes a name flag, validates it, and prints a colored, formatted greeting with the current date.

### Phase 2 — Control flow & data (3–4 days)

5. **Conditionals & case** — Rewrite an if/elif chain as a `case`. Learn `[[ ]]` file tests.
6. **Loops + safe file reading** — Master `while IFS= read -r`. Avoid the pipe-subshell trap.
7. **Parameter expansion** — Do filename manipulation (basename/extension) with `${}` only — no `sed`/`basename`.
8. **Arrays** — Build an indexed list and an associative counter.

**Mini-project: a log-analysis tool** — read an access log line by line, count status codes into an associative array, and print a sorted frequency report (compare with the `awk`/`sort`/`uniq` one-liner version).

### Phase 3 — Robustness & tooling (3–5 days)

9. **set -euo pipefail + trap** — Add strict mode and EXIT cleanup to every script.
10. **ShellCheck** — Run it on everything you've written; fix every warning; learn the top SC codes.
11. **Text toolkit** — Solve five real tasks with grep/sed/awk/find/xargs. Write the `find -print0 | xargs -0` pattern from memory.
12. **Functions & libraries** — Extract a `log.sh` library and `source` it.

**Mini-project: a backup script** — tar+gzip a directory to a timestamped archive, use `mktemp`, clean up with `trap`, support a `--dry-run` flag, lock with `flock`, and log to a file.

### Phase 4 — Advanced & real-world (1 week+)

13. **Argument parsing** — Implement both a `getopts` version and a long-option `--flag` parser.
14. **Regex & validation** — Use `=~` and `BASH_REMATCH` to parse versions/dates/emails.
15. **Processes & parallelism** — Run jobs in parallel with `&`+`wait` and `xargs -P`; add `timeout` and retry-with-backoff.
16. **Debugging** — Practice `set -x`, custom `PS4`, and `BASH_XTRACEFD`.

**Capstone projects (pick two or more):**

- **Deploy script** — pull latest code, run a build, atomically swap a symlink to the new release, health-check with retry/backoff, roll back on failure, send a notification. Idempotent and `--dry-run` capable.
- **Project bootstrapper** — scaffold a new project: create the directory tree, render template files (here-docs), `git init`, install deps if a manager is present (`command -v`), and print next steps.
- **Dotfiles installer** — symlink dotfiles into `$HOME` idempotently, back up any existing files, detect OS (Linux vs macOS) and branch accordingly, and be safe to re-run.
- **Log-analysis CLI** — a full tool with subcommands (`top`, `errors`, `tail`), `getopts`/long flags, colored output (TTY-detected), and streaming with `tail -F`.
- **System healthcheck** — check disk, memory, key services and ports in parallel, with per-check timeouts, a summary table, and a non-zero exit if anything is unhealthy.

---

### Reference: tools you'll lean on

| Tool | Purpose |
|---|---|
| `bash` 5.2+ | the shell itself |
| `shellcheck` | static analysis / linting — **install it** |
| `printf` | safe, formatted output |
| `grep` / `sed` / `awk` | search, edit, process text |
| `cut` / `tr` / `sort` / `uniq` / `wc` | column & line utilities |
| `find` / `xargs` | locate files & run commands safely (`-print0` / `-0`) |
| `mktemp` / `flock` | temp files & locking |
| `timeout` | bound runtime |
| `jq` | JSON processing (when shell isn't enough) |
| `date` | timestamps & timing |
| `dos2unix` | fix CRLF (Windows) |

---

> **Final note:** Bash rewards two habits above all: **quote everything** (`"$var"`, `"$@"`, `"${arr[@]}"`) and **run ShellCheck on every script**. With strict mode, traps for cleanup, and the small-sharp-tools mindset (grep/sed/awk/find), you can automate almost anything on a Unix system without leaving the terminal. When a script grows beyond ~200 lines of real logic or needs complex data structures, that's your signal to reach for Python — but until then, Bash is the fastest glue there is.
