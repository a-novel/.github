# Backend

This is generic documentation about backend development across A-Novel repositories.

## Stack

Backend services are written in Go.

Services may also export packages to interact with them. These packages can be written in Go, or any language that
may be used in the target environment. Usually, those are typescript packages for the browser clients.

Commonly used technologies:

- [Go](https://go.dev/): main business logic.
- [Postgres](https://www.postgresql.org/): generic-purpose database.
- [OpenTelemetry](https://opentelemetry.io/): distributed tracing and observability.
- [Podman](https://podman.io/): containers runtime.

## Layers

The backend repositories follow a clean-layered architecture:

```
┌─────────────────────────────────────────────────────┐
│                     HTTP Layer                      │
│              (handlers/, middlewares/)              │
│  • Request parsing, validation, response formatting │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│                    Service Layer                    │
│                     (services/)                     │
│  • Business logic, orchestration                    │
│  • Transaction management                           │
│  • Input validation, error handling                 │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│                  Data Access Layer                  │
│                       (dao/)                        │
│  • External data source abstraction                 │
│  • Query execution and response mapping             │
└──────────────────────────┬──────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   PostgreSQL            LLMs          Other sources
```

In addition to the layers above, the backend repositories also contain a `/lib` layer (for custom helpers) while
a special `/config` layer centralizes shared configurations.

## Coding practices

This section documents the coding practices used in the backend repositories.

### Dependencies

Go has a feature-rich standard library. Generally speaking, we prefer stdlib over external packages (e.g. `net/http`).

Still, this is not enough, and we must sometimes resort to external packages. Those are carefully selected: packages
must be well-maintained, preferably belong to an organization and be open-source. Below are the preferred solutions
when external packages are required:

- [Chi](https://github.com/go-chi/chi): provides neat helpers over the standard library `net/http` package.
- [Validator](https://github.com/go-playground/validator): Provides tag-based validation for structs.
- [Goccy YAML](github.com/goccy/go-yaml): yaml parser, we use yaml for human-maintained configuration files.
- [Google UUID](https://github.com/google/uuid): we need the uuid type basically everywhere, so...
- [Gorilla Schema](https://github.com/gorilla/schema): we don't use mux, but gorilla provides a nice decoder that allows
  us to parse url search params into structs, just like we do with json bodies.
- [Samber/Lo](https://github.com/samber/lo): collection of Go utilities inspired by Lodash. Can help reduce boilerplate.
- [Testify](https://github.com/stretchr/testify): test helpers.
- [Bun](https://github.com/uptrace/bun): we don't really use ORMs, but Bun offers a complete, integrated solution for
  interacting with SQL databases in Go. It does not require any external dependencies for operations like struct
  mapping (unlike `pgx`).
- [Otel](https://github.com/open-telemetry/opentelemetry-go): has an official Go SDK.

For custom helpers, you may put them inside the [`/lib` layer](#layers).

> We provide a collection of custom helpers through our managed package [golib](https://github.com/a-novel-kit/golib).
> This contains helpers that are used across multiple repositories, and helps keeping the lib layer clean.

### Go Coding Conventions

#### File Naming

| Type     | Pattern                   | Example                                        |
| -------- | ------------------------- | ---------------------------------------------- |
| Services | `{operation}.go`          | `tokenCreate.go`, `credentialsExist.go`        |
| Handlers | `http.{resource}.go`      | `http.credentials.go`, `http.token.go`         |
| DAO      | `{source}.{operation}.go` | `pg.credentialsList.go`, `openai.summarize.go` |
| Tests    | `{file}_test.go`          | `tokenCreate_test.go`                          |
| Mocks    | Generated in `mocks/`     | `mocks/mocks.go`                               |

#### Function Naming

```go
// Constructors: New[Type]
func NewTokenCreate(deps Dependencies) *TokenCreate

// Main business method: Exec
func (s *TokenCreate) Exec(ctx context.Context, req *Request) (*Response, error)

// HTTP handlers: ServeHTTP
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request)

// Private helpers: camelCase
func (s *service) validateInput(req *Request) error
```

#### Service Pattern

Every service follows this structure:

```go
// 1. Define interfaces for dependencies
type MyServiceRepository interface {
    GetData(ctx context.Context, id string) (*Data, error)
}

// 2. Define request/response structs
type MyServiceRequest struct {
    ID string `validate:"required,uuid"`
}

// 3. Define the service struct
type MyService struct {
    repository MyServiceRepository
}

// 4. Constructor
func NewMyService(repository MyServiceRepository) *MyService {
    return &MyService{repository: repository}
}

// 5. Main method with tracing and validation
func (s *MyService) Exec(ctx context.Context, request *MyServiceRequest) (*Response, error) {
    ctx, span := otel.Tracer().Start(ctx, "service.MyService")
    defer span.End()

    // Validate
    if err := validate.Struct(request); err != nil {
        return nil, otel.ReportError(span, errors.Join(err, ErrInvalidRequest))
    }

    // Execute logic
    result, err := s.repository.GetData(ctx, request.ID)
    if err != nil {
        return nil, otel.ReportError(span, err)
    }

    return otel.ReportSuccess(span, result), nil
}
```

#### Error Handling

```go
// Define package-level errors with Err prefix
var (
    ErrInvalidRequest = errors.New("invalid request")
    ErrNotFound       = errors.New("resource not found")
)

// Wrap errors with context
return nil, fmt.Errorf("fetch user: %w", err)

// Join errors when adding classification
return nil, errors.Join(err, ErrInvalidRequest)

// Check errors with errors.Is
if errors.Is(err, ErrNotFound) {
    // handle not found
}
```

#### Handler Pattern

##### HTTP Handlers

```go
type MyHandler struct {
    service MyServiceInterface
}

func NewMyHandler(service MyServiceInterface) *MyHandler {
    return &MyHandler{service: service}
}

func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer().Start(r.Context(), "handler.MyHandler")
    defer span.End()

    // Parse request
    var request MyRequest
    if err := httpf.DecodeJSON(r, &request); err != nil {
        httpf.HandleError(ctx, w, span, nil, err)
        return
    }

    // Call service
    result, err := h.service.Exec(ctx, &request)
    if err != nil {
        httpf.HandleError(ctx, w, span, httpf.ErrMap{
            ErrNotFound: http.StatusNotFound,
            ErrInvalidRequest: http.StatusUnprocessableEntity,
        }, err)
        return
    }

    // Return response
    httpf.WriteJSON(ctx, w, span, result, http.StatusOK)
}
```

##### GRPC Handlers

```go
type MyHandler struct {
    protogen.UnimplementedMyServiceServer

    service MyServiceInterface
}

func NewMyHandler(service MyServiceInterface) *MyHandler {
    return &MyHandler{service: service}
}

func (h *MyHandler) MyMethod(ctx context.Context, req *protogen.MyRequest) (*protogen.MyResponse, error) {
    ctx, span := otel.Tracer().Start(ctx, "handler.MyMethod")
    defer span.End()

    // Parse and validate request
    id, err := uuid.Parse(req.GetId())
    if err != nil {
        _ = otel.ReportError(span, err)
        return nil, status.Error(codes.InvalidArgument, "invalid id")
    }

    // Call service
    result, err := h.service.Exec(ctx, &ServiceRequest{ID: id})
    if errors.Is(err, ErrNotFound) {
        _ = otel.ReportError(span, err)
        return nil, status.Error(codes.NotFound, "resource not found")
    }
    if err != nil {
        _ = otel.ReportError(span, err)
        return nil, status.Error(codes.Internal, "internal error")
    }

    // Return response
    return &protogen.MyResponse{...}, nil
}
```

#### Dependency Injection

All services receive dependencies via constructor:

```go
// Good: Dependencies injected
func NewService(repo Repository, logger Logger) *Service {
    return &Service{repo: repo, logger: logger}
}

// Bad: Global dependencies
func NewService() *Service {
    return &Service{repo: globalRepo}
}
```

#### Validation

Validate at service layer entry, not in handlers:

```go
func (s *Service) Exec(ctx context.Context, req *Request) (*Response, error) {
    // Validate here
    if err := validate.Struct(req); err != nil {
        return nil, errors.Join(err, ErrInvalidRequest)
    }
    // Business logic...
}
```

### Database Conventions

#### Entity Naming

- Tables: `snake_case` plural (e.g., `jwks`, `key_metadata`)
- Columns: `snake_case` (e.g., `created_at`, `expires_at`)

#### Bun ORM Tags

```go
type Jwk struct {
    bun.BaseModel `bun:"table:jwks"`

    ID        uuid.UUID  `bun:"id,pk,type:uuid,default:uuid_generate_v4()"`
    Payload   []byte     `bun:"payload,notnull"`
    ExpiresAt *time.Time `bun:"expires_at"`
    CreatedAt time.Time  `bun:"created_at,notnull,default:current_timestamp"`
}
```

#### Embedded SQL (PostgreSQL DAO)

For PostgreSQL operations, DAO files use **raw SQL queries in external `.sql` files**, not Bun's query builder. This
avoids typical ORM pitfalls (unexpected query generation, N+1 problems, abstraction leaks) and gives full control over
the executed SQL:

```go
//go:embed pg.jwkSelect.sql
var jwkSelectQuery string

func (dao *JwkSelect) Exec(ctx context.Context, req *Request) (*Jwk, error) {
    var result Jwk
    err := dao.db.NewRaw(jwkSelectQuery, req.ID).Scan(ctx, &result)
    return &result, err
}
```

Bun is used only for connection management and result scanning — the query itself is always explicit SQL.

---

### Protocol Buffers

The gRPC API is defined using Protocol Buffers in `internal/models/proto/`. These files are the source of truth for all
available endpoints, request/response schemas, and service definitions.

#### Directory Structure

```
internal/models/proto/
├── jwk.proto           # Common JWK message types
├── jwk_get.proto       # JwkGetService definition
├── jwk_list.proto      # JwkListService definition
├── claims_sign.proto   # ClaimsSignService definition
└── status.proto        # StatusService definition
```

#### Editing Proto Files

When adding or modifying proto files:

1. Define your messages and services in `.proto` files
2. Run `make generate` to regenerate Go code
3. Run `make lint-proto` to validate the proto files

#### Code Generation

Proto files are compiled to Go code using `buf`:

```bash
# Generate Go code from proto files
make generate

# Lint proto files
make lint-proto

# Format proto files
make format-proto
```

Generated code lives in `internal/handlers/protogen/` and should **never be edited manually**.

#### Buf Configuration

The project uses [Buf](https://buf.build/) for proto management. Configuration is in `buf.yaml` and `buf.gen.yaml`.

**Resources:**

- [Buf Documentation](https://buf.build/docs/)
- [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [gRPC Go Documentation](https://grpc.io/docs/languages/go/)

---

### Testing

#### Running Tests

```bash
make test       # All tests
make test-unit  # Go unit tests only
make test-pkg   # Package integration tests
```

#### Table-Driven Tests

All tests use the **table-driven pattern**: test cases are defined as a slice of structs, each describing inputs,
expected outputs, and mock behavior. This approach:

- **Centralizes test logic** — the assertion code is written once and reused for all cases
- **Makes adding cases trivial** — just append a new struct to the slice
- **Improves readability** — each case is self-documenting with a descriptive name
- **Enables parallel execution** — independent cases can run concurrently

```go
testCases := []struct {
    name      string
    request   *Request
    expect    *Response
    expectErr error
}{
    {name: "Success", request: &Request{ID: "123"}, expect: &Response{...}},
    {name: "Error/NotFound", request: &Request{ID: "bad"}, expectErr: ErrNotFound},
}

for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
        // test logic here
    })
}
```

#### Test Case Naming

Use descriptive names that indicate the scenario:

| Pattern           | Usage                               |
| ----------------- | ----------------------------------- |
| `Success`         | Happy path                          |
| `Success/Variant` | Happy path with specific conditions |
| `Error/Reason`    | Expected failure with cause         |

Examples: `"Success"`, `"Success/WithExpiredKey"`, `"Error/InvalidInput"`, `"Error/NotFound"`

#### Parallel Execution

Use `t.Parallel()` to run tests concurrently, reducing total test time:

```go
func TestMyService(t *testing.T) {
    t.Parallel() // Run this test in parallel with other top-level tests

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel() // Run subtests in parallel with each other
            // ...
        })
    }
}
```

**When to use:** Services and handlers tests (with mocks) — these have no shared state.

**When to omit:** DAO tests wrapped in transactions — isolation is managed by the transaction, not parallelism.

#### DAO Tests (Real Database)

DAO tests run against a real PostgreSQL instance to verify actual SQL behavior. Each test case is wrapped in a
**transaction that rolls back automatically**, ensuring:

- Tests don't pollute each other's data
- No manual cleanup required
- Database state is reset between cases

```go
func TestJwkSelect(t *testing.T) {
    repository := dao.NewJwkSelect()

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            postgres.RunTransactionalTest(t, config.PostgresPresetTest, func(ctx context.Context, t *testing.T) {
                // 1. Insert fixtures (test-specific data)
                // 2. Run the DAO method under test
                // 3. Assert results
                // Transaction auto-rollbacks after this function returns
            })
        })
    }
}
```

**Fixtures** are defined in the test case struct alongside expected inputs/outputs, then inserted at the start of the
transactional callback.

#### Service/Handler Tests (Mocks)

Services and handlers are tested with **mocked dependencies**, making tests fast and deterministic. Mock behavior is
defined per test case:

```go
testCases := []struct {
    name           string
    request        *Request
    repositoryMock *repositoryMockSetup // Define expected mock behavior
    expect         *Response
    expectErr      error
}{...}
```

Mocks are generated by mockery. After adding or modifying interfaces:

```bash
make generate
```

---

# Appendix

## A. Common commands

Those commands are generally available in all repositories.

| Command         | Description                              |
| --------------- | ---------------------------------------- |
| `make run`      | Start a fully functional service locally |
| `make test`     | Run all tests                            |
| `make lint`     | Run all linters                          |
| `make format`   | Format code (fix lint issues)            |
| `make build`    | Build Docker images locally              |
| `make generate` | Generate mocks and templates             |

## B. Directory Structure

```
service-name/
├── cmd/                          # Entry points (Go)
│   ├── rest/main.go              # REST API server (when applicable)
│   ├── grpc/main.go              # GRPC API server (when applicable)
│   ├── migrations/main.go        # Database migration runner
│   └── init/main.go              # Patch the server with initial state
│
├── internal/                     # Private Go packages (core logic)
│   ├── config/                   # Configuration management
│   ├── dao/                      # Data Access Objects (external data sources)
│   ├── handlers/                 # HTTP request handlers
│   │   └── middlewares/          # HTTP middleware
│   ├── services/                 # Business logic layer
│   ├── lib/                      # Utilities
│   └── models/                   # Domain models
│       ├── migrations/           # SQL migration files
│       └── ...                   # Any internal model files (e.g. mail templates, etc.)
│
├── pkg/                          # Public packages
│   ├── *.go                      # Public go package
│   ├── rest-js/                  # TypeScript REST client    (when applicable)
│   └── rest-js-test/             # TypeScript test utilities (when applicable)
│
├── builds/                       # Docker build files
│   ├── *.Dockerfile              # Image definitions
│   └── podman-compose.yaml       # Local development stack
│
├── scripts/                      # Build and test scripts
└── .github/workflows/            # CI/CD pipelines
```
