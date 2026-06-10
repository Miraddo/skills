---
name: go-concurrency
description: Write correct Go (golang) concurrency — goroutines with clear lifecycles, channels, context cancellation, sync primitives, errgroup, worker pools, and avoiding goroutine leaks, data races, and deadlocks. Use when writing or reviewing concurrent Go code.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Concurrency

Concurrency in Go is cheap to start and easy to get wrong. The discipline:
**every goroutine has a clear owner and a clear exit**, communicate by
passing data rather than sharing memory, and use `context` for lifecycle.
Pairs with [[go-idioms]] and [[go-commands]] (always test with `-race`).

## Core proverbs
- **"Don't communicate by sharing memory; share memory by communicating."**
  Prefer channels for handing off data; use a mutex only to protect simple
  shared state.
- **"Channels orchestrate; mutexes serialize."** Pick the simpler one for the
  job — a mutex around a map is often clearer than a channel.
- **A goroutine you start is yours to stop.** Never `go f()` without knowing
  how and when it ends.

## Structured concurrency: every goroutine has an exit

The #1 bug is the **goroutine leak** — a goroutine blocked forever on a
channel send/receive because no one is listening anymore. Rules:
- Always provide a cancellation/exit path (a `context`, a `done` channel, or
  a closed input channel).
- The function that **starts** goroutines should **wait** for them before
  returning (`sync.WaitGroup` or `errgroup`).
- A sender owns its channel and **closes it** when done; receivers range over
  it. Never close a channel from the receiver side; never close twice.

```go
func process(ctx context.Context, in <-chan Job) error {
    var wg sync.WaitGroup
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return                 // exit on cancel — no leak
                case job, ok := <-in:
                    if !ok { return }      // exit when input closed
                    handle(job)
                }
            }
        }()
    }
    wg.Wait()                              // owner waits for all workers
    return ctx.Err()
}
```

## context: lifecycle & cancellation
- Pass `context.Context` as the **first argument** (`ctx`); thread it down.
  Never store it in a struct.
- Derive scoped contexts: `context.WithCancel`, `WithTimeout`, `WithDeadline`.
  **`defer cancel()`** immediately to release resources even on the happy
  path.
- Select on `<-ctx.Done()` in any blocking loop so work stops on cancel.
- Propagate the **incoming** request's context to outbound calls so a
  client disconnect cancels downstream work.
- **Anti-pattern**: passing a `cancelFunc` down into callees. The scope that
  *creates* the context owns cancelling it.

## errgroup: concurrent tasks that can fail

`golang.org/x/sync/errgroup` runs a group of goroutines sharing a context;
the **first error cancels the group**, signaling siblings to stop, and
`Wait()` returns that error.

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8)                       // bound concurrency (needs x/sync ≥ v0.1.0)
for _, u := range urls {
    u := u                          // capture (pre-1.22) — or use Go 1.22 loopvar
    g.Go(func() error {
        return fetch(ctx, u)        // use the GROUP's ctx, not the outer one
    })
}
if err := g.Wait(); err != nil {    // first failure
    return err
}
```
Caveat: each `g.Go` func must respect `ctx` cancellation, or a sibling's
failure won't actually stop the others. errgroup fits short-lived,
bounded-scope fan-out; long-running daemons need hand-rolled supervision.

## Worker pool (bound concurrency)

When you have many tasks but want a fixed number of concurrent workers:
- N workers range over a `jobs` channel; results go to a `results` channel.
- Close `jobs` when all work is queued; workers exit when the range ends.
- Or simply `errgroup` + `SetLimit(N)` for the common case.

## Common pitfalls
- **Loop variable capture** (Go ≤1.21): `for _, v := range xs { go func(){ use(v) }() }`
  captured the *same* `v`. Fixed by `v := v` shadowing, or by passing as an
  arg. **Go 1.22+** makes the loop var per-iteration, so this is no longer a
  bug — but know which Go version the repo targets (`go.mod`).
- **Data race**: two goroutines touch the same variable, one writes, no
  synchronization. Detect with `go test -race ./...` and `go run -race`.
  **Run `-race` in CI** — races are invisible until they corrupt prod.
- **`WaitGroup` misuse**: call `wg.Add` *before* starting the goroutine, not
  inside it; `defer wg.Done()` first line of the goroutine.
- **Sending on a closed channel** → panic. Only the sole sender closes.
- **Unbuffered channel deadlock**: send with no ready receiver blocks
  forever; use `select` with `ctx.Done()` or a buffer where appropriate.
- **`sync.Mutex` copied**: never copy a struct containing a mutex (pass by
  pointer); `go vet` flags this.
- **`time.After` in loops** leaks timers until they fire — use
  `time.NewTimer` + `Stop()`, or Go 1.23+ `context`-aware patterns.

## Primitives quick map
| Need | Use |
|------|-----|
| Wait for N goroutines | `sync.WaitGroup` |
| First-error fan-out, bounded | `errgroup.Group` + `SetLimit` |
| Protect shared state | `sync.Mutex` / `sync.RWMutex` |
| One-time init | `sync.Once` |
| Cancellation / deadlines | `context` |
| Atomic counter/flag | `sync/atomic`, `atomic.Int64` |
| Hand off data | channel |

## Review checklist
- [ ] Every goroutine has a guaranteed exit (ctx, closed chan, or done).
- [ ] Starter waits for its goroutines (WaitGroup/errgroup) before returning.
- [ ] `context` threaded, `defer cancel()` present, blocking loops select on `ctx.Done()`.
- [ ] Channels closed by the sole sender, exactly once.
- [ ] Tests run under `-race` in CI.

## References
- [Go Concurrency Patterns (blog)](https://go.dev/blog/pipelines)
- [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)
- [Sharp edges of errgroup & context](https://peng.fyi/post/lessons-from-an-errgroup-and-context-mishap/)
- [context package](https://pkg.go.dev/context)
