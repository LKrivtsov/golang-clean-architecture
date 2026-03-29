---
name: golang-clean-architecture
description: >
  Opinionated guidance for building production-grade Go backends using Clean
  Architecture. Covers project structure, REST API patterns, error handling,
  migrations, middleware, Makefile, Docker Compose, Keycloak auth, PostgreSQL,
  and Swagger/OpenAPI. Trigger when scaffolding Go backend services or working
  with layered/hexagonal Go projects.
---

# Skill: Golang Backend — Clean Architecture

Opinionated guidance for building production-grade Go backends using Clean Architecture. Follow these conventions when scaffolding, extending, or reviewing Go backend services that use a layered/hexagonal structure.

---

## 1. When to Use This Skill

**Trigger conditions:**

- User is scaffolding a new Go backend service
- Project contains `go.mod` and follows a layered/hexagonal structure
- User asks about REST API design or clean architecture in Go
- Files match patterns like `internal/entity/`, `internal/usecase/`, `cmd/api/`

**Do NOT trigger when:**

- Project is a CLI tool, library, or non-HTTP service
- User is working on frontend, infrastructure-only, or non-Go code

---

## 2. Core Concepts

- **Entity layer**: Business logic and data structures. Each entity keeps its struct, repository interface, and domain errors in a single file.
- **Use-case (application) layer**: Orchestrates entity logic. Input/output ports. Transaction boundaries. Each entity's use-case service, its methods, and DTOs live in a single file.
- **Interface (adapter) layer**: HTTP handlers, gRPC servers, DB implementations, external service clients.
- **Infrastructure layer**: Framework glue, config, DI wiring, logging setup.
- **Dependency rule**: Dependencies point inward — entities never import from outer layers.
- **Inter-layer data transfer**: All data between layers MUST be passed via explicitly typed structs — never raw maps, `interface{}`, or leaked entities. Transport layer uses request/response DTOs; use-case layer defines its own input/output structs; repository layer accepts and returns entities or dedicated query/result structs.
- **UUID identifiers (required)**: All entity IDs MUST be of type UUID (`uuid.UUID` from `github.com/google/uuid`). Never use integer auto-increment IDs. Generate UUIDs at creation time (typically in the use-case or repository layer). In PostgreSQL, use the `uuid` column type. In JSON responses, UUIDs are serialized as strings (e.g., `"id": "550e8400-e29b-41d4-a716-446655440000"`).
- **Go version (required)**: Always use **Go 1.25**. The `go.mod` file MUST specify `go 1.25`, and all Dockerfile stages MUST use the `golang:1.25-alpine` base image. Never use an older Go version.

---

## 3. Project Structure

Every layer follows a **one file per entity** rule — no splitting across multiple files per entity. Entity files hold struct + repository interface + domain errors; use-case files hold all methods + input/output DTOs; handler files hold all HTTP handlers; repo files hold all queries.

```
cmd/
  api/              # main.go, server bootstrap
internal/
  entity/           # business entities — one file per entity
    user.go         # User struct, UserRepository interface, domain errors
  usecase/          # application services, input/output structs
    user.go           # all use-case methods + input/output DTOs for User
  adapter/
    http/            # handlers, routes, middleware
      v1/
        user_handler.go
        router.go
        dto.go              # request/response DTOs (transport layer)
        mapper.go           # DTO <-> use-case struct mapping
    postgres/        # repository implementations
      user_repo.go
      migrations/
  infrastructure/
    config/          # env parsing, config structs
    logger/          # structured logging setup
    server/          # HTTP server lifecycle
pkg/                 # shared utilities (optional, keep minimal)
docs/                # generated OpenAPI/Swagger spec (see section 19)
Dockerfile           # multi-stage build: dev (air) + production (see section 15)
Makefile             # project automation (see section 14)
docker-compose.yml   # local dev environment (see section 15)
.air.toml            # live reload config
.env                 # environment variables — DB_HOST=postgres (docker network) (see section 16)
.env.example         # template with placeholder values
```

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
GET    /api/v1/users              # List users (with pagination)
POST   /api/v1/users              # Create user
GET    /api/v1/users/{id}         # Get specific user
PUT    /api/v1/users/{id}         # Replace user (full update)
PATCH  /api/v1/users/{id}         # Update user fields (partial)
DELETE /api/v1/users/{id}         # Delete user
```

- **snake_case for multi-word parameters**: All query parameters, JSON request/response field names, and sorting keys MUST use snake_case (underscores) when the name consists of two or more words. Examples: `page_size`, `created_at`, `user_id`, `total_items`. Never use camelCase in API schemas.
- Resources are nouns, not verbs: `/users`, `/orders/{id}/items`
- Use HTTP methods correctly: GET (read), POST (create), PUT (full replace), PATCH (partial update), DELETE
- Use `Location` header on 201 Created
- HATEOAS optional — only if the team commits to it

---

## 6. Resource Collection Design

- Plural nouns: `/users`, `/products`
- Nested resources for strong ownership: `/users/{id}/addresses`
- Flat resources with query filters for weak associations: `/orders?user_id=123`
- Bulk operations: `POST /users/batch` or `PATCH /users` with array body

---

## 7. Pagination and Filtering

All list endpoints use **page-based pagination** with `page` and `page_size` query parameters:

```
GET /api/v1/users?page=1&page_size=20
GET /api/v1/users?page=2&page_size=20&status=active
```

- **Parameters**: `page` (1-based), `page_size` (default 20)
- **Filtering**: query params matching field names — `?status=active&created_after=2025-01-01`
- **Sorting**: `?sort=created_at:desc,name:asc`
- **Pagination response** is included in the `pagination` field of the standard envelope (see section 9):

```json
{
  "page": 1,
  "per_page": 20,
  "total_items": 145,
  "total_pages": 8,
  "has_next": true,
  "has_prev": false
}
```

**No cursor-based pagination.** Always use page + page_size.

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
    "request_id": "e7f1b6a2-2c3d-4f11-9b8e-123456789abc"
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
| `meta`       | Optional — `timestamp` and `request_id` for tracing               |

### 9.1 Successful response without pagination

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
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
    { "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", "name": "Item 1" },
    { "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901", "name": "Item 2" }
  ],
  "error": null,
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_items": 145,
    "total_pages": 8,
    "has_next": true,
    "has_prev": false
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
- **Developer error logging (required)**: Before returning any error response to the client, the handler MUST log the original (internal) error using the structured logger. The user-facing response contains a sanitized message; the full error details (including stack context) go to the server log so developers can debug. Example:

```go
// In the error response helper (adapter layer):
func respondWithError(w http.ResponseWriter, r *http.Request, logger *slog.Logger, status int, apiErr ErrorObject, internalErr error) {
    // 1. Log the real error for the developer
    logger.Error("request error",
        "request_id", middleware.GetRequestID(r.Context()),
        "status", status,
        "code", apiErr.Code,
        "internal_error", internalErr,
    )
    // 2. Return the sanitized error to the user
    respondJSON(w, status, APIResponse{
        Status: "error",
        Error:  &apiErr,
    })
}
```

---

## 11. DTO Structure

**Three DTO boundaries — all typed structs, no `interface{}` or raw maps:**

1. **Transport DTOs** (adapter/http layer):
   - `CreateUserRequest` — request body with validation tags (`validate:"required,email"`)
   - `UserResponse` — what the API returns, separate from entity
   - Lives in `adapter/http/v1/dto.go`
2. **Use-case DTOs** (usecase layer):
   - `CreateUserInput` — what the use-case accepts
   - `CreateUserOutput` / `UserResult` — what the use-case returns
   - Lives alongside use-case methods in `usecase/user.go`
3. **Entities** (entity layer):
   - `User` — the core entity struct
   - Repository interface accepts/returns entities
   - Lives in `entity/user.go`

**Mapping functions** convert between layers:

- `adapter/http/v1/mapper.go`: `requestToInput()`, `outputToResponse()`
- `usecase/user.go`: works directly with entities via repository interface

Never leak entities into API responses. Never reuse a request DTO as a response DTO.

**Struct change verification rule**: When modifying any request or response struct, you MUST verify that all handlers and methods that use that struct still compile and behave correctly. Check every caller and consumer of the changed struct across all layers (transport -> use-case -> repository). Run `go build ./...` after any struct change.

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

## 14. Makefile

Every project MUST have a `Makefile` at the root. It is the single entry point for all project automation:

```makefile
# --- Project ---
.PHONY: run build
run:              ## Run the server locally
	@go run ./cmd/api
build:            ## Build the binary
	@go build -o bin/api ./cmd/api

# --- Docker ---
.PHONY: up down stop
up:               ## Build and start all services (docker-compose) with live reload
	@docker-compose up -d --build
down:             ## Stop and remove all services
	@docker-compose down
stop:             ## Stop all services without removing them
	@docker compose stop

# --- Database / Migrations ---
# Migrations run on the HOST machine, so DATABASE_URL uses localhost (not the docker service name).
DATABASE_URL ?= postgres://postgres:postgres@localhost:5432/app?sslmode=disable

.PHONY: migrate-up migrate-down migrate-create
migrate-up:       ## Apply all pending migrations
	@migrate -path internal/adapter/postgres/migrations -database "$(DATABASE_URL)" up
migrate-down:     ## Roll back the last migration
	@migrate -path internal/adapter/postgres/migrations -database "$(DATABASE_URL)" down 1
migrate-create:   ## Create a new migration (usage: make migrate-create name=add_users)
	@migrate create -ext sql -dir internal/adapter/postgres/migrations -seq $(name)


# --- Linting ---
.PHONY: lint
lint:             ## Run linter
	@golangci-lint run ./...

# --- Swagger / OpenAPI ---
.PHONY: swagger
swagger:          ## Generate OpenAPI spec from annotations
	@swag init -g cmd/api/main.go -o docs/

# --- Init (full project bootstrap) ---
.PHONY: init
init:             ## Generate docs, start docker, wait for DB, run migrations, print success
	@echo "Generating Swagger documentation..."
	@swag init -g cmd/api/main.go -o docs/
	@echo "Starting Docker services..."
	@docker-compose up -d --build
	@echo "Waiting for PostgreSQL to be ready..."
	@until docker-compose exec -T postgres pg_isready -U $${DB_USER:-postgres} > /dev/null 2>&1; do sleep 1; done
	@echo "Running migrations..."
	@migrate -path internal/adapter/postgres/migrations -database "$(DATABASE_URL)" up
	@echo "Project successfully started!"

# --- Helpers ---
.PHONY: help
help:             ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'
```

---

## 15. Dockerfile, Docker Compose & Live Reload

Every project MUST have a `Dockerfile` and a `docker-compose.yml` for local development. Use `air` for live reload of the Go server.

### Dockerfile

The Dockerfile MUST always be created at the project root. It uses a multi-stage build: a development stage with `air` for live reload, and a production stage with a minimal image.

```dockerfile
# --- Development stage (used by docker-compose for live reload) ---
FROM golang:1.25-alpine AS dev

RUN apk add --no-cache git curl
RUN go install github.com/air-verse/air@latest
RUN go install github.com/swaggo/swag/cmd/swag@latest
RUN go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["air", "-c", ".air.toml"]

# --- Build stage ---
FROM golang:1.25-alpine AS build

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /bin/api ./cmd/api

# --- Production stage ---
FROM alpine:3.19 AS production

RUN apk add --no-cache ca-certificates tzdata

COPY --from=build /bin/api /bin/api

EXPOSE 8080

ENTRYPOINT ["/bin/api"]
```

### docker-compose.yml

```yaml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    ports:
      - "${APP_PORT:-8080}:8080"
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy

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

**Key points:**

- The `target: dev` directive builds the development stage from the Dockerfile, which has `air` pre-installed and configured as the default `CMD`.
- No explicit `command` override is needed — the dev stage's `CMD ["air", "-c", ".air.toml"]` handles live reload automatically.
- The volume mount `.:/app` ensures source file changes on the host trigger `air` rebuilds inside the container.
- After `make up`, the container builds, starts, and immediately begins watching for file changes with live reload.

### .air.toml (live reload)

```toml
root = "."
tmp_dir = "tmp"

[build]
cmd = "migrate -path internal/adapter/postgres/migrations -database \"$DATABASE_URL\" up && swag init -g cmd/api/main.go -o docs/ && go build -o ./tmp/api ./cmd/api"
bin = "tmp/api"
watch_dir = ["cmd", "internal", "pkg"]
include_ext = ["go", "toml", "yaml", "yml", "sql"]
exclude_dir = ["tmp", "vendor", "node_modules", "docs"]
delay = 1000
```

**Key points:**

- The `cmd` chains three steps: (1) run pending migrations, (2) regenerate Swagger docs, (3) build the binary. On every restart (including file changes), migrations are applied first.
- `sql` is included in `include_ext` so that new migration files trigger a rebuild and migration run.
- `docs` is added to `exclude_dir` to prevent infinite rebuild loops (since `swag init` writes to `docs/`).
- The dev Dockerfile must install both `swag` and `migrate` (see Dockerfile dev stage).

---

## 16. Environment Variables (.env)

Every project MUST have:

- `.env` — actual environment variables for local dev (gitignored)
- `.env.example` — template with placeholder values (committed to git)

When setting up the project, always create `.env.example`. If any env var value is unknown (secrets, external service URLs, API keys), add a `# TODO: fill in` comment and notify the user that they must fill it in before running the project.

### .env.example

The `.env` file is used by docker-compose and the containerized app. The `DB_HOST` MUST use the **docker-compose service name** (e.g., `postgres`) so the API container can reach the database over the internal Docker network.

```env
# --- App ---
APP_PORT=8080
APP_ENV=development

# --- Database (PostgreSQL) ---
# DB_HOST uses the docker-compose service name for internal Docker network connectivity
DB_HOST=postgres
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

**Important**: The `DATABASE_URL` in `.env` targets the Docker internal network (`DB_HOST=postgres`). The Makefile's `migrate-*` targets run on the **host machine** and MUST use `localhost` instead. See the Makefile section for the correct `DATABASE_URL` override.

---

## 17. Authentication (Keycloak)

If the project uses authentication, the identity provider is **Keycloak**. Always.

- Keycloak runs as a service in `docker-compose.yml` (add when auth is needed)
- JWT tokens issued by Keycloak are validated in the auth middleware
- Use the Keycloak OIDC well-known endpoint for key discovery: `{KEYCLOAK_URL}/realms/{realm}/.well-known/openid-configuration`
- Roles come from Keycloak realm/client roles, mapped to Go RBAC in the authorization middleware
- If Keycloak credentials or URLs are unknown, add them to `.env.example` with `# TODO: fill in` and notify the user

---

## 18. Database (PostgreSQL)

The database is always **PostgreSQL**.

- Use `pgx` as the driver (preferred) or `database/sql` with `lib/pq`
- Connection pooling: configure `max_open_conns`, `max_idle_conns`, `conn_max_lifetime` in config
- Migrations target PostgreSQL syntax
- Connection string format: `postgres://user:pass@host:port/dbname?sslmode=disable`

---

## 19. Swagger / OpenAPI Documentation

Every API handler MUST have `swaggo/swag` annotations for automatic OpenAPI spec generation.

### Required annotations

**Main entry point** (`cmd/api/main.go`):

```go
// @title           My API
// @version         1.0
// @description     API server description
// @host            localhost:8080
// @BasePath        /api/v1
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
func main() { ... }
```

**Important**: The `@BasePath` MUST include the API version prefix (e.g., `/api/v1`). This ensures Swagger UI displays the correct full paths (e.g., `/api/v1/users` instead of `/api/users`). When adding a new API version, update `@BasePath` accordingly or use separate Swagger specs per version.

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

### Docs import (required)

The `cmd/api/main.go` file MUST always contain a blank import of the generated `docs` package, even if the `docs/` directory has not been created yet. This import is required for `swag` to register the generated spec at runtime:

```go
import (
	// ... other imports ...

	_ "<module>/docs" // Swagger generated docs — MUST always be present
)
```

Replace `<module>` with the actual module path from `go.mod` (e.g., `_ "myapp/docs"`). Never remove this import, even if the IDE or linter reports it as unused before the first `swag init` run. The `docs/` package will be generated by `make swagger` or `make init`.

### Swagger UI endpoint

The project MUST **always** register the Swagger UI route — it is not optional and must never be removed or guarded behind a feature flag or environment check. The Swagger route is a permanent part of every project's router setup:

```go
import httpSwagger "github.com/swaggo/http-swagger"

// This route MUST always be registered — never conditionally skip it.
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
