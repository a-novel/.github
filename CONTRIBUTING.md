# Contributing to a-novel services

The shared vocabulary behind every backend service in the
[`a-novel`](https://github.com/a-novel) organization — its terms, its architecture, and the
reasoning behind both. Each service specifies its own taxonomy in its `CONTRIBUTING.md`, a mere
extension of this document.

## Taxonomy

This section defines the concepts every service shares. It will grow to cover the wider contributor
process in time; for now it is the vocabulary.

### Services and domains

A service is bounded by its **domain** — a cohesive area of the product. You draw a domain by the
product capability it serves, not by the technology behind it: take the features that always change
together and would make no sense apart, and that set is a domain. A service owns exactly one.

A service is not a single program. It is the whole bounded **environment** for its domain: the
**APIs** that expose it, the **jobs** that maintain it, and the **infrastructure** it runs on — a
database, a cache, whatever the domain needs. Nothing reaches inside; other services speak only to
its APIs.

The goal is for a service to be **self-contained** — able to serve its features from end to end on
its own. Owning a whole domain is what makes that possible. Split a domain across many small services
instead, and you need a gateway or an orchestrator to stitch them back together, plus a sync bus
(such as Kafka) to keep them aligned: one endpoint becomes a pipeline of services. Owning the domain
keeps that endpoint a single, layered call. (A technical concern occasionally earns its own service,
but that is the exception.)

This is why our services sit closer to **macro-services** than to micro- or nano-services: a few
larger services, each shaped to a domain, are simpler to build and change than many tiny ones.

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
  [State, data, and jobs](#state-data-and-jobs).

### Inside a service: layers and contracts

A request enters through the API and is **dispatched down the service's layers in order** — each
layer does its part, then the response flows back up.

Separately, a layer may reach _past_ the service to a dependency it calls (the core, for example,
often sends mail). So a request's flow is not confined to the service; see
[A service is also a caller](#a-service-is-also-a-caller).

```
  caller ──API──►  layer ─►  layer ─►  layer        (request down, response back up)
                                  └──►  a dependency the service calls
```

The **layer order is itself a contract**. Layers form a hierarchy: a layer answers to the one above
it and calls the one below it. Calling sideways is allowed but discouraged — the core sometimes does
it to share logic — and skipping a layer is not.

| Layer       | Role                                                                                                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | The **translation layer**: it parses an incoming API call into the internal types the layers below use, and turns their answer back into a response. |
| **Core**    | The **logical layer**: it turns a domain feature into running code, calling the layers below as dependencies.                                        |
| **DAO**     | The boundary to an **external data source** — a database, a cache server. (Data the service keeps in memory is not its concern.)                     |

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

#### A service is also a caller

A service does not only serve an API; it **calls** others. Those calls go to plain APIs too — another
service (service to service), a database, a mail server. A service reaches each from whichever layer
needs it: the DAO for its data, the core for mail, and so on.

#### Layering rationale

The lower a layer sits, the closer it is to constraints the service does not control — a wire format,
a database's quirks, a provider's API. Concentrating those constraints in small, low-level layers
frees the layers above to follow only the contract they chose. **Testing** is one beneficiary:
because a layer depends only on a contract, a test can replace the layer below it and exercise the
upper layer with no live dependency.

### State, data, and jobs

A service holds two kinds of information, and the difference is what lets us draw the boundary of a
**job**.

- **Data** is what a service is _about_: the domain records it stores and serves. The service works
  on its data through the API, while serving requests. More data does not change how the service
  behaves — it is more of the same.
- **State** is the smaller set of facts that decide _how_ a service behaves and what it is capable
  of: configuration, a feature flag, a signing key, an elevated user, the schema version. State is
  set deliberately and changes rarely; it may sit in the database next to the data, or in memory.
  Changing data is the service doing its work; changing state changes the work itself.

Both can change two ways. **In-band**, an API request changes them while serving a caller — most
data, and some state, moves this way. **Out-of-band**, a **job** changes them with no caller
involved.

A **job** is an operation that no request drives. An operator or a schedule triggers it; it runs
once, to completion, and exits. Its role is to **establish or maintain the ground a service runs
on** — applying a schema **migration**, seeding or correcting state, rotating keys, a scheduled
cleanup. That is also its boundary: a job never serves a request and never has a caller waiting on
its result. If the work answers to a caller, it belongs in the API, not a job.

### Runnable units

A service compiles to several small binaries; each is a **target** — the unit built, started, and
supervised on its own. A target is a **server** while it stays up to serve an API, or a **job** (see
above) when it runs once and exits.

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
