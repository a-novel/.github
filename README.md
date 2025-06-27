This document is a comprehensive list of the infrastructure components used for deploying and running the Agora
services.

# Tech stack

- [Go](https://go.dev/): for backend services and scripting
- [PostgreSQL](https://www.postgresql.org/): database management
- [Tanstack start](https://tanstack.com/start/latest): client framework
  - [Node.js](https://nodejs.org/en): used to power SSR and build the client
  - [React](https://react.dev/): for building the user interface
- [Podman](https://podman.io/): to run containers locally
- [MJML](https://mjml.io/): email templating language

# Code quality

- [Prettier](https://prettier.io/): formatter used for many languages
- [SQLFluff](https://sqlfluff.com/): SQL linter and formatter
- [Golang-CI lint](https://github.com/golangci/golangci-lint): for Go projects

# Infra

- [Groq](https://console.groq.com/home): LLM inference
- [Codecov](https://app.codecov.io/): code coverage reporting
- [Sentry](https://agora-storyverse.sentry.io/): error tracking and performance monitoring
- [Tolgee](https://app.tolgee.io/): translation management system

# Ports cheat sheet

Because we expose our applications through docker containers, here is a cheat sheet of the ports used by each service:

| Service                                                                         | Local Port | Test Port | Infra Port |
|---------------------------------------------------------------------------------|------------|-----------|------------|
| [Service JSON Keys](https://github.com/a-novel/service-json-keys)               | 4001       | 5001      | 4001       |
| Service JSON Keys Postgres                                                      | 4002       | 5002      |            |
| [Service Authentication](https://github.com/a-novel/service-authentication)     | 4011       | 5011      | 4011       |
| Service Authentication Postgres                                                 | 4012       | 5012      |            |
| [Service Story-Schematics](https://github.com/a-novel/service-story-schematics) | 4021       | 5021      | 4021       |
| Service Story-Schematics Postgres                                               | 4022       | 5022      |            |

| Platform Service                                                              | Local Port | Infra Port |
|-------------------------------------------------------------------------------|------------|------------|
| [Platform authentication](https://github.com/a-novel/platform-authentication) | 6001       | 6001       |
| [Platform authentication](https://github.com/a-novel/platform-studio)         | 6011       | 6011       |
| [Platform authentication](https://github.com/a-novel/platform-back-office)    | 6021       | 6021       |