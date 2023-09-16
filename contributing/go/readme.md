# Backend

## Philosophy

Agora uses a macro-service architecture: we focus on monolithic services, and only split things up when we can fully
decouple them. The idea is to avoid the necessity for tools such as API gateways for managing messy and countless
back-and-forth calls between services.

Each service is self-contained and fully autonomous, meaning no infrastructure must be shared. The only dependencies
allowed are occasional API calls to other services, done through their exposed API client (if they ever provide one).

If data has to be shared for cross-service features (for example, a full-text search index), data must be duplicated
through dedicated channels (using tools such as kafka).

## Tools

Agora services are written using the following stack:

- [Go](https://go.dev/): for the business logic
  - [Bun](https://bun.uptrace.dev/): ORM tool, see [About using ORMs](#about-using-orms) for more information
  - [Ogen](https://github.com/ogen-go/ogen): to generate OpenAPI stubs
  - [Mockery](https://github.com/vektra/mockery): to generate mocks for testing
- [PostgreSQL](https://www.postgresql.org/): to manage data
- [OpenAPI](https://www.openapis.org/): to define the API specs

The following tools are used to help with development:

- [Podman](https://github.com/containers/podman): to manage containers
- [Python](https://www.python.org/) and [Node.js](https://nodejs.org/en): to run extra scripts and tools
  - [Prettier](https://prettier.io/): initially a JS tool, used to format / lint most non-Go files (.md, .yaml, etc.) 
  - [SQLFluff](https://sqlfluff.com/): to lint and format SQL files

## Basics

- Learn Go: https://go.dev/tour/
- Learn PostgreSQL: https://www.postgresql.org/docs/current/tutorial.html
- Learn OpenAPI: https://learn.openapis.org/

It is also recommended to take a look at our in-deep documentation:

- [Code Quality](code-quality.md): how to write code that is compliant with our standards
- [Mocking](mocking.md): how to use mocks in your tests

## Code structure

Each service shares the following structure, exceptions being documented in the Contributing guidelines of
each service.

- `.github`: contains CI configuration
- `api`: contains the API handlers, along with [Ogen](https://github.com/ogen-go/ogen) generated code.
- `build`: contains all the compose files and Docker images used to run the service.
- `cmd`: the entrypoints of the service
- `config`: contains the configuration files
- `docs`: [OpenAPI](https://www.openapis.org/) specs
- `internal`: contains the business logic of the service, split into layers:
  - `services`: contains the service interfaces and their implementations
  - `dao`: contains the data access layer, used to interact with data sources
  - `lib`: contains utility functions used across the service
- `migrations`: contains the PostgreSQL migration files
- `models`: contains the shared data models used across the service
- `scripts`: contains scripts used to run the service locally

> An arrow on the graph indicates the dependency flow. `A -> B` means that `A` may import / depend on `B`, but not the
> other way around. This is important to prevent circular dependencies.

```text
               ┌──────────────────────────────┐
               │ SCRIPTS                      │
               │                              ◄── External User
               │ Bash scripts for local tasks │
               └─────────────────┬────────────┘         │
                                 │                      │
                                 │               ┌──────▼──────┐
                                 │               │ CMD         │
                                 └───────────────►             │
                                                 │ Executables │
                                                 └──────┬──────┘
                                                        │
                    ┌───────────────────────────────────┼───────────────────────────┐
┌────────────────┐  │                          ┌────────▼─────────┐                 │
│ DOCS           │  │                          │ API              │                 │
│                ◄──┼──────────────────────────┤                  │                 │
│ Spec documents │  │                          │ OpenAPI handlers │                 │
└────────────────┘  │                          └──┬───────────────┘                 │
                    │                             │                                 │
                    │                             │                                 │
                    │  INTERNAL                   │                                 │
                    │                             │                                 │
                    │  Application Layers         │                                 │
                    │                             │                                 │
                    │ ┌───────────────────────────▼───┐      ┌────────────────────┐ │
                    │ │ SERVICES                      │      │ DAO                │ │
                    │ │                               ├──────►                    │ │
                    │ │ Business Logic implementation │      │ Data-Access Object │ │
                    │ └──────────────┬────────────────┘      └──────────┬─────────┘ │
                    │                │                                  │           │
                    │                │     ┌────────────────┐           │           │
                    │                │     │ LIB            │           │           │
                    │                └─────►                ◄───────────┘           │
                    │                      │ Internal tools │                       │
                    │                      └────────────────┘                       │
                    └──────┬─────────────────────┬────────────────────────┬─────────┘
                           │                     │                        │
     ┌─────────────────────▼──────┐ ┌────────────▼────────┐   ┌───────────▼────────┐
     │ MIGRATIONS                 │ │ CONFIG              │   │ MODELS             │
     │                            │ │                     ├───►                    │
     │ PostgreSQL migration files │ │ Configuration files │   │ Shared definitions │
     └────────────────────────────┘ └─────────────────────┘   └────────────────────┘
```

### Internal

Internal directory contains the core business logic of the service.

Except for the `lib` (which has little to no rules), the internal layers are based on single-purpose interfaces.
Each file exposes a single interface, that itself performs a single task.

```go
// Always define a wrapping error for your service.
var ErrMyService = errors.New("MyService.MyTask")

func NewErrMyService(err error) error {
	return errors.Join(err, ErrMyService)
}

type MyService struct {}

func (service *MyService) MyTask(ctx context.Context, arg1 Arg1Type, arg2 Arg2Type) (Output1Type, Output2Type, error) {
	// Do something
	if err != nil {
		// Always wrap returned errors.
		return Output1Type{}, Output2Type{}, NewErrMyService(err)
	}
	
	return output1, output2, nil
}

func NewMyService() *MyService {
	return &MyService{}
}
```

Things to consider:

- Always define a wrapping error for your service, so you can easily identify where the error comes from.
- The first argument should always be a `context.Context`. This is useful for transactions and tracing.

Depending on your layer, you'll often need dependencies. A dependency is an interface that comes either from the
same layer, or another layer lower in the hierarchy (for example, the service interfaces will often import a DAO).
While not required, it is highly recommended you define a single interface for all the dependencies of an interface.
The standard naming for such interface is a Source:

```go
type MyServiceSource interface {
	MyDependencyMethod1()
	MyDependencyMethod2()
}

type MyService struct {
	source MyServiceSource
}

func (service *MyService) MyTask(ctx context.Context) (OutputType, error) {
	// ...	
	
	serviceResp, serviceErr := service.source.MyDependencyMethod1()
	
	// ...	
}

func NewMyService(source MyServiceSource) *MyService {
	return &MyService{source: source}
}
```

If your source only represents a single interface, then you can pass an implementation of that interface directly:

```go
service := NewMyService(myDependencyImplementation)
```

However, if your source has to be implemented by multiple interfaces, then you should provide a custom initializer
as well.

```go
type MyServiceSource interface {
	MyDependencyMethod1()
	MyDependencyMethod2()
}

type MyService struct {
	source MyServiceSource
}

func (service *MyService) MyTask(ctx context.Context) (OutputType, error) {}

func NewMyServiceSource(
	dependency1 Dependency1Interface,
	dependency2 Dependency2Interface,
) MyServiceSource {
	return &struct {
		Dependency1Interface
		Dependency1Interface
	}{
		Dependency1: dependency1,
		Dependency2: dependency2,
	}
}

func NewMyService(source MyServiceSource) *MyService {
	return &MyService{source: source}
}
```

