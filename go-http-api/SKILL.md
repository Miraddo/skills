---
name: go-http-api
description: Build HTTP/REST APIs in Go (golang) with the standard library — net/http, Go 1.22+ ServeMux routing (methods, wildcards, PathValue), middleware chains, JSON decode/encode, request validation, handler patterns, and timeouts. Use when writing HTTP servers, REST endpoints, routers, or middleware.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go HTTP / REST APIs

Build production HTTP services with the **standard library** — since Go 1.22,
`net/http` is enough for routing, methods, path params, and middleware. Reach
for a third-party router (chi, echo, gin) only when you need a specific
feature the stdlib lacks. Pairs with [[go-idioms]], [[go-monolith]],
[[go-microservices]], and [[go-testing]] (httptest).

## Routing (Go 1.22+ ServeMux)
The enhanced `http.ServeMux` supports **method matching** and **wildcards** —
express routes as patterns, not handler-body `if`s.

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUser)        // method + path param
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("GET /files/{path...}", serveFile) // trailing wildcard = rest of path

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")                        // extract path param
    ...
}
```
- Unmatched method on a known path → automatic **405 Method Not Allowed**.
- Specificity wins: `/users/{id}` vs `/users/me` — the more specific pattern
  is chosen.
- Subroute / group by mounting a mux with `StripPrefix` or composing handlers.

## Handler signature & the error-return pattern
The stdlib handler returns nothing, which makes error handling repetitive. A
common idiom is a handler type that returns `error`, wrapped to satisfy
`http.Handler`:

```go
type apiHandler func(http.ResponseWriter, *http.Request) error

func (h apiHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := h(w, r); err != nil {
        writeError(w, err)        // central error → status + JSON body mapping
    }
}
```
Keep handlers thin: decode → validate → call a service ([[go-monolith]]
layering) → encode. No business logic in the handler.

## Middleware
Middleware is `func(http.Handler) http.Handler` — it wraps a handler, can act
before/after, and **chains** because input and output are both `http.Handler`.

```go
func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        slog.Info("request", "method", r.Method, "path", r.URL.Path,
            "dur", time.Since(start))
    })
}

// chain helper
func Chain(h http.Handler, mw ...func(http.Handler) http.Handler) http.Handler {
    for i := len(mw) - 1; i >= 0; i-- { h = mw[i](h) }
    return h
}
srv := Chain(mux, Recover, Logging, Auth, CORS)
```
Essential middleware: **Recover** (defer/recover → 500, prevents a panic
killing the server), **Logging**, **Auth** (validate `Authorization: Bearer`),
**CORS** (handle `OPTIONS` preflight), request-ID/trace injection. Store
per-request values via `r.Context()` with a private key type — never globals.

## JSON decode / encode + validation
```go
func createUser(w http.ResponseWriter, r *http.Request) error {
    var in CreateUserRequest
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()                 // reject stray fields
    if err := dec.Decode(&in); err != nil {
        return badRequest("invalid json: %w", err)
    }
    if err := in.Validate(); err != nil {       // validate BEFORE use
        return badRequest("%w", err)
    }
    ...
    return writeJSON(w, http.StatusCreated, out)
}

func writeJSON(w http.ResponseWriter, status int, v any) error {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    return json.NewEncoder(w).Encode(v)
}
```
- Separate **request / response / domain** types — don't expose DB models on
  the wire.
- Validate every input (it's an attack surface — see [[go-security]]).
  Hand-written `Validate()` methods are idiomatic; `go-playground/validator`
  (struct tags) is the common library if you want declarative rules.
- Always set `Content-Type` and an explicit status.

## Server hardening (don't skip)
A bare `http.ListenAndServe` has **no timeouts** — a slow client can pin a
connection forever. Always configure an explicit server:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           rootHandler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      15 * time.Second,
    IdleTimeout:       60 * time.Second,
    MaxHeaderBytes:    1 << 20,
}
```
- **Graceful shutdown**: `signal.NotifyContext` → `srv.Shutdown(ctx)` to
  drain in-flight requests (see [[go-concurrency]]).
- Limit body size with `http.MaxBytesReader`.
- Terminate TLS (or sit behind a proxy that does); set security headers
  (`Strict-Transport-Security`, `X-Content-Type-Options`) — [[go-security]].

## Testing handlers
Use `net/http/httptest`: `httptest.NewRecorder()` for unit tests,
`httptest.NewServer` for integration. Details in [[go-testing]].

## Anti-patterns
- Business logic / DB calls inside handlers.
- `ListenAndServe` with no timeouts.
- Returning raw DB structs as JSON.
- Reaching for gin/echo before you've hit a real stdlib limitation.
- Panicking on bad input instead of returning a 4xx.

## References
- [Routing Enhancements for Go 1.22](https://go.dev/blog/routing-enhancements)
- [Making and Using HTTP Middleware — Alex Edwards](https://www.alexedwards.net/blog/making-and-using-middleware)
- [Which Go Router Should I Use? — Alex Edwards](https://www.alexedwards.net/blog/which-go-router-should-i-use)
- [net/http package](https://pkg.go.dev/net/http)
