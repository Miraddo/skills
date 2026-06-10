---
name: go-monolith
description: Design and structure a single-deployable Go (golang) monolith — layered/clean-ish architecture, one binary, shared database, model/repository/service/handler layering. Use when building or organizing a Go app that ships as one service before splitting into modules or microservices.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Monolith Architecture

A monolith is **one codebase, one binary, one database, one deployment**. It
is the correct default for most new Go services — start here, earn the
complexity of [[go-modular-monolith]] or [[go-microservices]] only when real
pressure demands it. See [[go-structure]] for the directory mechanics and
[[go-commands]] for the build/test workflow.

## When a monolith is the right call
- New product / unproven domain — boundaries aren't known yet.
- Small team (1–8 devs) sharing the codebase.
- You want a single transaction across the whole data model.
- Simple deploy and local dev are worth more than independent scaling.

Move off a monolith only when you hit concrete limits: slow deploy cadence,
unclear code ownership, or parts that must scale independently.

## Go-flavored layering (keep it simple)

**Do not cargo-cult full Clean Architecture into Go** — the interface/DI
ceremony usually adds more friction than value here. Use a thin, pragmatic
layering and Go's `internal/` to enforce a public surface.

Per feature/domain package, the conventional files:

| File | Responsibility |
|------|----------------|
| `model.go` | Domain types / DB models. |
| `repository.go` | Data access. Define a **repository interface** + a concrete (e.g. Postgres) impl. |
| `service.go` | Business logic. Depends on the repository *interface*, not the impl. |
| `handler.go` | Transport (HTTP/gRPC) — decode request, call service, encode response. |
| `router.go` | Route wiring for this feature. |

**Dependency rule (the one that matters)**: transport → service →
repository. Inner layers never import outer layers. The handler knows about
the service; the service never imports `net/http`.

## Recommended layout

```
myapp/
  go.mod
  cmd/
    myapp/
      main.go              # thin: load config, build deps, start server
  internal/
    config/                # env/flags loading
    platform/              # shared infra: db, logger, http server, middleware
      database/
      httpserver/
    user/                  # feature package
      model.go
      repository.go
      service.go
      handler.go
      router.go
    order/                 # another feature
      ...
  migrations/              # SQL schema migrations
  api/                     # OpenAPI / proto contracts (optional)
```

Why this works:
- **`cmd/myapp/main.go` is thin** — it is the composition root: load config,
  open the DB, construct each feature's repo→service→handler, mount routes,
  start the server, handle graceful shutdown. No business logic.
- **`internal/`** prevents anyone outside the module importing your guts and
  keeps the public surface honest.
- **`internal/platform`** holds cross-cutting infra (db pool, logger,
  middleware) so feature packages depend on it, not the reverse.
- Organize by **feature/domain** (`user/`, `order/`), not by technical layer
  (`controllers/`, `services/`) — domain grouping keeps related code together
  and is the natural seam if you later extract a module.

## Composition root sketch (`main.go`)

```go
func main() {
    cfg := config.Load()
    log := platform.NewLogger(cfg)

    db, err := database.Open(cfg.DatabaseURL)
    if err != nil { log.Fatal(err) }
    defer db.Close()

    // wire each feature: repo -> service -> handler
    userRepo := user.NewPostgresRepository(db)
    userSvc  := user.NewService(userRepo)
    userAPI  := user.NewHandler(userSvc)

    r := httpserver.NewRouter()
    user.RegisterRoutes(r, userAPI)
    // order.RegisterRoutes(r, ...)

    srv := httpserver.New(cfg.Addr, r)
    // graceful shutdown on SIGINT/SIGTERM
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    srv.Run(ctx)
}
```

## Practices to enforce
- **One shared database**, but give each feature its own tables and access
  it only through that feature's repository — no cross-feature SQL. This
  discipline is what makes a future split painless.
- **Interfaces at the consumer**: define the repository interface in the
  package that *uses* it (the service), keep it small. Accept interfaces,
  return structs.
- **Context everywhere**: pass `context.Context` as the first arg through
  handler → service → repository for cancellation and deadlines.
- **Graceful shutdown** via `signal.NotifyContext`.
- **Config from env/flags**, never hardcoded; secrets out of source.
- Keep features from importing each other directly; if they must talk, go
  through a service interface — that's the bridge to [[go-modular-monolith]].

## Anti-patterns
- A giant `models/` and `utils/` dump shared by everything → tight coupling.
- Business logic in `main.go` or in HTTP handlers.
- Splitting into microservices before the domain boundaries are proven.
- Full hexagonal/Clean ceremony on a small app.

## References
- [Building a monolith in Go — Layout](https://dev.to/daunderworks/building-a-monolith-in-go-layout-1no8)
- [Why Clean Architecture struggles in Go (and what works better)](https://dev.to/lucasdeataides/why-clean-architecture-struggles-in-golang-and-what-works-better-m4g)
- [go-monolith-example (powerman)](https://github.com/powerman/go-monolith-example)
