---
name: go-testing
description: Write effective Go (golang) tests — table-driven tests with subtests, t.Helper/Cleanup/Parallel, httptest, golden files, fakes over heavy mocks, fuzzing, benchmarks, and coverage as a signal. Use when writing or improving tests for Go code.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Writing Go Tests

How to write Go tests that are fast, readable, and trustworthy, using the
standard `testing` package. For *running* tests (flags, `-race`, coverage,
caching) see [[go-commands]]; for style see [[go-idioms]].

## Conventions
- Test files: `foo_test.go` next to `foo.go`, same package.
- Use **`package foo_test`** (external test package) to test only the
  exported API — it keeps tests honest about the public surface. Use internal
  (`package foo`) tests only when you must reach unexported helpers.
- Names: `func TestXxx(t *testing.T)`, `BenchmarkXxx(b *testing.B)`,
  `ExampleXxx()`, `FuzzXxx(f *testing.F)`.
- **Examples double as docs**: an `Example` with an `// Output:` comment is
  compiled, run, and shown in godoc.

## Table-driven tests + subtests (the default pattern)

Express cases as a slice of structs; run each as a `t.Run` subtest so
failures name the case and you can run one with `-run TestX/case_name`.

```go
func TestParse(t *testing.T) {
    tests := []struct {
        name    string
        in      string
        want    int
        wantErr bool
    }{
        {name: "simple", in: "42", want: 42},
        {name: "negative", in: "-7", want: -7},
        {name: "bad input", in: "x", wantErr: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Parse(tt.in)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("Parse(%q) = %d, want %d", tt.in, got, tt.want)
            }
        })
    }
}
```

`t.Error*` reports and continues; `t.Fatal*` reports and stops the current
test/subtest (use when later assertions would panic on the bad value).

## The helper trio
- **`t.Helper()`** — first line of any assertion helper, so failures point at
  the caller's line, not the helper's.
- **`t.Cleanup(fn)`** — register teardown that runs at test end (LIFO);
  cleaner than `defer` across helpers.
- **`t.Parallel()`** — mark independent tests to run concurrently. Capture
  loop vars (pre-1.22) and don't share mutable state between parallel tests.

## Fakes over heavy mocks
Idiomatic Go prefers **small interfaces + hand-written fakes** over
mock-generation frameworks. Because you "accept interfaces" ([[go-idioms]]),
a test can pass a tiny fake:

```go
type fakeStore struct{ users map[int]User }
func (f *fakeStore) Get(_ context.Context, id int) (User, error) {
    u, ok := f.users[id]
    if !ok { return User{}, ErrNotFound }
    return u, nil
}
```

Reach for `testify/mock` or `go.uber.org/mock` (gomock) only when call-order
/ call-count verification genuinely matters. `testify/require` &
`testify/assert` are widely used for terser assertions — fine, but stdlib-only
is perfectly idiomatic too. Match what the repo already uses.

## HTTP handler testing (`net/http/httptest`)
- **`httptest.NewRecorder()`** — unit-test a handler in isolation: build a
  `*http.Request`, call the handler, assert on the recorder.
- **`httptest.NewServer(handler)`** — spin a real server on a random port for
  client/integration tests; `defer srv.Close()`.

```go
req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
rec := httptest.NewRecorder()
handler.ServeHTTP(rec, req)
if rec.Code != http.StatusOK { t.Fatalf("status = %d", rec.Code) }
```

## Golden files (large/complex expected output)
Store expected output in `testdata/` instead of giant string literals.
Add a `-update` flag to regenerate them:

```go
var update = flag.Bool("update", false, "update golden files")
// ...
golden := filepath.Join("testdata", tt.name+".golden")
if *update { os.WriteFile(golden, got, 0o644) }
want, _ := os.ReadFile(golden)
if !bytes.Equal(got, want) { t.Errorf("mismatch; run -update") }
```
`testdata/` is ignored by the Go toolchain, so it's the conventional fixture
home.

## Fuzzing & benchmarks
- **Fuzz** (`FuzzXxx`, Go 1.18+): seed with `f.Add`, assert invariants /
  round-trips in `f.Fuzz`. Run: `go test -fuzz=FuzzParse -fuzztime=30s`.
- **Benchmark**: loop `for i := 0; i < b.N; i++`; `b.ReportAllocs()`; reset
  costly setup with `b.ResetTimer()`. Run with `-benchmem` (see [[go-commands]]).

## Concurrency & flake hygiene
- **Always `go test -race`** for code with goroutines (see [[go-concurrency]]).
- **`testing/synctest`** (stable since Go 1.25; experimental in 1.24 behind
  `GOEXPERIMENT=synctest`) makes timer/ticker-based concurrent code
  deterministic — prefer it over real `time.Sleep` in tests.
- No `time.Sleep` to "wait for" async work; synchronize on channels/signals.
- Tests must be hermetic: no network/clock/global-state dependence; inject a
  clock and use `t.TempDir()` for files.

## Coverage — signal, not target
`go test -cover` / `-coverprofile`. Treat ~70–80%+ as a health signal, not a
goal to game. Cover behavior and edge cases, not lines for their own sake.

## Review checklist
- [ ] Table-driven with named `t.Run` subtests.
- [ ] Helpers call `t.Helper()`; teardown via `t.Cleanup`/`t.TempDir`.
- [ ] Errors checked with `errors.Is/As`, not string compare.
- [ ] Fakes/small interfaces over heavyweight mocks unless order matters.
- [ ] Hermetic: no sleeps, no real network/clock; `-race` for concurrent code.
- [ ] External `package foo_test` where testing the public API.

## References
- [testing package](https://pkg.go.dev/testing)
- [net/http/httptest](https://pkg.go.dev/net/http/httptest)
- [Go Fuzzing](https://go.dev/doc/security/fuzz/)
- [Go Unit Testing: Structure & Best Practices](https://www.glukhov.org/post/2025/11/unit-tests-in-go/)
