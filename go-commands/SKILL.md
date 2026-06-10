---
name: go-commands
description: Run Go (golang) toolchain commands correctly and idiomatically — build, run, test (with race/coverage/table tests), vet, fmt, mod tidy, generate, benchmarks. Use when building, testing, formatting, linting, or managing dependencies of a Go project.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Running Go Commands

How to drive the Go toolchain the idiomatic way. Sourced from the official
[`go` command docs](https://pkg.go.dev/cmd/go) and
[`go vet`](https://pkg.go.dev/cmd/vet).

## The `./...` rule

Most commands take package patterns. **`./...` means "this directory and all
subpackages recursively"** — use it to operate on the whole module from the
root:

```bash
go build ./...
go vet ./...
go test ./...
```

A bare command with no pattern acts on the current directory's package only.
Prefer `./...` from the module root for project-wide actions.

## Core commands

| Command | Use |
|---------|-----|
| `go run .` | Compile + run the package in the current dir (no binary left behind). `go run ./cmd/server` for a specific entrypoint. |
| `go build ./...` | Compile everything; reports errors. Produces a binary only for `main` packages built individually. |
| `go build -o bin/app ./cmd/app` | Build a named binary to a path. |
| `go install ./cmd/app` | Build and install to `$GOBIN` (on `PATH`). |
| `go test ./...` | Run all tests in the module. |
| `go vet ./...` | Static analysis — catches bugs the compiler misses (bad `Printf` args, lost struct tags, etc.). |
| `go fmt ./...` | Format all sources in place (`gofmt -l -w`). |
| `go mod tidy` | Add missing + remove unused deps. Run before every commit. |
| `go generate ./...` | Run `//go:generate` directives. |
| `go doc <pkg> [sym]` | Show docs for a package/symbol. |
| `go env` | Print Go environment (`GOPATH`, `GOMODCACHE`, …). |
| `go clean -cache` | Clear the build cache. |

## Standard pre-commit / CI sequence

Run these in order from the module root; stop at the first failure:

```bash
go mod tidy          # deps are accurate
gofmt -l .           # lists unformatted files; non-empty = fail. Or: go fmt ./...
go vet ./...         # static checks
go build ./...       # everything compiles
go test ./... -race  # tests pass under the race detector
```

`go test` automatically runs `go vet` on test sources first and won't run
the tests if vet finds serious problems — so a clean `go test` already
implies a partial vet pass, but run `go vet ./...` explicitly for full
coverage.

## Testing

```bash
go test ./...                    # all packages
go test ./pkg/auth               # one package
go test -run TestLogin ./...     # tests matching a regex
go test -v ./...                 # verbose: per-test PASS/FAIL
go test -race ./...              # race detector (use in CI)
go test -count=1 ./...           # disable the test result CACHE (force re-run)
go test -cover ./...             # coverage % per package
go test -coverprofile=cover.out ./... && go tool cover -html=cover.out  # HTML report
go test -short ./...             # skip tests that check testing.Short()
go test -timeout 30s ./...       # fail tests that hang
```

**Key gotcha — test caching**: Go caches passing test results. If a test
"passes" without seeming to run, it was cached. Use `-count=1` to force a
real run.

### Test conventions to follow / enforce
- Test files: `xxx_test.go`, same directory as the code.
- Functions: `func TestXxx(t *testing.T)`, `func BenchmarkXxx(b *testing.B)`,
  `func ExampleXxx()`, `func FuzzXxx(f *testing.F)`.
- Prefer **table-driven tests** with subtests (`t.Run(name, func(t *testing.T){…})`)
  to cover many cases without duplication.
- Use `t.Helper()` in assertion helpers; `t.Cleanup()` for teardown;
  `t.Parallel()` for independent tests.
- Use `package foo_test` (external) to test only the exported API.

## Benchmarks & profiling

```bash
go test -bench=. ./...                    # run all benchmarks
go test -bench=. -benchmem ./...          # include allocation stats
go test -bench=BenchmarkX -benchtime=10s  # longer run for stable numbers
go test -cpuprofile=cpu.out -bench=. ./...
go tool pprof cpu.out
```

## Modules & dependencies

```bash
go mod init github.com/user/repo   # create go.mod (run once, at root)
go mod tidy                        # sync deps with imports (run often)
go get example.com/pkg@latest      # add / upgrade a dependency
go get example.com/pkg@v1.4.0      # pin a version
go get -u ./...                    # upgrade deps to latest minor/patch
go mod download                    # populate module cache
go mod verify                      # check deps haven't been modified
go mod why example.com/pkg         # explain why a dep is needed
go list -m all                     # list all modules in the build
```

`go.sum` records checksums — commit both `go.mod` and `go.sum`.

## Build constraints & cross-compilation

```bash
GOOS=linux GOARCH=amd64 go build -o app ./cmd/app    # cross-compile
go build -tags=integration ./...                     # enable build tag
go vet -tags=integration ./...                       # vet tag-gated code too
```
On Windows PowerShell, set env per-command differently:
```powershell
$env:GOOS="linux"; $env:GOARCH="amd64"; go build -o app ./cmd/app
```

## Linting beyond vet

`go vet` is the built-in baseline. For deeper checks, projects commonly use
[`golangci-lint`](https://golangci-lint.run/) (aggregates many linters):
```bash
golangci-lint run ./...
```
Check whether the repo has a `.golangci.yml` before assuming it's available.

## Workflow when applying this skill

1. Confirm you're at the module root (a `go.mod` is present — `Glob go.mod`).
2. For "does it build/pass" requests, run the **pre-commit sequence** above
   and report the first failure with its output verbatim.
3. When tests look suspiciously fast/green, re-run the relevant package with
   `-count=1 -v` to defeat caching before declaring success.
4. Report failures honestly with the actual command output; don't claim
   green unless you saw it.

## References
- [`go` command reference](https://pkg.go.dev/cmd/go)
- [`go vet` reference](https://pkg.go.dev/cmd/vet)
- [Go testing package](https://pkg.go.dev/testing)
