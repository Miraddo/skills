---
name: go-tooling-ci
description: Set up Go (golang) project tooling and CI — gofumpt formatting, golangci-lint config, govulncheck, a standard Makefile, GitHub Actions workflow, and pre-commit hooks. Use when configuring linting, formatting, CI pipelines, or quality gates for a Go repo.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Tooling & CI

Wire up the quality gates every Go repo should have: deterministic
formatting, linting, vulnerability scanning, and a CI pipeline that enforces
them. Builds on [[go-commands]] and feeds [[go-security]] (scanners) and
[[go-testing]] (`-race`/coverage in CI).

## The tools
| Tool | Role |
|------|------|
| `gofmt` / **`gofumpt`** | Formatting. `gofumpt` = stricter, opinionated superset of `gofmt`. |
| **`golangci-lint`** | Meta-linter aggregating 50+ linters — the de-facto standard (used by Kubernetes, Prometheus, Terraform). One config, fast. |
| **`govulncheck`** | Official vuln scanner; call-graph aware ([[go-security]]). |
| `go vet` | Built-in correctness checks (also run by `go test`). |
| `go mod tidy` / `verify` | Dependency hygiene. |
| `benchstat` | Compare benchmark runs ([[go-performance]]). |

## `.golangci.yml` (single source of truth)
A unified config gives the whole team + CI the same rules. A sane starting set:

```yaml
version: "2"
run:
  timeout: 3m
linters:
  enable:
    - errcheck      # unchecked errors
    - govet
    - staticcheck   # the big one (SA/ST/QF checks)
    - revive        # style/lint
    - ineffassign
    - unused
    - gosec         # security
    - bodyclose     # http resp bodies closed
    - sqlclosecheck # rows/stmts closed
    - misspell
formatters:
  enable:
    - gofumpt
    - goimports
```
Tune to taste, but keep `staticcheck`, `errcheck`, `govet`, and `gosec`.
Don't enable *every* linter — curate to reduce noise.

## Makefile (consistent entry points)
```makefile
.PHONY: fmt lint vet test test-race cover tidy vuln check ci

fmt:        ; gofumpt -w .
lint:       ; golangci-lint run --timeout=3m
vet:        ; go vet ./...
test:       ; go test ./...
test-race:  ; go test -race ./...
cover:      ; go test -coverprofile=cover.out ./... && go tool cover -func=cover.out
tidy:       ; go mod tidy && git diff --exit-code go.mod go.sum
vuln:       ; govulncheck ./...
check: fmt vet lint test-race    # run locally before pushing
ci: tidy vet lint test-race vuln # the gate CI enforces
```

## GitHub Actions
```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v6
        with:
          go-version: stable      # resolves to the latest stable Go (1.26.x as of Jun 2026)
          cache: true
      - name: tidy is clean
        run: go mod tidy && git diff --exit-code go.mod go.sum
      - name: format check
        run: |
          go install mvdan.cc/gofumpt@latest
          test -z "$(gofumpt -l .)" || (gofumpt -l . && exit 1)
      - name: lint
        uses: golangci/golangci-lint-action@v9   # v9 supports golangci-lint v2; v6/v7 are for v1/early-v2
        with: { version: latest }                # latest is v2.12.x (Jun 2026)
      - name: test
        run: go test -race -coverprofile=cover.out ./...
      - name: vulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
```
Key gates: **formatting** (fail if `gofumpt -l` is non-empty), **lint**,
**`go test -race`**, and **`govulncheck`** — CI must reject builds with known
vulnerabilities ([[go-security]]). Use `setup-go`'s build cache to keep it
fast. The shell snippets above assume a **Linux runner** (`ubuntu-latest`),
which is standard for Go CI even when you develop on Windows/macOS. Pin action
majors to the latest you've tested (`@v9`/`@v6` here) and let Dependabot bump
them.

## Pre-commit hooks (catch it before CI)
A `.pre-commit-config.yaml` running `gofumpt`, `golangci-lint`, and a local
`go mod tidy` hook gives fast local feedback so failures don't first surface
in CI. Run `make check` before pushing.

## Conventions
- **Pin the Go version** in `go.mod` (`go 1.XX`); CI uses `stable` or a
  matrix.
- Commit `go.mod` **and** `go.sum`; CI proves `go mod tidy` is a no-op.
- Vendoring is optional (`go mod vendor`) — useful for hermetic/air-gapped
  builds; otherwise rely on the module cache.
- Keep linter config in-repo so editors and CI agree.

## Anti-patterns
- Relying on developers to "remember" to format/lint — automate it.
- Enabling all linters → noise → people ignore the linter.
- CI that runs tests but not `-race`, lint, or `govulncheck`.
- Letting `go.sum`/`go.mod` drift (no `tidy` check).

## References
- [golangci-lint](https://golangci-lint.run/)
- [gofumpt](https://github.com/mvdan/gofumpt)
- [Automate Your Go Project: CI/CD with GitHub Actions](https://dev.to/sbshobhit/automate-your-go-project-best-practices-cicd-with-github-actions-4bo4)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
