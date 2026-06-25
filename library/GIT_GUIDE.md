# Git — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I `git commit` and `git push` by copying commands and pray" to "I understand exactly what Git is doing to my repository and can recover from any mess" — without an internet connection. Git is not a set of magic incantations; it is a small, beautifully consistent data model with a famously inconsistent command surface on top. Once you hold the *model* in your head, the commands stop being scary. Every concept below leads with prose that explains *what it is*, *why Git works this way* (the mental model is everything), *when you reach for it*, and *how to use it* — followed by heavily-commented, runnable commands. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Git 2.45+** (current in 2026). Modern conventions assumed throughout:
> - **`main`** is the default initial branch name (not `master`). Git made this configurable via `init.defaultBranch` years ago; GitHub, GitLab, and most teams default to `main` now. This guide uses `main` everywhere.
> - **`git switch`** (change/create branches) and **`git restore`** (discard/unstage changes) are preferred over the old, overloaded **`git checkout`**. `checkout` still works and you'll see it everywhere online, but `switch`/`restore` were introduced precisely because `checkout` did too many unrelated things. We teach the modern verbs first and show the `checkout` equivalent.
> - **`--force-with-lease`** is preferred over `--force` for pushing rewritten history (§13).
> - **`git filter-repo`** (not the deprecated, dangerous `filter-branch`) is the tool for rewriting history to remove secrets/large files (§14).
> - SHA-1 is still the default object hash, but Git supports **SHA-256** repositories (`git init --object-format=sha256`). Interop is still limited in 2026, so most repos remain SHA-1; we note where it matters.
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (line endings / `core.autocrlf`, path separators, credential managers, shells) are called out. This guide cross-references the **GitHub Actions / CI-CD guide** (`GITHUB_ACTIONS_GUIDE.md`) wherever Git meets automation. Confirm fast-moving details against `git help <command>` (which works fully offline).

---

## Table of Contents

1. [What Git Is & First-Time Setup](#1-what-git-is--first-time-setup) **[B]**
2. [The Mental Model — How Git Actually Works](#2-the-mental-model--how-git-actually-works) **[B/I]**
3. [The Basics — init, add, commit, status, diff, log](#3-the-basics--init-add-commit-status-diff-log) **[B]**
4. [Branching & Merging](#4-branching--merging) **[B/I]**
5. [Remotes — fetch, pull, push, tracking](#5-remotes--fetch-pull-push-tracking) **[B/I]**
6. [Rebasing](#6-rebasing) **[I/A]**
7. [Undoing Things — restore, reset, revert, amend, reflog](#7-undoing-things--restore-reset-revert-amend-reflog) **[I]**
8. [Stashing](#8-stashing) **[I]**
9. [Inspecting & Finding — blame, bisect, grep, pickaxe](#9-inspecting--finding--blame-bisect-grep-pickaxe) **[I/A]**
10. [Tags & Releases](#10-tags--releases) **[I]**
11. [Collaboration Workflows](#11-collaboration-workflows) **[I]**
12. [Advanced — cherry-pick, submodules, worktrees, hooks, LFS](#12-advanced--cherry-pick-submodules-worktrees-hooks-lfs) **[A]**
13. [Fixing Mistakes & Recovery](#13-fixing-mistakes--recovery) **[I/A]**
14. [Best Practices](#14-best-practices) **[I/A]**
15. [Gotchas & Command Reference](#15-gotchas--command-reference)
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects)

---

## 1. What Git Is & First-Time Setup

Before a single command, you need the *why*. Git is a **distributed version control system (DVCS)**. "Version control" means it records the history of your project — every change, who made it, when, and why — so you can review the past, compare versions, undo mistakes, and collaborate without overwriting each other. "Distributed" is the part that makes Git different from what came before, and understanding that difference is the foundation for everything else.

### 1.1 Distributed vs centralized — the core idea **[B]**

In an older **centralized** VCS (CVS, Subversion/SVN), there is exactly one repository — it lives on a central server — and developers have only a *working copy* of the files. To see history, branch, diff old versions, or commit, you must talk to that server. If the server is down or you're offline, you're mostly stuck; if the server's disk dies and there's no backup, the project's entire history is gone.

Git flips this. **Every clone is a full, independent repository** — it contains the complete history, every version of every file, all branches and tags, stored locally in a hidden `.git` folder. The "server" (GitHub, GitLab, a bare repo on a colleague's machine) is just another clone you've agreed to treat as the shared meeting point, conventionally named **`origin`**. There is nothing technically special about it.

The consequences of this design are everything people love about Git:

- **Almost every operation is local and instant.** Commit, branch, diff, view log, merge — none of these need a network. You commit on a plane, in a tunnel, with no Wi-Fi. You sync (`push`/`pull`) only when you choose to.
- **It's a safety net.** Because every clone is a full backup, losing the server is an inconvenience, not a catastrophe. Anyone's clone can re-seed `origin`.
- **Branching is cheap and central to the workflow.** A branch in Git is just a tiny pointer (§2), not a server-side copy of files, so creating and discarding branches costs nothing. This is *why* Git-based workflows revolve around branches in a way SVN never did.
- **You decide when to share.** Your local history is yours to refine — squash messy work-in-progress commits, rewrite messages — *before* you push it to others. (With the one golden rule that you don't rewrite history others already have — §6.)

The mental shift from SVN: there is no single "the repository." There are many repositories that occasionally synchronize by exchanging commits. `origin` is a social convention, not a technical authority.

### 1.2 Installing Git **[B]**

```bash
# Windows: download "Git for Windows" from git-scm.com, OR use a package manager:
winget install Git.Git                 # winget (built into Windows 11)
# choco install git                    # Chocolatey, if you use it

# macOS:
brew install git                       # Homebrew (recommended; bundled git is older)
# (running `git` the first time may prompt to install Apple's command-line tools)

# Linux (pick your distro's manager):
sudo apt install git                   # Debian/Ubuntu
sudo dnf install git                   # Fedora/RHEL
sudo pacman -S git                     # Arch

git --version                          # verify — you want 2.45 or newer in 2026
```

> **⚡ Windows note — "Git for Windows" gives you more than git.** The installer bundles **Git Bash** (a real Bash/POSIX shell — the easiest way to follow Unix-style tutorials on Windows), the `git` CLI for PowerShell/cmd, and **Git Credential Manager** (handles GitHub/Azure auth, including OAuth and 2FA, so you never type a password). During install, the two choices worth thinking about are the **default editor** (pick VS Code or Notepad over the default Vim if Vim scares you) and **line-ending handling** (`core.autocrlf` — covered in §14.7; the safe default on Windows is "Checkout Windows-style, commit Unix-style").

### 1.3 First-time configuration — do this once per machine **[B, important]**

Git records *who* made each commit. Before your first commit, set your name and email. This identity is stamped into every commit you create (it is not a login — it's just metadata), so use the email you want associated with your work (on GitHub, match it to your account email so commits link to your profile).

Configuration lives at three levels, each overriding the one above it. Understanding the hierarchy saves confusion later:

- **System** (`--system`): every user on the machine. Rarely edited. File: Git's install dir.
- **Global** (`--global`): you, across all your repos. File: `~/.gitconfig` (i.e. `C:\Users\You\.gitconfig` on Windows). This is where 95% of your personal config goes.
- **Local** (`--local`, the default): this one repository only. File: `.git/config` inside the repo. Use it to override global per-project (e.g. a different email for work repos).

More specific wins: local overrides global overrides system.

```bash
# Identity — set globally so it applies to every repo (do this first!):
git config --global user.name  "Ada Lovelace"
git config --global user.email "ada@example.com"

# Make `main` the default branch name for NEW repos (modern convention):
git config --global init.defaultBranch main

# Default editor for commit messages, interactive rebase, etc.:
git config --global core.editor "code --wait"   # VS Code (the --wait is REQUIRED:
                                                 #   it makes git block until you close the tab)
# git config --global core.editor "nano"        # or nano (simple, on most Unix systems)
# git config --global core.editor "vim"         # the traditional default

# How `git pull` reconciles divergence — set this explicitly to avoid a nag (see §5/§6):
git config --global pull.rebase false   # default: merge.  Set `true` for rebase-on-pull.

# Push only the current branch to its upstream (modern, least-surprising default):
git config --global push.default simple   # this is the default in modern Git; set it anyway

# Use colour in output (usually on by default, but be explicit):
git config --global color.ui auto

# View your config and find WHERE each value came from (which file set it):
git config --list                         # all effective values
git config --list --show-origin           # same, annotated with the file each came from
git config user.email                     # read one specific value
git config --global --edit                # open your ~/.gitconfig in your editor
```

> **Windows line endings — set this now (full discussion in §14.7).** On Windows, configure `core.autocrlf` so you don't pollute repos with carriage returns:
> ```bash
> git config --global core.autocrlf true   # Windows: CRLF in working dir, LF in the repo
> # macOS/Linux would use: git config --global core.autocrlf input
> ```

### 1.4 Aliases — teach Git your shortcuts **[B]**

An **alias** is a short name for a longer command. Git pros lean on a handful heavily because they type the same commands hundreds of times a day. Aliases live in your global config, so they follow you across every repo.

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.sw switch
git config --global alias.cm "commit -m"
git config --global alias.unstage "restore --staged"
# A genuinely useful pretty log graph — worth memorizing the payoff, not the syntax:
git config --global alias.lg "log --oneline --graph --decorate --all"

# Now these work:
git st          # = git status
git lg          # = a colourful commit graph of all branches
```

You can also alias a *shell* command by prefixing with `!`:

```bash
# `git last` -> show the most recent commit; the ! means "run this as a shell command":
git config --global alias.last '!git log -1 HEAD'
```

### 1.5 Getting help — entirely offline **[B]**

Git ships its complete manual locally. You never need the internet to learn a command's flags.

```bash
git help            # the list of common commands and a starting point
git help <command>  # full man page for a command (opens in a pager/browser)
git help commit     # e.g. everything about commit
git commit --help   # identical to the above
git commit -h       # SHORT inline help (just the flag summary) — fast and stays in the terminal
git help -g         # list the "guides" (gitglossary, gittutorial, gitworkflows, ...)
git help glossary   # the official glossary — great for nailing down terminology
```

`git <command> -h` (single dash, short) prints a quick flag summary right in the terminal — use this constantly. `git help <command>` (or `--help`) opens the full prose manual.

---

## 2. The Mental Model — How Git Actually Works

This is **the most important section in the guide**. Most people's pain with Git comes from memorizing commands without the model underneath. Once the model clicks, commands become *predictable*: you can reason about what any command will do, and recover from anything. Read this section slowly. Re-read it. Everything later assumes it.

### 2.1 The three areas: working directory, staging area, repository **[B, crucial]**

A Git project has **three "trees"** (states your files can be in). Picture them left to right:

```
   WORKING DIRECTORY            STAGING AREA (INDEX)            REPOSITORY (.git)
  ────────────────────         ─────────────────────         ────────────────────
  your actual files            a proposed next                permanent, immutable
  on disk that you             commit — a snapshot            history: every commit
  edit in your editor          you're building up             you've ever made
        │                              │                              │
        │   git add <file>             │      git commit              │
        │ ───────────────────────────► │ ───────────────────────────►│
        │                              │                              │
        │   git restore <file>         │   git restore --staged <f>   │
        │ ◄─────────────────────────── │ ◄─────────────────────────── │
        │      (discard edits)         │       (unstage)              │
```

- **Working directory** (a.k.a. *working tree*): the files you see and edit on disk. This is a single checked-out version of your project that you're free to modify. Changes here are *not* tracked by Git until you tell it about them.
- **Staging area** (a.k.a. the **index**): a middle zone — Git's defining and most-misunderstood feature. It's a *draft of your next commit*. You selectively `git add` changes into it. The commit you eventually make captures *exactly what's in the staging area*, not necessarily everything you changed on disk.
- **Repository** (the `.git` directory): the database of committed history. Once something is committed, it's recorded permanently (and recoverable — §13). This is the part that gets pushed and pulled.

A file's lifecycle: it starts **untracked** (Git has never seen it) → you `git add` it so it becomes **staged** → you `git commit` so it becomes **tracked/committed**. After that, editing it makes it **modified**; `git add` stages the modification; `git commit` records it. `git status` always tells you which state every file is in.

### 2.2 Why a staging area? The logic most tutorials skip **[B/I]**

Beginners ask: why the extra step? Why not just commit my changes? The staging area exists so you can **craft commits deliberately** instead of dumping whatever happens to be on disk.

Imagine you sat down and fixed a bug *and* reformatted a function *and* started a new feature — three unrelated changes, maybe across five files. A good history has these as **three separate, atomic commits** (each one self-contained and revertible — §14.1), not one muddy "stuff" commit. The staging area lets you pick exactly which changes (even which *lines* within a file) belong in the next commit:

```bash
git add bugfix.js            # stage only the bug fix
git commit -m "fix: off-by-one in pagination"
git add -p formatter.js      # interactively stage ONLY the formatting hunks (see below)
git commit -m "style: reformat parseUser"
# the half-finished feature stays unstaged, uncommitted, untouched
```

`git add -p` (`--patch`) walks you through each *hunk* (a contiguous block of changed lines) and asks whether to stage it. This is how you split a messy working directory into clean commits. The staging area is therefore a **commit-composition workspace** — that's its whole reason to exist.

### 2.3 Git stores SNAPSHOTS, not diffs **[B/I, crucial]**

Here is the single biggest conceptual difference between Git and almost every VCS before it. SVN, CVS, and friends store history as a **list of changes (deltas)**: file F started as version 1, then here's the diff to version 2, then the diff to version 3. To reconstruct version 3, they replay diffs.

**Git does not think in diffs. Git thinks in snapshots.** Every time you commit, Git takes a picture of what *all your tracked files* look like at that moment and stores a reference to that complete snapshot. It's as if Git photographs your entire project on every commit.

"But that sounds wasteful — wouldn't storing a full copy every commit be huge?" No, because of a clever optimization: **if a file hasn't changed between commits, Git doesn't store it again — it just stores a pointer (a SHA hash) to the identical previous version.** Content is stored once and shared by reference. Identical content = identical hash = stored once. So you get the *mental simplicity* of snapshots with the *storage efficiency* of deduplication.

Why does this matter to you, practically? Because it explains Git's behaviour:
- Branching and switching are fast — Git just points you at a different snapshot.
- Git can compute a diff between *any* two snapshots on demand (diffs are *derived*, not stored), which is why `git diff` between two distant commits is instant.
- Merges and history operations reason about snapshots and their relationships, which makes them robust.

### 2.4 The object model — blobs, trees, commits, content addressing **[I, the heart of it]**

Underneath, the `.git` database stores everything as **objects**, and each object is identified by the **SHA-1 hash of its content** (a 40-character hex string like `e3c9a8f...`). This is called **content addressing**: the *name* of an object *is* a fingerprint of what's inside it. Change one byte of content and you get a completely different hash. This is the bedrock of Git's integrity — corruption or tampering is detectable because the content would no longer match its hash.

There are four object types; three matter day-to-day:

1. **Blob** ("binary large object") — the *contents of a file*. Just the bytes. A blob has no filename and no metadata; it's pure content. Two files with identical contents (even in different folders) are the *same blob*, stored once.
2. **Tree** — a *directory*. A tree lists names and points to the blobs (files) and sub-trees (subdirectories) it contains, along with their modes (permissions). A tree is essentially a snapshot of one directory level. The top-level tree of a commit, plus its sub-trees, *is* the project snapshot.
3. **Commit** — a *snapshot plus metadata*. A commit object points to exactly **one top-level tree** (the whole project state at that moment) and stores: the author and committer (name, email, timestamp), the commit message, and **the hash(es) of its parent commit(s)**. A normal commit has one parent; the very first commit has none; a merge commit has two (or more).
4. **Tag object** — an annotated tag (§10); a named, optionally signed pointer to a commit.

Because a commit records its parent's hash, and that parent records *its* parent, commits form a **chain backwards through time** — actually a **DAG (directed acyclic graph)**, since merges join chains. The whole history is this graph of commits, each pointing to a tree (a snapshot), each tree pointing to blobs (file contents).

```
   commit C3 ──────► tree ──► blob (README.md)
      │  (parent)         └──► blob (app.js)
      ▼
   commit C2 ──────► tree ──► blob (README.md, unchanged → SAME blob as C3)
      │  (parent)         └──► blob (app.js, older version → different blob)
      ▼
   commit C1 (no parent — the root commit)
```

You can see this machinery directly. These "plumbing" commands aren't for daily use, but running them *once* makes the model concrete:

```bash
# Hash some content the way Git would store it (creates a blob's hash):
echo "hello" | git hash-object --stdin     # prints the SHA Git would give this content

# Inspect any object by its hash:
git cat-file -t <hash>     # -t = TYPE of the object: "blob", "tree", "commit", or "tag"
git cat-file -p <hash>     # -p = PRETTY-PRINT its contents

# Walk a real commit's innards:
git cat-file -p HEAD       # show the commit object: its tree, parent, author, message
git cat-file -p HEAD^{tree}  # show the top-level TREE: the files/dirs in that snapshot
git log --pretty=raw -1    # show a commit including its tree & parent hashes
```

**Why content addressing is genius:** because identical content always hashes to the same value, Git deduplicates automatically, detects corruption for free, and can compare or transfer history by exchanging hashes ("do you have object e3c9a8?"). Two repos can determine *exactly* which objects they're missing from each other just by comparing hashes — that's how `fetch`/`push` are efficient.

### 2.5 Refs, HEAD, and branches as movable pointers **[B/I, crucial]**

Now the payoff. We said history is a graph of commits chained by parent hashes. But hashes are 40 hex characters — unusable by humans. **Refs** (references) are human-friendly *names that point to commits*. A ref is literally a tiny file containing one commit hash.

**A branch is just a movable pointer to a commit.** That's it. `main` is a 41-byte file (`.git/refs/heads/main`) containing the hash of the latest commit on that branch. When you make a new commit on `main`, Git creates the commit object and then **advances the `main` pointer forward** to the new commit. The branch "moves." This is why creating a branch is instant and free — you're just writing one more tiny file with a hash in it; you are *not* copying any files.

```
        C1 ◄─── C2 ◄─── C3   (arrows show parent links: C3's parent is C2)
                         ▲
                       main   ← the branch "main" is a pointer to commit C3
                         ▲
                       HEAD   ← HEAD points to main (you are "on" main)
```

**HEAD** is a special ref meaning **"where you are right now" / "what the next commit's parent will be."** Normally HEAD points to a *branch name* (e.g. it contains `ref: refs/heads/main`), and that branch points to a commit. So HEAD → main → C3. When you switch branches, you're just moving HEAD to point at a different branch ref, and Git updates your working directory to match that branch's snapshot.

Make a commit and watch the pointers move:

```
   Before commit:  C1 ◄─ C2 ◄─ C3 ◄─ main ◄─ HEAD

   git commit       creates C4 (parent = C3), then advances main:

   After commit:   C1 ◄─ C2 ◄─ C3 ◄─ C4 ◄─ main ◄─ HEAD
```

Other refs work the same way:
- **Tags** (`refs/tags/v1.0`) — pointers to a commit that *don't move* (a fixed label, §10).
- **Remote-tracking branches** (`refs/remotes/origin/main`) — your local memory of where `origin`'s `main` was the last time you talked to it (§5).

You can inspect refs directly:

```bash
git rev-parse HEAD            # print the actual commit hash HEAD resolves to
cat .git/HEAD                 # usually: "ref: refs/heads/main"  (HEAD -> a branch)
git symbolic-ref HEAD         # the same, the clean way
git branch                    # list local branches; the * marks the one HEAD points to
git show-ref                  # dump every ref and the hash it points to
```

**The detached HEAD state** (you'll meet it in §13): if you check out a *commit hash* or a *tag* directly instead of a branch, HEAD points straight at a commit rather than at a branch. You're "detached." Commits you make are valid but no branch points to them — so when you switch away, they're easy to lose (recoverable via reflog, §13). Git warns you loudly when this happens.

### 2.6 Putting it together — what a commit *really* does

When you run `git commit`, Git:
1. Takes everything currently in the **staging area** and writes any new file contents as **blobs**.
2. Builds **tree** objects capturing the directory structure (reusing unchanged blobs/trees by hash).
3. Creates a **commit** object pointing to the top-level tree, recording your identity, the message, and the **current HEAD commit as its parent**.
4. **Advances the current branch pointer** to this new commit (and HEAD comes along, since HEAD → branch).

Everything else in Git — branching, merging, rebasing, resetting, reverting — is *manipulating these pointers and creating new objects*. Reset moves a branch pointer. Merge creates a commit with two parents. Rebase replays commits onto a new base. Hold the model and the rest is detail.

### 2.7 Reachability, garbage collection & why nothing is really lost **[I, the safety-net model]**

One more idea ties §2.6 to the "I can always recover" confidence you'll lean on in §13: **reachability**. A commit is "alive" as long as *something points to it* — a branch, a tag, HEAD, the reflog, or another commit that has it as an ancestor. Starting from every ref, Git can walk the parent links and reach a set of commits (and through them, trees and blobs). Those are **reachable** objects. Anything *not* reachable from any ref is an **orphan** (a "dangling" object).

Here's the reassuring part: when you `reset --hard` past a commit, or abandon a detached-HEAD commit, or `branch -D` a branch — you didn't *delete* those commit objects. You only removed the *pointer* that made them reachable through a branch. The objects still sit in `.git`, and crucially the **reflog still points at them** (§7.6), keeping them reachable (and recoverable) for the reflog's expiry window (~90 days by default). That's *why* the reflog can resurrect "lost" work: the work was never gone, just temporarily unreferenced.

Orphaned objects are eventually cleaned up by **garbage collection** (`git gc`), which Git runs automatically now and then. `gc` also **packs** loose objects: instead of one file per object, Git compresses many objects together into efficient *packfiles* (`.git/objects/pack/`), applying delta compression *between* similar objects (note: this is a storage optimization *layered on top of* the snapshot model — it does not change the fact that each commit conceptually references a full snapshot).

```bash
git fsck --lost-found          # find DANGLING (unreachable) commits/blobs — a recovery aid
git fsck --unreachable         # list everything currently unreachable
git gc                         # manually pack loose objects & prune old unreachable ones
git gc --aggressive            # spend more time for a smaller repo (rarely needed)
git count-objects -vH          # how many objects / how much disk the repo uses
```

> **Takeaway:** "deleting" in Git almost always means "removing a pointer," not "destroying data." Until `gc` prunes an *unreferenced* object past the reflog window, you can get it back. The exceptions — the things with *no* pointer ever — are **uncommitted** working-directory changes (`reset --hard`, `restore`, `clean` destroy these for good) and untracked files. So the practical rule is simple: **commit or stash before any destructive operation**, and the reflog covers the rest.

### 2.8 Two-second model recap **[B]**

If you remember nothing else, remember this chain. You edit files in the **working directory**. You stage a chosen snapshot of them into the **index**. `git commit` freezes the index as an immutable **commit** (a snapshot + parent + metadata, named by the hash of its content). A **branch** is just a sticky-note with a commit hash on it, and **HEAD** is the sticky-note marking "you are here." Sharing is exchanging commits with **remotes**; your `origin/*` refs are cached memories of where the remote was. Every command you'll learn either *creates objects* or *moves pointers* — usually both. That's Git.

---

## 3. The Basics — init, add, commit, status, diff, log

With the model in hand, the everyday commands become obvious. This section is the daily loop: change files → stage → commit → inspect.

### 3.1 Starting a repository — `init` and `clone` **[B]**

There are two ways to get a Git repo: create a fresh one (`init`) or copy an existing one (`clone`).

```bash
# init — turn the CURRENT folder into a new, empty Git repository:
mkdir my-project && cd my-project
git init                       # creates the hidden .git/ folder; you're now in a repo
git init -b main my-project    # create folder AND init with branch "main" in one step
#                                (-b sets the initial branch name; needs Git 2.28+)

# clone — copy a remote repository (full history) to your machine:
git clone https://github.com/user/repo.git          # clones into ./repo
git clone https://github.com/user/repo.git myname    # clone into folder "myname" instead
git clone git@github.com:user/repo.git              # clone over SSH (needs an SSH key set up)
git clone --depth 1 https://github.com/user/repo.git # SHALLOW: only latest commit (§12.9)
```

`git init` makes the `.git` directory — that single folder *is* the repository; delete it and you have plain files again. `git clone` does several things at once: creates a new repo, adds the source as a remote named **`origin`**, fetches all its objects, and checks out its default branch. After cloning you're ready to work immediately.

### 3.2 `git status` — your constant companion **[B]**

`git status` answers "what's going on right now?" — which branch you're on, what's staged, what's modified but unstaged, what's untracked, and whether you're ahead/behind the remote. Run it *constantly*; it's free and it tells you Git's view of the world (which is the view that matters).

```bash
git status            # the full, friendly report
git status -s         # SHORT format: two columns of status codes, compact
git status -sb        # short + branch line at the top
# Short codes:  M = modified, A = added (staged new), ?? = untracked, D = deleted,
#               R = renamed.  Left column = staging area, right column = working dir.
```

### 3.3 `git add` — staging changes **[B]**

`git add` copies the *current* state of a file into the staging area (the next commit's draft). Remember: it stages a *snapshot of the file as it is now*. If you edit the file again after `git add`, that newer edit is **not** staged until you `git add` again.

```bash
git add file.txt              # stage one specific file
git add src/ docs/            # stage everything under these directories (recursively)
git add .                     # stage all changes in the CURRENT directory and below
git add -A                    # stage ALL changes in the whole repo (incl. deletions, anywhere)
git add -u                    # stage modifications & deletions of TRACKED files only
                              #   (ignores brand-new untracked files)

# Interactive, surgical staging — stage SOME hunks of a file, not all of it:
git add -p                    # walk each hunk; answer: y=stage, n=skip, s=split, q=quit, ?=help
git add -p file.txt           # patch-stage just this file
```

> **Gotcha — `git add .` vs `git add -A`.** Historically `git add .` only staged the current directory subtree. In modern Git both stage new and modified files, but `-A` also catches deletions repo-wide regardless of your current directory. When in doubt, `git add -A` from the repo root stages "everything."

### 3.4 `git commit` — recording a snapshot **[B]**

`git commit` takes whatever is in the staging area and writes it as a permanent commit object, advancing your branch (§2.6). A commit needs a **message** explaining *why* the change was made.

```bash
git commit -m "Add user login form"     # commit staged changes with an inline message
git commit                              # opens your editor for a multi-line message
git commit -a -m "msg"                  # -a = auto-stage all TRACKED modified/deleted files,
                                        #   THEN commit. (Skips the explicit add — but it does
                                        #   NOT include untracked/new files. Convenient but
                                        #   defeats deliberate staging; use sparingly.)
git commit -am "msg"                    # same, combined flags

git commit --amend                      # REPLACE the last commit (edit its message and/or add
                                        #   newly-staged changes). Rewrites history — only on
                                        #   commits you haven't pushed/shared! (See §7.5/§13.)
git commit --amend --no-edit            # amend to include staged changes but KEEP the old message
```

**Writing good commit messages** is a real skill (and conventions in §14.2). The widely-followed format:
- A **subject line** ≤ ~50 chars, in the **imperative mood** ("Add login" not "Added login" — read it as "this commit will *Add login*"), capitalized, no trailing period.
- A **blank line**, then a **body** wrapping at ~72 chars explaining *what* and especially *why* (the diff already shows *how*).

```bash
git commit          # then in the editor:
# Add rate limiting to the login endpoint
#                                          ← blank line
# Brute-force attempts were hitting the auth route unthrottled.
# This adds a 5-attempts-per-minute limit keyed by IP, returning
# 429 with a Retry-After header. Closes #142.
```

### 3.5 `.gitignore` — telling Git what to ignore **[B]**

Most projects have files you *never* want in version control: build output, dependencies (`node_modules/`), logs, secrets, editor cruft, OS junk (`.DS_Store`, `Thumbs.db`). A **`.gitignore`** file lists patterns of paths Git should leave untracked (it won't show them in `status` or stage them with `git add .`).

The `.gitignore` file itself **should** be committed (so the whole team shares the same rules). Patterns:

```gitignore
# A line starting with # is a comment.

# Bare name: ignore anywhere in the tree (any directory level):
node_modules
*.log                 # ignore all .log files anywhere (* = wildcard, any chars but /)
.env                  # ignore secrets file (NEVER commit secrets — §14.6)

# Trailing slash: match DIRECTORIES only:
build/                # ignore the build directory and everything under it
dist/

# Leading slash: anchor to the repo ROOT only (not nested copies):
/config.local.json    # only the top-level one; a nested config.local.json is NOT ignored

# ** matches across directories:
**/temp/              # any "temp" directory at any depth
logs/**/*.log         # all .log under logs/, at any depth

# ! NEGATES a previous pattern (re-include something you'd otherwise ignore):
*.secret              # ignore all .secret files
!public.secret        # ...except this one (order matters; a later ! can re-include)

# ? matches a single char; [abc] matches a char set:
file?.txt             # file1.txt, fileA.txt — but not file10.txt
```

```bash
# Common gotcha: a file is ALREADY tracked, so .gitignore is ignored for it.
# .gitignore only affects UNTRACKED files. To stop tracking an already-committed file:
git rm --cached secret.env       # remove from the index (staging/repo) but KEEP it on disk
git commit -m "Stop tracking secret.env; add to .gitignore"
# (then ensure it's listed in .gitignore so it stays out)

# Debug why a path is or isn't ignored:
git check-ignore -v path/to/file   # prints which .gitignore rule (and line) matched
```

> **Tip:** `github.com/github/gitignore` hosts curated `.gitignore` templates per language/tool — but since you're offline, just remember the principle: ignore generated, downloaded, secret, and machine-specific files; track source and config. A global ignore (`git config --global core.excludesFile ~/.gitignore_global`) is handy for OS/editor cruft that shouldn't burden every project's `.gitignore`.

### 3.6 `git diff` — seeing what changed **[B]**

`diff` shows the line-by-line differences between two states. The confusing part is *which two states*, and that maps directly onto the three areas (§2.1):

```bash
git diff                  # WORKING DIR vs STAGING — "what have I changed but not yet staged?"
git diff --staged         # STAGING vs LAST COMMIT — "what will the next commit include?"
git diff --cached         # exact synonym for --staged
git diff HEAD             # WORKING DIR vs LAST COMMIT — all changes, staged or not

git diff main feature     # compare the tips of two branches
git diff abc123 def456    # compare two commits by hash
git diff HEAD~3 HEAD      # what changed across the last 3 commits
git diff main feature -- src/app.js   # restrict to one path (after the --)

# Reading a diff: lines starting with - were removed, + were added. The @@ -a,b +c,d @@
# "hunk header" tells you the line ranges. Colour: red = removed, green = added.

git diff --stat           # SUMMARY: files changed + insertion/deletion counts (no full diff)
git diff --word-diff      # highlight changes word-by-word rather than whole lines
git diff -w               # ignore whitespace-only changes (handy reviewing reformatted code)
```

**Mental shortcut:** no flag = "unstaged work," `--staged` = "the commit I'm about to make," `HEAD` = "everything since the last commit."

### 3.7 `git log` — browsing history **[B]**

`log` walks the commit graph backwards from HEAD, showing commits. The default is verbose; the useful skill is knowing the formatting flags.

```bash
git log                       # full log: hash, author, date, message — newest first
git log --oneline             # one compact line per commit (short hash + subject) — the workhorse
git log --oneline --graph     # + an ASCII graph of branch/merge structure
git log --oneline --graph --all --decorate
#   --all = include every branch (not just current);  --decorate = show branch/tag labels
#   ^ memorize this combo (or alias it as `git lg`, §1.4) — it's the best history view

git log -5                    # only the last 5 commits
git log -p                    # show the full DIFF introduced by each commit (patch)
git log --stat                # show per-commit file change stats
git log --author="Ada"        # filter by author
git log --since="2 weeks ago" # filter by date (also --until, --after, --before)
git log --grep="fix"          # filter by message text (regex)
git log -S "functionName"     # PICKAXE: commits that ADDED/REMOVED that string (§9.4)
git log main..feature         # commits on `feature` that are NOT on `main` (very useful!)
git log --follow file.txt     # history of one file, even across renames
git log -- path/to/file       # history touching a given path

# Custom one-line format (great for aliases):
git log --pretty=format:"%h %ad %an %s" --date=short
#   %h short hash, %ad author date, %an author name, %s subject. See `git help log` for all.
```

`git log main..feature` deserves a callout: the `A..B` syntax means "commits reachable from B but not from A" — i.e. "what's on feature that main doesn't have yet." You'll use it constantly to preview what a branch or PR adds.

### 3.8 `git show` — inspecting one commit/object **[B]**

```bash
git show               # the most recent commit: its metadata AND its full diff
git show HEAD~2        # the commit two before HEAD
git show abc123        # any commit by hash
git show v1.0          # what a tag points to
git show HEAD:src/app.js   # the CONTENTS of a file AS IT WAS at that commit (handy!)
git show --stat HEAD   # just the file-change summary for that commit
```

---

## 4. Branching & Merging

Branching is where Git stops being "a backup tool" and becomes "a parallel-development tool." Because branches are free (§2.5), the standard workflow is: never work directly on `main`; instead branch for every feature/fix, then merge back. This section makes that fluent.

### 4.1 What a branch *really* is (again, because it matters) **[B]**

Recall §2.5: a branch is a **movable pointer to a commit**, and `HEAD` points to the branch you're "on." Creating a branch writes one tiny file. Switching branches moves `HEAD` and updates your working directory to that branch's snapshot. Committing advances the current branch pointer. Hold this and merging makes sense instantly.

### 4.2 Creating and switching — `branch`, `switch` (and the old `checkout`) **[B]**

```bash
git branch                    # LIST local branches (* = current)
git branch -a                 # list all branches INCLUDING remote-tracking ones
git branch -v                 # list with the latest commit on each
git branch feature-login      # CREATE a branch named feature-login (does NOT switch to it)

# SWITCH to a branch (the modern verb — clearer than checkout):
git switch feature-login      # move HEAD onto feature-login (working dir updates to match)
git switch -c feature-login   # CREATE and switch in one step (-c = create)
git switch -                  # switch to the PREVIOUS branch (like `cd -`)
git switch main               # back to main

# The classic equivalents you'll see everywhere online (checkout is overloaded — §0):
git checkout feature-login    # = git switch feature-login
git checkout -b feature-login # = git switch -c feature-login
```

> **⚡ Why `switch`/`restore` exist.** `git checkout` historically did *three* unrelated jobs: switch branches, create branches, AND discard file changes. Overloading "checkout" caused dangerous mistakes (people meaning to switch branches accidentally nuked their edits). Git 2.23 split it: **`git switch`** for branches, **`git restore`** for files. Use them. `checkout` still works for muscle-memory and old tutorials.

When you create a branch, it points at *your current commit*. Now your new commits go onto that branch, leaving `main` where it was:

```
   Before:   C1 ◄─ C2 ◄─ C3 ◄─ main ◄─ HEAD
   git switch -c feature; (work); commit C4:

             C1 ◄─ C2 ◄─ C3 ◄─ main
                              ◄─ C4 ◄─ feature ◄─ HEAD
   main still points at C3; feature points at C4. History has diverged.
```

### 4.3 Merging — bringing a branch's work back **[B/I]**

`git merge` integrates another branch's commits into your current branch. There are two fundamentally different outcomes, and knowing which you'll get prevents confusion.

**Fast-forward merge:** if the current branch hasn't diverged — i.e. `main` is a direct ancestor of `feature`, so `feature` is simply "ahead" — Git can just **slide the `main` pointer forward** to `feature`'s commit. No new commit is created; the history stays linear. This happens when nobody committed to `main` while you worked on `feature`.

```
   Before merge (main hasn't moved since you branched):
       C1 ◄─ C2 ◄─ C3 ◄─ main
                        ◄─ C4 ◄─ C5 ◄─ feature

   git switch main && git merge feature   →  FAST-FORWARD (just move the pointer):
       C1 ◄─ C2 ◄─ C3 ◄─ C4 ◄─ C5 ◄─ main, feature
   Linear history, no merge commit.
```

**Three-way merge:** if *both* branches advanced after they split (history diverged), Git can't just move a pointer. It finds the **merge base** (the common ancestor commit), looks at the two branch tips, and combines the changes, creating a brand-new **merge commit** that has **two parents**. This permanently records "these two lines of work joined here."

```
   Diverged history:
       C1 ◄─ C2 ◄─ C3 ◄─ C6 ◄─ main      (someone added C6 to main)
                  └─ C4 ◄─ C5 ◄─ feature  (you added C4, C5)

   git switch main && git merge feature   →  3-WAY MERGE creates M:
       C1 ◄─ C2 ◄─ C3 ◄─ C6 ◄─ M ◄─ main
                  └─ C4 ◄─ C5 ──┘
   M has TWO parents (C6 and C5). The diamond shape records the merge.
```

```bash
git switch main               # be ON the branch you want to merge INTO
git merge feature-login       # bring feature-login's commits into main

# Force a merge commit even when a fast-forward was possible (preserves "this was a branch"):
git merge --no-ff feature-login   # ALWAYS creates a merge commit; good for feature branches
                                  #   on shared history — keeps the branch visible in the graph

# The opposite — refuse to merge unless it's a clean fast-forward (no merge commit):
git merge --ff-only feature-login # errors out if a real merge would be needed

# Abort a merge that's in progress (e.g. conflicts you don't want to resolve right now):
git merge --abort             # restores the pre-merge state
```

> **Best practice:** many teams use `--no-ff` for merging feature branches into `main` so the history shows *that a feature existed as a unit* (the merge commit groups its commits), while using fast-forward or rebase for trivial updates. Others prefer a strictly linear history via rebase (§6). Both are valid — agree as a team.

### 4.4 Resolving merge conflicts — step by step **[I, essential skill]**

A **conflict** happens when both branches changed *the same lines of the same file* (or one edited a file the other deleted), so Git can't decide which version wins. Git pauses the merge and asks *you* to resolve it. Conflicts are normal — don't panic. Here's the exact procedure.

When a conflict occurs, Git tells you which files conflicted and marks the conflicting regions *inside* those files with markers:

```text
<<<<<<< HEAD
the version from YOUR current branch (e.g. main)
=======
the version from the branch you're MERGING IN (e.g. feature)
>>>>>>> feature-login
```

The block between `<<<<<<<` and `=======` is *your* side; between `=======` and `>>>>>>>` is *their* side. Your job: edit the file so it contains the correct final result, and **delete all three marker lines**.

```bash
# Step 1: a merge conflict happened. See exactly what's unresolved:
git status                    # lists files "both modified" / "Unmerged paths"

# Step 2: open each conflicted file. Find the <<<<<<< / ======= / >>>>>>> markers.
#         Edit the file so it has the CORRECT final content. Remove ALL marker lines.
#         (A merge tool can help: `git mergetool` opens a configured 3-way diff tool.)

# Step 3: stage each file you've resolved — staging tells Git "this conflict is handled":
git add conflicted-file.js

# Step 4: when ALL conflicts are staged, complete the merge:
git commit                    # for a merge, this finalizes the merge commit
                              #   (Git pre-fills a sensible merge message; just save it)

# Want to bail out instead of resolving? Reset to before the merge:
git merge --abort             # cleanly undo the whole merge attempt

# Helpful during resolution:
git diff                      # show the conflict hunks
git checkout --ours file      # take YOUR side entirely for this file (current branch)
git checkout --theirs file    # take THEIR side entirely (the branch being merged in)
#   (then `git add file`)
git log --merge -p file       # show the commits from each side that touch the conflicted file
```

> **⚡ Tip — `rerere` for repeated conflicts.** If you resolve similar conflicts over and over (common when maintaining long-lived branches), enable `git config --global rerere.enabled true`. Git then *records how you resolved a conflict* and *auto-applies the same resolution* next time the identical conflict appears (§12.9).

> **Gotcha — "ours/theirs" flips during rebase.** In a `merge`, `--ours` is your current branch and `--theirs` is the incoming branch. During a `rebase` (§6) the meaning **reverses** (because rebase replays *your* commits *onto* the other branch, so "ours" is the branch you're replaying onto). When in doubt, read the conflict markers — `HEAD` is always the side currently being built.

### 4.4b How three-way merge actually decides — the merge base **[I, the model behind merges]**

To trust merges (and to debug surprising conflicts), understand *how* Git produces the merged result. A three-way merge has exactly three inputs, hence the name:

1. **Your branch tip** ("ours" — where you are).
2. **Their branch tip** ("theirs" — the branch being merged in).
3. The **merge base**: the *best common ancestor* of the two tips — the commit where the two lines of history last agreed before diverging.

For each file (really, each region), Git compares both tips *against the merge base*. The logic is intuitive: "compared to our shared starting point, what did each side change?"

- If **only one side** changed a region, that side's change wins automatically — no conflict.
- If **both sides changed the *same* region differently**, Git can't choose, so it emits conflict markers and asks you (§4.4).
- If both sides made the **identical** change, that's fine — they agree.

This is why a merge isn't "newer wins" or "bigger wins" — it's a *structured combination of two independent sets of changes relative to a common base*. You can see the base yourself:

```bash
git merge-base main feature        # print the common-ancestor commit of these two branches
git merge-base --all main feature  # (rarely) multiple bases if history is gnarly
git diff $(git merge-base main feature) feature   # what `feature` changed since the split
```

> **⚡ The merge engine: `ort`.** Modern Git (2.34+) uses the **`ort`** merge strategy by default ("Ostensibly Recursive's Twin") — a faster, more correct rewrite of the old `recursive` strategy. It handles renames, directory moves, and criss-cross histories better, so you'll hit fewer spurious conflicts than years-old tutorials warn about. You rarely set this by hand; it's the default. (`git merge -s ort`, or `-s ours`/`-s theirs` for the deliberate "take one whole side" strategies, or `-X ours`/`-X theirs` to auto-resolve *conflicting hunks only* in favour of one side while still merging the rest.)

```bash
git merge -X theirs feature    # merge normally, but auto-resolve any CONFLICTS toward "theirs"
git merge -s ours feature      # record a merge but KEEP our tree entirely (ignore their changes;
                               #   useful to mark a branch as "merged" without taking its code)
```

### 4.5 Deleting and renaming branches **[B]**

Once a branch is merged, delete it — branches are cheap to make and cheap to clean up; stale branches clutter the repo.

```bash
git branch -d feature-login   # delete a branch — SAFE: refuses if it has unmerged commits
git branch -D feature-login   # FORCE delete even if unmerged (you lose those commits unless
                              #   you recover via reflog — §13). Capital -D = "I'm sure."
git branch -m old-name new-name   # rename a branch (-m = move)
git branch -m new-name        # rename the CURRENT branch

# Delete a branch on the REMOTE too (local delete doesn't touch the server):
git push origin --delete feature-login    # delete the remote branch
git push origin :feature-login            # older equivalent syntax (push "nothing" to it)
```

---

## 5. Remotes — fetch, pull, push, tracking

A **remote** is a named reference to *another copy of the repository* — typically the shared one on GitHub/GitLab (named `origin`). Collaboration is just exchanging commits with remotes. The key to never being confused here is understanding **remote-tracking branches** and the **fetch vs pull** distinction.

### 5.1 Managing remotes **[B]**

```bash
git remote                    # list remote names (after a clone, you'll see "origin")
git remote -v                 # list remotes WITH their URLs (v = verbose) — fetch & push URLs
git remote add origin https://github.com/user/repo.git   # add a remote named origin
git remote add upstream https://github.com/orig/repo.git # common: "upstream" = the repo you forked
git remote show origin        # detailed info: URL, branches, what's tracked, push/pull config
git remote rename origin old  # rename a remote
git remote remove old         # delete a remote reference (doesn't touch the server)
git remote set-url origin git@github.com:user/repo.git   # change a remote's URL (e.g. HTTPS→SSH)
```

### 5.2 Remote-tracking branches — the mental model **[B/I, crucial]**

When you clone or fetch, Git creates **remote-tracking branches** like `origin/main`. These are **your local, read-only snapshots of where the remote's branches were the last time you communicated with it.** They are NOT live — `origin/main` only updates when you `fetch` (or `pull`). This is the distributed model in action: you have *your* `main` (a local branch you commit to) and `origin/main` (your cached memory of the server's `main`).

```
   After cloning:
       origin/main ──► C3        ← your snapshot of the server's main
       main ─────────► C3        ← your local main (you commit on this)
       HEAD ─────────► main

   You commit C4 locally:
       origin/main ──► C3        ← UNCHANGED (you haven't talked to the server)
       main ─────────► C4 ──► C3 ← your local main moved ahead
   git status now says "ahead of origin/main by 1 commit"
```

```bash
git branch -r                 # list REMOTE-tracking branches (origin/main, origin/dev, ...)
git branch -vv                # list local branches + which remote branch each tracks + ahead/behind
```

### 5.3 `fetch` vs `pull` — the difference that confuses everyone **[B/I, crucial]**

This is the single most important distinction in this section:

- **`git fetch`** downloads new commits/objects from the remote and **updates your remote-tracking branches** (`origin/main`), but **does NOT touch your local branches or working directory.** It's the *safe, read-only sync*: "show me what's new on the server, but don't change my work." After fetching, *you* decide what to do.
- **`git pull`** is **`fetch` + `merge`** (or `fetch` + `rebase`) in one step. It downloads *and immediately integrates* the remote's changes into your current branch. Convenient, but it changes your working state — and if there are conflicts or surprises, it does so mid-stride.

The pro habit: **`fetch` then look, then merge/rebase deliberately** — especially on a shared branch. `pull` is fine for a personal branch where you just want to catch up.

```bash
git fetch                     # fetch from origin: update ALL remote-tracking branches. Safe.
git fetch origin              # explicit remote
git fetch --all              # fetch from every configured remote
git fetch -p                  # also PRUNE: delete local origin/* refs for branches deleted on server

# After a fetch, INSPECT before integrating:
git log main..origin/main     # commits on the server's main you don't have yet
git diff main origin/main     # what changed
git merge origin/main         # NOW integrate, knowingly (or `git rebase origin/main`)

# pull = fetch + integrate, in one go:
git pull                      # fetch + merge into current branch (default)
git pull --rebase             # fetch + REBASE your local commits on top (linear history — §6.6)
git pull --ff-only            # only fast-forward; REFUSE if a real merge is needed (safest pull)
```

> **⚡ Set a sane pull policy.** Modern Git nags if you haven't chosen how `pull` reconciles divergence. Decide once: `git config --global pull.ff only` (refuse surprising merges) is the safest default; teams favouring linear history use `git config --global pull.rebase true`. The classic merge behaviour is `pull.rebase false`.

### 5.4 `push` — sending your commits to the remote **[B/I]**

`git push` uploads your local branch's new commits to the remote and advances the remote's branch pointer. The first push of a new branch needs to establish a **tracking relationship** (which remote branch this local branch syncs with) via `-u`.

```bash
git push                      # push current branch to its configured upstream
git push origin main          # push local main to origin's main (explicit)

# First push of a NEW branch — set the upstream so future `push`/`pull` need no args:
git push -u origin feature-x  # -u (--set-upstream) links feature-x ↔ origin/feature-x.
                              #   After this, plain `git push` and `git pull` just work here.

git push --all origin         # push all local branches
git push --tags               # push your tags too (tags are NOT pushed by default — §10)
git push origin --delete old  # delete a branch on the remote

# Pushing rewritten history (after rebase/amend) — needs force. Use the SAFE variant:
git push --force-with-lease   # force, but ABORT if the remote moved since you last fetched
                              #   (protects you from clobbering a teammate's push — §13.6)
git push --force              # DANGEROUS: overwrite the remote unconditionally. Avoid.
```

> **Push is rejected ("non-fast-forward")?** It means the remote has commits you don't have locally — someone pushed since you last fetched. Git refuses to overwrite them. The fix is **not** `--force`; it's to integrate first: `git pull --rebase` (or `git fetch` then `git rebase origin/main`), resolve any conflicts, then push. Only force-push (with-lease) when you *intend* to rewrite shared history and have coordinated it (§6.3, §13.6).

### 5.5 Authentication — HTTPS vs SSH, credential helpers **[B]**

```bash
# HTTPS: you authenticate with a Personal Access Token (PAT), not your password.
#   A credential helper caches it so you don't retype it. On Windows, Git Credential
#   Manager (bundled with Git for Windows) handles GitHub OAuth/2FA in a browser popup.
git config --global credential.helper manager      # Windows (Git Credential Manager)
git config --global credential.helper "cache --timeout=3600"   # cache in memory (Unix)
git config --global credential.helper store        # store on disk in plaintext — convenient
                                                   #   but INSECURE; prefer the OS keychain/manager

# SSH: generate a key once, add the public key to GitHub, then clone/push over git@ URLs.
ssh-keygen -t ed25519 -C "ada@example.com"   # create a modern Ed25519 key pair
# (add ~/.ssh/id_ed25519.pub to GitHub → Settings → SSH keys)
ssh -T git@github.com                         # test the connection ("Hi user!" = success)
```

> **HTTPS vs SSH — which to pick?** HTTPS "just works" through corporate firewalls/proxies and pairs with the credential manager (no key setup), so it's the easy default — especially on Windows with Git Credential Manager. SSH avoids token prompts entirely once your key is added and is the norm for people pushing many times a day. Both are equally secure; it's ergonomics. You can switch a repo over anytime with `git remote set-url origin <new-url>`.

### 5.6 A worked example — branches that diverged **[B/I]**

Tie it together with the most common real situation: you and a teammate both committed since the last sync, so the histories diverged. Here's the full decision and the commands.

```bash
# You're on `main`, you have local commit X; meanwhile origin/main gained commit Y.
git fetch                       # update origin/main to include Y (your local main is untouched)
git status                      # "Your branch and 'origin/main' have diverged, and have
                                #  1 and 1 different commits each, respectively."

# Inspect BEFORE integrating — see exactly what each side has:
git log --oneline main..origin/main   # what THEY added that you lack (commit Y)
git log --oneline origin/main..main   # what YOU added that they lack (commit X)

# Now choose how to reconcile:
git merge origin/main           # OPTION A: 3-way merge → a merge commit joining X and Y
#   ...or...
git rebase origin/main          # OPTION B: replay your X on top of Y → linear history (X')
#   ...or do both fetch+integrate at once with the policy you prefer:
git pull --rebase               # = fetch + rebase  (linear; common for personal branches)
git pull --no-rebase            # = fetch + merge   (merge commit)

# Resolve any conflicts (§4.4), then push:
git push                        # now a normal fast-forward push succeeds
```

The takeaway from §5: a rejected push almost always means "you diverged — `fetch`, look, integrate, then push." It is *never* a reason to reach for `--force` unless you *deliberately* rewrote shared history (§13.6).

---

## 6. Rebasing

Rebasing is the topic that intimidates people, but it's conceptually simple once you have the model: **rebase replays your commits onto a new base.** It rewrites history to make it *linear* and *clean*. It's powerful and a little dangerous — hence the famous golden rule. Let's build it up carefully.

### 6.1 What rebase does, and why **[I]**

Suppose you branched `feature` off `main` at C3, did work (C4, C5), and meanwhile `main` advanced to C6. Two ways to integrate `main`'s new work:

- **Merge** (§4.3): create a merge commit M joining the two lines. History keeps the diamond — it's *truthful* (it really happened in parallel) but the graph gets bushy with many branches.
- **Rebase**: **take your commits (C4, C5), set them aside, move to the tip of `main` (C6), then re-apply your commits one by one on top.** The result looks as if you'd started your work *from C6 all along*. History is **linear** — no merge commit, no diamond.

```
   Before:
       C1 ◄─ C2 ◄─ C3 ◄─ C6 ◄─ main
                  └─ C4 ◄─ C5 ◄─ feature

   git switch feature && git rebase main   →  replay C4,C5 onto C6 as NEW commits C4',C5':
       C1 ◄─ C2 ◄─ C3 ◄─ C6 ◄─ C4' ◄─ C5' ◄─ feature
                              (main still at C6)
   Linear! But note: C4',C5' are NEW commits with NEW hashes — the originals are rewritten.
```

The crucial detail: **rebasing creates new commit objects** (C4', C5' have different hashes than C4, C5) because a commit's hash depends on its parent, and you changed the parent. The old commits become unreferenced (recoverable via reflog until garbage-collected). *This is why rebase "rewrites history."*

### 6.2 Basic rebase usage **[I]**

```bash
git switch feature
git rebase main               # replay feature's commits on top of main's tip

# If a conflict occurs DURING a rebase, it pauses at the offending commit:
#   1. resolve the conflict in the files (same markers as a merge, §4.4)
#   2. git add <resolved files>
#   3. git rebase --continue       # proceed to the next commit
# Other controls mid-rebase:
git rebase --skip                 # drop the current commit and continue (rare; you lose it)
git rebase --abort                # bail out entirely, restoring the pre-rebase state

# Rebase onto a specific commit, or move a branch onto another base:
git rebase --onto newbase oldbase feature   # advanced: transplant feature's commits
```

### 6.3 THE GOLDEN RULE OF REBASING **[I, memorize this]**

> **Never rebase commits that exist outside your local repository — i.e. never rebase history you've already pushed/shared.**

Here's *why*, precisely. Rebasing replaces commits with new ones (new hashes). If you rebase commits that a teammate has already pulled, the history they have and the history you now have **diverge** — your C5' is a different object than their C5, even though it's "the same change." When you force-push your rewritten branch, their next pull sees conflicting histories: duplicate-looking commits, painful merges, and confusion. You've effectively yanked the ground out from under everyone who based work on those commits.

So the rule in practice:
- **Rebase freely on your *own* local, un-pushed commits** — to clean them up before sharing. This is rebase's best use.
- **Once commits are pushed to a shared branch, treat them as immutable.** To change shared history, prefer `revert` (§7.4), which adds a *new* commit rather than rewriting.
- If you *must* rewrite a shared branch (e.g. a personal feature branch that only you use, or a coordinated cleanup), force-push with **`--force-with-lease`** and tell collaborators to re-sync (they should `git fetch` then reset their local branch to the remote).

**Merge vs rebase — when to use which:**

| Situation | Prefer | Why |
|---|---|---|
| Clean up YOUR local commits before pushing | **rebase -i** | tidy, linear, no one else affected |
| Update your feature branch with latest `main` (local-only) | **rebase** (or pull --rebase) | avoids noisy "merge main into feature" commits |
| Integrating a finished feature into shared `main` | **merge** (often `--no-ff`) | preserves true history; safe; non-destructive |
| Undoing a commit on a shared branch | **revert** | adds a new commit; never rewrites shared history |
| Team wants a strictly linear `main` | **rebase** workflow + squash-merge PRs | consistent, bisect-friendly history |

### 6.4 Interactive rebase — `rebase -i` (the power tool) **[I/A]**

Interactive rebase lets you **rewrite a series of your own commits**: reorder them, combine them, edit their messages, edit their contents, or drop them. This is how you turn a messy local branch ("wip", "fix typo", "actually fix it") into a clean, reviewable set of commits *before* you push.

```bash
git rebase -i HEAD~4          # interactively rewrite the LAST 4 commits
git rebase -i main            # rewrite every commit on this branch that isn't on main
```

Git opens an editor listing the commits **oldest-first**, each prefixed with a command word you can change:

```text
pick   a1b2c3 Add login form
pick   d4e5f6 wip
pick   g7h8i9 fix typo in login
pick   j1k2l3 Add password validation

# Commands (change "pick" to one of these):
#   p, pick    = use the commit as-is
#   r, reword  = use the commit, but EDIT its message
#   e, edit    = pause AT this commit so you can amend its CONTENT (split it, fix it)
#   s, squash  = MERGE this commit into the PREVIOUS one; combine both messages
#   f, fixup   = like squash, but DISCARD this commit's message (keep the previous one)
#   d, drop    = DELETE this commit entirely
#   (reorder lines to reorder commits; delete a line = same as drop)
```

To clean up the example — squash the "wip"/"typo" noise into the real commits and reword:

```text
pick   a1b2c3 Add login form
fixup  d4e5f6 wip                 # fold "wip" into "Add login form", drop its message
fixup  g7h8i9 fix typo in login   # fold the typo fix in too
reword j1k2l3 Add password validation   # save, then editor reopens to fix this message
```

Save and close. Git replays the commits applying your instructions. The result: two clean commits instead of four. The workflow for each command:

```bash
# After choosing "edit" for a commit, Git stops there. You can:
git commit --amend            # change its content/message
#   ...or split it: `git reset HEAD^` to unstage its changes, then make 2+ smaller commits
git rebase --continue         # resume once you're done at that commit

# A super-common pattern: make a fix for an earlier commit, then auto-squash it later.
git commit --fixup=a1b2c3     # creates a commit marked "fixup! Add login form"
# ...later:
git rebase -i --autosquash main   # automatically reorders & marks fixup commits to fold in
```

> **Best practice:** rebase-i your branch into logical, atomic commits *just before* opening a Pull Request. Reviewers thank you, and `git bisect` (§9.2) and `revert` work far better on clean, single-purpose commits.

### 6.5 Pull with rebase **[I]**

```bash
git pull --rebase             # fetch, then replay YOUR local commits on top of the remote's
                              #   new commits — instead of creating a merge commit.
                              #   Keeps your branch linear. Great for personal branches.
git config --global pull.rebase true   # make --rebase the default for `git pull` everywhere
```

`git pull --rebase` is the everyday way to "catch up to the team's latest without polluting history with merge commits." Since it only rebases *your local, un-pushed* commits, it respects the golden rule.

---

## 7. Undoing Things — restore, reset, revert, amend, reflog

This is one of the most practically valuable sections. "I made a mistake — how do I undo it?" has *many* answers in Git because there are many *kinds* of mistake and many *areas* (working dir, index, history). The key is matching the right tool to the situation and knowing **what's safe** (doesn't lose work / doesn't rewrite shared history) versus **what's destructive**. We go from gentlest to most forceful.

### 7.1 `git restore` — discard working/staged changes (modern) **[I]**

`git restore` undoes changes to *files* (it does not touch commit history). It replaced the file-related half of `checkout`. Two main jobs:

```bash
# UNSTAGE a file (move it out of the staging area, keep your edits on disk):
git restore --staged file.txt     # = git reset HEAD file.txt (old way). Edits stay in working dir.

# DISCARD working-directory edits (revert a file to its last-committed state):
git restore file.txt              # DESTRUCTIVE: your unsaved-to-git edits are GONE. No undo!
git restore .                     # discard ALL unstaged changes in the tree — be careful

# Restore a file as it was in a specific commit (without changing history):
git restore --source=HEAD~2 file.txt   # bring back the version from two commits ago
git restore -s abc123 -- path/         # from a specific commit

# Old checkout equivalents (still seen everywhere):
git checkout -- file.txt          # = git restore file.txt  (discard edits)
git reset HEAD file.txt           # = git restore --staged file.txt  (unstage)
```

> **Gotcha:** `git restore file.txt` *permanently* throws away your uncommitted edits to that file — there's no reflog for un-committed work. Pause before running it. If you might want the changes later, `git stash` (§8) instead.

### 7.2 `git reset` — moving the branch pointer (--soft / --mixed / --hard) **[I, learn precisely]**

`git reset` is the powerful, sometimes-scary one. Mentally, `reset <commit>` does up to three things in order, and the flag controls *how far* it goes:

1. **Move the current branch pointer** to `<commit>` (always happens). This is the "rewind history" part — commits after `<commit>` are no longer referenced by this branch.
2. **Update the staging area** to match `<commit>` (unless `--soft`).
3. **Update the working directory** to match `<commit>` (only with `--hard`).

So the three modes, from gentlest to most destructive:

```bash
# --soft: move the branch pointer ONLY. Staging area and working dir UNTOUCHED.
#   Effect: the commits' changes are now STAGED, ready to re-commit. Nothing lost.
git reset --soft HEAD~1       # "undo the last commit but keep all its changes staged"
#   ^ THE classic "I want to redo my last commit / combine it" move.

# --mixed (the DEFAULT): move the pointer AND reset the staging area. Working dir UNTOUCHED.
#   Effect: changes from undone commits become UNSTAGED edits in your working dir. Nothing lost.
git reset HEAD~1              # "undo the last commit; changes return as unstaged edits"
git reset --mixed HEAD~1      # explicit form of the same thing
git reset                     # with no commit: just UNSTAGE everything (keep edits) = unstage-all

# --hard: move the pointer, reset staging AND working dir to match the target commit.
#   Effect: DESTRUCTIVE — uncommitted changes are DELETED, and the undone commits are
#   discarded from this branch. Use only when you truly want to throw work away.
git reset --hard HEAD~1       # "blow away the last commit AND any uncommitted changes"
git reset --hard origin/main  # "make my branch IDENTICAL to origin/main, discarding local work"
```

A table to keep these straight — *what each mode keeps*:

| Mode | Branch pointer | Staging area | Working dir | Use when |
|---|---|---|---|---|
| `--soft` | moved | kept (changes staged) | kept | redo/combine last commit(s); nothing at risk |
| `--mixed` (default) | moved | reset (changes unstaged) | kept | uncommit + re-pick what to stage; nothing at risk |
| `--hard` | moved | reset | **reset (wiped)** | genuinely discard commits AND edits — destructive |

> **Safety:** Even after `git reset --hard`, the *committed* work isn't truly gone for ~90 days — `git reflog` (§7.6) still knows the old commit hash, and you can `git reset --hard <that-hash>` to get it back. But *uncommitted* changes destroyed by `--hard` are gone for good (no reflog for the working dir). So: commit (or stash) before a `--hard` if there's any doubt.

> **Golden-rule reminder:** `reset` *rewrites the current branch's history* (it drops commits). That's fine on local, un-pushed commits. To undo something already pushed/shared, use **`revert`** (§7.4), not `reset`.

### 7.3 `git commit --amend` — fix the last commit **[I]**

`--amend` replaces your most recent commit with a new one (new hash). Use it to fix a typo in the message or to include a file you forgot to stage — *before* you've pushed.

```bash
git commit --amend                 # edit the last commit's message (opens editor)
git commit --amend --no-edit       # keep the message; just fold in newly-staged changes
git add forgotten-file.js          # stage what you forgot...
git commit --amend --no-edit       # ...and tuck it into the previous commit silently
```

Because amend rewrites the commit, the golden rule applies: amend only un-pushed commits, or you'll have to force-push and disrupt others.

### 7.4 `git revert` — the safe undo for shared history **[I, important]**

`git revert` undoes a commit by **creating a NEW commit that applies the inverse changes.** It does *not* rewrite history — the original commit stays in the log, and a new "Revert ..." commit cancels its effect. Because it only *adds* a commit, it's **safe on shared/pushed history** — everyone can pull it normally.

```bash
git revert abc123             # create a new commit that undoes the changes in commit abc123
git revert HEAD               # undo the most recent commit (safely, as a new commit)
git revert HEAD~2..HEAD       # revert a RANGE of commits (creates several revert commits)
git revert -n abc123          # -n / --no-commit: stage the inverse changes but don't commit yet
                              #   (lets you revert several and bundle into one commit)
git revert --abort            # if a revert hits conflicts and you want out
```

**`reset` vs `revert` — the decision:**
- **`reset`** *removes* commits by moving the branch pointer back. Rewrites history. **Use only on local, un-shared commits.**
- **`revert`** *adds* a counter-commit. Preserves history. **Use on shared/pushed commits** (and any time you want an auditable "we undid this" record).

```text
   reset --hard HEAD~1:   C1 ◄─ C2 ◄─ C3   →   C1 ◄─ C2          (C3 gone from branch)
   revert HEAD:           C1 ◄─ C2 ◄─ C3   →   C1 ◄─ C2 ◄─ C3 ◄─ C3⁻¹  (new commit cancels C3)
```

### 7.5 `git clean` — remove untracked files **[I]**

`reset --hard` doesn't touch *untracked* files (Git doesn't manage them). To delete untracked clutter (stray build artifacts, generated files), use `clean`. It's destructive — these files aren't in Git, so they're unrecoverable. **Always dry-run first.**

```bash
git clean -n                  # DRY RUN: list what WOULD be deleted (n = no-op). ALWAYS do this first.
git clean -f                  # actually delete untracked FILES (f = force; required to act)
git clean -fd                 # also delete untracked DIRECTORIES (d)
git clean -fdx                # ALSO delete ignored files (x) — e.g. wipe node_modules, build/.
                              #   Very destructive: makes the tree pristine. Be sure.
git clean -fdi                # INTERACTIVE: choose what to delete (i)
```

### 7.6 `git reflog` — the safety net that saves your bacon **[I/A, lifesaver]**

This is the most reassuring command in Git. The **reflog** records **every time HEAD (or a branch tip) moves** — every commit, checkout, reset, rebase, merge, amend — in your *local* repo, with the previous hash, for ~90 days by default. So even when you "lose" commits (a bad `reset --hard`, a botched rebase, a deleted branch, a detached-HEAD orphan), **the old commit hash is still in the reflog**, and you can jump right back to it.

The mental model: branches and HEAD move around, but the reflog is a journal of *every position they've ever held*. Nothing committed is truly lost until garbage collection runs (and that respects the reflog window).

```bash
git reflog                    # show the movement history of HEAD: each line has a hash + action
#   e.g.:  a1b2c3 HEAD@{0}: reset: moving to HEAD~1
#          d4e5f6 HEAD@{1}: commit: Add password validation   ← the commit I "lost"!
git reflog show main          # reflog for a specific branch's tip

# Recover: I did `git reset --hard HEAD~1` and want that commit back —
git reset --hard d4e5f6       # jump the branch back to the lost commit (by hash from reflog)
git reset --hard HEAD@{1}     # or by the reflog position (HEAD@{1} = "where HEAD was 1 move ago")

# Recover a DELETED branch (you ran `git branch -D feature` by mistake):
git reflog                    # find the last commit that was on `feature` (its hash)
git switch -c feature <hash>  # recreate the branch pointing at that commit. Saved.
```

> **Remember the reflog exists.** Whenever you think "I just destroyed hours of work," the answer is almost always `git reflog`. The reflog is *local and per-clone* — it isn't pushed, so it only knows about movements *on this machine*.

---

## 8. Stashing

`git stash` **shelves your uncommitted changes** (working-dir + staged) so your working directory goes back to a clean state, *without committing*. You get the changes back later with `pop`/`apply`. It's a quick "save my half-done work aside" button.

### 8.1 When and why **[I]**

The classic scenario: you're mid-change on `feature` and suddenly need to switch to `main` to fix something urgent — but Git won't let you switch with conflicting uncommitted changes, and you're not ready to commit half-finished work. Stash it, switch, fix, switch back, un-stash. Other uses: pulling when you have local edits; quickly trying a clean build; reordering when you should-have-branched.

Stashes are stored as a stack (`stash@{0}` is newest). Each stash is actually a couple of hidden commits behind the scenes — so stashes are recoverable via reflog even after dropping (advanced).

### 8.2 Core stash commands **[I]**

```bash
git stash                     # shelve tracked, modified+staged changes; working dir goes clean
git stash push -m "wip auth"  # same, with a descriptive message (recommended)
git stash -u                  # ALSO stash UNTRACKED files (-u / --include-untracked)
git stash -a                  # ALSO stash ignored files too (-a / --all) — rare

git stash list                # list all stashes: stash@{0}, stash@{1}, ... with their messages
git stash show stash@{0}      # summary of what's in a stash
git stash show -p stash@{0}   # the full diff of a stash

git stash pop                 # re-apply the NEWEST stash AND remove it from the stack
git stash pop stash@{1}       # pop a specific stash
git stash apply               # re-apply the newest stash but KEEP it in the stack (reusable)
git stash apply stash@{2}     # apply a specific one without dropping it

git stash drop stash@{0}      # delete one stash without applying it
git stash clear               # delete ALL stashes (irreversible-ish — only reflog can help)

# Turn a stash into a branch (useful if it conflicts with current work):
git stash branch new-branch stash@{0}   # create a branch from the stash's base & apply it there
```

> **`pop` vs `apply`:** `pop` = apply *and* remove (the common choice). `apply` = apply but *leave it on the stack* (use when you want the same stash on multiple branches, or you're unsure it'll apply cleanly). If `pop` hits a conflict, it does *not* drop the stash (so you don't lose it) — resolve, then `git stash drop` manually.

> **Gotcha:** stashing is per-repo but *not* tied to a branch — you can pop a stash onto a *different* branch than where you made it (sometimes what you want, sometimes a surprise). Also, untracked files are **not** stashed unless you pass `-u`; forgetting this is a common "where did my new file go / why is it still here?" confusion.

---

## 9. Inspecting & Finding — blame, bisect, grep, pickaxe

Git's history isn't just for backup — it's a searchable database of *who changed what, when, and why*. These tools turn that history into a debugging superpower.

### 9.1 `git blame` — who last touched each line **[I]**

`blame` annotates every line of a file with the commit, author, and date that last changed it. It answers "who wrote this line and in which commit?" — invaluable for understanding *why* code exists (find the commit, read its message/PR).

```bash
git blame file.js             # annotate each line with commit/author/date
git blame -L 40,60 file.js    # only lines 40–60 (-L = line range) — focus your search
git blame -w file.js          # ignore whitespace-only changes (skip reformatting commits)
git blame -C file.js          # detect lines MOVED or COPIED from elsewhere (track true origin)
git blame -L 40,60 -- file.js HEAD~10   # blame as of an older commit
# Once you find the commit hash, dig in:
git show <hash>               # see the full change and message that introduced the line
```

> **Etiquette:** "blame" is a historical name — use it to *understand* code, not to assign fault. Many editors show this inline as "git lens"/"annotations."

### 9.2 `git bisect` — find the commit that introduced a bug by binary search **[I/A, powerful]**

This is one of Git's most underused gems. Suppose a bug exists *now* but worked at some older commit. Somewhere between "good" and "bad" a commit broke it. Rather than reading every commit, `bisect` does a **binary search**: you mark one known-good and one known-bad commit, and Git repeatedly checks out the *midpoint* and asks you "good or bad?" Each answer halves the search space — 1000 commits → ~10 checks.

```bash
git bisect start              # begin a bisect session
git bisect bad                # the CURRENT commit is bad (bug present)
git bisect good v1.2          # this older commit/tag was good (bug absent)
#   Git now checks out a commit halfway between. Test it (run the app / a test), then:
git bisect good               # ...if the bug is ABSENT at this commit
git bisect bad                # ...if the bug is PRESENT at this commit
#   Repeat. Git narrows down and finally prints:
#     "<hash> is the first bad commit"  — the exact commit that introduced the bug.
git bisect reset              # END the session: return to where you were before bisecting

# AUTOMATE it — let a test script decide good/bad at each step (exit 0 = good, non-0 = bad):
git bisect start HEAD v1.2    # bad=HEAD, good=v1.2 in one line
git bisect run npm test       # Git runs `npm test` at each midpoint automatically — hands-free!
git bisect run ./check.sh     # any command/script that returns 0 for good, non-zero for bad
```

`git bisect run` is magic: give it a command that exits 0 when the code is good and non-zero when bad, and Git finds the culprit commit completely automatically. Write a one-line repro script and let it churn.

### 9.3 `git grep` — search the working tree (and history) **[I]**

`git grep` searches *tracked* files — faster and more relevant than plain `grep` because it skips `.git`, ignored files, and `node_modules` (if ignored), and it can search *any commit*.

```bash
git grep "TODO"               # find "TODO" in tracked files of the working tree
git grep -n "loginUser"       # -n = show line numbers
git grep -i "error"           # -i = case-insensitive
git grep -c "import"          # -c = count matches per file
git grep "foo" -- "*.js"      # restrict to a path/glob
git grep "deprecated" v1.0    # search the files AS THEY WERE at tag/commit v1.0
git grep -e "cat" --or -e "dog"   # match either term
git grep -l "TODO"            # -l = just list the files that match
```

### 9.4 The pickaxe — `log -S` and `log -G` (find *when* code appeared/vanished) **[I/A]**

`grep` searches the *current* content. The **pickaxe** searches *history* for *when a string or pattern was introduced or removed*. This answers "when did this function get added?" or "which commit deleted this config line?" — extremely useful for archaeology.

```bash
git log -S "functionName"     # commits where the NUMBER of occurrences of "functionName"
                              #   changed — i.e. where it was ADDED or REMOVED. ("S" = string)
git log -S "API_KEY" -p       # ...and show the diffs, so you SEE the add/remove
git log -G "regex.*pattern"   # like -S but the argument is a REGEX, and it matches any diff
                              #   line that contains the pattern (broader than -S). ("G")
git log -S "secret" --all     # search across ALL branches (great for hunting leaked secrets)
git log --oneline -S "TODO"   # compact view of when TODOs came and went
```

**`-S` vs `-G`:** `-S` finds commits that change *how many times* a literal string appears (added or deleted) — precise for "when did this exact token enter/leave the code." `-G` matches commits whose diff contains a line matching a *regex* — broader, catches modifications too. Reach for `-S` first; escalate to `-G` for patterns.

---

## 10. Tags & Releases

A **tag** is a fixed, human-readable pointer to a specific commit — typically marking a release (`v1.0.0`). Unlike branches, tags **don't move**: once `v1.0.0` points at a commit, it stays there forever. Tags are how you say "this exact snapshot is what we shipped."

### 10.1 Lightweight vs annotated tags **[I]**

There are two kinds, and the distinction matters:

- **Lightweight tag:** just a name pointing at a commit — like a branch that never moves. No extra metadata. Fine for private/temporary bookmarks.
- **Annotated tag:** a full **tag object** in Git's database, storing the **tagger's name/email, date, a message, and (optionally) a GPG/SSH signature.** This is what you want for releases — it's a real, dated, attributable, verifiable record. Tooling (GitHub releases, `git describe`) expects annotated tags.

**Best practice: use annotated tags for anything you publish.**

```bash
# Annotated (recommended for releases) — -a, with a message via -m:
git tag -a v1.0.0 -m "Release 1.0.0 — first stable"
git tag -a v1.0.0 abc123 -m "..."     # tag a SPECIFIC older commit, not just HEAD

# Lightweight (a bare pointer, no metadata):
git tag v1.0.0-rc1

# Signed (annotated + cryptographic signature — proves authenticity, §14.5):
git tag -s v1.0.0 -m "Signed release 1.0.0"     # -s signs with your configured GPG/SSH key
git tag -v v1.0.0                                # VERIFY a signed tag's signature
```

### 10.2 Listing, viewing, deleting, pushing **[I, important gotcha]**

```bash
git tag                       # list all tags (alphabetical)
git tag -l "v1.*"             # list tags matching a pattern
git show v1.0.0               # show the tag (and, for annotated, the tagger + message + commit)

git tag -d v1.0.0             # delete a tag LOCALLY

# ⚡ CRUCIAL GOTCHA: tags are NOT pushed by `git push`. You must push them explicitly:
git push origin v1.0.0        # push ONE tag to the remote
git push origin --tags        # push ALL local tags
git push --follow-tags        # push commits AND any annotated tags reachable from them (cleaner)

# Delete a tag on the REMOTE (local delete doesn't propagate):
git push origin --delete v1.0.0
git push origin :refs/tags/v1.0.0     # older equivalent
```

> **⚡ Forgetting `--tags` is the #1 tag mistake.** You tag a release, push, and the tag isn't on GitHub because plain `push` ignores tags. Use `git push --follow-tags` habitually (and consider `git config --global push.followTags true`).

### 10.3 `git describe` and semantic versioning **[I]**

```bash
git describe --tags           # human name for the current commit relative to the nearest tag:
                              #   e.g. "v1.2.0-14-gab12cd3" = 14 commits after v1.2.0, at ab12cd3.
                              #   Great for embedding a version string in builds.
git describe --tags --always  # fall back to a bare hash if there are no tags
```

**Semantic Versioning (SemVer)** is the dominant tag-naming scheme: **`MAJOR.MINOR.PATCH`** (e.g. `2.4.1`). Bump **MAJOR** for breaking changes, **MINOR** for backward-compatible features, **PATCH** for backward-compatible bug fixes. Pre-releases append `-alpha.1`, `-rc.2`, etc. Tags should be the SemVer with a `v` prefix by convention (`v2.4.1`). Releases on GitHub are built *from* annotated tags, and CI release pipelines typically trigger on a `v*` tag push — see the **GitHub Actions / CI-CD guide** (`GITHUB_ACTIONS_GUIDE.md`) for tag-triggered release workflows.

---

## 11. Collaboration Workflows

Git gives you the *mechanics*; a **workflow** is the *agreement* your team makes about how to use them — which branches exist, how code gets reviewed, how it reaches production. The right workflow depends on team size, release cadence, and risk tolerance. This section covers the common ones and their trade-offs.

### 11.1 The feature-branch workflow **[I, the baseline]**

The near-universal foundation: **`main` is always deployable; all work happens on short-lived feature branches; changes merge back via review.** Nobody commits directly to `main`.

```bash
# 1. Start from an up-to-date main:
git switch main
git pull                      # get the latest

# 2. Branch for your task (name it descriptively — §14.3):
git switch -c feature/user-avatars

# 3. Work: commit in small, atomic steps:
git add -p && git commit -m "feat: add avatar upload endpoint"
# ...more commits...

# 4. Keep up with main as you go (rebase your local branch — golden rule OK, it's yours):
git fetch origin
git rebase origin/main        # replay your work on the latest main (resolve conflicts as needed)

# 5. Push and open a Pull Request (see 11.2):
git push -u origin feature/user-avatars

# 6. After the PR is approved & merged, clean up:
git switch main && git pull
git branch -d feature/user-avatars               # delete local branch
git push origin --delete feature/user-avatars    # delete remote branch (or let the PR do it)
```

### 11.2 Pull Requests — the review flow **[I, central to modern Git]**

A **Pull Request** (PR; "Merge Request"/MR on GitLab) is *not* a Git feature — it's a **platform feature** (GitHub/GitLab/Bitbucket) layered on top. It's a request to merge your branch into another (usually `main`), wrapped in a web UI for **code review**: teammates see the diff, leave line comments, request changes, and approve; CI runs automated checks (tests, lint, build); once approved and green, someone merges.

The PR lifecycle:
1. Push your feature branch to the remote.
2. Open a PR (`main` ← `feature/...`) with a clear title and description (what + why; link issues).
3. **CI runs** — automated tests/lint/build (configured via the **GitHub Actions / CI-CD guide**). PRs are where CI gates code.
4. Reviewers comment; you push more commits to address feedback (the PR auto-updates).
5. Approved + CI green → **merge**. Three merge strategies the platform offers:
   - **Merge commit** — keeps all your commits + adds a merge commit (full history).
   - **Squash and merge** — collapses the whole PR into *one* commit on `main` (clean, linear `main`; loses intermediate commits). Very popular.
   - **Rebase and merge** — replays your commits onto `main` with no merge commit (linear, keeps individual commits).
6. Delete the branch.

```bash
# Using GitHub's CLI (`gh`) to create and manage PRs from the terminal — handy and offline-ish:
gh pr create --base main --head feature/avatars --title "Add avatars" --body "Closes #42"
gh pr status                  # see your PRs and their CI/review state
gh pr checkout 123            # check out PR #123 locally to test/review it
gh pr merge 123 --squash      # merge a PR (squash strategy)
```

### 11.3 Forking workflow — contributing to repos you can't push to **[I]**

For open source (and any repo where you lack write access), you **fork**: make your own server-side copy of the repo, push to *your* fork, then open a PR from your fork back to the original ("upstream"). The convention is two remotes: **`origin`** = your fork, **`upstream`** = the original.

```bash
# After forking on GitHub, clone YOUR fork:
git clone git@github.com:you/repo.git
cd repo
git remote add upstream git@github.com:original/repo.git   # add the ORIGINAL as "upstream"
git remote -v                 # origin = your fork, upstream = the source

# Keep your fork in sync with upstream (do this before starting new work):
git fetch upstream            # get upstream's latest
git switch main
git merge upstream/main       # (or: git rebase upstream/main) — update your main
git push origin main          # push the updated main to your fork

# Then branch, work, push to origin (your fork), and open a PR to upstream.
```

### 11.4 Git Flow vs GitHub Flow vs trunk-based — trade-offs **[I/A]**

There are three archetypal models. Choosing well matters; over-engineering your branching is a real cost.

**GitHub Flow (simple, continuous delivery):** one long-lived branch (`main`, always deployable) + short-lived feature branches → PR → merge → deploy. Minimal ceremony. Best for web apps that deploy frequently. This is what most teams should use.

**Trunk-based development (high-velocity, CI-heavy):** everyone integrates into `main` ("trunk") very frequently — feature branches live hours to a day, or developers commit to `main` directly behind **feature flags**. Demands strong CI and a culture of small changes. Minimizes merge hell and "integration debt." Favoured by high-performing teams and large monorepos. Long-lived branches are the enemy.

**Git Flow (structured, scheduled releases):** a heavier model with multiple long-lived branches — `main` (production), `develop` (integration), plus `feature/*`, `release/*`, and `hotfix/*` branches with prescribed merge rules. Powerful for software with **explicit, versioned releases** and multiple maintained versions (e.g. installed/desktop software). But it's **overkill for most web apps** that deploy continuously — the extra branches add overhead and merge complexity without payoff.

| Workflow | Branches | Best for | Cost / downside |
|---|---|---|---|
| **GitHub Flow** | `main` + short feature branches | most web apps, continuous deploy | minimal; assumes good CI |
| **Trunk-based** | `main` (+ very short branches), feature flags | high-velocity teams, monorepos | needs strong CI + flags discipline |
| **Git Flow** | `main`, `develop`, feature/release/hotfix | versioned/desktop releases, multiple supported versions | complex; overkill for continuous deploy |

> **Guidance:** default to **GitHub Flow** or **trunk-based**. Adopt **Git Flow** only if you genuinely ship discrete, versioned releases and maintain several at once. The trend in 2026 is toward trunk-based + feature flags + strong CI.

---

## 12. Advanced — cherry-pick, submodules, worktrees, hooks, LFS

Power tools for specific situations. You won't use all of these daily, but knowing they exist (and their gotchas) saves you when the situation arises.

### 12.1 `git cherry-pick` — copy a specific commit onto your branch **[I/A]**

Cherry-pick **applies the changes from one specific commit onto your current branch as a new commit.** Use it to grab a single fix from another branch without merging the whole branch — e.g. backporting a hotfix from `main` to a release branch.

```bash
git cherry-pick abc123        # apply commit abc123's changes here as a new commit
git cherry-pick abc123 def456 # cherry-pick several commits in order
git cherry-pick abc123..def456    # a RANGE (exclusive of abc123)
git cherry-pick -x abc123     # -x: append "(cherry picked from commit abc123)" to the message
                              #   (good provenance — records where it came from)
git cherry-pick -n abc123     # -n: apply changes but DON'T commit (stage only; bundle later)
# Conflicts? Resolve like a merge, then:
git cherry-pick --continue    # proceed
git cherry-pick --abort       # bail out
```

> **Caution:** cherry-picking creates a *duplicate* of the change (new hash) on another branch. If both branches later merge, Git usually reconciles it, but excessive cherry-picking causes confusing "this commit appears twice" history. Prefer merging/rebasing when you want a whole branch; cherry-pick for surgical single-commit transplants.

### 12.2 Submodules — a repo inside a repo **[A, use with caution]**

A **submodule** embeds *another* Git repository inside yours at a fixed path, pinned to a *specific commit* of that other repo. Your repo doesn't store the submodule's files — it stores a *pointer* (the submodule's URL + the exact commit). Use case: you depend on another repo's source and want to control exactly which commit you build against (vendored libraries, shared components).

```bash
git submodule add https://github.com/user/lib.git libs/lib   # add a submodule at libs/lib
git commit -m "Add lib submodule"           # commits the .gitmodules file + the pinned commit

# Cloning a repo WITH submodules — they're empty by default! You must init+update:
git clone --recurse-submodules <url>        # clone AND pull all submodules in one go
# ...or, if you already cloned without it:
git submodule update --init --recursive     # populate submodules after the fact

# Update a submodule to a newer commit of ITS repo:
cd libs/lib && git fetch && git checkout <newer-commit> && cd ../..
git add libs/lib && git commit -m "Bump lib submodule"   # record the new pinned commit

git submodule update --remote               # update submodules to their tracked branch's latest
git submodule status                        # show each submodule's pinned commit
```

> **Why people avoid submodules.** They're notoriously easy to get wrong: forgetting `--recurse-submodules` leaves empty folders; the detached-HEAD nature of submodule checkouts confuses people; updates require a two-step "change inside, record outside" dance; and CI/teammates frequently end up on the wrong commit. They *work*, but the ergonomics are poor. Modern alternatives: a real **package manager** (npm/cargo/go modules), a **monorepo**, or **subtrees** (below). Reach for submodules only when you specifically need to pin and build another repo's *source* at an exact commit.

### 12.3 Subtrees — merge another repo's history into a folder **[A]**

`git subtree` is the main alternative to submodules. Instead of a pointer, it **copies another repo's content into a subdirectory of yours and merges its history in**, so the files are *really there* — clones just work, no special commands for collaborators. The trade-off is a more complex history and trickier two-way syncing.

```bash
# Add an external repo's contents into a subfolder, squashing its history:
git subtree add --prefix=libs/lib https://github.com/user/lib.git main --squash
# Pull updates from that external repo later:
git subtree pull --prefix=libs/lib https://github.com/user/lib.git main --squash
# Push your local changes back to the external repo:
git subtree push --prefix=libs/lib https://github.com/user/lib.git main
```

**Submodule vs subtree, in a nutshell:** submodule = *pointer* (lightweight repo, but fiddly UX, must init/update). Subtree = *copied content* (heavier history, but transparent to clones). Most teams that need vendoring pick subtree (or a package manager) over submodules.

### 12.4 Worktrees — multiple working directories from one repo **[A, underrated]**

A **worktree** lets you check out **multiple branches at once, in separate folders, sharing one `.git` database.** Normally a repo has one working directory and you switch branches in place (which forces you to stash/commit to switch). With worktrees, you can have `main` checked out in one folder and `feature` in another *simultaneously* — great for running two branches side-by-side, doing a hotfix without disturbing your in-progress feature, or building one branch while editing another.

```bash
git worktree add ../repo-hotfix hotfix/urgent   # create a NEW folder ../repo-hotfix with the
                                                #   hotfix/urgent branch checked out
git worktree add -b experiment ../repo-exp main # create branch "experiment" off main in a new dir
git worktree list                               # list all worktrees and their branches
git worktree remove ../repo-hotfix              # remove a worktree when done
git worktree prune                              # clean up stale worktree metadata
```

Worktrees share the object database, so they're cheap (no re-clone) and stay consistent. A branch can be checked out in only one worktree at a time. This is far nicer than juggling stashes when you genuinely need two branches live at once.

### 12.5 Hooks — run scripts on Git events **[A]**

**Hooks** are scripts Git runs automatically at certain points (before a commit, after a merge, before a push, etc.). They live in `.git/hooks/` as executable scripts named after the event. Use them to enforce standards locally: run a linter/tests before allowing a commit, validate commit-message format, prevent pushing to `main`, etc.

Key client-side hooks:
- **`pre-commit`** — runs *before* a commit is created; **exit non-zero to abort.** Most common: lint/format/test the staged changes. (Don't confuse with the *tool* named "pre-commit" — below.)
- **`commit-msg`** — runs with the message file as an argument; validate/enforce message format (e.g. Conventional Commits). Abort by exiting non-zero.
- **`pre-push`** — runs before a push; e.g. run the full test suite, or block pushing to protected branches.
- **`post-merge`, `post-checkout`** — run *after* the event; e.g. auto-install dependencies when `package-lock.json` changed.

```bash
# Hooks live here; samples are provided with a .sample suffix (rename to activate):
ls .git/hooks/                 # pre-commit.sample, commit-msg.sample, pre-push.sample, ...

# A minimal pre-commit hook (.git/hooks/pre-commit) — must be executable (chmod +x on Unix):
#!/bin/sh
# Block a commit if the linter fails:
npm run lint || { echo "Lint failed — commit aborted."; exit 1; }
```

> **Gotcha:** hooks in `.git/hooks/` are **local and NOT committed/shared** (they live in `.git`, which isn't versioned). To share hooks across a team you need a tool or `core.hooksPath`:

```bash
git config core.hooksPath .githooks   # point Git at a COMMITTED .githooks/ folder instead
```

**The hook ecosystem (what teams actually use):**
- **`pre-commit`** (the Python tool, pre-commit.com) — a framework managing multi-language hooks declaratively via a committed `.pre-commit-config.yaml`. Installs into `.git/hooks` for everyone consistently. Language-agnostic; very popular.
- **Husky** (Node/JS ecosystem) — sets up Git hooks from your repo, commonly paired with **lint-staged** (run linters only on staged files). The default for JS/TS projects.

```yaml
# .pre-commit-config.yaml (the pre-commit tool) — committed, so the whole team gets it:
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict     # block accidentally committed <<<<<<< markers
```

> Hooks run *locally* and can be bypassed (`git commit --no-verify`), so they're a convenience/early-warning, **not** a security boundary. The real gate is **server-side CI** (PR checks, branch protection) — see the **GitHub Actions / CI-CD guide**.

### 12.6 `sparse-checkout` — check out only part of a huge repo **[A]**

In a giant monorepo you often only need a subdirectory. **Sparse-checkout** populates your working directory with *only chosen paths* (the rest stays in the object DB but not on disk), making the working tree smaller and faster.

```bash
git sparse-checkout init --cone          # enable sparse mode ("cone" = simple dir-based, fast)
git sparse-checkout set apps/web libs/ui # only check out these directories
git sparse-checkout list                 # show current sparse paths
git sparse-checkout disable              # back to a full checkout
# Combine with a partial clone (below) for the leanest monorepo experience.
```

### 12.7 Partial & shallow clone — download less history/data **[A]**

For huge repos or CI, you often don't need *all* history or *all* file blobs.

```bash
# SHALLOW clone — only the latest N commits of history (no deep past):
git clone --depth 1 <url>          # just the latest commit — fast; common in CI builds
git fetch --unshallow              # later, deepen to the full history if needed

# PARTIAL clone — skip downloading file blobs until you actually need them (lazy fetch):
git clone --filter=blob:none <url>     # get commits/trees now; fetch blobs on demand
git clone --filter=tree:0 <url>        # even leaner (also defer trees); for tooling/CI
```

Shallow clones speed up CI dramatically (see the **GitHub Actions / CI-CD guide** — `actions/checkout` shallow-clones by default). Partial clones keep working with huge repos manageable.

### 12.8 Git LFS — large file storage **[A]**

Git is built for text and small files; committing large binaries (videos, datasets, PSDs, game assets) bloats every clone forever (since every clone holds all history). **Git LFS (Large File Storage)** replaces large files in the repo with tiny *pointer files* and stores the actual bytes on a separate LFS server, downloading them on checkout. This keeps the Git repo small.

```bash
git lfs install                    # one-time: set up LFS in your Git
git lfs track "*.psd"              # tell LFS to manage all .psd files (writes a .gitattributes rule)
git lfs track "assets/**/*.mp4"
git add .gitattributes            # COMMIT the .gitattributes so the team shares the tracking rules
git add design.psd && git commit -m "Add design (via LFS)"
git lfs ls-files                  # list files currently managed by LFS
git lfs pull                      # download LFS file contents for the current checkout
```

> **Gotcha:** LFS must track a file type *before* you commit those files, or the big bytes go into normal Git history (and removing them later requires history rewriting — §14.6). Set up `git lfs track` early.

### 12.9 `rerere` — reuse recorded conflict resolutions **[A]**

**`rerere`** = "**re**use **re**corded **re**solution." When enabled, Git remembers how you resolved a particular conflict and **auto-applies the same resolution** if that exact conflict shows up again. This is a big time-saver when rebasing a long-lived branch repeatedly or resolving the same merge conflict across many cherry-picks.

```bash
git config --global rerere.enabled true   # turn it on (it then records/replays automatically)
# Now, after you resolve a conflict once, Git silently reuses that resolution next time the
# identical conflict appears. `git rerere status` / `git rerere diff` inspect recorded states.
```

---

## 13. Fixing Mistakes & Recovery

A focused playbook for "oh no" moments. Most are fixable — the reflog (§7.6) is your safety net, and the rule of thumb is: **rewrite freely on local/un-pushed history, but on *shared* history, prefer additive fixes (`revert`) and coordinate any force-push.**

### 13.1 Detached HEAD — what it is and how to get out **[I]**

You enter **detached HEAD** when you check out a commit hash or tag directly instead of a branch (§2.5). HEAD points at a *commit*, not a *branch*. You can look around and even commit, but those commits belong to no branch — switch away and they're orphaned (recoverable via reflog, but easy to lose). Git warns you when it happens.

```bash
git switch --detach abc123    # intentionally detach to inspect an old commit
git switch -c rescue          # if you made commits here and want to KEEP them: branch now!
git switch main               # if you made NO commits: just go back to a branch — no harm

# If you already detached, committed, then switched away and "lost" the commits:
git reflog                    # find the orphaned commit's hash...
git switch -c rescue <hash>   # ...and rescue it onto a new branch
```

> **Rule:** if you ever find yourself committing in detached HEAD and want to keep the work, **create a branch right there** (`git switch -c name`) before switching away.

### 13.2 Recover a deleted branch **[I]**

```bash
git reflog                    # find the last commit the deleted branch pointed to
git switch -c recovered-branch <hash>   # recreate it
# (If you deleted it on the remote too, just re-push: git push -u origin recovered-branch)
```

### 13.3 Recover commits lost to `reset --hard` / a bad rebase **[I]**

```bash
git reflog                    # every position HEAD held — find the good one (e.g. HEAD@{4})
git reset --hard HEAD@{4}     # jump the branch back to that state
# (a botched interactive rebase? `git reflog` shows the pre-rebase tip; reset to it.)
```

### 13.4 Undo a *pushed* commit safely **[I, important]**

The commit is already on the shared remote, so **don't** rewrite history (golden rule). Use `revert` to add a counter-commit, then push normally — collaborators just pull it.

```bash
git revert <bad-commit-hash>  # create a new commit that undoes it
git push                      # push the revert — safe, no force, no disruption
```

If you *must* truly remove a pushed commit (e.g. a leaked secret — §14.6) and you've coordinated with everyone, you'll rewrite and force-push-with-lease (§13.6) — but `revert` is the default for "undo a shared mistake."

### 13.5 Split or reorder commits **[I/A]**

Use interactive rebase (§6.4). To **split** a commit, mark it `edit`; when Git stops there, un-commit its changes and re-commit them as multiple smaller commits:

```bash
git rebase -i HEAD~3          # mark the target commit as "edit", save
# Git pauses at that commit. Un-stage its contents, then make several commits:
git reset HEAD^               # move the commit's changes back to the working dir (un-commit)
git add part-one-files && git commit -m "feat: part one"
git add part-two-files && git commit -m "feat: part two"
git rebase --continue         # resume the rebase

# To REORDER commits: just rearrange the lines in the `git rebase -i` editor and save.
```

### 13.6 Force-push safely — `--force-with-lease` vs `--force` **[I/A, critical]**

After rewriting history (rebase/amend/squash) on a branch you've already pushed, the remote rejects a normal push (it's "non-fast-forward"). You need to overwrite the remote — but how you do it matters enormously.

- **`git push --force`** overwrites the remote branch **unconditionally**. If a teammate pushed commits since you last fetched, **`--force` silently destroys them.** Dangerous.
- **`git push --force-with-lease`** overwrites the remote **only if it still points where you last saw it** (your remote-tracking ref). If someone else pushed in the meantime, the remote has *moved*, the "lease" is broken, and Git **refuses** — protecting their work. You then fetch, integrate, and try again.

```bash
git push --force-with-lease   # SAFE force: aborts if the remote moved unexpectedly. Use THIS.
git push --force-with-lease origin feature   # explicit
git push --force              # AVOID — clobbers anything the remote gained since your last fetch
```

> **Rule:** never `--force`; always `--force-with-lease`. And only force-push branches that are yours/agreed-upon — **never** force-push shared branches like `main` without team coordination (many teams *protect* `main` server-side to forbid it entirely).

---

## 14. Best Practices

Mechanics get you working; practices keep a repository *healthy* over years and across a team. These are the habits that separate painful repos from pleasant ones.

### 14.1 Atomic commits **[I]**

A commit should be **one logical change** — self-contained, and ideally able to be reverted on its own without breaking anything. Don't mix a bug fix with a refactor with a new feature in one commit. Atomic commits make history readable, `git bisect` (§9.2) precise, `git revert` (§7.4) surgical, and reviews easier. Use the staging area and `git add -p` (§3.3) to compose them.

```bash
# Bad:  one commit "stuff" touching 12 unrelated files.
# Good: three focused commits —
git add auth.js && git commit -m "fix: reject expired tokens"
git add format/ && git commit -m "style: run formatter on the auth module"
git add feature/ && git commit -m "feat: add SSO login button"
```

### 14.2 Commit message conventions — Conventional Commits **[I, widely adopted]**

A consistent message format makes history scannable and **machine-readable** (enables auto-generated changelogs and automated SemVer bumps). The dominant standard is **Conventional Commits**: `type(optional scope): description`.

```text
feat: add password reset flow            # a new feature  → bumps MINOR
fix: handle null user in profile page    # a bug fix      → bumps PATCH
docs: update README install steps        # docs only
style: format with prettier              # formatting, no logic change
refactor: extract validation helper      # code change, no behaviour change
perf: memoize expensive selector         # performance
test: add cases for token expiry         # tests
build: bump webpack to 5                 # build system / deps
ci: cache node_modules in pipeline       # CI config
chore: update .gitignore                 # misc maintenance

# Breaking change → bumps MAJOR. Mark with "!" or a "BREAKING CHANGE:" footer:
feat!: drop support for Node 18
feat: change auth API
BREAKING CHANGE: the /login response shape changed; clients must update.
```

Tools like **commitlint** (often via a `commit-msg` hook, §12.5) enforce the format; **semantic-release**/**changesets** read these messages to automate versioning and changelogs in CI (see the **GitHub Actions / CI-CD guide**).

### 14.3 Branch naming **[I]**

Consistent branch names keep a busy remote navigable. A common convention: `type/short-description`, optionally with an issue number.

```text
feature/user-avatars
fix/login-redirect-loop
hotfix/payment-timeout
chore/upgrade-deps
docs/api-reference
feat/1234-add-export        # include the issue/ticket number for traceability
```

Use lowercase, hyphens (not spaces/underscores), and keep them short but descriptive. Avoid generic names like `test` or `mybranch`.

### 14.4 `.gitignore` and `.gitattributes` **[I]**

You met `.gitignore` (§3.5). Its sibling, **`.gitattributes`**, tells Git how to *treat* specific paths — line endings, diff/merge strategies, LFS tracking, export rules. Commit both so the whole team shares the rules.

```gitattributes
# .gitattributes — normalize line endings and control handling per file type:
* text=auto                  # let Git auto-detect text and normalize line endings to LF in repo
*.sh   text eol=lf           # shell scripts ALWAYS LF (even on Windows checkouts)
*.bat  text eol=crlf         # Windows batch files ALWAYS CRLF
*.png  binary                # never try to diff/merge or alter line endings on binaries
*.lock -diff                 # don't show diffs for noisy lockfiles (still tracked)
*.psd  filter=lfs diff=lfs merge=lfs -text   # route PSDs through Git LFS (§12.8)
```

`.gitattributes` is the *robust* way to control line endings across a mixed-OS team — more reliable than each developer's `core.autocrlf` (§14.7), because it lives in the repo and applies to everyone identically.

### 14.5 Signing commits (GPG / SSH) **[I/A]**

A commit's author field is just text — anyone can set `user.name`/`user.email` to impersonate you. **Signing** cryptographically proves a commit/tag really came from you. Platforms show a "Verified" badge for signed commits. Modern Git can sign with a **GPG key** *or* (simpler) an **SSH key** you already use.

```bash
# SSH signing (simplest if you already have an SSH key):
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true     # sign EVERY commit automatically
git config --global tag.gpgsign true        # sign tags too
git commit -S -m "msg"                       # sign a single commit explicitly (-S)
git tag -s v1.0 -m "msg"                      # sign a tag
git log --show-signature -1                   # verify the signature on a commit

# GPG signing (traditional):
git config --global user.signingkey <GPG-KEY-ID>
git config --global commit.gpgsign true
# (upload the public key to GitHub/GitLab so they can show "Verified")
```

### 14.6 Security — never commit secrets, and how to purge them **[I/A, critical]**

**Never commit secrets** (API keys, passwords, tokens, private keys, `.env` files). Because Git keeps history forever, a secret committed once is in *every clone's history* even if you delete it in a later commit — it's still reachable. Prevention first:

- Add secret files to `.gitignore` (`*.env`, `*.pem`, `secrets/`) **before** they exist.
- Use a `pre-commit` hook / scanner (e.g. **gitleaks**, **trufflehog**) to block commits containing secret-looking strings.
- Keep secrets in environment variables / a secrets manager, not in the repo.

If a secret *does* get committed, the only real fix is to **rewrite history to remove it from every commit** *and* **rotate (invalidate) the secret immediately** (assume it's compromised — it was, after all, pushed). Tools:

```bash
# git-filter-repo — the modern, recommended history-rewriting tool (NOT the old filter-branch):
pip install git-filter-repo                  # install once
git filter-repo --path secrets.env --invert-paths   # remove this file from ALL of history
git filter-repo --replace-text expressions.txt      # redact matching strings everywhere

# BFG Repo-Cleaner — a fast, simpler Java tool for the common cases:
#   java -jar bfg.jar --delete-files secrets.env
#   java -jar bfg.jar --replace-text passwords.txt   # replace listed secrets with ***REMOVED***

# After rewriting history you MUST force-push (coordinate with everyone first!):
git push --force-with-lease --all
git push --force-with-lease --tags
# Everyone else must re-clone or hard-reset their local copies (the old commits are gone).
# AND rotate the leaked credential — rewriting doesn't help if someone already grabbed it.
```

> **⚡ `filter-branch` is deprecated and dangerous** (slow, error-prone, and Git itself warns against it). Use **`git filter-repo`** (or **BFG**). Rewriting history changes every commit hash after the change point, so it's disruptive — do it deliberately and communicate.

### 14.7 Line endings — `core.autocrlf` (Windows-relevant) **[I]**

Windows uses **CRLF** (`\r\n`) for newlines; macOS/Linux use **LF** (`\n`). If a Windows dev commits CRLF and a Mac dev commits LF, you get noisy diffs ("the whole file changed!") and broken shell scripts. The fix is to **store LF in the repo** and let each OS use its native endings in the *working directory*.

```bash
# Windows: convert CRLF→LF on commit, LF→CRLF on checkout (so the repo stays LF):
git config --global core.autocrlf true

# macOS/Linux: convert any stray CRLF→LF on commit, but don't touch on checkout:
git config --global core.autocrlf input

# Disable conversion entirely (rely on .gitattributes instead — the robust team approach):
git config --global core.autocrlf false
```

> **Best practice:** prefer a committed **`.gitattributes`** (§14.4) with `* text=auto` plus explicit `eol=lf`/`eol=crlf` for specific file types. That enforces consistent endings for *everyone* regardless of their personal `core.autocrlf`, eliminating the "works on my machine" line-ending churn. `core.autocrlf` is a per-developer fallback; `.gitattributes` is the team-wide source of truth.

### 14.8 General hygiene **[I]**

- **Pull/rebase before you start work** each day so you build on the latest.
- **Push frequently** so your work is backed up on the remote and visible to teammates.
- **Keep branches short-lived** — long branches drift from `main` and merge painfully.
- **Review your own diff before committing** (`git diff --staged`) — catch debug prints, stray files, secrets.
- **Delete merged branches** to keep the remote tidy (§4.5).
- **Protect `main`** on the server (require PRs + passing CI + reviews; forbid force-push). Server-side rules are the real safety; local hooks are convenience.
- **One repo, one project** generally — or a deliberate **monorepo** with the tooling (sparse-checkout, partial clone, code owners) to support it.

---

## 15. Gotchas & Command Reference

### 15.1 Gotchas that bite everyone

- **`git add` snapshots the file *now*.** Edit after staging and you must `git add` again — the later edits aren't in the staged version. `git status` shows a file as both "staged" and "modified" when this happens.
- **`.gitignore` doesn't affect already-tracked files.** Ignoring only works on *untracked* paths. Already committed it? `git rm --cached file` to stop tracking (§3.5).
- **`git pull` can create surprise merge commits** on a shared branch. Prefer `git fetch` + look + integrate, or `pull --rebase`/`pull --ff-only`.
- **Tags aren't pushed by default.** Use `git push --follow-tags` or `--tags` (§10.2).
- **`git reset --hard` destroys uncommitted work irretrievably** (no reflog for the working dir). Stash or commit first. *Committed* work it drops is still in the reflog (§7.6).
- **`git restore file` (discard) is unrecoverable** for uncommitted edits — there's no undo.
- **Force-push (`--force`) can erase a teammate's commits.** Always `--force-with-lease` (§13.6).
- **The golden rule:** never rebase/amend/reset commits you've already pushed to a shared branch — use `revert` instead (§6.3, §7.4).
- **Detached HEAD commits are orphaned** when you switch away — branch them first (§13.1).
- **`ours`/`theirs` flips between merge and rebase** (§4.4). Trust the `<<<<<<< HEAD` marker, not the words.
- **Line endings** silently change whole-file diffs on mixed-OS teams — set `core.autocrlf`/`.gitattributes` (§14.7).
- **A merge conflict left unresolved** (markers still in the file) will commit the `<<<<<<<` markers into your code if you `git add` and commit without finishing. The `check-merge-conflict` hook (§12.5) guards against this.
- **`git checkout` is overloaded** — prefer `switch` (branches) and `restore` (files) to avoid foot-guns.
- **Stash doesn't include untracked files** unless you pass `-u` (§8.2).

### 15.2 Command reference table

| Goal | Command |
|---|---|
| Configure identity | `git config --global user.name/​user.email "..."` |
| New repo / copy a repo | `git init` / `git clone <url>` |
| See current state | `git status` (`-sb` short) |
| Stage changes | `git add <path>` / `-A` (all) / `-p` (interactive hunks) |
| Commit | `git commit -m "msg"` / `--amend` (fix last) |
| What changed (unstaged) | `git diff` |
| What's staged | `git diff --staged` |
| History | `git log --oneline --graph --all --decorate` |
| Inspect one commit | `git show <ref>` |
| List/create branch | `git branch` / `git switch -c <name>` |
| Switch branch | `git switch <name>` |
| Merge a branch | `git merge <branch>` (`--no-ff` to force a merge commit) |
| Abort a merge | `git merge --abort` |
| Delete branch | `git branch -d <name>` (`-D` force) |
| Add/list remotes | `git remote add origin <url>` / `git remote -v` |
| Safe sync (download only) | `git fetch` (`-p` to prune) |
| Sync + integrate | `git pull` (`--rebase` / `--ff-only`) |
| Upload commits | `git push` (`-u` first time) |
| Rebase onto latest | `git rebase main` |
| Clean up local commits | `git rebase -i <base>` |
| Safe force-push | `git push --force-with-lease` |
| Unstage a file | `git restore --staged <file>` |
| Discard working edits | `git restore <file>` |
| Undo last commit, keep changes | `git reset --soft HEAD~1` |
| Undo + unstage | `git reset HEAD~1` (mixed) |
| Discard commits + edits | `git reset --hard <ref>` (destructive) |
| Safe undo (shared history) | `git revert <commit>` |
| Delete untracked files | `git clean -fd` (`-n` dry-run first) |
| Recover lost work | `git reflog` then `git reset --hard <hash>` |
| Shelve work | `git stash` (`-u` incl. untracked) / `git stash pop` |
| Who changed a line | `git blame <file>` |
| Find the bug commit | `git bisect start/good/bad` (`run <cmd>`) |
| Search tracked files | `git grep "<text>"` |
| Find when code changed | `git log -S "<string>"` / `-G "<regex>"` |
| Tag a release | `git tag -a v1.0.0 -m "..."` then `git push --follow-tags` |
| Copy one commit | `git cherry-pick <hash>` |
| Multiple working dirs | `git worktree add <path> <branch>` |
| Remove secrets from history | `git filter-repo --invert-paths --path <file>` |
| Get help offline | `git help <cmd>` / `git <cmd> -h` |

---

## 16. Study Path & Build-to-Learn Projects

Git is learned by *using it on real work and recovering from real mistakes*. Read for the model, then drill the commands until they're muscle memory — and deliberately break things in a throwaway repo so recovery is familiar *before* you need it under pressure.

**Suggested order:**
1. **§1–3 (setup, the mental model, basics).** Do *not* skip §2 — re-read it until the three areas, snapshots-not-diffs, the object model, and "a branch is a pointer" feel obvious. Everything else rests on it.
2. **§4–5 (branching/merging, remotes).** Get fluent with `switch`, `merge`, conflict resolution, and especially the **fetch-vs-pull** distinction.
3. **§7 (undoing) + §13 (recovery) + §8 (stash).** The confidence section. Practise `reset` modes and `reflog` recovery deliberately.
4. **§6 (rebasing) + §11 (workflows).** Now you're collaborating like a pro: clean local history with `rebase -i`, integrate via PRs.
5. **§9–10, §12, §14 (finding, tags, advanced, best practices)** as you encounter the need.

**Build these to cement it (each targets specific sections):**
1. **A "mistakes sandbox" repo** — in a throwaway repo, deliberately: make messy commits then `rebase -i` to clean them (§6.4); `reset --hard` and recover via `reflog` (§7.6); create and resolve a merge conflict by hand (§4.4); delete a branch and bring it back (§13.2). The single best Git exercise. *Goal: never fear "breaking" Git again.*
2. **Solo project with discipline** — build any small app, but enforce: a feature branch per task, atomic commits, Conventional Commit messages, annotated release tags, a clean `.gitignore`/`.gitattributes`. Exercises §3, §4, §10, §14.
3. **Simulated team collaboration** — clone the same repo into two folders (or use two GitHub accounts/a fork). Have "Alice" and "Bob" edit overlapping code, push/pull, hit and resolve conflicts, and practise `fetch`-then-integrate vs `pull`. Exercises §5, §11.
4. **Open-source contribution** — fork a real project, sync with `upstream`, branch, commit cleanly, and open a real **Pull Request**. Exercises §11.2–11.3 end to end.
5. **Bisect a real bug** — introduce a bug deep in a project's history, then find it with `git bisect run` and an automated test. Exercises §9.2.
6. **Set up local + server-side guardrails** — add a `pre-commit` hook (or the pre-commit/Husky tooling) for lint + Conventional Commits, then wire CI to gate PRs. This bridges directly into the **GitHub Actions / CI-CD guide** (`GITHUB_ACTIONS_GUIDE.md`) — Git provides the events (push, PR, tag) that CI reacts to. Exercises §12.5, §14.2.

**Next steps after this guide:** dive into the **GitHub Actions / CI-CD guide** to automate testing, releases (triggered by tag pushes, §10), and deployment from the Git events you now understand. Then explore platform features that build on Git — branch protection rules, CODEOWNERS, required reviews, and release automation (semantic-release/changesets reading your Conventional Commits, §14.2).

---

*Part of the offline developer study library. Written for Git 2.45+ as of 2026, using the modern `main`/`switch`/`restore`/`--force-with-lease`/`filter-repo` conventions. When in doubt, `git help <command>` works fully offline. Cross-reference the GitHub Actions / CI-CD guide for automating the Git events covered here.*
