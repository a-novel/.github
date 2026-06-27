# Contributing to a-novel services

The shared vocabulary behind every backend service in the
[`a-novel`](https://github.com/a-novel) organization — its terms, its architecture, and the reasons
for both. Each service's own `CONTRIBUTING.md` builds on this; a concept defined here once means the
same thing in every repository.

## Services and domains

A service is bounded by its **domain** — a cohesive area of the product, drawn by what the service
is _about_ rather than by any technical concern. Identity, signing keys, and narrative state are
each a domain; a service owns exactly one.

A service is not a single program but the **whole bounded environment** for its domain. It bundles
the **APIs** that expose the domain, the background **jobs** that maintain it, and the
**infrastructure** it depends on — a database, a cache, whatever the domain needs. Nothing outside
reaches into that environment; other services speak only to its **APIs**.

We grow the platform by **adding domains, not by splitting technical concerns**. Capabilities that
belong to the same domain stay in one service, so the number of services tracks the breadth of the
product rather than the depth of its plumbing. That is deliberately closer to **macro-services**
than to micro- or nano-services: fewer, larger, domain-shaped environments are easier to reason
about and evolve than a sprawl of tiny ones.

## Interacting with a service

A service is reached through its **API** — the request/response contract it publishes for callers.
The API _is_ what a service promises the outside world; everything below
([Inside a service](#inside-a-service-layers-and-contracts)) is how it keeps that promise. Which
protocols carry the API is an [implementation detail](#implementation-details), covered separately.

- A **client package** is a thin wrapper around an API for one programming language — an "API with
  benefits". It spares every caller from re-declaring the same API contract in its own repository,
  and adds a little ergonomics on top. It is not a second contract: it is the same API, reached
  through generated code instead of hand-built requests.
- A **job** is a one-off run that acts _on_ a service rather than serving a request — applying a
  schema change, seeding data, a scheduled maintenance pass. A job changes the service's state and
  exits.

## Inside a service: layers and contracts

A request enters through the API and is **dispatched down the service's layers in order**; each
layer does its part and may dispatch further down, until — at the lowest layers — the request can
reach _past_ the service to an external provider, then flow back up as the response.

```
  caller ──API──►  layer ─►  layer ─►  layer ──►  external provider (itself an API)
                   └─────────────  response flows back up  ─────────────┘
```

The **layer order is itself a contract**. Layers form a strict hierarchy: a layer answers only to
the one directly above it and may call only the one directly below it — never sideways, never
skipping. That single rule is what keeps the architecture predictable.

| Layer       | Role                                                                                                                                                                                |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | The **translation layer**: it turns an external contract — a serialized API call — into the internal logical types the layers below understand, and the result back. No rules.      |
| **Core**    | The domain logic: the rules, validation, and orchestration that define the service. It depends only on the contract beneath it, never on how a request arrived or where data lives. |
| **DAO**     | The data boundary: it speaks to whatever holds the service's data.                                                                                                                  |

Not everything is a layer: **configuration** and small local **helpers** sit beside the stack as
support code (listed under [implementation details](#implementation-details)).

### Contracts: external and internal

Layers — and services — communicate only through **contracts**, and the two kinds differ on purpose:

- **External contracts** — a service's API — are written in **generic, serialized standards**
  (OpenAPI, protobuf) built to travel over a network nobody controls. Portable, but verbose.
- **Internal contracts** — between a service's own layers — are written in the **implementation
  language** itself (Go interfaces and types). Efficient and precise, but meaningless outside the
  service.

A **model** is part of a contract, not merely a struct: it is to a layer what a request/response is
to an API, expressed in the layer's own language. The **handler** is the layer that routinely
**converts** between models — translation is its whole job; the other layers pass their models
along plainly.

### A service is also a caller

The flow does not stop at the service boundary. A service is itself a **caller of other APIs**:
Postgres is an API for talking to a database, an SMTP server an API for sending mail. So a request
can **bleed past the boundary** to external providers — and not only from the DAO. The core may
reach an SMTP provider directly; any layer may call out where its work requires it. Think of those
providers as **internal APIs** the service indirectly proxies.

### Why the layering

The lower a layer sits, the closer it is to hardware and to constraints the service does not control
— a wire format, a database's quirks, a provider's API. Concentrating those constraints in tightly
scoped low-level layers **frees the upper layers** — above all the logic-heavy core — to obey only
the contract they chose. **Testing** is one beneficiary among many: because a layer depends only on
a contract, a test can mock the layer below it — exercising core logic with no live provider and
forcing rare conditions at will.

## State and data

**State** is any data that can be changed from _outside_ the service without touching its code — a
configuration value, a feature flag, a power-user grant, a cached secret. It usually lives in the
store behind the DAO, but it can extend past the DAO's reach (an in-memory cache is state too). The
test is external patchability: if changing it needs a code change, it is not state.

## Runnable units

A service compiles to several small binaries; each is a **target** — the unit built, started, and
supervised on its own. A target is a **server** while it stays up to serve an API, or a **job** when
it runs once and exits. The split is by lifecycle, not a fixed catalogue.

## Implementation details

The concepts above are independent of the tools that implement them today; the specifics live here,
so adopting a new one updates this section rather than the vocabulary.

- **APIs**: **OpenAPI / REST** for the public API, **protobuf / gRPC** for the internal one. The
  proto definition generates the gRPC server and its client; the store schema the DAO depends on
  evolves through ordered **migrations**.
- **Data**: **Postgres** behind the DAO, through the **bun** query builder.
- **Observability**: **OpenTelemetry** tracing, threaded through every layer.

## External resources

- [Developer onboarding guide](https://github.com/a-novel-kit/.github/blob/master/README.md) — the
  toolchain and the `a-novel` CLI.
- [Libraries, tooling & platform concepts](https://github.com/a-novel-kit/.github/blob/master/CONTRIBUTING.md)
  — the two-org model, repository kinds, the shared libraries, and the CLI.
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
  — the layering these services follow.
- [Go](https://go.dev/) — the implementation language; the inter-layer interface contracts are
  written in it.
- [OpenAPI](https://www.openapis.org/) and [gRPC](https://grpc.io/) — the API standards (public and
  internal).
- [YAML](https://yaml.org/) — service configuration.
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
  — where service images publish for container orchestration.
