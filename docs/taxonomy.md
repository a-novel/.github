# Taxonomy

This document is the shared vocabulary of the [`a-novel`](https://github.com/a-novel) organization, the
**product**: its terms, its architecture, and the reasoning behind both. It covers the **services** that
make up the backend today; the client-facing **platforms** will join it as they land. The product runs on
[`a-novel-kit`](https://github.com/a-novel-kit), the shared platform of libraries and tooling it depends
on ([platform taxonomy](https://github.com/a-novel-kit/.github/blob/master/docs/taxonomy.md)). Each
repository extends this document with its own `CONTRIBUTING.md`.

The backend is a set of **isolated services**. Each owns one slice of the product and runs on its own, and
client code reaches them only through their **APIs**.

## Services and domains

A service is bounded by its **domain**, a cohesive area of the product. You draw a domain by the product
capability it serves, not by the technology behind it: take the features that always change together and
would make no sense apart, and that set is a domain. A service owns exactly one.

A service is not a single program. It is the whole bounded **environment** for its domain: the **APIs**
that expose it, the **jobs** that maintain it, and the **infrastructure** it runs on, whether a database,
a cache, or whatever the domain needs. The only way in is the API; nothing else reaches inside.

A service should be **self-contained**, able to serve its features from end to end on its own. Owning a
whole domain is what makes that possible.

> Split a domain across many small services instead, and you need a gateway or an orchestrator to stitch
> them back together, plus a sync bus (such as Kafka) to keep them aligned: one endpoint becomes a
> pipeline of services. Owning the domain keeps that endpoint a single, layered call.

So we prefer to start with **large services and split them later**, over small services we have to merge.
We sit closer to **macro-services** than to micro- or nano-services.

A **technical concern** earns its own service only when it is large enough _and_ shared across several
services. Such a service usually exposes **internal APIs** only: it exists to be called by other services,
not by external clients.

## Interacting with a service

A service is reached through its **API**, the request/response contract it publishes for callers. The API
is what a service promises the outside world; everything below
([Inside a service](#inside-a-service-layers-and-contracts)) is how it keeps that promise. Which protocols
carry the API is an [implementation detail](#implementation-details).

- A **client package** wraps an API for one programming language, an "API with benefits." It bundles the
  code to call the API, generated or written by hand, so callers need not re-declare the contract
  themselves. (The authentication service, for example, ships a ready-made permission helper.) It is not a
  second contract; it is the same API, bundled.
- A **job** is a one-off run that acts _on_ a service rather than serving a request (see [Jobs](#jobs)).

## Authentication and authorization

Most APIs are not open: a caller must prove **who it is** (authentication) and be allowed the operation it
asks for (authorization). The platform solves both once, with a small shared backbone, rather than
re-solving them per service.

- A **session** is a pair of JSON Web Tokens. A short-lived **access token** authorizes individual API
  calls; a long-lived **refresh token** mints a new pair when the access token expires, so a caller
  authenticates once and rolls its session without re-sending credentials. The **authentication service**
  owns identities and issues both.
- **Signing is centralized.** One service holds the keys and signs and verifies every token; no other
  service keeps a signing key, not even the one issuing them. That signer rotates its keys on a schedule
  and publishes the **public** half, so a token stays verifiable across a rotation while the private key
  never leaves it.
- **Authorization is local.** A token carries the caller's **role**, which maps to a set of
  **permissions**. A service does not call the authentication service on every request: it mounts that
  service's [client package](#interacting-with-a-service) as middleware, verifies the token against the
  signer's published public keys, and checks the route's required permission itself. A network round-trip
  happens only to refresh a session or change an identity, never just to authorize a call.

So the authentication service is reached as an [internal API](#services-and-domains) and consumed as a
client package, and the signer is another internal API behind it. A service's own `CONTRIBUTING.md` records
how it _uses_ this backbone (which permission guards which route), not how the backbone works.

## Inside a service: layers and contracts

A request enters through the API and is **dispatched down the service's layers in order**: each layer does
its part, then the response flows back up.

```
  caller ──► API ──► layer ──► layer ──► layer        request down, response back up
```

The **layer order is itself a contract**. Layers form a hierarchy: a layer answers to the one above it and
calls the one below it. Within a layer, components may also call each other **sideways** to share logic,
which the core does often. A layer is never skipped.

| Layer       | Role                                                                                                                                                                        |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | The **translation layer**: it parses an incoming, serialized API call into the internal types the layers below use, and turns their answer back into a serialized response. |
| **Core**    | The **logical layer**: it turns a domain feature into running code, calling the layers below as dependencies.                                                               |
| **DAO**     | The boundary to an **external storage source**: a database, a cache server, any store that lives outside the program's own memory.                                          |

Some modules sit outside the layer system, like configuration and local helpers; see
[implementation details](#implementation-details).

### Contracts: external and internal

Layers, and services, communicate only through **contracts**, of two kinds:

- An **external contract** (a service's API) is **serialized** into a standard format (OpenAPI, protobuf)
  so it can travel over a network nobody controls. A serialized value cannot be used as is: the receiver
  must **parse** it into native types first.
- An **internal contract** (between a service's layers) is expressed in the **implementation language**
  directly, usable without parsing but meaningless off the machine.

Internal contracts are shaped by **models**: the equivalent of an API's request and response, but in the
layer's own language.

### Layering rationale

Layers separate concerns by how controllable they are. A **low** layer holds simple logic but little
control over its surroundings: it sits against the uncontrolled outside, a wire format, a database, a
provider. A **high** layer holds the complex logic, but can **mock** the layers beneath it, and so
controls its whole environment in a test. Layering balances the two, leaving each layer testable in the way
that suits it: the low ones for their narrow logic, the high ones with everything below them mocked.

### A service is also a caller

A service does not only serve an API; it also **calls** others. Reaching an external dependency is **not
tied to a layer**: it is not part of the hierarchy, so any layer may do it where its work needs it, within
that layer's boundaries. Each call goes to another API, whether a sibling service, a database, or a mail
server, so a request's work carries on past the service's own boundary.

## State and data

A service holds two kinds of information.

- **Data** is what a service is _about_: the domain records it stores and serves. The service works on its
  data while serving requests; more data does not change how it behaves.
- **State** is the smaller set of facts that decide _how_ a service behaves and what it is capable of:
  configuration, a feature flag, a signing key, an elevated user, the schema version. It may sit in the
  database next to the data, or in memory. Changing data is the service doing its work; changing state
  changes the work itself.

## Jobs

A **job** is how a service is changed **out-of-band**, outside the request flow. Where an API call changes
data or state while serving a caller, a job does so with no caller involved:

- Its call is **intentional**: an operator or a schedule triggers it, not an external actor's request.
- It runs **once**, to completion, and exits.
- Its role is to **establish or maintain the ground a service runs on**: applying a schema **migration**,
  seeding or correcting state, rotating keys, running a scheduled cleanup.

Its boundary follows: a job never serves a request and never has a caller waiting on its result. If the
work answers to a caller, it belongs in the API, not a job.

## Runnable units

A service compiles to several small binaries, each a **target**: the unit built, started, and supervised
on its own. A target is either:

- a **server**, while it stays up to serve an API; or
- a **job** (see above), which runs once and exits.

## Implementation details

The concepts above are independent of the tools that implement them today. The current choices, and why
each:

- **[Go](https://go.dev/)** is the implementation language; the internal contracts between layers are Go
  interfaces and types.
- **[OpenAPI](https://www.openapis.org/)** describes the public REST API; **[gRPC](https://grpc.io/)** with
  **[protobuf](https://protobuf.dev/)** the internal one, and the proto file generates the gRPC server and
  its client.
- **[Postgres](https://www.postgresql.org/)**, through the **[bun](https://bun.uptrace.dev/)** query
  builder, backs the DAO; schema changes ship as ordered **migrations**.
- **[YAML](https://yaml.org/)** carries configuration.
- **[OpenTelemetry](https://opentelemetry.io/)** traces every layer.
- Services publish as container images to the
  **[GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)**
  for orchestration.

### Code structure

The layers above map to the same Go packages in every service, so a contributor moving between them finds
the same shape; only the domain resource differs:

- **`internal/dao`** — the DAO layer; one file per query (a `.go`, plus a `.sql` when the query is
  hand-written), built on bun.
- **`internal/core`** — the core layer; one file per feature. Each operation **owns the interface of the
  dependency it calls**: a `<Resource>Dao` Go interface, held in a `dao` field, while `internal/dao`
  provides the implementation. The interface lives with the **caller**, not the callee: the core depends on
  a contract it defines, never on the database package directly. This is what lets a high layer mock
  everything beneath it (see [Layering rationale](#layering-rationale)).
- **`internal/handlers`** — the handler layer; `http.*` files serve the REST API, `grpc.*` the gRPC one.
- **`cmd/<target>`** — one `main.go` per [runnable target](#runnable-units): a server or a job.
- **`pkg/go`, `pkg/js`** — the [client packages](#interacting-with-a-service), one per language.

Configuration and local helpers sit outside the layers. The Go interfaces between layers have their test
mocks **generated**, not written by hand. A service's own `CONTRIBUTING.md` only maps these packages to its
concrete resource and files; it does not restate the model.

## External resources

- [Developer onboarding guide](https://github.com/a-novel-kit/.github/blob/master/README.md) — the
  toolchain and the `a-novel` CLI.
- [Libraries, tooling & platform concepts](https://github.com/a-novel-kit/.github/blob/master/CONTRIBUTING.md)
  — the two-org model, repository kinds, libraries, and the CLI.
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) —
  the layering these services follow.
