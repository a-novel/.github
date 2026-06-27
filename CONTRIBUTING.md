# Contributing to a-novel services

The shared vocabulary every backend service in the
[`a-novel`](https://github.com/a-novel) organization is built on. It is the **spine**: a term is
defined here once, and every repository — through its own `CONTRIBUTING.md` — uses it with that one
meaning. A repository's own file covers what is specific to it; this document covers what they all
have in common.

## Services and domains

A **service** is one deployable backend that owns a coherent slice of the product, and nothing
outside it. Services never share a data store or reach into each other's internals; they collaborate
only through each other's published interfaces.

Where that slice is drawn is the decision that matters. a-novel does not pursue fine-grained
micro-services — splitting behaviour to the smallest technical unit and accumulating many tiny
repositories. Instead it groups behaviour by **domain**: a cohesive area of the product that is
naturally reasoned about and changed together. The result sits between a monolith and classic
micro-services — each service is independently deployable and bounded, but it is sized to a whole
domain, which keeps the services few and each one navigable.

A **domain** is therefore not an implementation detail; it is the unit that _defines a service's
boundary_. Choosing what a service contains is choosing which domain it owns: two capabilities share
a service when they share a domain, and separate when their domains do. Everything else in this
document — the layers, the contracts, the runnable pieces — exists to implement one domain cleanly.

Within that boundary every service is built the same way, following a
[clean architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html):
concerns are separated into layers that depend inward, so the rules of the domain never entangle
with a wire format or a storage engine. The uniformity is deliberate — learn one service and you can
navigate any of them.

## Interacting with a service

A service is reached three ways. They serve different callers and carry different guarantees, so
keeping them distinct is the first thing to get straight.

- **API** — the request/response interface a service exposes so callers can drive its behaviour. The
  API _is_ the contract; **REST** and **gRPC** are two protocols that carry it — a public HTTP/REST
  one for outside callers, an internal gRPC one for sibling services — not two different APIs. A
  long-running process, a **server**, hosts the API and answers requests for as long as it runs.
- **Jobs** — one-off executions that act _on_ a service rather than serve traffic: applying a schema
  change, seeding data, rotating a key, running a scheduled maintenance pass. A job changes the
  service's state and exits.
- **Client packages** — libraries a caller imports instead of assembling requests by hand. A client
  package _implements_ the service's API for one ecosystem (a Go package over gRPC, a JS package over
  REST); it is a **wrapper** over an existing contract, not a contract of its own.

```
  caller  ──►  API  ──►  server [ handler → core → DAO ]  ──►  data source
  caller  ──►  client package   (a wrapper over the service's API)
  trigger ──►  job              (a one-off run that changes the service's state)
```

## Inside a service: layers and contracts

Within a server, a request crosses a fixed stack of **layers**, each named by its **role** and each
depending only on the one beneath it.

| Layer       | Role                                                                                                                              |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Handler** | The request boundary. Translates an API call into a call on the logic below, and the result back into a response. Holds no rules. |
| **Core**    | The domain logic — the validation, rules, and orchestration that define what the service does. Independent of transport or store. |
| **DAO**     | The data boundary. Reads and writes the service's data through whatever source holds it. Holds no rules.                          |

Not everything in a service is a layer: configuration (read once at start-up) and small local
helpers are **support code** — they serve the layers without sitting in the request path or holding
domain rules.

A layer never reaches past its neighbour; it talks to it through a **contract**. Contracts come in
two kinds, and the difference is the point:

- **External contracts** are the API — the promise a service makes to others. Changing one is a
  cross-service event.
- **Internal contracts** are the interfaces between layers — one narrow interface per operation,
  passing **models** (plain structs) as the shared currency. Changing one is a local refactor.

Two artifacts version these contracts, pointing opposite ways. The external API is generated from a
**proto** definition — the source of truth for the gRPC surface and its client, facing outward to
callers. The store schema the DAO is written against evolves through ordered **migrations**, applied
before any server starts and facing inward to the store.

Separating concerns this way pays off most in **testing**. Because a layer depends only on a
contract, a test for an upper layer substitutes a stand-in for the layer below: core logic is
exercised without a live data source, awkward inputs — a fixed clock, a rare error — are trivial to
force, and only the DAO's own tests handle real data. Each layer is tested for exactly what it owns.

### Models across layers

A **model** is a struct for one domain entity at one layer — a storage-shaped model at the DAO, a
plain domain model in the core, a transport-shaped model at the handler — with small converters
between them. This is insulation: the schema, the domain object, and the wire format change for
different reasons and on different schedules, and a separate model at each boundary stops a change to
one from rippling into the others.

## State versus data

A service holds two kinds of information, and conflating them muddles its design. **Data** is the
bulk a service accumulates in the ordinary course of work — records that grow in number but rarely
change how it behaves. **State** is the smaller set of facts that change what the service _does_ or
_can do_: an elevated user, a secret a server keeps in hand to avoid recomputing it. State often
lives in fast local storage or an in-memory cache rather than the primary store, because it gates
behaviour and is read constantly. Most of what a service persists is data; state is the deliberate
exception.

## Runnable units

A service compiles to several small binaries. Each is a **target** — the unit built, started, and
supervised on its own. The term is defined apart from its uses so it stays stable: a target is _any_
runnable piece of a service, whatever that piece does.

Targets differ by what they are _for_:

- a **server** is a long-running target that hosts an API;
- a **job** is a target that runs once and exits.

The division is by lifecycle and purpose, not a closed list — the model holds whether a service has
two kinds of target or five.

A service also depends on things it does not build — a data store, a message broker, a mail relay.
These are **infra**. A service cannot run correctly without its infra, so the infra is always brought
up first, by whatever runs the service.

> Implementation: a target is one `cmd/<name>/` entry in a service repository; that repo's own
> `CONTRIBUTING.md` lists its targets.

## Technology choices

The concepts above are independent of the technologies that currently implement them — a DAO is
defined by its role, not by SQL — so concrete choices live here, not in the glossary. Recording them
in one place means adopting a new technology updates this list rather than a definition.

- **Postgres** — the default data store behind the DAO, accessed through the **bun** query builder.
- **OpenTelemetry** — distributed tracing threaded through every layer.

A service's protocol stack (its REST and gRPC servers) and anything domain-specific are listed in
that service's own `CONTRIBUTING.md`.

## External resources

- [Developer onboarding guide](https://github.com/a-novel-kit/.github/blob/master/README.md) — the
  toolchain and the `a-novel` CLI.
- [Libraries, tooling & platform concepts](https://github.com/a-novel-kit/.github/blob/master/CONTRIBUTING.md)
  — the two-org model, repository kinds, the shared libraries, and the CLI that runs all of this.
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
  — the layering these services follow.
