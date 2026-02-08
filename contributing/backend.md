# Backend

Backend services are written in Go. Each service owns a single business domain and exposes its API
over HTTP, gRPC, or both.

The guides below cover the architecture, conventions, and tooling shared across all backend services.

## Guides

1. [**Services**](./backend/1.service.md) — What a service is, how it differs from a microservice or
   monolith, and how repositories are organized.
2. [**Layers**](./backend/2.layers.md) — The three-layer architecture (DAO, Services, Handlers), clean
   architecture principles, and dependency rules.
3. [**Testing**](./backend/3.testing.md) — Table-driven unit tests, mocking with mockery, and handler
   testing for both HTTP and gRPC.
4. [**Error Handling**](./backend/4.errors.md) — Sentinel errors, error composition (`fmt.Errorf` vs
   `errors.Join`), and protocol-level error mapping.
5. [**Observability**](./backend/5.observability.md) — OpenTelemetry integration, span conventions,
   reporting helpers, and sensitive field masking.
6. [**Packaging**](./backend/6.packaging.md) — The `/pkg` public API surface, type re-exports, gRPC
   client wrappers, and adapters.
7. [**OpenAPI**](./backend/7.openapi.md) — Writing and publishing OpenAPI specs for HTTP services.
8. [**Integration Testing**](./backend/8.integration.md) — TypeScript REST client stubs, Zod validation,
   and end-to-end tests against a local service stack.
