# Contributing to a-novel services

## Taxonomy

The shared vocabulary behind every backend service in the
[`a-novel`](https://github.com/a-novel) organization — its terms, its architecture, and the
reasoning behind both. Each service specifies its own taxonomy in its `CONTRIBUTING.md`, which acts
as an extension of this document.

The backend is a set of **isolated services**. Each owns one slice of the product and runs on its
own; client code reaches them only through their **APIs**.

### Services and domains

A service is bounded by its **domain** — a cohesive area of the product. You draw a domain by the
product capability it serves, not by the technology behind it: take the features that always change
together and would make no sense apart, and that set is a domain. A service owns exactly one.

A service is not a single program. It is the whole bounded **environment** for its domain: the
**APIs** that expose it, the **jobs** that maintain it, and the **infrastructure** it runs on — a
database, a cache, whatever the domain needs. The only way in is the API; nothing else reaches
inside.

A service should be **self-contained** — able to serve its features from end to end on its own.
Owning a whole domain is what makes that possible.

> Split a domain across many small services instead, and you need a gateway or an orchestrator to
> stitch them back together, plus a sync bus (such as Kafka) to keep them aligned: one endpoint
> becomes a pipeline of services. Owning the domain keeps that endpoint a single, layered call.

So we prefer to start with **large services and split them later**, over small services we have to
merge — and we sit closer to **macro-services** than to micro- or nano-services.

A **technical concern** earns its own service only when it is large enough _and_ shared across
several services. Such a service usually exposes **internal APIs** only: it exists to be called by
other services, not by external clients.

### Interacting with a service

A service is reached through its **API** — the request/response contract it publishes for callers.
The API is what a service promises the outside world; everything below
([Inside a service](#inside-a-service-layers-and-contracts)) is how it keeps that promise. Which
protocols carry the API is an [implementation detail](#implementation-details).

- A **client package** wraps an API for one programming language — an "API with benefits". It bundles
  the code to call the API (generated, or written by hand — the authentication service, for example,
  ships a ready-made permission helper) so callers need not re-declare the contract themselves. It is
  not a second contract; it is the same API, bundled.
- A **job** is a one-off run that acts _on_ a service rather than serving a request — see
  [Jobs](#jobs).

### Inside a service: layers and contracts

A request enters through the API and is **dispatched down the service's layers in order**: each layer
does its part, then the response flows back up.

```
  caller ──► API ──► layer ──► layer ──► layer        request down, response back up
```

The **layer order is itself a contract**. Layers form a hierarchy: a layer answers to the one above
it and calls the one below it. Within a layer, components may also call each other **sideways** to
share logic — the core does this often. A layer is never skipped.

| Layer       | Role                                                                                                                                                                        |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | The **translation layer**: it parses an incoming, serialized API call into the internal types the layers below use, and turns their answer back into a serialized response. |
| **Core**    | The **logical layer**: it turns a domain feature into running code, calling the layers below as dependencies.                                                               |
| **DAO**     | The boundary to an **external storage source** — a database, a cache server: any store that lives outside the program's own memory.                                         |

Some modules sit outside the layer system — configuration and local helpers; see
[implementation details](#implementation-details).

#### Contracts: external and internal

Layers — and services — communicate only through **contracts**, of two kinds:

- An **external contract** (a service's API) is **serialized** into a standard format (OpenAPI,
  protobuf) so it can travel over a network nobody controls. A serialized value cannot be used as is:
  the receiver must **parse** it into native types first.
- An **internal contract** (between a service's layers) is expressed in the **implementation
  language** directly — usable without parsing, but meaningless off the machine.

Internal contracts are shaped by **models**: the equivalent of an API's request and response, but in
the layer's own language.

#### Layering rationale

Layers separate concerns by how controllable they are. A **low** layer holds simple logic but little
control over its surroundings: it sits against the uncontrolled outside — a wire format, a database,
a provider. A **high** layer holds the complex logic, but can **mock** the layers beneath it, and so
controls its whole environment in a test. Layering balances the two, leaving each layer testable in
the way that suits it: the low ones for their narrow logic, the high ones with everything below them
mocked.

#### A service is also a caller

A service does not only serve an API — it **calls** others. Reaching an external dependency is **not
tied to a layer**: it is not part of the hierarchy, so any layer may do it where its work needs it,
within that layer's boundaries. Each call goes to another API — a sibling service, a database, a mail
server, any external dependency — so a request's work carries on past the service's own boundary.

### State and data

A service holds two kinds of information.

- **Data** is what a service is _about_: the domain records it stores and serves. The service works
  on its data while serving requests; more data does not change how it behaves.
- **State** is the smaller set of facts that decide _how_ a service behaves and what it is capable
  of: configuration, a feature flag, a signing key, an elevated user, the schema version. It may sit
  in the database next to the data, or in memory. Changing data is the service doing its work;
  changing state changes the work itself.

### Jobs

A **job** is how a service is changed **out-of-band** — outside the request flow. Where an API call
changes data or state while serving a caller, a job does so with no caller involved:

- Its call is **intentional**: an operator or a schedule triggers it, not an external actor's
  request.
- It runs **once**, to completion, and exits.
- Its role is to **establish or maintain the ground a service runs on** — applying a schema
  **migration**, seeding or correcting state, rotating keys, a scheduled cleanup.

Its boundary follows: a job never serves a request and never has a caller waiting on its result. If
the work answers to a caller, it belongs in the API, not a job.

### Runnable units

A service compiles to several small binaries, each a **target** — the unit built, started, and
supervised on its own. A target is either:

- a **server**, while it stays up to serve an API; or
- a **job** (see above), which runs once and exits.

### Implementation details

The concepts above are independent of the tools that implement them today. The current choices, and
why each:

- **[Go](https://go.dev/)** is the implementation language; the internal contracts between layers are
  Go interfaces and types.
- **[OpenAPI](https://www.openapis.org/)** describes the public REST API; **[gRPC](https://grpc.io/)**
  with **[protobuf](https://protobuf.dev/)** the internal one — the proto file generates the gRPC
  server and its client.
- **[Postgres](https://www.postgresql.org/)**, through the **[bun](https://bun.uptrace.dev/)** query
  builder, backs the DAO; schema changes ship as ordered **migrations**.
- **[YAML](https://yaml.org/)** carries configuration.
- **[OpenTelemetry](https://opentelemetry.io/)** traces every layer.
- Services publish as container images to the
  **[GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)**
  for orchestration.

## External resources

- [Developer onboarding guide](https://github.com/a-novel-kit/.github/blob/master/README.md) — the
  toolchain and the `a-novel` CLI.
- [Libraries, tooling & platform concepts](https://github.com/a-novel-kit/.github/blob/master/CONTRIBUTING.md)
  — the two-org model, repository kinds, libraries, and the CLI.
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
  — the layering these services follow.
