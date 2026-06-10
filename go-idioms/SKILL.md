---
name: go-idioms
description: Write idiomatic Go (golang) — naming conventions, error handling (wrap with %w, errors.Is/As, sentinel errors), accept-interfaces-return-structs, small consumer-defined interfaces, zero values, and avoiding "Java in Go". Use when writing or reviewing Go code for idiomatic style.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Idiomatic Go

How to write Go the way the language wants, per
[Effective Go](https://go.dev/doc/effective_go) and the
[Go proverbs](https://go-proverbs.github.io/). Pairs with [[go-structure]],
[[go-commands]], [[go-testing]], and [[go-concurrency]]. The recurring
failure mode is writing "Java in Go" — heavy abstraction, upfront
interfaces, getters/setters. Don't.

## Naming
- **MixedCaps / mixedCaps**, never `snake_case`. Exported = `UpperCase`,
  unexported = `lowerCase`.
- **Initialisms keep case**: `URL`, `ID`, `HTTP`, `API` → `ServeHTTP`,
  `userID`, `parseURL` — never `Url`, `Id`, `Http`.
- **Short names for short scopes**: `i`, `r`, `buf`, `ctx`. Longer, clearer
  names for package-level identifiers.
- **No stutter**: in package `user`, name the type `User` so callers write
  `user.User`? No — prefer `user.Service`, `user.Repository`. Avoid
  `user.UserService` (`user.User...` stutters).
- **Getters drop `Get`**: field `name` → method `Name()`, not `GetName()`.
  Setters keep `Set`: `SetName()`.
- **Interface names**: single-method interfaces end in `-er`: `Reader`,
  `Writer`, `Stringer`, `Closer`.
- Package names are short, lowercase, no underscores or plurals: `http`,
  `strconv` — not `utils`, `helpers`, `common`, `models`.

## Errors are values
- Return `error` as the **last** return value; handle it explicitly. Don't
  use `panic` for ordinary, expected failures.
- **Error strings**: lowercase, no trailing punctuation — they get wrapped
  into larger messages. `fmt.Errorf("open config: %w", err)`, not
  `"Open config failed."`.
- **Wrap with `%w`** to preserve the chain so callers can inspect it:
  ```go
  if err != nil {
      return fmt.Errorf("loading user %d: %w", id, err)
  }
  ```
- **Inspect with `errors.Is` / `errors.As`**, never string-match:
  ```go
  if errors.Is(err, sql.ErrNoRows) { ... }      // sentinel match through the chain
  var perr *fs.PathError
  if errors.As(err, &perr) { use(perr.Path) }    // typed match
  ```
- **Sentinel errors**: `var ErrNotFound = errors.New("not found")` for values
  callers branch on. **Typed errors** (a struct implementing `error`) when
  callers need fields.
- Add context as you go *up* the stack; don't log-and-return (that
  double-reports). Either handle it or return it — not both.
- `defer` for cleanup; check errors from deferred `Close()` on writers.

## Accept interfaces, return structs
- **Accept the narrowest interface** you actually use as a parameter →
  callers can pass anything that satisfies it (flexibility, easy test fakes).
- **Return concrete structs** (often pointers) → you can add methods later
  without breaking callers, and they get the full type.
- **Define interfaces in the consumer package**, not the implementer. Keep
  them tiny (1–2 methods). Don't write an interface until a second
  implementation or a test fake actually needs one — "the bigger the
  interface, the weaker the abstraction."

```go
// good: small, consumer-defined, accepts interface / returns struct
type Store interface{ Save(context.Context, User) error } // defined where used

func NewService(s Store) *Service { return &Service{store: s} }   // return *struct
```

## Other idioms that matter
- **Zero values are useful**: design types so the zero value works
  (`sync.Mutex`, `bytes.Buffer`). Avoid mandatory constructors when a zero
  value suffices.
- **`context.Context` first param**, named `ctx`; never store it in a struct;
  pass it down call chains. (See [[go-concurrency]].)
- **Early return / guard clauses** over nested `if/else`; keep the happy path
  at minimum indentation.
- **Composition over inheritance**: embed structs/interfaces; there is no
  class hierarchy.
- **Slices/maps**: prefer returning `nil` slices over empty; `append` to nil
  is fine. Pre-size with `make([]T, 0, n)` when length is known.
- **`any`/generics**: use type parameters when they remove real duplication,
  not reflexively. Concrete types first.
- **Don't over-abstract**: no DI frameworks, no getter/setter boilerplate, no
  one-interface-per-struct. Simplicity is a feature.
- Run `gofmt`/`go vet` always (see [[go-commands]]); use `golangci-lint` for
  deeper style enforcement.

## Review checklist
- [ ] Names MixedCaps, initialisms cased, no stutter, no `Get` prefix.
- [ ] Errors wrapped with `%w`, inspected via `errors.Is/As`, lowercase strings.
- [ ] Functions accept interfaces, return concrete types; interfaces small &
      consumer-defined.
- [ ] `context.Context` threaded, not stored.
- [ ] No `panic` for expected errors; no log-and-return.
- [ ] No `utils`/`common` dumping grounds; packages named by what they provide.

## References
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Writing Idiomatic Go — not "Java in Go"](https://www.gocloudstudio.com/post/writing-idiomatic-go-patterns-that-separate-clean-code-from-java-in-go)
