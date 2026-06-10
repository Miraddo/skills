---
name: go-modular-monolith
description: Design a Go (golang) modular monolith — independent domain modules with private internals, a single public facade each, an in-process event bus, and a composition root. Use when building one deployable that needs enforced module boundaries, or when evolving a monolith toward extractable services.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Modular Monolith

A modular monolith is **one deployable** built from **independent modules**
with enforced boundaries. It is the sweet spot between a plain
[[go-monolith]] (simple but tends toward a big ball of mud) and
[[go-microservices]] (independent but operationally heavy) — you get module
isolation without distributed-systems pain. See [[go-structure]] for layout
mechanics.

## The core idea

> A **module** owns its domain, exposes a **minimal public interface (a
> facade)**, and never leaks its internals.

Treat each module as if it were a separate application that happens to run
in the same process. Modules may NOT reach into each other's
repositories/services/models — they talk only through published facades or
events.

## Layout

```
myapp/
  go.mod
  cmd/myapp/main.go          # entrypoint: calls app.New(), starts server
  internal/
    app/                     # COMPOSITION ROOT — builds & wires all modules
      app.go
    platform/                # shared infra: db, logger, http, event bus
      eventbus/
      database/
      httpserver/
    modules/
      user/
        user.go              # PUBLIC FACADE — the only exported surface
        events.go            # integration events this module publishes
        internal/            # PRIVATE — compiler-blocked from other modules
          model.go
          repository.go
          service.go
          handler.go
      order/
        order.go             # public facade
        events.go
        internal/
          ...
```

The key mechanic: each module keeps its guts in its **own nested
`internal/`** directory. Go's compiler forbids any package outside
`modules/order/` from importing `modules/order/internal/...`. Boundaries are
enforced by the toolchain, not by convention or code review.

## The four rules of a module

1. **One public facade.** Each module exposes a single small interface (e.g.
   `user.Module` or a set of methods) — the only thing other modules and the
   composition root may call. Everything else lives under `internal/`.
2. **Private internals.** Models, repositories, services, handlers are
   unexported and live in the module's `internal/`. No external import path
   reaches them.
3. **Depend on interfaces.** A module that needs another's capability accepts
   it as a small interface injected at construction — never imports the other
   module's concrete types.
4. **Communicate via facade calls or events.** Synchronous need → call the
   other module's facade method. Reaction to "something happened" →
   publish/subscribe on the in-process event bus. Never share tables.

## Composition root (`internal/app`)

`internal/app` is the only place that knows concrete implementations. It
instantiates every module, injects dependencies (as interfaces), and wires
the event bus. `main.go` just calls `app.New(...)` and runs the server.

```go
// internal/app/app.go
func New(cfg config.Config) (*App, error) {
    db  := database.MustOpen(cfg.DatabaseURL)
    bus := eventbus.New()
    r   := httpserver.NewRouter()

    // each module gets its deps as interfaces + the bus
    userMod  := user.New(db, bus)
    orderMod := order.New(db, bus, userMod)  // order depends on user's facade

    userMod.RegisterRoutes(r)
    orderMod.RegisterRoutes(r)

    return &App{router: r, db: db}, nil
}
```

## In-process event bus (decoupling)

For "X happened, others may care" use a lightweight in-process pub/sub —
**no external broker**. Modules publish integration events; subscribers
react. This decouples module lifecycles and is the seam along which you'd
later swap in a real message broker if a module becomes a microservice.

```go
bus.Subscribe("user.registered", func(ctx context.Context, e Event) error {
    return welcome.Send(ctx, e.UserID)
})
// in user module, after creating a user:
bus.Publish(ctx, Event{Name: "user.registered", UserID: id})
```

Prefer async events over synchronous facade calls when the caller doesn't
need a result — it keeps coupling low.

## Drawing module boundaries
- Use **DDD bounded contexts**: one module per sub-domain (billing, catalog,
  identity), grouping things that change together and share a ubiquitous
  language.
- A good module is **cohesive inside, loosely coupled outside** — most calls
  stay within the module; few cross it.
- If two modules constantly call each other, the boundary is wrong — merge
  them or extract the shared concept into a third module.

## Why this is worth it
- **Scales team size** without distributed-systems overhead — modules can be
  owned by different people/teams.
- **Independently testable** modules (inject fakes for the interfaces).
- **Clean extraction path**: a module with a facade + events boundary becomes
  a microservice by replacing in-process calls with RPC and the in-process
  bus with a broker — see [[go-microservices]].

## Anti-patterns
- Importing another module's `internal/` (the compiler stops this — don't
  fight it by flattening internals up).
- A shared `common`/`shared` module that everything depends on (becomes a
  hidden coupling hub) — keep truly-shared code minimal in `platform/`.
- Cross-module database joins / shared tables.
- Building all the module ceremony for a 3-endpoint CRUD app — use a plain
  [[go-monolith]] until modules earn their keep.

## References
- [Designing a Modular Monolith in Go — Structure, Boundaries, Patterns](https://daveamit.com/posts/2026-02-13-modular-monolith/)
- [GO Modular Monolith Part I](https://medium.com/@arkjuniork.yudistira/go-modular-monolith-part-i-f963da742e81)
- [Building Modular Monoliths with Logical Boundaries & Internal Messaging](https://www.softwareseni.com/building-modular-monoliths-with-logical-boundaries-hexagonal-architecture-and-internal-messaging/)
