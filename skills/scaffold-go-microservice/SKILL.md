---
name: scaffold-go-microservice
description: Scaffold a production-ready Go HTTP microservice — chi router + sqlc + golang-migrate + structured logging + OpenTelemetry + healthchecks + Dockerfile + GitHub Actions. Use when starting a new Go service, not when adding structure to an existing one.
---

# Scaffold a Go Microservice

The opinionated "this is the layout you'd land on after building 3 Go services" scaffold. No frameworks beyond `chi`. No code generators that create magic. Idiomatic Go that an experienced reader recognizes immediately.

## When to use

- New Go service, day 1.
- Internal service or public API; HTTP/JSON.
- You want sqlc (typed SQL) over GORM (ORM with reflection cost).
- You'll deploy to Kubernetes or Cloud Run / Fly / Railway.

## When NOT to use

- gRPC-first service → use `grpc-services` skill (different scaffold).
- Tiny CLI tool → use `scaffold-cli-tool`.
- Adding structure to an existing service — different problem.
- You want a heavy framework (Gin, Fiber, Echo) — this scaffold is `net/http` + chi.

## Decisions made for you

| Decision | Choice | Why |
|---|---|---|
| Router | `go-chi/chi` | Stdlib-shaped middleware, fastest mental model |
| DB access | `sqlc` (codegen from `.sql`) | Compile-time SQL safety, no ORM magic |
| Migrations | `golang-migrate` | Dialect-agnostic, plays well with CI |
| Config | `kelseyhightower/envconfig` | Struct + tags > viper for service-shaped configs |
| Logging | `slog` (stdlib, Go 1.21+) | Structured, levelled, zero deps |
| Tracing/Metrics | OpenTelemetry SDK + OTLP exporter | One vendor, all signals |
| Errors | `errors.Is/As` + sentinel errors | No `pkg/errors`, no custom error stacks |
| Tests | stdlib `testing` + `testify/require` | Familiar, minimal |
| Docker | distroless base, multi-stage | 20MB images, no shell |
| CI | GitHub Actions | Most teams have it |

## File structure

```
my-service/
  cmd/
    server/main.go              # entrypoint, wires deps
  internal/
    api/                        # HTTP handlers (one file per resource)
      health.go
      user.go
      middleware.go
    domain/                     # business logic, no HTTP/DB types
      user.go
    db/
      queries/                  # *.sql files for sqlc
      migrations/               # *.up.sql / *.down.sql
      sqlc.yaml
      generated/                # sqlc output (committed)
    config/
      config.go                 # envconfig struct
    obs/
      logger.go
      tracing.go
  Dockerfile
  docker-compose.yml            # local postgres + otel collector
  Makefile                      # run, test, lint, sqlc, migrate
  .golangci.yml
  go.mod
  README.md
  .github/workflows/ci.yml
```

**`internal/` over `pkg/`** — these are private to the service. `pkg/` invites accidental public APIs.

## main.go pattern (the whole service in one readable file)

```go
// cmd/server/main.go
package main

import (
    "context"
    "errors"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/kelseyhightower/envconfig"

    "myservice/internal/api"
    "myservice/internal/config"
    "myservice/internal/db/generated"
    "myservice/internal/obs"
)

func main() {
    if err := run(); err != nil {
        slog.Error("fatal", "err", err)
        os.Exit(1)
    }
}

func run() error {
    var cfg config.Config
    if err := envconfig.Process("", &cfg); err != nil {
        return err
    }

    logger := obs.NewLogger(cfg.Env)
    slog.SetDefault(logger)

    shutdownTracing, err := obs.SetupTracing(context.Background(), cfg)
    if err != nil {
        return err
    }
    defer shutdownTracing()

    pool, err := pgxpool.New(context.Background(), cfg.DatabaseURL)
    if err != nil {
        return err
    }
    defer pool.Close()

    queries := generated.New(pool)

    r := chi.NewRouter()
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(api.RequestLogging(logger))
    r.Use(middleware.Recoverer)
    r.Use(middleware.Timeout(30 * time.Second))

    api.MountHealth(r, pool)
    api.MountUsers(r, queries)

    srv := &http.Server{
        Addr:              cfg.Addr,
        Handler:           r,
        ReadHeaderTimeout: 5 * time.Second,
    }

    go func() {
        slog.Info("listening", "addr", cfg.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("listen failed", "err", err)
            os.Exit(1)
        }
    }()

    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
    <-stop
    slog.Info("shutting down")

    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()
    return srv.Shutdown(ctx)
}
```

A new dev should read this and immediately know: where config comes from, where the DB connects, where routes mount, how shutdown works. No framework magic.

## sqlc — the killer feature

`internal/db/queries/users.sql`:
```sql
-- name: GetUser :one
SELECT * FROM users WHERE id = $1;

-- name: CreateUser :one
INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *;

-- name: ListActiveUsers :many
SELECT * FROM users WHERE deleted_at IS NULL ORDER BY created_at DESC LIMIT $1 OFFSET $2;
```

`make sqlc` generates `internal/db/generated/users.sql.go` with typed Go functions:

```go
user, err := queries.GetUser(ctx, userID)            // user is users.User struct
created, err := queries.CreateUser(ctx, generated.CreateUserParams{
    Email: "x@y.com",
    Name:  pgtype.Text{String: "Mahesh", Valid: true},
})
```

If your SQL has a typo, `make sqlc` fails at *codegen time*, not runtime. ORMs ship the typo.

## Healthchecks (the version K8s wants)

```go
// internal/api/health.go
func MountHealth(r chi.Router, pool *pgxpool.Pool) {
    // Liveness: are we running? Cheap.
    r.Get("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    // Readiness: can we serve traffic? Checks deps.
    r.Get("/readyz", func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()
        if err := pool.Ping(ctx); err != nil {
            http.Error(w, "db down", http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    })
}
```

K8s docs and most blog posts conflate these. They are **different signals**: liveness restarts the pod; readiness yanks it from the load balancer. Get them right or you'll have a service that never gets traffic, or a service that thrashes-restarts forever.

## Dockerfile (distroless, multi-stage)

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

20MB image, no shell, no package manager, runs as nonroot. Audit findings shrink dramatically.

## What the scaffold emits

1. The directory tree above.
2. `cmd/server/main.go` ready to run with example `users` resource.
3. sqlc.yaml + sample query + matching migration.
4. Dockerfile + docker-compose with Postgres + OTel collector.
5. Makefile with `run`, `test`, `lint`, `sqlc`, `migrate-up`, `migrate-down`, `migrate-create`.
6. `.golangci.yml` with sane defaults (errcheck, govet, gosimple, ineffassign, staticcheck, gosec).
7. GitHub Actions: lint, test (with race detector), sqlc verify, migration verify, docker build.
8. README with the "first 5 minutes" runbook.

## Anti-patterns

- **GORM/Ent for new services** — reflection cost, query opacity, and you'll fight it within 6 months. `sqlc` instead.
- **`pkg/` instead of `internal/`** — accidentally publishes types. Use `internal/`.
- **Heavy frameworks (Gin, Fiber, Echo)** — they wrap `net/http` for ergonomics you don't need. Stdlib + chi is enough.
- **Initializing OTel without graceful shutdown** — drops in-flight spans. The defer matters.
- **One `/health`** for both liveness and readiness — guarantees a bad K8s deployment day. Split them.
- **Logging `fmt.Println` / `log.Printf`** — unstructured logs are unsearchable in production. `slog` everywhere.
- **`time.Now()` in domain code** — untestable. Inject a `Clock` interface, default to real time.
- **Goroutines without context cancellation** — leaks on shutdown. Every `go func()` accepts a `ctx`.
- **DB queries without `ctx`** — request timeout doesn't propagate; pool exhaustion under load.

## Verify it worked

- [ ] `make run` starts the service against the docker-compose Postgres; `curl /healthz` returns 200.
- [ ] Killing the service with `Ctrl+C` shuts down cleanly within 15s (no panics, no leaked goroutines).
- [ ] `make sqlc` regenerates Go from SQL with no diff (or expected diff if SQL changed).
- [ ] `make test` passes with `-race` enabled.
- [ ] `make lint` runs golangci-lint with zero issues on the scaffold.
- [ ] Docker image builds < 30s, final size < 30MB.
- [ ] `/readyz` returns 503 when Postgres is killed; `/healthz` keeps returning 200.
- [ ] An OTel trace shows the full request path: HTTP → DB query, with parent-child relationship intact.
- [ ] CI green on first push.
