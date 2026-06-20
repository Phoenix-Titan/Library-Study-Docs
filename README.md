# 📚 Developer Study Library & Workspace

A self-contained, **offline-first** collection of in-depth technical guides plus a real demo project. Everything here is written to be learned and built from **without an internet connection** — each guide has concepts, runnable commented examples, reference tables, gotchas, and a study path, accurate **as of 2026**.

> **How to use this:** All guides live in [`library/`](library/). Pick a track below, open the guide, and work top-to-bottom. Each guide ends with a "Study Path" and build-to-learn projects.

---

## 🗂️ The Guides (`library/`)

### Frontend — React Ecosystem
| Guide | What it covers | Lines |
|---|---|---|
| [React 19](library/REACT_19_GUIDE.md) | Core + all React 19 APIs (`use`, Actions, `useActionState`, `useOptimistic`, ref-as-prop, document metadata, Compiler), migration from 18 | 2686 |
| [Next.js 16](library/NEXTJS_16_GUIDE.md) | App Router, Server/Client Components, rendering, caching, Server Actions, SEO, performance | 803 |
| [TanStack Query v5](library/TANSTACK_QUERY_GUIDE.md) | Server-state: `useQuery`/`useMutation`, caching, invalidation, optimistic updates, Next.js hydration | 612 |
| [Zustand](library/ZUSTAND_GUIDE.md) | Client state: stores, selectors, middleware (persist/immer/devtools), slices, Next.js SSR pattern | 1452 |
| [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) | `useForm`/`register`, validation, `Controller`, zod resolvers, `useFieldArray`, shadcn + Next.js integration | 1889 |
| [Motion (animation)](library/MOTION_ANIMATION_GUIDE.md) | `motion` components, transitions, gestures, variants, `AnimatePresence`, layout & scroll animations, Next.js usage | 2384 |
| [Tailwind CSS (v4)](library/TAILWIND_CHEATSHEET.md) | Full utility cheatsheet, variants, arbitrary values, dark mode, common combos | 487 |
| [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) | Component-by-component reference, props, the Form pattern, theming | 677 |
| [Material UI (MUI)](library/MATERIAL_UI_GUIDE.md) | Theming, `sx`/`styled`, v6 Grid, component reference, DataGrid, Next.js setup, dark mode | 2365 |

### Mobile
| Guide | What it covers | Lines |
|---|---|---|
| [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md) | Expo Router, core components, SDK device APIs, EAS Build/Submit/Update, animations, shipping to stores | 2791 |

### Backend — Node.js
| Guide | What it covers | Lines |
|---|---|---|
| [NestJS](library/NESTJS_GUIDE.md) | Modules, DI, controllers, pipes/guards/interceptors/filters, validation + **custom DTO validators**, Prisma/TypeORM, JWT auth, Swagger, WebSockets, testing | 3071 |
| [Prisma ORM](library/PRISMA_ORM_GUIDE.md) | Schema, relations, migrations, full CRUD/query API, transactions, raw queries, seeding, Next.js singleton | 1875 |

### Backend — Go
| Guide | What it covers | Lines |
|---|---|---|
| [Go `net/http` REST API](library/GO_NET_HTTP_REST_API_GUIDE.md) | Go 1.22+ routing, handlers, middleware, full CRUD API, context, graceful shutdown, testing | 1964 |
| [Go Gin — REST API & File Upload](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) | Routing, binding/validation, middleware, CRUD, deep image-upload section, JWT, CORS, testing | 2130 |
| [Go Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) | Upgrader, two-goroutine model, ping/pong, full Hub chat server, JS + Go clients, scaling | 1565 |
| [Go ent ORM](library/GO_ENT_ORM_GUIDE.md) | Schema-as-code, fields/edges, codegen, CRUD, predicates, eager loading, transactions, hooks | 1897 |
| [Go JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) | Argon2id hashing (PHC, constant-time verify), golang-jwt v5, access/refresh tokens, auth middleware, RBAC | 1979 |

### Infrastructure & Protocols
| Guide | What it covers | Lines |
|---|---|---|
| [Docker & Docker Compose](library/DOCKER_GUIDE.md) | Images/containers, Dockerfiles, multi-stage builds, volumes, networking, Compose, real stacks, deployment | 802 |
| [FTP Server — Go & Node](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) | FTP/FTPS/SFTP, Go (`ftpserverlib`) + Node (`ftp-srv`) servers, SFTP skeletons, firewall/Docker, security | 1709 |

---

## 🧭 Suggested Learning Tracks

Pick the track that matches your goal. Within a track, follow the order listed.

**① Frontend (React ecosystem)**
1. [React 19](library/REACT_19_GUIDE.md) → 2. [Tailwind](library/TAILWIND_CHEATSHEET.md) → 3. [Next.js 16](library/NEXTJS_16_GUIDE.md) → 4. [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) → 5. [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) → 6. [Motion](library/MOTION_ANIMATION_GUIDE.md) → 7. [TanStack Query](library/TANSTACK_QUERY_GUIDE.md) → 8. [Zustand](library/ZUSTAND_GUIDE.md) → 9. [Material UI](library/MATERIAL_UI_GUIDE.md)

**② Full-stack with Node**
[Next.js 16](library/NEXTJS_16_GUIDE.md) → [NestJS](library/NESTJS_GUIDE.md) → [Prisma](library/PRISMA_ORM_GUIDE.md) → [Docker](library/DOCKER_GUIDE.md)

**③ Backend with Go**
[Go net/http](library/GO_NET_HTTP_REST_API_GUIDE.md) → [Gin + uploads](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) → [JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) → [ent ORM](library/GO_ENT_ORM_GUIDE.md) → [Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) → [Docker](library/DOCKER_GUIDE.md)

**④ Mobile**
[React 19](library/REACT_19_GUIDE.md) → [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md)

**⑤ Specialized**
[FTP Server (Go & Node)](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) — when you need a file-transfer service.

---

## ✅ Accuracy & Verification Notes

- **Written for 2026.** Fast-moving APIs are flagged inside each guide with **⚡ Version notes** (and **🆕 React 19** markers in the React guide). Always confirm exact package versions against official docs when you build.
- **Go code was syntax-verified.** All ~230 Go code blocks across the six Go guides were parsed with the Go toolchain (`gofmt`); one real bug (an invalid variadic parameter in the JWT guide) was found and fixed. Note: this verifies *syntax*, not full type-checking of examples that depend on third-party packages.
- **Security-critical code** (Argon2 params, JWT algorithm validation, file-upload validation) follows current best practices but should be reviewed against official guidance before production use.

---

*This library is a personal study + reference workspace. Each guide is standalone — start anywhere.*
