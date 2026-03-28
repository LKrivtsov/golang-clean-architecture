---
name: golang-clean-architecture
description: >
  Opinionated guidance for building production-grade Go backends using Clean
  Architecture and Domain-Driven Design. Covers project structure, REST API
  patterns, error handling, migrations, middleware, testing, Makefile, Docker
  Compose, Keycloak auth, PostgreSQL, and Swagger/OpenAPI. Trigger when
  scaffolding Go backend services or working with layered/hexagonal Go projects.
---

# Skill: Golang Backend — Clean Architecture & DDD

Opinionated guidance for building production-grade Go backends using Clean Architecture and Domain-Driven Design. Follow these conventions when scaffolding, extending, or reviewing Go backend services that use a layered/hexagonal structure.

---

## 1. When to Use This Skill

**Trigger conditions:**

- User is scaffolding a new Go backend service
- Project contains `go.mod` and follows a layered/hexagonal structure
- User asks about REST API design, DDD patterns, or clean architecture in Go
- Files match patterns like `internal/domain/`, `internal/usecase/`, `cmd/api/`

**Do NOT trigger when:**

- Project is a CLI tool, library, or non-HTTP service
- User is working on frontend, infrastructure-only, or non-Go code

---

## 2. Core Concepts

- **Domain layer**: Pure business logic, no framework dependencies. Entities, value objects, domain events, repository interfaces.
- **Use-case (application) layer**: Orchestrates domain logic. Input/output ports. Transaction boundaries.
- **Interface (adapter) layer**: HTTP handlers, gRPC servers, DB implementations, external service clients.
- **Infrastructure layer**: Framework glue, config, DI wiring, logging setup.
- **Dependency rule**: Dependencies point inward — domain never imports from outer layers.
- **Inter-layer data transfer**: All data between layers MUST be passed via explicitly typed structs — never raw maps, `interface{}`, or leaked domain entities. Transport layer uses request/response DTOs; use-case layer defines its own input/output structs; repository layer accepts and returns domain entities or dedicated query/result structs.

---

## 3. Project Structure

Designed for 100% test coverage. Every layer has a clear interface boundary that can be mocked.

```
cmd/
  api/              # main.go, server bootstrap
internal/
  domain/           # entities, value objects, repository interfaces, domain errors
    user/
      entity.go
      repository.go     # interface — mockable
      errors.go
      entity_test.go    # domain logic unit tests
  usecase/          # application services, input/output structs
    user/
      create.go
      create_test.go    # unit tests with mocked repo interface
      get.go
      get_test.go
      dto.go            # use-case input/output structs
  adapter/
    http/            # handlers, routes, middleware
      v1/
        user_handler.go
        user_handler_test.go  # unit tests with mocked use-case
        router.go
        dto.go              # request/response DTOs (transport layer)
        mapper.go           # DTO <-> use-case struct mapping
    postgres/        # repository implementations
      user_repo.go
      user_repo_test.go     # integration tests against test DB
      migrations/
  infrastructure/
    config/          # env parsing, config structs
    logger/          # structured logging setup
    server/          # HTTP server lifecycle
pkg/                 # shared utilities (optional, keep minimal)
docs/                # generated OpenAPI/Swagger spec (see section 20)
Makefile             # project automation (see section 15)
docker-compose.yml   # local dev environment (see section 16)
.air.toml            # live reload config
.env                 # environment variables (see section 17)
.env.example         # template with placeholder values
```

**Testing rule**: Every piece of code MUST have unit tests. When writing any new code — handler, use-case, entity, repository — always create the corresponding `_test.go` file. Use table-driven tests. Mock all external dependencies via interfaces.

---

## 4. API Versioning Strategies

- **URL-path versioning** (recommended): `/api/v1/users`, `/api/v2/users`
- Version routers independently so v1 and v2 can coexist
- Deprecation: add `Sunset` and `Deprecation` headers on old versions
- Internal DTOs per version — never share request/response structs across versions

---

## 5. REST API Design Patterns

Standard resource endpoints follow this pattern:

```
GET    /api/users              # List users (with pagination)
POST   /api/users              # Create user
GET    /api/users/{id}         # Get specific user
PUT    /api/users/{id}         # Replace user (full update)
PATCH  /api/users/{id}         # Update user fields (partial)
DELETE /api/users/{id}         # Delete user
```

- Resources are nouns, not verbs: `/users`, `/orders/{id}/items`
- Use HTTP methods correctly: GET (read), POST (create), PUT (full replace), PATCH (partial update), DELETE
- Use `Location` header on 201 Created
- HATEOAS optional — only if the team commits to it

---

## 6. Resource Collection Design

- Plural nouns: `/users`, `/products`
- Nested resources for strong ownership: `/users/{id}/addresses`
- Flat resources with query filters for weak associations: `/orders?userId=123`
- Bulk operations: `POST /users/batch` or `PATCH /users` with array body

---

## 7. Pagination and Filtering

All list endpoints use **page-based pagination** with `page` and `pageSize` query parameters:

```
GET /api/users?page=1&pageSize=20
GET /api/users?page=2&pageSize=20&status=active
```

- **Parameters**: `page` (1-based), `pageSize` (default 20)
- **Filtering**: query params matching field names — `?status=active&createdAfter=2025-01-01`
- **Sorting**: `?sort=createdAt:desc,name:asc`
- **Pagination response** is included in the `pagination` field of the standard envelope (see section 9):

```json
{
  "page": 1,
  "perPage": 20,
  "totalItems": 145,
  "totalPages": 8,
  "hasNext": true,
  "hasPrev": false
}
```

**No cursor-based pagination.** Always use page + pageSize.

---

## 8. Migrations

- Use a migration tool (`golang-migrate`, `goose`, or `atlas`)
- Migrations live in `internal/adapter/postgres/migrations/`
- Naming: `YYYYMMDDHHMMSS_description.up.sql` / `.down.sql`
- Every migration must have a rollback (`down`)
- Never modify a migration after it has been applied — create a new one
- Run migrations at startup or via a separate CLI command, not in request handlers
- Run migrations via Makefile: `make migrate-up`, `make migrate-down`, `make migrate-create`

---

## 9. Standard API Response Envelope

All API responses MUST follow this structure:

```json
{
  "status": "success",
  "data": {},
  "error": null,
  "pagination": null,
  "meta": {
    "timestamp": "2026-03-28T12:00:00Z",
    "requestId": "e7f1b6a2-2c3d-4f11-9b8e-123456789abc"
  }
}
```

### Fields

| Field        | Description                                                      |
| ------------ | ---------------------------------------------------------------- |
| `status`     | `"success"`, `"error"`, or optionally `"partial_success"`        |
| `data`       | Main payload — object, array, or `null`                          |
| `error`      | Error object (see 9.3) or `null` on success                      |
| `pagination` | Pagination info (see section 7) or `null` for non-list endpoints |
| `meta`       | Optional — `timestamp` and `requestId` for tracing               |

### 9.1 Successful response without pagination

```json
{
  "status": "success",
  "data": {
    "id": 123,
    "name": "Test"
  },
  "error": null,
  "pagination": null
}
```

### 9.2 Successful response with list and pagination

```json
{
  "status": "success",
  "data": [
    { "id": 1, "name": "Item 1" },
    { "id": 2, "name": "Item 2" }
  ],
  "error": null,
  "pagination": {
    "page": 1,
    "perPage": 20,
    "totalItems": 145,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### 9.3 Error response

```json
{
  "status": "error",
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "email": "Field email is required"
    }
  },
  "pagination": null
}
```

---

## 10. Error Handling and Status Codes

- Domain errors are typed: `ErrUserNotFound`, `ErrEmailTaken`
- Map domain errors to HTTP status codes in the adapter layer, not in use cases
- Error object format (always consistent):

```json
{
  "code": "STRING_CODE",
  "message": "Human-readable description",
  "details": {}
}
```

- `code` — used by frontend/other services for logic
- `message` — can be shown to humans
- `details` — additional context fields
- Status code mapping:
  - 400: validation errors
  - 401: unauthenticated
  - 403: unauthorized
  - 404: resource not found
  - 409: conflict (duplicate, version mismatch)
  - 422: unprocessable entity (valid syntax, invalid semantics)
  - 500: unexpected internal errors
- Wrap errors with `fmt.Errorf("...: %w", err)` for stack context

---

## 11. DTO Structure

**Three DTO boundaries — all typed structs, no `interface{}` or raw maps:**

1. **Transport DTOs** (adapter/http layer):
   - `CreateUserRequest` — request body with validation tags (`validate:"required,email"`)
   - `UserResponse` — what the API returns, separate from domain entity
   - Lives in `adapter/http/v1/dto.go`
2. **Use-case DTOs** (usecase layer):
   - `CreateUserInput` — what the use-case accepts
   - `CreateUserOutput` / `UserResult` — what the use-case returns
   - Lives in `usecase/user/dto.go`
3. **Domain entities** (domain layer):
   - `User` — the core domain struct
   - Repository interface accepts/returns domain entities
   - Lives in `domain/user/entity.go`

**Mapping functions** convert between layers:

- `adapter/http/v1/mapper.go`: `requestToInput()`, `outputToResponse()`
- `usecase/user/`: works directly with domain entities via repository interface

Never leak domain entities into API responses. Never reuse a request DTO as a response DTO.

**Struct change verification rule**: When modifying any request or response struct, you MUST verify that all handlers and methods that use that struct still compile and behave correctly. Check every caller and consumer of the changed struct across all layers (transport -> use-case -> repository). Run `go build ./...` and `go test ./...` after any struct change.

---

## 12. Best Practices for REST APIs

- Idempotency keys for POST/PATCH: `Idempotency-Key` header
- Rate limiting: return `429` with `Retry-After` header
- Request IDs: generate or propagate `X-Request-ID` for tracing
- Content negotiation: support `application/json`, reject unsupported types with 415
- Health check: `GET /healthz` (liveness), `GET /readyz` (readiness)
- Graceful shutdown: drain connections on SIGTERM
- Input validation at the handler level, business validation in use cases

---

## 13. Middleware Creation

- Middleware signature: `func(next http.Handler) http.Handler`
- Common middleware stack (in order):
  1. Recovery (panic -> 500)
  2. Request ID injection
  3. Structured logging (method, path, status, duration)
  4. CORS
  5. Rate limiting
  6. Authentication (JWT via Keycloak — see section 18)
  7. Authorization (RBAC/ABAC)
- Keep middleware stateless — inject dependencies via closures or DI
- Per-route middleware for auth scopes: `router.With(RequireRole("admin")).Get("/admin/users", ...)`

---

## 14. Testing Strategy

**Every file gets a test file. No exceptions.**

- **Domain tests**: Pure unit tests — no mocks needed, test entity behavior and validation directly
- **Use-case tests**: Mock repository interfaces (use `gomock` or hand-written mocks). Test business logic in isolation.
- **Handler tests**: Mock use-case interfaces. Test HTTP request parsing, validation, response formatting.
- **Repository tests**: Integration tests against a test PostgreSQL database (use `testcontainers-go` or a shared test DB)
- **Table-driven tests**: Default pattern for all tests
- **Interface-driven design**: Every cross-layer dependency is an interface -> every dependency is mockable -> 100% unit test coverage is achievable

```go
// Example: use-case depends on repository interface
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
}

// In tests, inject a mock:
type mockUserRepo struct { ... }
func (m *mockUserRepo) Create(ctx context.Context, user *User) error { ... }
```

---

## 15. Makefile

Every project MUST have a `Makefile` at the root. It is the single entry point for all project automation:

```makefile
# --- Project ---
.PHONY: run build
run:              ## Run the server locally
	@go run ./cmd/api
build:            ## Build the binary
	@go build -o bin/api ./cmd/api

# --- Docker ---
.PHONY: up down
up:               ## Start all services (docker-compose)
	@docker-compose up -d
down:             ## Stop all services
	@docker-compose down

# --- Database / Migrations ---
.PHONY: migrate-up migrate-down migrate-create
migrate-up:       ## Apply all pending migrations
	@migrate -path internal/adapter/postgres/migrations -database "$(DATABASE_URL)" up
migrate-down:     ## Roll back the last migration
	@migrate -path internal/adapter/postgres/migrations -database "$(DATABASE_URL)" down 1
migrate-create:   ## Create a new migration (usage: make migrate-create name=add_users)
	@migrate create -ext sql -dir internal/adapter/postgres/migrations -seq $(name)

# --- Testing ---
.PHONY: test test-coverage
test:             ## Run all tests
	@go test ./... -v
test-coverage:    ## Run tests with coverage report
	@go test ./... -coverprofile=coverage.out && go tool cover -html=coverage.out -o coverage.html

# --- Code Generation ---
.PHONY: mocks
mocks:            ## Generate mocks (mockgen)
	@go generate ./...

# --- Linting ---
.PHONY: lint
lint:             ## Run linter
	@golangci-lint run ./...

# --- Swagger / OpenAPI ---
.PHONY: swagger
swagger:          ## Generate OpenAPI spec from annotations
	@swag init -g cmd/api/main.go -o docs/

# --- Helpers ---
.PHONY: help
help:             ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'
```

---

## 16. Docker Compose & Live Reload

Every project MUST have a `docker-compose.yml` for local development. Use `air` for live reload of the Go server.

### docker-compose.yml

```yaml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "${APP_PORT:-8080}:8080"
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
    command: ["air", "-c", ".air.toml"]

  postgres:
    image: postgres:16-alpine
    ports:
      - "${DB_PORT:-5432}:5432"
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-app}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### .air.toml (live reload)

```toml
root = "."
tmp_dir = "tmp"

[build]
cmd = "go build -o ./tmp/api ./cmd/api"
bin = "tmp/api"
watch_dir = ["cmd", "internal", "pkg"]
include_ext = ["go", "toml", "yaml", "yml"]
exclude_dir = ["tmp", "vendor", "node_modules"]
delay = 1000
```

---

## 17. Environment Variables (.env)

Every project MUST have:

- `.env` — actual environment variables for local dev (gitignored)
- `.env.example` — template with placeholder values (committed to git)

When setting up the project, always create `.env.example`. If any env var value is unknown (secrets, external service URLs, API keys), add a `# TODO: fill in` comment and notify the user that they must fill it in before running the project.

### .env.example

```env
# --- App ---
APP_PORT=8080
APP_ENV=development

# --- Database (PostgreSQL) ---
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=app
DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}?sslmode=disable

# --- Keycloak (if auth is used) ---
# KEYCLOAK_URL=http://localhost:8180  # TODO: fill in if using auth
# KEYCLOAK_REALM=app                  # TODO: fill in if using auth
# KEYCLOAK_CLIENT_ID=backend          # TODO: fill in if using auth
# KEYCLOAK_CLIENT_SECRET=             # TODO: fill in
```

---

## 18. Authentication (Keycloak)

If the project uses authentication, the identity provider is **Keycloak**. Always.

- Keycloak runs as a service in `docker-compose.yml` (add when auth is needed)
- JWT tokens issued by Keycloak are validated in the auth middleware
- Use the Keycloak OIDC well-known endpoint for key discovery: `{KEYCLOAK_URL}/realms/{realm}/.well-known/openid-configuration`
- Roles come from Keycloak realm/client roles, mapped to Go RBAC in the authorization middleware
- If Keycloak credentials or URLs are unknown, add them to `.env.example` with `# TODO: fill in` and notify the user

---

## 19. Database (PostgreSQL)

The database is always **PostgreSQL**.

- Use `pgx` as the driver (preferred) or `database/sql` with `lib/pq`
- Connection pooling: configure `max_open_conns`, `max_idle_conns`, `conn_max_lifetime` in config
- Migrations target PostgreSQL syntax
- Repository integration tests run against a real PostgreSQL instance (via docker-compose or testcontainers-go)
- Connection string format: `postgres://user:pass@host:port/dbname?sslmode=disable`

---

## 20. Swagger / OpenAPI Documentation

Every API handler MUST have `swaggo/swag` annotations for automatic OpenAPI spec generation.

### Required annotations

**Main entry point** (`cmd/api/main.go`):

```go
// @title           My API
// @version         1.0
// @description     API server description
// @host            localhost:8080
// @BasePath        /api
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
func main() { ... }
```

**Every handler function**:

```go
// CreateUser godoc
// @Summary      Create a new user
// @Description  Creates a user with the provided data
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        request body CreateUserRequest true "User data"
// @Success      201 {object} APIResponse{data=UserResponse}
// @Failure      400 {object} APIResponse{error=ErrorObject}
// @Failure      409 {object} APIResponse{error=ErrorObject}
// @Failure      500 {object} APIResponse{error=ErrorObject}
// @Security     BearerAuth
// @Router       /users [post]
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) { ... }
```

### Swagger UI endpoint

The project MUST serve Swagger UI so developers can browse and test all endpoints:

```go
import httpSwagger "github.com/swaggo/http-swagger"

router.Get("/swagger/*", httpSwagger.WrapHandler)
```

Access at: `http://localhost:8080/swagger/index.html`

### Generation

Run `make swagger` to regenerate the OpenAPI spec from annotations. The generated files live in `docs/` (committed to git so CI can serve them).

### Rules

- Every new or modified handler MUST have complete swag annotations before the PR is merged
- Annotations must match the actual request/response DTOs — keep them in sync when structs change
- Include `@Security` annotation on all protected endpoints
- Use `@Tags` to group endpoints by resource
