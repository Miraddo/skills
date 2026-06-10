---
name: go-observability
description: Instrument Go (golang) services with the three pillars — structured logging via log/slog (JSON, levels, trace/request IDs), distributed tracing and metrics via OpenTelemetry, Prometheus /metrics, context propagation, and graceful exporter shutdown. Use when adding logging, metrics, tracing, or health checks to a Go service.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Observability

Make a service debuggable in production via the **three pillars**: logs,
metrics, traces — correlated by IDs. Especially important for
[[go-microservices]]. Pairs with [[go-http-api]] (middleware) and
[[go-concurrency]] (graceful shutdown).

## Logs: `log/slog` (stdlib, structured)
`log/slog` (stable since Go 1.21) is the standard. Emit **structured JSON**,
not `fmt.Printf` lines.

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

slog.InfoContext(ctx, "user created",
    "user_id", id, "request_id", reqID)
```
Rules:
- **One queryable event per line** (`JSONHandler`); key/value attributes, not
  string interpolation.
- Use **levels** (Debug/Info/Warn/Error); make the threshold configurable.
- Attach **stable fields** that stitch the pillars together: `request_id`,
  `trace_id`, `span_id`. Inject them in middleware and pass `ctx` so
  `*Context` log calls pick them up.
- **Never log secrets/PII**; do redaction in a custom handler ([[go-security]]).
- Pass a `*slog.Logger` via context or as a dependency; avoid scattered
  globals beyond `slog.Default()`.

## Traces & metrics: OpenTelemetry (vendor-neutral)
OTel is the standard instrumentation API/SDK for **traces** and **metrics**
(and increasingly logs). Wire it once at startup:

1. Build a `TracerProvider` / `MeterProvider` with an OTLP exporter and a
   `resource` describing the service (`service.name`, version, env).
2. **Set up context propagation** (`otel.SetTextMapPropagator` with W3C
   `tracecontext` + `baggage`) so trace IDs cross service boundaries.
3. **Auto-instrument** HTTP/gRPC with `otelhttp` / `otelgrpc` middleware
   instead of hand-rolling spans — minimal code pollution.
4. Bridge logs with **`otelslog`**: it implements `slog.Handler`, so your
   existing `slog` calls flow through the OTel Logs SDK with trace context
   attached automatically.

```go
handler := otelhttp.NewHandler(mux, "api")     // spans per request
// inside handlers, child spans:
ctx, span := tracer.Start(ctx, "GetUser")
defer span.End()
span.SetAttributes(attribute.Int64("user.id", id))
if err != nil { span.RecordError(err); span.SetStatus(codes.Error, "lookup failed") }
```

**Graceful exporter shutdown**: defer `tp.Shutdown(ctx)` / `mp.Shutdown(ctx)`
on SIGTERM so buffered spans/metrics flush before exit ([[go-concurrency]]).

## Metrics: Prometheus
Expose a **`/metrics`** endpoint scraped by Prometheus. Either use the
OTel metrics SDK with a Prometheus exporter, or `prometheus/client_golang`
directly. Track the **RED** signals for request handlers (Rate, Errors,
Duration) and **USE** for resources; plus DB pool stats ([[go-database]]),
queue depths, and Go runtime metrics. Use a histogram for latencies, not a
gauge.

## Health & readiness
Add `/healthz` (liveness) and `/readyz` (readiness — checks DB/deps) endpoints
for Kubernetes-style probes. Keep liveness cheap; readiness reflects whether
the service can actually serve.

## Correlation is the whole point
A request flows: middleware generates/extracts `request_id` + `trace_id` →
puts them in `ctx` → logs (`slog` via ctx), spans (OTel), and metric exemplars
all carry the same IDs → you can pivot from a slow metric to its trace to its
logs. Wire the IDs once in middleware; everything downstream inherits them via
`context`.

## Anti-patterns
- `fmt.Println`/unstructured logs in services.
- Logging secrets, tokens, full request bodies, or PII.
- Manual spans everywhere instead of auto-instrumentation middleware.
- No exporter shutdown → lost telemetry on deploy.
- Metrics with unbounded label cardinality (e.g. user ID as a label).
- Treating logs as the only tool — add metrics + traces for distributed flows.

## References
- [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/)
- [log/slog package](https://pkg.go.dev/log/slog)
- [OpenTelemetry-Native Logging in Go with otelslog](https://www.dash0.com/guides/opentelemetry-logging-in-go)
- [prometheus/client_golang](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus)
