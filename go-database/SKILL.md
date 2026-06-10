---
name: go-database
description: Access SQL databases in Go (golang) safely — database/sql vs pgx/pgxpool, parameterized queries (no injection), connection pool tuning, context deadlines, transactions done right, sqlc codegen, and migrations. Use when writing data-access/repository code, queries, transactions, or DB migrations.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Databases (SQL)

Write the data-access layer (the repository in [[go-monolith]]) correctly:
safe queries, a tuned pool, context everywhere, and disciplined
transactions. Pairs with [[go-idioms]] and [[go-security]].

## Driver choice
- **`database/sql` + a driver** is the portable baseline; `sql.DB` is a
  concurrency-safe connection **pool**, not a single connection.
- **`pgx` / `pgxpool`** is preferred for new **PostgreSQL** projects —
  faster, better maintained than `lib/pq`, binary protocol, purpose-built
  pool. Use `pgx` over `lib/pq` for new Postgres code.
- **`sqlc`** generates type-safe Go from your SQL (compile-time checked,
  no runtime reflection) — excellent default. Set `sql_package: pgx/v5` in
  `sqlc.yaml` to pair it with pgxpool.
- **`sqlx`** = small ergonomic helpers over `database/sql`. **GORM** = full
  ORM; convenient but heavier and easy to misuse — prefer explicit SQL or
  sqlc unless the team wants an ORM.

## Never build SQL by string concatenation
Always use **parameterized queries** (`$1`, `?`) — the driver sends params
separately, which prevents SQL injection. Interpolating user input into query
text is the #1 DB vulnerability ([[go-security]]).

```go
// GOOD — parameterized
row := db.QueryRowContext(ctx, `SELECT name FROM users WHERE id = $1`, id)

// NEVER
// fmt.Sprintf("SELECT name FROM users WHERE id = %s", id)  // injection
```

## Always use the `...Context` methods
Pass `context.Context` to every call (`QueryContext`, `ExecContext`,
`QueryRowContext`, `BeginTx`) so queries inherit request deadlines and
cancellation. A request that's cancelled should stop its DB work, not hold a
connection.

## Configure the pool (don't leave defaults)
An unconfigured `sql.DB` can exhaust the database or starve under load. Set
limits to your workload and DB capacity:

```go
db.SetMaxOpenConns(25)                 // cap concurrent connections
db.SetMaxIdleConns(5)                  // keep a few warm
db.SetConnMaxLifetime(time.Hour)       // recycle (helps with failover/LB)
db.SetConnMaxIdleTime(5 * time.Minute)
```
(pgxpool has equivalents: `MaxConns`, `MaxConnLifetime`, `MaxConnIdleTime`.)
Tune from metrics ([[go-observability]]) — these numbers are a starting
point, not gospel.

## Scan & resource hygiene
- `Query` returns `*Rows` you **must close**: `defer rows.Close()`, then
  check `rows.Err()` after the loop.
- Distinguish no-rows: `errors.Is(err, sql.ErrNoRows)` (pgx:
  `pgx.ErrNoRows`) — map it to your own `ErrNotFound`, don't leak driver
  errors upward ([[go-idioms]]).
- Use `sql.Null*` types or pointers for nullable columns.

## Transactions, the good way
Begin with a context, **always resolve** the tx (commit or rollback). The
robust pattern is `defer tx.Rollback()` — it's a no-op after a successful
commit:

```go
func transfer(ctx context.Context, db *sql.DB, from, to int, amt int64) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
    if err != nil { return fmt.Errorf("begin: %w", err) }
    defer tx.Rollback()                       // safe: no-op if committed

    if _, err = tx.ExecContext(ctx,
        `UPDATE accounts SET bal = bal - $1 WHERE id = $2`, amt, from); err != nil {
        return fmt.Errorf("debit: %w", err)
    }
    if _, err = tx.ExecContext(ctx,
        `UPDATE accounts SET bal = bal + $1 WHERE id = $2`, amt, to); err != nil {
        return fmt.Errorf("credit: %w", err)
    }
    return tx.Commit()
}
```
- All statements in a tx must run on the **`tx`**, never the pool `db`.
- To thread a tx through a repository without leaking it, store it in
  `context` under a **private key type** and have repo methods pick `tx` or
  `db` from the context.
- Keep transactions short; don't do network calls / slow work inside one.

## Migrations
Change schema **only** through versioned migration files — never ad-hoc DDL.
`golang-migrate` (or `goose`, `atlas`) is standard: numbered `up`/`down` SQL
files, run on deploy. Keep migrations forward-compatible during rolling
deploys (add columns before using them; drop later).

## Repository shape (fits [[go-monolith]])
```go
type UserRepository interface {                 // define where USED (the service)
    GetByID(ctx context.Context, id int64) (User, error)
    Create(ctx context.Context, u User) (int64, error)
}
type pgUserRepo struct{ pool *pgxpool.Pool }    // concrete, returned as struct
```

## Anti-patterns
- String-concatenated SQL.
- Non-context query methods.
- Unbounded/default pool under production load.
- Forgetting `rows.Close()` / `rows.Err()`.
- A transaction that commits on some paths and silently leaks on others.
- Leaking driver errors (`sql.ErrNoRows`, pq codes) into business/HTTP layers.

## References
- [SQL Transactions in Go: The Good Way](https://blog.thibaut-rousseau.com/blog/sql-transactions-in-go-the-good-way/)
- [Using Go and pgx — sqlc docs](https://docs.sqlc.dev/en/stable/guides/using-go-and-pgx.html)
- [Go + PostgreSQL: Best Practices for Performance and Safety](https://dev.to/mx_tech/go-with-postgresql-best-practices-for-performance-and-safety-47d7)
- [database/sql package](https://pkg.go.dev/database/sql)
