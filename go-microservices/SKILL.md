---
name: go-microservices
description: Design and structure Go (golang) microservices — independent deployables with their own data, gRPC/REST boundaries, proto contracts, config, structured logging, graceful shutdown, and observability. Use when splitting into or building independently deployable Go services.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Microservices Architecture

Microservices are **many independently deployable services**, each owning its
own data and lifecycle, communicating over the network. They buy independent
scaling, deploy cadence, and clear ownership — at the cost of real
operational and distributed-systems complexity. See [[go-structure]] and
[[go-commands]] for the per-service mechanics.

## Earn it first
> Prove value with a [[go-monolith]] (or [[go-modular-monolith]]) first, then
> split out services deliberately.

Only accept microservice overhead when you hit concrete limits:
- Slow/blocked deploys because everything ships together.
- Unmanageable code ownership across one big codebase.
- Parts that must scale independently (one hot path, the rest idle).

A [[go-modular-monolith]] with facades + an event bus is the cheapest way to
get the boundaries right *before* you pay the network tax — extract a module
into a service when it genuinely needs independence.

## Service boundaries & granularity
- **One service = one autonomous responsibility** (a bounded context), owning
  **its own database** — no shared DB, no cross-service joins. Data is reached
  only through the owning service's API.
- Balance granularity: **too few** services and you lose independence; **too
  many** and you drown in operational overhead. Start coarse; split further
  only under pressure.
- Services must be **loosely coupled** — a change in one shouldn't force a
  lockstep deploy of another.

## Per-service layout

Each service is its own Go module / repo, structured like a small monolith:

```
order-service/
  go.mod
  cmd/
    server/main.go        # entrypoint: config, deps, start gRPC+HTTP, shutdown
  internal/
    config/               # env-based config
    domain/               # models + business logic
    repository/           # this service's OWN data store
    transport/
      grpc/               # gRPC server impl
      http/               # REST gateway / health endpoints
    platform/             # logger, db, telemetry, middleware
  api/
    proto/                # .proto contracts (source of truth)
      order/v1/order.proto
  migrations/
  Dockerfile
  Makefile                # proto gen, build, test, lint, docker targets
```

Contracts (`.proto` or OpenAPI) are versioned (`order/v1/`) and are the
**single source of truth** for the service's interface — generate client/
server code from them, don't hand-write the wire types.

## Communication
- **gRPC for service-to-service**: Protocol Buffers over HTTP/2 — ~7–10×
  faster than JSON/REST, with multiplexing and a typed contract. Default for
  internal east-west traffic.
- **REST/JSON for external/public APIs** and simple integrations — easy to
  test and consume. Often a gateway translates REST → gRPC at the edge.
- **Async events (message broker)** for "something happened" fan-out — keeps
  producers and consumers decoupled. Use when no immediate response is needed.

Pick sync (gRPC) when the caller needs a result now; async (events) to
decouple lifecycles and absorb load.

## Production essentials (every service)
- **Config** from environment (12-factor); never hardcode endpoints/secrets.
- **Structured logging** (JSON, e.g. `slog`) with a request/correlation ID
  threaded via `context.Context`.
- **Graceful shutdown**: `signal.NotifyContext(ctx, os.Interrupt,
  syscall.SIGTERM)`; stop accepting new requests, drain in-flight, close DB.
- **Health checks**: liveness + readiness endpoints (gRPC health protocol or
  `/healthz`, `/readyz`).
- **Timeouts, retries, deadlines** on every outbound call — propagate the
  incoming context's deadline.
- **Resilience**: circuit breakers / backoff for downstream failures so one
  slow dependency doesn't cascade.
- **Observability**: metrics (Prometheus) + distributed tracing (OpenTelemetry)
  + logs, correlated by trace ID.
- **Containerized** (Dockerfile) with a `Makefile` for proto-gen, build, test,
  lint, and image targets.

## Graceful shutdown sketch

```go
func main() {
    cfg := config.Load()
    ctx, stop := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer stop()

    srv := grpc.NewServer(/* interceptors: logging, recovery, tracing */)
    orderpb.RegisterOrderServiceServer(srv, order.NewServer(deps))

    go func() {
        lis, _ := net.Listen("tcp", cfg.Addr)
        srv.Serve(lis)
    }()

    <-ctx.Done()                 // SIGINT/SIGTERM received
    srv.GracefulStop()           // drain in-flight RPCs, then return
}
```

## Cross-cutting concerns
- **API gateway** at the edge: routing, auth, rate limiting, REST↔gRPC.
- **Service discovery** / a registry (or platform DNS like Kubernetes
  services) so services find each other without hardcoded hosts.
- **Schema/versioning discipline**: evolve protos backward-compatibly (add
  fields, never renumber/reuse tags); version the package (`v1`, `v2`).
- **CI/CD per service** so each deploys independently.

## Anti-patterns
- **Distributed monolith**: services that must deploy together / share a DB —
  worst of both worlds. If this is you, go back to [[go-modular-monolith]].
- A shared database across services.
- Synchronous call chains 5 services deep (latency + failure amplification) —
  prefer async events or aggregation.
- Premature decomposition before the domain boundaries are proven.
- Hand-rolled wire types instead of generated proto/OpenAPI code.

## References
- [Go Microservices in 2025: Architecture, gRPC vs REST & Frameworks](https://medium.com/@QuarkAndCode/go-microservices-in-2025-architecture-grpc-vs-rest-frameworks-09159c95a8d0)
- [Architecting a Maintainable Go Microservice: Structure, Logging, Reliability](https://dev.to/sagarmaheshwary/go-microservices-boilerplate-series-from-hello-world-to-production-part-1-46k5)
- [Building Production-Grade Microservices with Go and gRPC](https://dev.to/nikl/building-production-grade-microservices-with-go-and-grpc-a-step-by-step-developer-guide-with-example-2839)
