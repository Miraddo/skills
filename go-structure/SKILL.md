---
name: go-structure
description: Lay out and organize Go (golang) projects idiomatically — choosing between flat, internal/, pkg/, and cmd/ layouts; placing packages, tests, and entrypoints. Use when starting a Go module, adding packages/commands, or reviewing/refactoring Go project structure.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Project Structure

Idiomatic layout for Go modules, based on the official [Go module layout
guide](https://go.dev/doc/modules/layout) and the community
[golang-standards/project-layout](https://github.com/golang-standards/project-layout).

## Guiding principle: start flat, grow as needed

**Do not over-structure.** A new project is a single `main.go` + `go.mod`.
Add directories only when complexity demands it. The standard layout is a
menu, not a mandate — adopt only the directories you actually need. Heavy
structure on a small project is a smell.

Decision order:
1. **One package, library** → flat (files in root).
2. **One binary** → flat, all `package main` in root.
3. **Library + private helpers** → add `internal/`.
4. **Multiple public packages** → add named subdirs (`auth/`, `store/`).
5. **One or more binaries + shared code** → add `cmd/<name>/main.go`.

## Layouts (copy the one that fits)

### Basic package (library)
```
mymod/
  go.mod          # module example.com/mymod
  mymod.go        # package mymod — name matches last path element of module
  mymod_test.go
```

### Basic command (single binary)
```
myapp/
  go.mod
  main.go         # package main, func main()
  client.go       # package main
  client_test.go
```

### Library with private supporting packages
```
mymod/
  go.mod
  mymod.go
  internal/       # importable ONLY within this module — compiler-enforced
    auth/
      auth.go
      auth_test.go
    hash/
      hash.go
```

### Multiple public + internal packages
```
mymod/
  go.mod
  mymod.go
  auth/           # public: importable by anyone
    auth.go
    token/
      token.go
  internal/       # private to this module
    trace/
      trace.go
```

### Multiple binaries + shared code
```
myproj/
  go.mod
  internal/       # shared private packages
    ...
  cmd/
    server/
      main.go     # package main
    cli/
      main.go     # package main
```

## The directories that matter

| Dir          | Purpose | Rule |
|--------------|---------|------|
| `internal/`  | Private packages | **Compiler-enforced**: only code rooted at the parent of `internal/` may import it. Default home for app logic you don't want others importing. |
| `cmd/<name>/`| Binary entrypoints | One subdir per executable; dir name = binary name. Keep `main.go` thin — wire dependencies, call into packages. |
| `pkg/`       | Public libraries | Only if you *intend* external reuse. **Controversial** — many teams skip it and put public packages at root. Don't add it by default. |
| `api/`       | API contracts | OpenAPI/protobuf/JSON schema definitions. |
| `web/`/`assets/` | Static/web assets | Templates, SPA, static files. |
| `test/`      | External test data/fixtures | Not for `_test.go` files — those live next to the code. |

**Avoid**: a `src/` directory (not a Go convention), and dumping everything
in a `utils`/`common`/`helpers` grab-bag package — name packages for what
they provide, not what they are.

## Conventions checklist

- **Module path**: `go mod init github.com/user/repo` (or your VCS host).
  The module path is the import prefix for every package in it.
- **Package name = directory name** (lowercase, no underscores or
  mixedCaps, e.g. `package httputil` in `httputil/`). Exception: `main`.
- **Tests live beside code**: `foo.go` → `foo_test.go` in the same dir.
  Use a `package foo_test` (external) test package when you want to test
  only the public API.
- **One package per directory** — Go enforces this (except `_test` packages).
- **Exported identifiers** start uppercase; keep the public surface small.
- **No import cycles** — if two packages need each other, the shared piece
  belongs in a third (often `internal/`) package.
- **Avoid deep nesting** — flat-ish trees are easier to navigate than
  deeply nested ones.

## Workflow when applying this skill

1. Read existing layout first (`Glob **/*.go`, find `go.mod`). Don't impose
   structure that fights the current one.
2. Identify which layout above the project is closest to and what it needs.
3. Prefer the **smallest** change: move code into `internal/` before
   inventing `pkg/`; add `cmd/` only when there's a second binary or mixed
   lib+binary.
4. After moving files, fix import paths and run the checks from the
   companion **go-commands** skill (`go build ./...`, `go vet ./...`,
   `go test ./...`).

## References
- [Organizing a Go module (official)](https://go.dev/doc/modules/layout)
- [golang-standards/project-layout](https://github.com/golang-standards/project-layout)
- [Eleven Tips for Structuring Your Go Projects — Alex Edwards](https://www.alexedwards.net/blog/11-tips-for-structuring-your-go-projects)
