# Service & architecture concepts

The shared vocabulary for every backend **service** in the
[`a-novel`](https://github.com/a-novel) organization — what each term means and how a service is
put together. Each service's `CONTRIBUTING.md` links here instead of redefining the model, so a word
means the same thing in every repo.

<!-- TODO(project-docs): this doc states the canonical vocabulary. Two renames it relies on
     (`internal/services/` → `internal/core/`, and `repository*` constructors → `dao*`) land as the
     per-service tasks of the taxonomy epic (a-novel/.github#133); until they merge, the live code
     still uses the old names. -->

It pairs with two other documents. The
[developer onboarding guide](https://github.com/a-novel-kit/.github/blob/master/README.md) covers
the toolchain and the `a-novel` CLI; the
[libraries, tooling & platform concepts](https://github.com/a-novel-kit/.github/blob/master/CONCEPTS.md)
cover the two-org model, repo kinds, the libraries, and the CLI's operational view. Read those for
the platform picture — this document is the service interior.

## What a service is

A **service** is one deployable backend microservice: a single repository named `service-*` that
owns one slice of the product's behaviour — user identities, signing keys, narrative state. It never
owns another service's slice; services collaborate by calling each other, not by sharing a database.

Every service is built in the same **clean-architecture** layering. A request crosses a fixed
sequence of layers, each depending only on the one beneath it, so the business rules never touch a
wire protocol or a SQL driver directly. That uniformity is the point: once you know one service, you
can find your way around any of them.

## The global picture

```
                    ┌─────────────────────── service-<name> ───────────────────────┐
   another service  │   server (a target)                                          │
   ──────────────►  │   ┌────────────┐     ┌────────┐     ┌─────┐                   │
   via client pkg   │   │  handler   │ ──► │  core  │ ──► │ DAO │ ──►  Postgres ◄───┼── infra
   (REST or gRPC)   │   │ (REST/gRPC)│     │ (logic)│     │(sql)│      (a dependency)│
                    │   └────────────┘     └────────┘     └─────┘                   │
                    │        ▲ import direction: handler → core → DAO               │
                    │                                                               │
                    │   migrations · bootstrap … = jobs (one-shot targets)          │
                    └───────────────────────────────────────────────────────────────┘
```

A service compiles to a handful of small binaries — **targets** — and keeps its state in
**Postgres**. Long-running targets are **servers** that expose an **API**; one-off lifecycle work
runs as **jobs**. Postgres and anything else the service merely depends on is **infra**. Inside a
server, a request flows down the **layers** — handler → core → DAO — and the answer flows back up.
Other services reach it through its published **client packages**.

## Runnable units

What a service _runs_ — and the difference between a process that serves traffic and one that does a
job and exits.

| Term            | What it is                                                                                                                                                 |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Target**      | One runnable binary, one per `cmd/<name>/` directory. The unit the CLI starts, stops, and supervises. A service has several.                               |
| **Target kind** | What that binary is _for_: a **server** or a **job**. Detected from whether it declares a container healthcheck (servers do; jobs don't).                  |
| **Server**      | A long-running target that exposes an **API** and runs until stopped (`cmd/rest`, `cmd/grpc`). The thing that answers requests.                            |
| **API**         | What a server exposes. Every server has one; its **protocol** is the technology that carries it — **HTTP/REST** (the public API) or **gRPC** (internal).   |
| **Job**         | A one-shot target that runs to completion and exits (`cmd/migrations`, bootstrap, key rotation). Lifecycle work, not traffic.                              |
| **Infra**       | A backing dependency a service needs but does not build — Postgres, an SMTP relay. The CLI brings it up _before_ the targets; it is never a target itself. |

The split that matters: a **server** is a kind of target, not a separate thing — and "REST" or
"gRPC" is a _protocol_, not a fourth concept. A service typically runs two servers (a public REST
API and an internal gRPC API) over the same core logic.

## The layer stack

Inside a server, every request crosses the same three layers in one direction — **handler → core →
DAO** — and never the reverse. Each layer depends only on the contract of the one below it, so you
can change the database without touching business rules, or add a second protocol without
duplicating logic.

| Layer       | Directory            | Responsibility                                                                                                                            |
| ----------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | `internal/handlers/` | The protocol boundary. Translates a REST or gRPC request into a core call and the result back into a response. Holds no business rules.   |
| **Core**    | `internal/core/`     | The business logic — validation, orchestration, the rules that define the service. Knows nothing about HTTP, gRPC, or SQL.                |
| **DAO**     | `internal/dao/`      | Data access. One file per operation (`pg.<entity><Op>.{go,sql}`), queries run through bun against Postgres. The only layer that does SQL. |

Two layers support the stack rather than sit in it:

- **Config** (`internal/config/`) — environment-driven configuration, loaded once at start-up.
- **Lib** (`internal/lib/`) — small service-local helpers (hashing, encryption) that don't warrant a
  layer of their own.

### Models and the boundaries between layers

A **model** is a struct representing one domain entity _at one layer_. The same entity exists as a
DAO model (bun column tags), a core model (no tags, the canonical form), and a handler model
(JSON or protobuf tags) — and small converter functions translate between them at each boundary.

This looks like repetition; it is insulation. The database schema, the business object, and the wire
format change for different reasons and on different schedules. Giving each its own struct means a
column rename never ripples into the public API, and a new API field never forces a migration.

## Contracts and artifacts

The fixed surfaces a service publishes — what other code (and other services) depend on.

| Term               | What it is                                                                                                                                                                  |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Migration**      | A timestamped SQL script in `internal/models/migrations/` that evolves the schema. Applied to completion by the `migrations` job before any server starts.                  |
| **Proto**          | The gRPC contract in `internal/models/proto/`. The source of truth for the internal API and the generated Go client.                                                        |
| **Client package** | The published SDK other services import: `pkg/go` speaks gRPC (internal callers), `pkg/js` speaks REST (web callers). A consumer never hand-rolls requests against the API. |

## Domain vs. the layers

**Domain** is the _problem space_ a service covers — "the authentication domain", "the signing-key
domain". It is emphatically **not** a layer or a directory: it is the subject matter the layers
implement. Keeping "domain" for the problem and **core** for the logic layer is deliberate — one
word, one meaning.

Each service grows **domain patterns**: ideas specific to its problem that a contributor must
understand before changing it — a token pair, a single-use short code, a master-key encryption
scheme, a validity-window view. These are named and explained once, in that service's own
`CONTRIBUTING.md`, not here — this document defines the _shared_ model; the per-service doc defines
its _domain_. When a pattern turns out to be reusable across services, it graduates into this
document.

## How this maps to the CLI

The `a-novel` CLI operates a service through exactly the terms above: it starts **targets**,
distinguishes **servers** from **jobs** by their healthcheck, and brings up **infra** first. The
operational detail — how targets are discovered, the compose contract, run modes — lives in the
[libraries, tooling & platform concepts](https://github.com/a-novel-kit/.github/blob/master/CONCEPTS.md),
defined where the CLI itself lives. This document is the source of truth for what those words _mean_;
that one is the source of truth for how the CLI _runs_ them.
