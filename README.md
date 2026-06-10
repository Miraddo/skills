# Go Best-Practice Skills for Claude Code

A collection of **14 [Claude Code](https://claude.com/claude-code) skills** that
encode idiomatic, production-grade Go (golang) practice — project structure, the
toolchain, concurrency, testing, HTTP/DB/observability/security, and the
monolith → modular-monolith → microservices progression.

Each skill is a self-contained `SKILL.md` with decision guidance, idiomatic code,
a review checklist, anti-patterns, and links to authoritative sources
(go.dev, Effective Go, OWASP, golang-standards). They are cross-linked so the
model can pull in related guidance automatically.

> Verified against the Go ecosystem as of **June 2026** (Go 1.26, golangci-lint v2,
> `golangci-lint-action@v9`, pgx v5, sqlc 1.31, OpenTelemetry-Go).

## Skills

### Foundations
| Skill | Use it for |
|-------|-----------|
| [`go-structure`](go-structure/SKILL.md) | Project/package layout: flat → `internal/` → `cmd/` → `pkg/`, tests beside code. |
| [`go-commands`](go-commands/SKILL.md) | Running the toolchain: `build`/`run`/`test`/`vet`/`fmt`/`mod tidy`, race, coverage, the test-cache gotcha. |
| [`go-idioms`](go-idioms/SKILL.md) | Naming, error wrapping (`%w`, `errors.Is/As`), accept-interfaces-return-structs, avoiding "Java in Go". |
| [`go-concurrency`](go-concurrency/SKILL.md) | Goroutine lifecycles, channels, `context`, `errgroup`, worker pools; leaks/races/deadlocks. |
| [`go-testing`](go-testing/SKILL.md) | Table-driven tests + subtests, `t.Helper/Cleanup/Parallel`, `httptest`, golden files, fuzzing. |

### Building services
| Skill | Use it for |
|-------|-----------|
| [`go-http-api`](go-http-api/SKILL.md) | `net/http`, Go 1.22+ ServeMux routing, middleware chains, JSON + validation, server timeouts. |
| [`go-database`](go-database/SKILL.md) | `database/sql` vs pgx/pgxpool, parameterized queries, pool tuning, transactions, sqlc, migrations. |
| [`go-observability`](go-observability/SKILL.md) | `slog` structured logging, OpenTelemetry traces/metrics, Prometheus, trace/request-ID correlation. |
| [`go-security`](go-security/SKILL.md) | govulncheck/gosec, input validation, no SQL injection, secrets, `crypto/rand`, TLS, OWASP API risks. |
| [`go-performance`](go-performance/SKILL.md) | pprof (CPU/heap/block/mutex), benchmarks + benchstat, escape analysis, cutting allocations. |
| [`go-tooling-ci`](go-tooling-ci/SKILL.md) | gofumpt + golangci-lint config, Makefile, GitHub Actions, govulncheck, pre-commit hooks. |

### Architecture (a deliberate progression)
| Skill | Use it for |
|-------|-----------|
| [`go-monolith`](go-monolith/SKILL.md) | One binary, one DB, model/repository/service/handler layering. The default for new apps. |
| [`go-modular-monolith`](go-modular-monolith/SKILL.md) | One deployable, independent modules with private `internal/`, a facade each, in-process event bus. |
| [`go-microservices`](go-microservices/SKILL.md) | Independently deployable services, own data, gRPC/REST + proto contracts, graceful shutdown. |

**Progression:** start `go-monolith` → enforce boundaries with `go-modular-monolith`
→ extract to `go-microservices` only when concrete limits (deploy cadence, ownership,
independent scaling) demand it.

## Install

Skills live in `~/.claude/skills/`. Copy the skill folders there:

**macOS / Linux**
```bash
git clone https://github.com/Miraddo/skills.git
cp -r skills/go-* ~/.claude/skills/
```

**Windows (PowerShell)**
```powershell
git clone https://github.com/Miraddo/skills.git
Copy-Item skills\go-* "$env:USERPROFILE\.claude\skills\" -Recurse
```

Restart Claude Code (or start a new session). The skills auto-trigger from their
descriptions, or invoke one explicitly with `/go-structure`, `/go-commands`, etc.

## How these work

Each skill is a Markdown file with YAML frontmatter Claude Code reads:

```yaml
---
name: go-structure
description: Lay out and organize Go projects idiomatically...
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---
```

`description` is what the model matches against to decide when a skill is relevant.
`[[double-bracket]]` references inside the body link related skills.

## Contributing

Issues and PRs welcome. Keep each skill self-contained, cite authoritative sources,
and prefer the standard library and idiomatic Go over framework-specific advice.

## License

[MIT](LICENSE) © Milad Poshtdari
