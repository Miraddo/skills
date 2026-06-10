---
name: go-performance
description: Profile and optimize Go (golang) — pprof (CPU/heap/block/mutex), benchmarks with benchstat, escape analysis (-gcflags=-m), reducing allocations, sync.Pool, and data-driven tuning. Use when investigating slowness, high memory/GC, or optimizing hot paths — measure before changing.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Performance

Optimize with **data, not guesswork**. Profiling is a tool, not a goal —
start from a question and a metric, measure under representative load, change
one thing, measure again. Pairs with [[go-commands]] (test/bench flags),
[[go-testing]] (benchmarks), and [[go-observability]] (production signals).

## First rule: measure, don't guess
- Start with a **question + metric** ("p99 latency of `/search`", "RSS
  growth over an hour"), not "make it fast."
- Profile under **representative load and data** — micro-toy inputs lie.
- Capture **CPU and allocs together**; cause and effect usually sit next to
  each other.
- Don't optimize what isn't hot. Readability first ([[go-idioms]]); optimize
  the proven bottleneck.

## Benchmarks + benchstat
Write benchmarks in `_test.go` ([[go-testing]]) and compare runs statistically
— a single run is noise.

```go
func BenchmarkEncode(b *testing.B) {
    b.ReportAllocs()
    data := makeInput()
    b.ResetTimer()             // exclude setup
    for i := 0; i < b.N; i++ {
        _ = Encode(data)
    }
}
```
```bash
go test -bench=. -benchmem -count=10 ./... > old.txt
# ...make a change...
go test -bench=. -benchmem -count=10 ./... > new.txt
benchstat old.txt new.txt     # is the delta statistically real?
```
`-benchmem` reports `B/op` and `allocs/op` — allocations are often the real
cost. Use `-count=10` + `benchstat` for stable comparisons.

## pprof — the four profiles
| Profile | Answers |
|---------|---------|
| **CPU** (`-cpuprofile`) | Where is time spent? |
| **Heap / allocs** (`-memprofile`) | Where do allocations happen / what's retained? |
| **Block** | Where do goroutines wait on channels/sync? |
| **Mutex** | Where is lock contention? |

```bash
go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out ./pkg
go tool pprof -http=:8080 cpu.out      # flame graph in browser
```
For a running service, expose `net/http/pprof` (guard it — not public) and
pull a live profile:
```go
import _ "net/http/pprof"   // registers /debug/pprof on the default mux
```
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```
Capture **≥30s** CPU windows in prod (shorter = noisy sampling); profile
during realistic load/spikes.

**Memory nuance**: *total allocated* (`-alloc_space`) finds allocation-heavy
paths but not leaks; *currently retained* (`-inuse_space`) is what grows RSS
and reveals leaks.

## Reduce allocations (the usual biggest win)
- **Escape analysis**: `go build -gcflags=-m ./...` shows what escapes to the
  heap. A variable that "escapes to heap" could often stay on the stack.
  Recent Go releases improved escape analysis/inlining — keep the toolchain
  current.
- **Preallocate** slices/maps with known size: `make([]T, 0, n)`.
- **Reuse buffers** with `sync.Pool` for hot, short-lived objects (e.g.
  per-request byte buffers) — but only when profiling shows alloc pressure;
  misused pools hurt.
- Avoid hidden allocs: `[]byte`↔`string` conversions in loops, `interface{}`
  boxing, unnecessary pointers, capturing closures in hot paths.
- Streaming over buffering: `io.Reader`/`Writer` instead of reading whole
  payloads into memory.

## Concurrency & GC
- Bound parallelism (worker pools / `errgroup.SetLimit`) — more goroutines
  isn't faster past CPU count ([[go-concurrency]]).
- Reduce lock contention (sharding, `sync.RWMutex`, atomics) — confirm with
  the **mutex/block** profiles, don't assume.
- GC pressure is driven by allocation rate — cutting allocations is usually
  better than tuning `GOGC`. Tune `GOGC`/`GOMEMLIMIT` only with evidence.

## Workflow
1. Reproduce with a benchmark or capture a prod profile.
2. Find the top cost in pprof (CPU time or `inuse`/`alloc` space).
3. Form a hypothesis; make **one** change.
4. Re-bench with `benchstat`; keep it only if the delta is real.
5. Re-profile — the bottleneck moves.

## Anti-patterns
- Optimizing before profiling; micro-optimizing cold code.
- Trusting a single benchmark run (no `-count`/`benchstat`).
- `sync.Pool` / unsafe tricks without measured need.
- Sacrificing clarity for unproven speed.
- Benchmarking with unrepresentative inputs or timer including setup.

## References
- [A Practical Guide to Profiling in Go — JetBrains](https://blog.jetbrains.com/go/2026/05/20/golang-profiling-guide/)
- [Profiling Go Programs (official blog)](https://go.dev/blog/pprof)
- [net/http/pprof](https://pkg.go.dev/net/http/pprof)
- [benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat)
