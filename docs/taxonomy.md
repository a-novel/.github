# Taxonomy

This document is the shared vocabulary of the [`a-novel`](https://github.com/a-novel) organization, the
**product**: its terms, its architecture and the reasoning behind both. It covers the **services** that
make up the backend. The product runs on [`a-novel-kit`](https://github.com/a-novel-kit), the shared
platform of libraries and tooling, which has
[its own taxonomy](https://github.com/a-novel-kit/.github/blob/master/docs/taxonomy.md). Each repository
extends this document with its own `CONTRIBUTING.md`.

The backend is a set of **isolated services**. Each owns one slice of the product and runs on its own, and
client code reaches them only through their **APIs**.

## A-Novel and Agora

**A-Novel** is the entity that owns the code. It spans both organizations and every service, library and
tool in them. **Agora** is the brand of the product family users see. Studio is its only product today,
and Storyverse will join it. Only those products and the copy their users read carry the Agora name:
pages, emails and the account. Everything else is A-Novel, and `a-novel` in identifiers.

A few names predate this rule and keep Agora for now:

- the logo, until A-Novel has its own;
- the `agora-storyverse` Google Cloud project and the `agorastoryverse.com` Google Workspace domain;
- the `agorastoryverse` social handles;
- the `agora-` prefix of production resource IDs, until their migration lands.

## Services and domains

A service is bounded by its **domain**, a cohesive area of the product. A domain follows the product
capability it serves, never the technology behind it. Take the features that always change together and
make no sense apart: that set is a domain. A service owns exactly one.

A service is not a single program. It is the whole bounded **environment** for its domain: the **APIs**
that expose it, the **jobs** that maintain it, and the **infrastructure** it runs on, whether a database,
a cache, or whatever the domain needs. The only way in is the API; nothing else reaches inside.

A service should be **self-contained**, able to serve its features from end to end on its own. Owning a
whole domain makes that possible.

> A domain split across many small services needs a gateway or an orchestrator to stitch them back
> together, plus a sync bus such as Kafka to keep them aligned. One endpoint becomes a pipeline of
> services. Owning the domain keeps that endpoint a single, layered call.

We therefore start with **large services and split them later**. Our services are closer to
**macro-services** than to micro- or nano-services.

A **technical concern** earns its own service only when it is large enough _and_ shared across several
services. Such a service usually exposes **internal APIs** only: it exists to be called by other services,
not by external clients.

## Interacting with a service

A service is reached through its **API**, the request/response contract it publishes for callers. The API
is the service's promise to the outside world; everything below
([Inside a service](#inside-a-service-layers-and-contracts)) is how it keeps that promise. The protocols
that carry the API are an [implementation detail](#implementation-details).

- A **client package** wraps an API for one programming language. It bundles the code to call the API,
  generated or written by hand, so callers need not declare the contract again. It can add helpers, such
  as the authentication service's permission middleware. It is the same API, bundled, and never a second
  contract.
- A **job** is a one-off run that acts _on_ a service rather than serving a request (see [Jobs](#jobs)).

## Authentication and authorization

Most APIs are not open. A caller must prove **who it is** (authentication) and be allowed the operation
it asks for (authorization). A small shared backbone solves both once for every service.

- A **session** is a pair of JSON Web Tokens. A short-lived **access token** authorizes individual API
  calls; a long-lived **refresh token** mints a new pair when the access token expires, so a caller
  authenticates once and rolls its session without re-sending credentials. The **authentication service**
  owns identities and issues both.
- **Signing is centralized.** One service, the signer, holds the keys and signs every token. No other
  service keeps a signing key, not even the one issuing tokens. The signer rotates its keys on a schedule
  and publishes their **public** half, so a token stays verifiable across a rotation while the private
  key never leaves the signer.
- **Authorization is local.** A token carries the caller's **role**, which maps to a set of
  **permissions**. A service mounts the authentication service's
  [client package](#interacting-with-a-service) as middleware. The middleware verifies the token against
  the signer's published public keys and checks the route's required permission. A network round-trip
  happens only to refresh a session or change an identity.

The authentication service is therefore consumed as a client package, and the signer is an
[internal API](#services-and-domains) behind it. A service's own `CONTRIBUTING.md` records how it uses
this backbone, such as which permission guards which route.

## Inside a service: layers and contracts

A request enters through the API and is **dispatched down the service's layers in order**: each layer does
its part, then the response flows back up.

```
  caller ──► API ──► layer ──► layer ──► layer        request down, response back up
```

The **layer order** is fixed. A layer answers to the one above it and calls the one below it, and no
layer is ever skipped. Components of one layer may also call each other **sideways** to share logic, as
the core often does.

| Layer       | Role                                                                                                                                                                       |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | The **translation layer**: it parses an incoming, serialized API call into the internal types the layers below use and turns their answer back into a serialized response. |
| **Core**    | The **logical layer**: it turns a domain feature into running code, calling the layers below as dependencies.                                                              |
| **DAO**     | The boundary to an **external storage source**: a database, a cache server, or any store that lives outside the program's own memory.                                      |

Some modules, such as configuration and local helpers, sit outside the layers.

### Contracts: external and internal

Layers and services communicate only through **contracts**, of two kinds:

- An **external contract** (a service's API) is **serialized** into a standard format (OpenAPI, protobuf)
  so it can travel over a network nobody controls. The receiver **parses** a serialized value into native
  types before using it.
- An **internal contract** (between a service's layers) is expressed in the **implementation language**
  directly, usable without parsing but meaningless off the machine.

Internal contracts are shaped by **models**: the equivalent of an API's request and response, but in the
layer's own language.

### Layering rationale

Layers separate concerns by how controllable they are. A **low** layer holds simple logic but little
control over its surroundings. It sits against the uncontrolled outside: a wire format, a database or a
provider. A **high** layer holds the complex logic and can **mock** the layers beneath it, so a test
controls its whole environment. Each layer is then testable in the way that suits it: a low layer for its
narrow logic, a high layer with everything below it mocked.

### A service is also a caller

A service also **calls** other APIs: a sibling service, a database or a mail server. These calls **sit
outside the layer hierarchy**. Any layer may make one where its work needs it, within that layer's
boundaries.

## State and data

A service holds two kinds of information.

- **Data** is what a service is _about_: the domain records it stores and serves. The service works on its
  data while serving requests; more data does not change how it behaves.
- **State** is the smaller set of facts that decide _how_ a service behaves and what it can do, such
  as configuration, a feature flag, a signing key, an elevated user or the schema version. State may sit
  in the database next to the data, or in memory. Changing data is the service doing its work. Changing
  state changes the work itself.

## Jobs

A **job** changes a service **out-of-band**, outside the request flow. An API call changes data or state
while serving a caller, and a job does so with no caller involved:

- Its call is **intentional**: an operator or a schedule triggers it, not an external actor's request.
- It runs **once**, to completion, and exits.
- Its role is to **establish or maintain the ground a service runs on**: applying a schema **migration**,
  seeding or correcting state, rotating keys, or running a scheduled cleanup.

A job never serves a request, and no caller waits on its result. Work that answers to a caller belongs
in the API.

## Runnable units

A service compiles to several small binaries, each a **target**: the unit built, started, and supervised
on its own. A target is either:

- a **server**, while it stays up to serve an API; or
- a **job** (see above), which runs once and exits.

## Implementation details

The concepts above hold whatever tools implement them. These are the current choices:

- **[Go](https://go.dev/)** is the implementation language; the internal contracts between layers are Go
  interfaces and types.
- **[OpenAPI](https://www.openapis.org/)** describes the public REST API; **[gRPC](https://grpc.io/)** with
  **[protobuf](https://protobuf.dev/)** the internal one, and the proto file generates the gRPC server and
  its client.
- **[Postgres](https://www.postgresql.org/)**, through the **[bun](https://bun.uptrace.dev/)** query
  builder, backs the DAO; schema changes ship as ordered **migrations**.
- **Environment variables** configure a deployment. Static policy, such as a permission map, ships
  embedded as **[YAML](https://yaml.org/)**.
- **[OpenTelemetry](https://opentelemetry.io/)** traces every layer.
- Services publish as container images to the
  **[GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)**
  for orchestration.

### Code structure

The layers map to the same Go packages in every service, so a contributor finds the same shape in each.
Only the domain resource differs.

- **`internal/dao`** — the DAO layer; one file per query (a `.go`, plus a `.sql` when the query is
  hand-written), built on bun.
- **`internal/core`** — the core layer; one file per feature. Each operation **owns the interface of the
  dependency it calls**: a `<Resource>Dao` Go interface, held in a `dao` field, while `internal/dao`
  provides the implementation. The core depends on a contract it defines, never on the database
  package, so a high layer can mock everything beneath it (see [Layering rationale](#layering-rationale)).
- **`internal/handlers`** — the handler layer; `rest.*` files serve the REST API, `grpc.*` the gRPC one.
- **`cmd/<target>`** — one `main.go` per [target](#runnable-units): a server or a job.
- **`pkg/go`, `pkg/js`** — the [client packages](#interacting-with-a-service), one per language.

The test mocks of the Go interfaces between layers are **generated**. A service's own `CONTRIBUTING.md`
maps these packages to its concrete resource and files.

## External resources

- [Developer guide](https://a-novel-kit.github.io/stack/) — the toolchain and the `a-novel` CLI.
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) —
  the layering these services follow.
